---
name: authaz-setup-hono
description: Use when adding Authaz authentication to a Hono backend on Node (with `@hono/node-server`). Single-shot — writes the server entry plus the auth-handler module using `@authaz/hono`. Triggers on "add Authaz to Hono", "set up authentication in Hono", "@authaz/hono".
---

# Set up Authaz in a Hono backend — single shot

> **Last verified:** `@authaz/hono` v1.9.10 (2026-05-21). If installed SDK exports don't match, trust the SDK source over this skill and report the drift.

Wires Authaz into a Hono server in one pass. Code lifted verbatim from `authaz-sdk-js/examples/react-hono/server/`.

- **Scope:** server only. Wiring a React SPA against this server → hand off to `authaz-setup-react` after this skill finishes (same example, other half).
- **Runtime:** Bun / Cloudflare Workers / Vercel Edge — skill still applies, skip `@hono/node-server`, use the runtime's native server. Auth handler module is identical.

## Required inputs — stop and ask if missing, they can't be inferred

Three values from the Authaz Dashboard:

| Env var | Where it comes from |
|---|---|
| `AUTHAZ_CLIENT_ID` | Dashboard → your application → Auth Flow Configuration |
| `AUTHAZ_CLIENT_SECRET` | Same place. Shown once at creation — if lost, rotate it |
| `AUTHAZ_TENANT_ID` | Dashboard → tenant. Single-tenant apps use the default tenant Authaz created for the org |

Also add the browser-facing callback URL to Dashboard → **Allowed callback URLs**:
- Server alone is the front door: `http://localhost:3000/auth/callback`
- Vite SPA paired in front (default port): `http://localhost:5173/auth/callback`

**Domains:** SDK defaults to `https://auth.authaz.io` and `https://api.authaz.com`. Only override with a custom identity domain via `authazIdentityDomain` on `createAuthazHandler()` — it's the sole override that reliably affects every endpoint here, including `/me` (reads `config.authazIdentityDomain` directly, no fallback). `apiDomain` has no effect on anything this handler calls — don't set it here.

## Step 1 — Install

```bash
pnpm add hono @authaz/hono
pnpm add -D @hono/node-server tsx
```
Bun: replace `@hono/node-server` with `bun run`. Workers: skip both.

## Step 2 — `.env`

```
AUTHAZ_CLIENT_ID=your_client_id
AUTHAZ_CLIENT_SECRET=your_client_secret
AUTHAZ_TENANT_ID=your_tenant_id
PORT=3000
```
Add `.env` to `.gitignore`.

## Step 3 — Source files

### `server/auth.ts`

```ts
import { createAuthazHandler } from "@authaz/hono";

export const authHandler = createAuthazHandler({
  clientId: process.env.AUTHAZ_CLIENT_ID!,
  clientSecret: process.env.AUTHAZ_CLIENT_SECRET!,
  tenantId: process.env.AUTHAZ_TENANT_ID!,
  afterLoginUrl: "/dashboard",
  afterLogoutUrl: "/",
  debug: process.env.NODE_ENV === "development",
});
```
`afterLoginUrl` / `afterLogoutUrl` = landing routes after the OAuth round-trip succeeds. Match to actual app routes.

### `server/index.ts`

```ts
import { serve } from "@hono/node-server";
import { Hono } from "hono";
import { authHandler } from "./auth.js";

const app = new Hono();

// Mount auth routes — exposes /api/auth/{login,callback,logout,me,refresh}
app.route("/api/auth", authHandler);

// Bridge page: the IdP redirects here with ?code=&state= on a GET request,
// but POST /api/auth/callback only reads form data, not query params.
// This page re-POSTs those values as a form. Required only if the server
// itself is the front door (no paired SPA already doing this).
app.get("/auth/callback", (c) => {
  return c.html(`<!doctype html>
<script>
  const params = new URLSearchParams(window.location.search);
  const form = document.createElement("form");
  form.method = "POST";
  form.action = "/api/auth/callback";
  for (const [key, value] of params) {
    const input = document.createElement("input");
    input.type = "hidden";
    input.name = key;
    input.value = value;
    form.appendChild(input);
  }
  document.body.appendChild(form);
  form.submit();
</script>`);
});

// Your application routes go here, e.g.:
// app.get("/api/me", async (c) => { ... });

const port = parseInt(process.env.PORT || "3000", 10);

console.log(`Server running at http://localhost:${port}`);

serve({ fetch: app.fetch, port });
```

Endpoints created under `/api/auth`:

| Route | Purpose |
|---|---|
| `GET /api/auth/login` | start the OAuth flow |
| `POST /api/auth/callback` | receive the auth code from the browser-side callback page |
| `POST /api/auth/logout` | end the session (POST-only by design, to prevent CSRF) |
| `GET /api/auth/me` | read the current user (401 if signed out) |
| `POST /api/auth/refresh` | refresh the access token |

Keep the prefix exactly `/api/auth` — `@authaz/react`'s client and the standalone callback page both expect this path.

### Optional — gating your own routes

```ts
// server/index.ts (add below the auth mount)
import type { Context, Next } from "hono";

const requireAuth = async (c: Context, next: Next) => {
  const meResponse = await app.fetch(
    new Request(`http://localhost:${process.env.PORT || 3000}/api/auth/me`, {
      headers: { cookie: c.req.header("cookie") || "" },
    })
  );
  if (!meResponse.ok) return c.json({ error: "Unauthorized" }, 401);
  return next();
};

app.use("/api/protected/*", requireAuth);
app.get("/api/protected/hello", (c) => c.json({ message: "hi, signed-in user" }));
```
`/api/auth/me` exists precisely so business routes can re-check session state via this call instead of implementing their own cookie parsing or JWT verifier.

## Step 4 — Run and verify

```bash
pnpm tsx watch server/index.ts
```

1. `curl http://localhost:3000/api/auth/me` → `401 Unauthorized` (endpoint exists; just not signed in).
2. Open `http://localhost:3000/api/auth/login` in a browser → redirects to `https://auth.authaz.io/...` → complete sign-in.
3. After the callback, browser has Authaz session cookies. Reload `curl http://localhost:3000/api/auth/me` with cookies (use DevTools to confirm no 401).
4. `curl -X POST http://localhost:3000/api/auth/logout` → clears cookies; `/me` is 401 again.

Pairing with a Vite SPA → also need the SPA-side callback page at `/auth/callback`; that's in `authaz-setup-react` — invoke it next.

## When something fails

| Symptom | Cause / fix |
|---|---|
| `invalid_redirect_uri` | URL the browser sent isn't on the application's allowed list. Hono runs on `:3000`, but if a Vite SPA proxies through, the *browser* URL is `:5173/auth/callback` — add the URL the browser actually used. |
| CORS errors | Only happens if the SPA isn't proxied through the Hono server. Example uses a Vite proxy (`/api → :3000`), so no CORS. If origins are split, add `app.use("*", cors({ origin: "...", credentials: true }))`. |
| 401 even after sign-in | Browser dropped the cookie. Check `Secure`/`SameSite`: dev — both servers on localhost; prod — both HTTPS, same site. |
| Handler 404s under `/api/auth/...` | `app.route("/api/auth", authHandler)` line missing or mounted under a different prefix. |

Other failures → hand off to `authaz-troubleshoot-oauth`.

## Production checklist

When you move beyond `localhost`:

- [ ] **Add the prod callback URL** to the Dashboard's Allowed callback URLs — the URL the *browser* lands on (if a CDN/SPA is in front, that's its domain, not the Hono server's).
- [ ] **Move env vars to your hosting platform's secret store.** Don't commit `.env`.
- [ ] **HTTPS only.** Session cookies are `Secure`.
- [ ] **`X-Forwarded-Proto: https`** must reach Hono so `Secure` cookies set correctly. With `@hono/node-server`, ensure the proxy forwards it.
- [ ] **Same-origin or CORS.** If the browser fetches `/api/auth/*` from a different origin than the Hono server, enable CORS with `credentials: true` and an explicit allowed origin (no wildcards).
- [ ] **Custom identity domain?** Pass `authazDomain` to `createAuthazHandler` and configure the same custom domain in the Dashboard.
- [ ] **Drop debug.** `debug: process.env.NODE_ENV === "development"` already handles it, but verify no upstream logs include tokens.

## Anti-patterns

- **Never leak `AUTHAZ_CLIENT_SECRET` to the browser.** Server-only.
- **Never sign your own session cookies.** The handler does it.
- **Keep the mount at `/api/auth`.** The browser callback page, the `AuthazProvider` `basePath` default, and the SPA hooks all assume this path.
- **Don't reimplement `/api/auth/me`.** The handler exposes it; use it for downstream auth checks.
- **Don't add `AUTHAZ_IDENTITY_DOMAIN`** unless the customer has a custom identity domain. SDK defaults to `https://auth.authaz.io`.

## Source of truth

Code above is lifted from `authaz-sdk-js/examples/react-hono/server/`. Corresponding SPA is in `authaz-sdk-js/examples/react-hono/src/` — see `authaz-setup-react`.

## References

- Real example: `authaz-sdk-js/examples/react-hono/`
- SDK source: `authaz-sdk-js/packages/hono/src/`
- `authaz-setup-react` — paired frontend
- `authaz-troubleshoot-oauth` — failure diagnostics
- `authaz-multi-tenant`, `authaz-permission-check`
