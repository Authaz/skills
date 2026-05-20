---
name: authaz-setup-hono
description: Use when adding Authaz authentication to a Hono backend (Node, Bun, or edge). Wires `@authaz/hono` for OAuth 2.1 + PKCE, cookie-based sessions, and `createAuthMiddleware` route protection. Triggers on "add Authaz to Hono", "@authaz/hono", "Hono auth".
---

# Set up Authaz in a Hono backend

Targets Hono on Node or Bun. Edge runtimes work too but cookie behavior differs — call out the difference if the user's project uses Cloudflare Workers / Vercel Edge.

## Pre-flight

1. **Confirm runtime.** Node, Bun, or edge? Affects the install (`@hono/node-server` vs `bun run`).
2. **Confirm single- vs multi-tenant.** Use the answer from `authaz-quickstart` if available.
3. **Confirm the app exists in Authaz** — `client_id`, `client_secret`, and identity domain.

## Step 1 — Application setup

Dashboard or:

```http
POST https://api.rorix.io/api/v1/applications
X-API-Key: sk_live_…
Content-Type: application/json

{ "name": "my-hono-api", "tenancy_type": "single_tenant" }
```

Allowed callback URL: `http://localhost:3000/auth/callback` (for local dev).

## Step 2 — Install

```bash
pnpm add hono @authaz/hono @authaz/sdk
pnpm add -D @hono/node-server tsx   # if Node
```

(Skip `@hono/node-server` for Bun.)

## Step 3 — Configure

`.env`:

```
AUTHAZ_CLIENT_ID=app_01abc…
AUTHAZ_CLIENT_SECRET=cs_live_…
AUTHAZ_IDENTITY_DOMAIN=https://auth.rorix.io
```

## Step 4 — Wire it up

Recipe:

- **Single-tenant**: <https://authaz.io/docs/recipes/hono-single-tenant>
- **Multi-tenant**: <https://authaz.io/docs/recipes/hono-multi-tenant>

The shape is:

1. **Mount the auth handler**: `app.route("/api/auth", createAuthazHandler({ clientId, clientSecret, authazIdentityDomain }))`. This exposes `/api/auth/{login,callback,logout,me,refresh}`.
2. **Add the middleware**: `app.use("*", createAuthMiddleware({ publicPaths: ["/", "/api/auth/*", "/auth/callback"], loginPath: "/api/auth/login" }))`.
3. **Use `isAuthenticated(c)` inside protected handlers** to short-circuit unauthenticated requests with a 401.
4. **For multi-tenant**, decode the JWT cookie with `jose.decodeJwt` to read `tenant_id`. Validate the signature against the JWKS at `${AUTHAZ_IDENTITY_DOMAIN}/universal/.well-known/jwks/${client_id}.json` before trusting any claim in business logic.

Copy from the recipe — don't paraphrase. The function names and middleware signature are exact.

## Step 5 — Verify

```bash
pnpm tsx src/index.ts   # or bun run src/index.ts
```

Then:

1. `curl http://localhost:3000/` — public, returns the welcome string.
2. Visit `http://localhost:3000/me` in a browser — bounces to Authaz Sign-In, then back to `/me` returning JSON `{ email, sub }`.
3. `curl -i http://localhost:3000/protected` (no cookie) → 401.
4. After signing in, the same request with the session cookie → 200.

If the pure-API client (no browser) needs to authenticate, that's M2M, not user auth — see `authaz-add-provider` and search for "client credentials".

## Anti-patterns

- **Don't roll your own cookie-signing logic.** The middleware handles signing, encryption, and rotation.
- **Don't trust the cookie's claims without verifying the JWT signature** when the data comes from the *access token* (the part the browser can't tamper with). The session cookie is HttpOnly and SDK-signed, but anything you read from `Authorization: Bearer` headers must be JWKS-verified.
- **Don't skip CORS configuration if the React SPA recipe pairs with this.** Allow credentials and the SPA's origin only.
- **Don't tee the auth handler under a path that conflicts** (e.g., `/api`). `/api/auth` is the convention — keep it.

## References

- Recipe (single-tenant): <https://authaz.io/docs/recipes/hono-single-tenant>
- Recipe (multi-tenant): <https://authaz.io/docs/recipes/hono-multi-tenant>
- `references/endpoints.md` — JWKS URL, claims
- `references/error-codes.md`
- `authaz-troubleshoot-oauth`, `authaz-multi-tenant`, `authaz-permission-check`
