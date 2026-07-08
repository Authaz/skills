---
name: authaz-troubleshoot-oauth
description: Use when an Authaz OAuth flow is broken — redirect_uri mismatch, PKCE error, JWKS / token validation failure, callback loop, expired code, clock skew. Walks the symptom → cause → fix path for the common failures. Triggers on "redirect_uri mismatch", "invalid_grant", "authaz login broken", "authaz callback error", "JWKS error".
---

# Debug an Authaz OAuth flow

Match the exact error to the table below — don't guess.

> URLs use `auth.authaz.io` (hosted default). Substitute the customer's custom identity domain if set (e.g., `auth.your-app.com`).

## Step 1 — Locate the failing stage

| Symptom | Likely stage |
|---|---|
| Browser never leaves your app | Authorize-redirect construction |
| Redirected to Authaz, but Authaz shows an error page | Authorize endpoint validation |
| Authaz signs you in, redirects back, your app errors | Callback / token exchange |
| Token exchange succeeds, but `requireUser`/middleware says no | Cookie or session issue |
| Works in dev, breaks in prod | Cookie attributes or HTTPS |

## Step 2 — Error → cause → fix

**`error=invalid_redirect_uri` (Authaz error page):** sent `redirect_uri` isn't in the app's allow list — trailing slash mismatch, port differs, http vs https, or prod URL not added. Fix: Dashboard → Application → Auth Flow Configuration → add the exact browser URL, or `PATCH /api/v1/applications/{appId}/auth-flow`. Add every dev/staging/prod URL up-front.

**`error=invalid_grant` (token endpoint)**, by frequency:
1. Code already used (single-use) — double POST from React StrictMode/buggy rerender. Guard the effect with a "posted yet" flag.
2. Code expired — TTL 60s. Remove work between landing and POSTing.
3. `code_verifier` mismatch — PKCE state lost between authorize and token exchange (SDK stores verifier in cookie/state; hand-rolled flows often drop it). Use the SDK or copy its persistence shape.
4. `redirect_uri` mismatch on token exchange — must exactly match the authorize call string; don't normalize or strip query params.

**`error=invalid_client`:** wrong `client_id`/`client_secret`, or secret rotated. Check `appsettings.json`/`.env.local`; confirm env var loads by printing only the prefix at startup (never the full secret).

**"kid not found in JWKS" / signature verification failure:** token signed with a key your verifier hasn't fetched, or wrong app's JWKS cached.
- JWKS URL: `https://auth.authaz.io/.well-known/jwks/{clientId}.json` (no `/universal/` prefix — verified in `authaz/Authaz.AuthServer/UniversalRoutes.cs`). Legacy `/.well-known/jwks.json` still works but needs `?appId=` and is deprecated.
- Cache TTL ≤ 1 hour; refetch on `kid` miss or after a key rotation.

**"token expired":** access token `exp` passed — use refresh-token flow (SDK's `/api/auth/refresh`) or force re-login. If already expired right after sign-in, server clock is wrong — NTP it.

**Callback redirect loop** (bounces `/api/auth/login` ↔ `/auth/callback`):
- Session cookie not set — likely `Secure`/`SameSite` mismatch in dev. Run dev over HTTPS, or set SDK cookie config `secure: false` only for `NODE_ENV=development`.
- Middleware matcher wrongly protects `/auth/callback` — confirm `publicPaths` in `createAuthMiddleware`.
- Reverse proxy strips the cookie — confirm `X-Forwarded-Proto: https` is forwarded.

**Multi-tenant: `tenant_id` claim missing:** user has no tenants, or exactly one (Authaz skips picker, binds to it — confirm). If they have multiple but claim is still null:
- Sign-in used a tenant hint bypassing selection — remove `?tenant_hint=`.
- App is `single_tenant` (no tenant claims issued) — check `GET /api/v1/applications/{appId}` → `tenancy_type` should be `multi_tenant`.

## Step 3 — Reproduce with cURL when stuck

```bash
# Build the authorize URL with PKCE
CV=$(openssl rand -base64 32 | tr -d '=' | tr '/+' '_-')
CC=$(printf "%s" "$CV" | openssl dgst -sha256 -binary | openssl base64 | tr -d '=' | tr '/+' '_-')

open "https://auth.authaz.io/universal/oauth2/authorize?client_id=app_01abc&response_type=code&scope=openid&redirect_uri=http://localhost:3000/auth/callback&state=abc&code_challenge=$CC&code_challenge_method=S256"
```

After Authaz redirects with `?code=...`:

```bash
curl -X POST https://auth.authaz.io/universal/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code" \
  -d "code=THE_CODE_FROM_REDIRECT" \
  -d "client_id=app_01abc" \
  -d "client_secret=cs_live_..." \
  -d "redirect_uri=http://localhost:3000/auth/callback" \
  -d "code_verifier=$CV"
```

Manual round-trip works → SDK config is wrong. Fails the same way → app config is wrong.

## Anti-patterns

- Don't disable PKCE to dodge errors — restore the `code_verifier` instead.
- Don't loosen `Secure`/`SameSite` in production — fix dev with HTTPS, or fix prod's cookie domain/proto.
- Don't catch the OAuth error and silently retry — a fresh code request masks the underlying bug.
- Don't log the full access token, refresh token, or `client_secret` — log a prefix at most (`token[0..6]`).

## References

- OAuth 2.1 reference: <https://authaz.io/docs/authentication/oauth2>
- `references/error-codes.md` — every error you might see
- `references/endpoints.md` — all endpoint URLs and JWKS path
