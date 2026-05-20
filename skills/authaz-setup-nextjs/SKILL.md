---
name: authaz-setup-nextjs
description: Use when adding Authaz authentication to a Next.js app (App Router). Wires `@authaz/next` for OAuth 2.1 + PKCE, server-side session, and middleware-based route protection. Picks single- vs multi-tenant based on the user's app shape. Triggers on "add Authaz to Next.js", "@authaz/next", "Next.js login".
---

# Set up Authaz in a Next.js app

For a Next.js 14+ App Router app. If the project uses the Pages Router, mention it but follow the App Router shape — that's what `@authaz/next` is built for.

## Pre-flight

1. **Confirm App Router.** Look for `app/` (App Router) vs `pages/` (Pages Router). If only `pages/` exists, ask whether they're willing to add an `app/` directory or if you should fall back to the older `pages/api/auth` shape.
2. **Confirm single- vs multi-tenant.** If `authaz-quickstart` already established this, use that answer. Otherwise ask now.
3. **Confirm they have an application created.** They need `client_id`, `client_secret`, and the identity domain. If not, walk through step 2 below.

## Step 1 — Create or pick the application

In the Authaz Dashboard, **New application** → choose the tenancy type that matches step pre-flight 2 → add `http://localhost:3000/auth/callback` as an allowed callback URL.

Or via the Management API:

```http
POST https://api.rorix.io/api/v1/applications
X-API-Key: sk_live_…
Content-Type: application/json

{ "name": "my-nextjs-app", "tenancy_type": "single_tenant" }
```

For multi-tenant: `"tenancy_type": "multi_tenant"` and `"tenancy_mode": "shared"` (default).

## Step 2 — Install

```bash
pnpm add @authaz/next @authaz/sdk
```

For multi-tenant, also install `jose` to decode the access token's `tenant_id` claim:

```bash
pnpm add jose
```

## Step 3 — Configure

Create `.env.local`:

```
AUTHAZ_CLIENT_ID=app_01abc…
AUTHAZ_CLIENT_SECRET=cs_live_…
AUTHAZ_IDENTITY_DOMAIN=https://auth.rorix.io
```

Add `.env.local` to `.gitignore` if it's not already there.

## Step 4 — Wire it up

Follow the canonical recipe for the tenancy mode:

- **Single-tenant**: <https://authaz.io/docs/recipes/nextjs-single-tenant>
- **Multi-tenant**: <https://authaz.io/docs/recipes/nextjs-multi-tenant>

Both walk through the same shape:

1. **Auth handler** at `app/api/auth/[...authaz]/route.ts` — calls `createAuthazHandler({ clientId, clientSecret, authazIdentityDomain })` and exports `GET, POST`.
2. **Callback page** at `app/auth/callback/page.tsx` — a "use client" component that converts Authaz's redirect (GET) into a POST to `/api/auth/callback`. Don't try to handle the OAuth callback at `/api/auth/callback` directly — Authaz redirects to a *page*.
3. **Middleware** at `middleware.ts` — calls `createAuthMiddleware({ publicPaths: ["/", "/api/auth/*", "/auth/callback"], loginPath: "/api/auth/login" })`.
4. **Reading the user** in a server component — single-tenant uses `requireUser({...}).getOrRedirect()`; multi-tenant decodes the access token via `jose`'s `decodeJwt` to read `tenant_id`.
5. **Login** is just a link to `/api/auth/login`. **Logout** is a `<form method="POST" action="/api/auth/logout">` — never a link.

Copy the code from the recipe directly. The shapes above are the contract; don't paraphrase them from memory.

## Step 5 — Verify

```bash
pnpm dev
```

Then in the browser:

1. Visit `http://localhost:3000` — should be public.
2. Click **Sign in** — should redirect to `auth.rorix.io`, complete the password flow.
3. Land on `/dashboard` (or your protected page) signed in.
4. Click **Sign out** — should clear the session and bounce you back to `/`.

For multi-tenant: confirm the `tenant_id` claim appears in the decoded token. Browser DevTools → Application → Cookies → find the Authaz session cookie → decode the JWT.

If anything fails, the most common cause is a callback URL mismatch — see `authaz-troubleshoot-oauth`.

## Anti-patterns

- **Don't store tokens in `localStorage`.** The SDK uses HttpOnly cookies. Don't fight it.
- **Don't put `AUTHAZ_CLIENT_SECRET` in `NEXT_PUBLIC_*`.** It's server-only.
- **Don't skip PKCE / `state`.** The SDK handles both; if you bypass the handler you lose them.
- **Don't read `tenant_id` from a query string or header** in multi-tenant — only from the token.
- **Don't add `Cache-Control: public` to protected pages.** Authenticated pages must not be CDN-cached.
- **Don't make `/auth/callback` an API route.** It's a page that POSTs into the handler — see step 4.2.

## References

- Recipe (single-tenant): <https://authaz.io/docs/recipes/nextjs-single-tenant>
- Recipe (multi-tenant): <https://authaz.io/docs/recipes/nextjs-multi-tenant>
- `references/endpoints.md` — endpoints, JWKS URL, claims
- `references/error-codes.md` — when something breaks
- `authaz-troubleshoot-oauth` — for redirect_uri / token / PKCE failures
- `authaz-multi-tenant` — for tenant-scoped queries and permission checks
