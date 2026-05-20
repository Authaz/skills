---
name: authaz-setup-react
description: Use when adding Authaz to a React single-page app (Vite, CRA, or similar). Wires `@authaz/react`'s `AuthazProvider`, `useUser`, `useRequireUser`. Requires a paired backend that runs the Authaz handler — usually the Hono backend. Triggers on "add Authaz to React", "@authaz/react", "useUser hook", "React SPA login".
---

# Set up Authaz in a React SPA

The React SDK is **client-side only**. It cannot complete the OAuth flow on its own — it talks to a backend that runs `createAuthazHandler` (Hono, Next.js API route, or any other Node/Bun server). Don't try to make the SPA hold the `client_secret`.

## Pre-flight

1. **Confirm a backend exists** that runs the Authaz handler at `/api/auth/*`. If not, pause this skill, run `authaz-setup-hono` first, then come back.
2. **Confirm bundler.** Vite is the recipe-supported path; CRA / Webpack work but the proxy config differs.
3. **Confirm tenancy mode** (from `authaz-quickstart` if available).

## Step 1 — Application setup

Already done if the paired backend was set up. The same `client_id` is reused — there is **one application**, not one per layer.

The redirect URI for the SPA must be added to the application's allowed callback URLs (e.g., `http://localhost:5173/auth/callback` for Vite).

## Step 2 — Install

```bash
pnpm add react react-dom @authaz/react @authaz/sdk react-router-dom
pnpm add -D vite @vitejs/plugin-react
```

## Step 3 — Configure

The SPA reads **no Authaz env vars** — it gets everything from the backend via `/api/auth/me`. The backend keeps the secret.

Configure the dev proxy so the cookie flows from the SPA's origin to the backend's. `vite.config.ts`:

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: { proxy: { "/api/auth": "http://localhost:3000" } },
});
```

In production, deploy the SPA and the backend behind the same origin (or set CORS with credentials, but same-origin is simpler).

## Step 4 — Wire it up

Recipe:

- **Single-tenant**: <https://authaz.io/docs/recipes/react-single-tenant>
- **Multi-tenant**: <https://authaz.io/docs/recipes/react-multi-tenant>

The shape:

1. **`AuthazProvider`** at the root of the tree: `<AuthazProvider basePath="/api/auth">`. Wraps your router.
2. **`/auth/callback` route** is a tiny component that POSTs the OAuth code into `/api/auth/callback` (same bridge as the Next.js recipe). Copy from the recipe.
3. **`useUser()`** in any component — returns `{ user, isLoading, isSignedIn }`.
4. **`useRequireUser()`** in protected pages — redirects to login if signed-out.
5. **`<SignInButton>` / `<SignOutButton>`** — convenience components that hit the right endpoints.
6. **For multi-tenant**, `useUser()` exposes `user.tenantId` — use it to scope your data fetches (`/api/invoices?...` becomes `/api/invoices?tenantId=${user.tenantId}` *only if* your backend re-validates from the JWT, never from the query).

## Step 5 — Verify

Run the backend (e.g., `pnpm tsx src/index.ts` on port 3000) and the SPA (`pnpm dev` on port 5173) at the same time.

1. Visit `http://localhost:5173` — public, no redirect.
2. Click **Sign in** — bounces to Authaz Sign-In.
3. After login, lands back on `/dashboard` showing the user's email.
4. Click **Sign out** — clears session, returns to `/`.
5. For multi-tenant: confirm `useUser().user.tenantId` is set in the React DevTools.

## Anti-patterns

- **Never put `client_secret` in the SPA bundle.** It's a public bundle. The backend holds the secret.
- **Don't use `localStorage` for tokens.** The cookie is HttpOnly; that's a feature, not a limitation.
- **Don't fetch protected APIs without `credentials: "include"`** — without it, the cookie won't be sent.
- **Don't gate routes with `useRequireUser` and *also* re-fetch the user inside the route.** It's already cached in the provider.
- **Don't build your own tenant picker.** Authaz Sign-In has one; let it run.

## References

- Recipe (single-tenant): <https://authaz.io/docs/recipes/react-single-tenant>
- Recipe (multi-tenant): <https://authaz.io/docs/recipes/react-multi-tenant>
- `authaz-setup-hono` — the usual paired backend
- `authaz-protect-route`, `authaz-multi-tenant`
