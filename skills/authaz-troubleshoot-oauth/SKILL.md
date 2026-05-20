---
name: authaz-troubleshoot-oauth
description: Use when an Authaz OAuth flow is broken — redirect_uri mismatch, PKCE error, JWKS / token validation failure, callback loop, expired code, clock skew. Walks the symptom → cause → fix path for the common failures. Triggers on "redirect_uri mismatch", "invalid_grant", "authaz login broken", "authaz callback error", "JWKS error".
---

# Debug an Authaz OAuth flow

When login is broken, the symptom usually points straight at the cause. Don't guess — match the exact error to the table below.

## Step 1 — Identify where the flow is failing

| Symptom | Likely stage |
|---|---|
| Browser never leaves your app | Authorize-redirect construction |
| Redirected to Authaz, but Authaz shows an error page | Authorize endpoint validation |
| Authaz signs you in, redirects back to your app, your app errors | Callback / token exchange |
| Token exchange succeeds, but `requireUser` / middleware says no | Cookie or session issue |
| Everything works in dev, breaks in prod | Cookie attributes or HTTPS |

## Step 2 — Match the error and apply the fix

### `error=invalid_redirect_uri` on the Authaz error page

The exact `redirect_uri` you sent is not in the application's allow list. Causes:

- Trailing slash mismatch (`/auth/callback` vs `/auth/callback/`)
- Port differs (3000 vs 5173)
- `http` vs `https`
- Production URL not yet added

Fix: Dashboard → Application → Auth Flow Configuration → add the **exact** URL the browser is using. Or `PATCH /api/v1/applications/{appId}/auth-flow` with the new entry. **Add every dev / staging / prod URL up-front.**

### `error=invalid_grant` on the token endpoint

Common causes, in order of frequency:

1. **Code already used.** OAuth codes are single-use. If your callback page POSTs twice (e.g., React StrictMode double-effect, or a buggy form rerender), the second POST sees `invalid_grant`. Fix by guarding the effect with a "have I posted yet" flag.
2. **Code expired.** TTL is 60 seconds. If your callback page hangs on something else, the code rots. Fix by removing the work between landing and POSTing.
3. **`code_verifier` mismatch.** Means PKCE state was lost between the authorize request and the token exchange. The SDK stores the verifier in the cookie/state — if you hand-rolled the flow, you're missing it. Use the SDK or copy its persistence shape.
4. **`redirect_uri` mismatch on token exchange.** It must be the *exact same string* sent on the authorize call. Don't normalize, don't strip query params.

### `error=invalid_client`

Wrong `client_id`, wrong `client_secret`, or the secret was rotated. Check `appsettings.json` / `.env.local`. Confirm the env var is loaded — print the prefix (never the full secret) at startup.

### "kid not found in JWKS" / signature verification failure

The token was signed with a key your verifier hasn't fetched, or it has fetched and cached the wrong app's JWKS.

- Confirm the JWKS URL: `https://auth.rorix.io/universal/.well-known/jwks/{client_id}.json`. Note the `client_id` is in the path — using the global `/.well-known/jwks.json` from another platform's docs gives you the wrong keys.
- If you cache JWKS, set the cache TTL to no more than 1 hour and refetch on `kid` miss.
- If a key was just rotated, force a refetch.

### "token expired"

The access token's `exp` has passed. Either:

- Use the refresh-token flow (the SDK's `/api/auth/refresh` does it for you), or
- Force re-login.

If the user just signed in and the token is already expired, your server's clock is wrong. NTP it.

### Callback redirect loop

Browser bounces between `/api/auth/login` and `/auth/callback`. Causes:

- Session cookie isn't being set — likely a `Secure`/`SameSite` mismatch in dev. Either run dev over HTTPS, or set the SDK's cookie config to `secure: false` for `NODE_ENV=development` only.
- Middleware matcher includes `/auth/callback` (it shouldn't be protected). Confirm `publicPaths` in `createAuthMiddleware`.
- A reverse proxy strips the cookie. Confirm `X-Forwarded-Proto: https` is forwarded.

### Multi-tenant: `tenant_id` claim missing

User belongs to no tenants, or to exactly one (Authaz skips the picker and binds to the only one — confirm). If they belong to multiple but `tenant_id` is still null:

- Sign-In was started with a tenant hint that bypassed selection — remove the `?tenant_hint=` if you set it.
- The application is set to `single_tenant`. The shape doesn't issue tenant claims. Check `GET /api/v1/applications/{appId}` → `tenancy_type` should be `multi_tenant`.

## Step 3 — Reproduce with cURL when stuck

If the SDK output is opaque, hit the OAuth endpoints directly:

```bash
# Build the authorize URL with PKCE
CV=$(openssl rand -base64 32 | tr -d '=' | tr '/+' '_-')
CC=$(printf "%s" "$CV" | openssl dgst -sha256 -binary | openssl base64 | tr -d '=' | tr '/+' '_-')

open "https://auth.rorix.io/universal/oauth2/authorize?client_id=app_01abc&response_type=code&scope=openid&redirect_uri=http://localhost:3000/auth/callback&state=abc&code_challenge=$CC&code_challenge_method=S256"
```

After Authaz redirects with `?code=...`:

```bash
curl -X POST https://auth.rorix.io/universal/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code" \
  -d "code=THE_CODE_FROM_REDIRECT" \
  -d "client_id=app_01abc" \
  -d "client_secret=cs_live_..." \
  -d "redirect_uri=http://localhost:3000/auth/callback" \
  -d "code_verifier=$CV"
```

If the manual round-trip works, your SDK config is wrong; if it fails the same way, the application config is wrong.

## Anti-patterns

- **Don't disable PKCE to get past errors.** Authaz requires it; the right fix is restoring the `code_verifier`.
- **Don't loosen `Secure`/`SameSite` in production** to make a flow work. The dev/prod gap means either dev is too lax (fix dev with HTTPS) or prod is misconfigured (cookie domain or proto).
- **Don't catch the OAuth error and silently retry.** The retry uses a fresh code request; you'll mask the underlying bug.
- **Don't log the full access token, refresh token, or `client_secret`.** Log a prefix at most (`token[0..6]`).

## References

- OAuth 2.1 reference: <https://authaz.io/docs/authentication/oauth2>
- `references/error-codes.md` — every error you might see
- `references/endpoints.md` — all endpoint URLs and JWKS path
