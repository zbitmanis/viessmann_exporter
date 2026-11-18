# Viessmann refresh token renewal (container runbook)

The exporter authenticates against Viessmann IAM with OAuth2 authorization code + PKCE.
Access tokens are short-lived (about 1 hour) and are renewed automatically by the poll loop
using the stored refresh token. **The refresh token itself is not extended by this process.**
It expires about 180 days after the last interactive login, and then a new login is required.

## How it works

| Step | Code | Behaviour |
|------|------|-----------|
| Interactive login | `GET /oauth/login` -> `ViessmannOAuthService.build_authorize_url` | Creates `state` + PKCE verifier, redirects the browser to Viessmann |
| Callback | `GET /oauth/callback` -> `exchange_code_for_token` | Exchanges the code and stores `access_token` and `refresh_token` |
| Access renewal | `poll_loop` -> `refresh_access_token` (also `POST /oauth/refresh-access`) | `grant_type=refresh_token`; overwrites `access_token` |
| Refresh token handling | `handle_token_response` | `refresh_token` is overwritten **only if the response contains one**; `refresh_updated_at` is set at that moment |

Tokens are persisted as JSON at `$VIESSMANN_TOKEN_STORE`
(default `$VIESSMANN_TOKEN_DIR/viessmann_tokens.json`, i.e. `/app/tokens/viessmann_tokens.json` in the image).

Because the refresh grant normally does not return a new refresh token, `refresh_updated_at`
stays at the time of the last interactive login. Treat that timestamp as the start of the 180-day clock.

## Prerequisites

1. **Persist the token directory.** The image does not declare a volume. Without one, every
   container recreate loses the tokens and requires a new login.

   ```bash
   docker run -d --name viessmann-exporter -p 9000:9000 \
     -e VIESSMANN_CLIENT_ID=... \
     -e VIESSMANN_CLIENT_SECRET=... \
     -e VIESSMANN_REDIRECT_URI=https://your.domain.com/oauth/callback \
     -v viessmann_tokens:/app/tokens \
     viessmann-exporter
   ```

   The directory must be writable by the container user.

2. **Redirect URI** must match the one registered in the Viessmann developer portal exactly.
   The default callback path is `/oauth/callback`; the README example `/callback` only works if
   `VIESSMANN_CALLBACK_URL` is set accordingly.

3. **Scope.** Default is `IoT`. Per the Viessmann API docs, add `offline_access` to
   `VIESSMANN_SCOPE` if no refresh token is issued (`has_refresh_token: false`):

   ```bash
   -e VIESSMANN_SCOPE="IoT offline_access"
   ```

   Changing the scope requires a new login.

4. **Browser access** to the exporter login page from a machine that can reach the
   redirect URI. Expose only the login, callback and status pages, and restrict them to trusted
   networks (see below).

## Reverse proxy example (nginx, prefix `/viessmann`)

Publish only what the login flow needs, restricted to private networks (`allow` your internal
ranges, then `deny all`). The browser used for login must be on the LAN or VPN.

| Public URL | Proxied to |
|------------|------------|
| `/viessmann/login` | `127.0.0.1:9000/viessmann/oauth/login` |
| `/viessmann/callback` | `127.0.0.1:9000/viessmann/oauth/callback` (query string preserved) |
| `/viessmann/success`, `/viessmann/fail` | same path, unchanged |
| `/viessmann/debug/token/status` | same path, unchanged (optional, no secrets) |

The app must serve the same paths nginx forwards to. Container environment:

```bash
-e VIESSMANN_REDIRECT_URI=https://your.domain.com/viessmann/callback \
-e VIESSMANN_LOGIN_URL=/viessmann/oauth/login \
-e VIESSMANN_CALLBACK_URL=/viessmann/oauth/callback \
-e VIESSMANN_SUCCESS_URL=/viessmann/success \
-e VIESSMANN_FAIL_URL=/viessmann/fail \
-e VIESSMANN_DEBUG_STATUS_URL=/viessmann/debug/token/status
```

Notes:

- The redirect URI registered in the Viessmann developer portal is the **public** one
  (`.../viessmann/callback`), not the internal `/viessmann/oauth/callback`.
- Do not proxy `/oauth/refresh-access`, `/metrics` or any other `/debug/*` route. Scrape
  `/metrics` from the internal network only.

Status check through the proxy:

```bash
curl -s https://your.domain.com/viessmann/debug/token/status | jq
```

Point the expiry-check script at this URL if it runs from another host.

## Renewal procedure

Do this before the refresh token reaches about 180 days (suggest at day 150), or after a failure.

1. Check state:

   ```bash
   curl -s http://localhost:9000/debug/token/status | jq
   ```

   Look at `refresh_updated_at` (epoch seconds, start of the clock), `last_error`, `has_refresh_token`.

2. Open `https://your.domain.com/viessmann/login` (without the proxy: `/oauth/login`) in a browser, log in to Viessmann, and approve.
   You are redirected to `/oauth/callback` and then `/success`.

3. Verify the new token was stored:

   ```bash
   curl -s http://localhost:9000/debug/token/status | jq '{refresh_updated_at, last_success, last_error}'
   ```

   `refresh_updated_at` must be newer than before and `last_success.refresh_rotated` must be `true`.
   No restart is needed: the token file is re-read on each poll.

4. Confirm the poll loop works: `/metrics` values change and the logs show
   `metrics updated - installation: ...`.

### Constraints

- `valid_states` is held in memory. Login and callback must reach the **same running
  container**, with no restart in between. Run a single replica, or use sticky routing.
- Do not run two exporters against the same Viessmann user with separate token files.
  Re-login in one may invalidate the other's refresh token (verify against the portal's behaviour).

## Expiry check

Warn when the refresh token is older than 150 days (cron or any monitor):

```bash
#!/bin/sh
ts=$(curl -sf http://localhost:9000/debug/token/status | jq -r '.refresh_updated_at // 0')
age_days=$(( ( $(date +%s) - ts ) / 86400 ))
[ "$age_days" -ge 150 ] && echo "Viessmann refresh token is ${age_days} days old - re-login at /oauth/login"
```

## Troubleshooting

| Symptom | Cause | Action |
|---------|-------|--------|
| Log: `Refresh failed: 400` and `last_error.reason` mentions invalid/expired grant | Refresh token expired or revoked | Run the renewal procedure |
| `has_refresh_token: false` after login | `offline_access` not in scope | Set `VIESSMANN_SCOPE`, re-login |
| `Invalid state` on callback | Container restarted or a different replica served the callback | Retry `/oauth/login` |
| `Missing PKCE verifier for state` | Token file was overwritten or unwritable between login and callback | Check volume permissions, retry |
| Tokens gone after redeploy | No volume on `/app/tokens` | Mount a volume, re-login |
| `No refresh_token stored` | First start or empty volume | Run login |

## Security notes

- The token file contains a long-lived refresh token. Keep the volume private to the container
  user (mode `0600`), exclude it from backups that leave your control, and never commit it.
- Pass `VIESSMANN_CLIENT_SECRET` through a secret store or an env file with restricted
  permissions, not on a shared shell command line.
- The exporter has unauthenticated debug routes under `/debug/*`, one of which returns the raw
  tokens. Never publish them; allow only the routes listed in the proxy example.
- Treat tokens as compromised if they appear in logs or tickets: re-run the login and, if needed,
  revoke the client credentials in the Viessmann developer portal.
