---
name: authaz-protect-route
description: Use when gating an endpoint or page behind authentication after the initial Authaz setup is done. Adds the right framework primitive — middleware matcher, `withAuth`, `[Authorize]`, `useRequireUser` — without re-doing the install. Triggers on "protect this route", "make this page require login", "require auth", "gate endpoint".
---

# Protect a route

For when Authaz is already wired up and you just need to add another protected route. If Authaz isn't set up yet, run the matching `authaz-setup-*` skill first.

## Step 1 — Decide where the protection goes

| Where the user comes from | Where to enforce |
|---|---|
| Browser, full page navigation | **Server-side** (middleware, `[Authorize]`, `requireUser`) — never client-only |
| Browser, fetching from your API | Server-side on the API — the SPA can hint by hiding UI but cannot be the gate |
| Service-to-service (M2M) | API key or M2M access token check, not a user session |

A common mistake: gating a page with only a `useRequireUser` hook in React. That redirects after the page loads — a user with the source can read the page's secrets in transit. **Always gate on the server too.**

## Step 2 — Apply the framework primitive

### Next.js (App Router)

Add the path to the middleware matcher (already configured by `authaz-setup-nextjs`). For finer control, use `requireUser` inside the page:

```tsx
// app/admin/page.tsx
import { requireUser } from "@authaz/next";

const user = requireUser({
  authazDomain: process.env.AUTHAZ_IDENTITY_DOMAIN,
  clientSecret: process.env.AUTHAZ_CLIENT_SECRET,
});

export default async function Admin() {
  const me = await user.getOrRedirect();
  return <main>Hi {me.email}</main>;
}
```

For an API route, wrap the handler with `withAuth`:

```ts
// app/api/admin/route.ts
import { withAuth } from "@authaz/next";

export const POST = withAuth(async (req) => {
  // req.user is populated; if not signed in, withAuth returned 401 already
  return Response.json({ ok: true });
});
```

### Hono

Either rely on the global middleware from `authaz-setup-hono` (and add the path *outside* `publicPaths`), or check inline:

```ts
import { isAuthenticated } from "@authaz/hono";

app.get("/admin", (c) => {
  if (!isAuthenticated(c)) return c.json({ error: "unauthorized" }, 401);
  return c.json({ ok: true });
});
```

### React SPA

`useRequireUser` for client-side redirects (UX), and the **server** must still enforce auth on every API call:

```tsx
import { useRequireUser } from "@authaz/react";

export default function AdminPage() {
  const { user, isLoading } = useRequireUser();
  if (isLoading) return <Spinner />;
  return <h1>Admin — {user.email}</h1>;
}
```

If your data loads via `fetch("/api/admin", { credentials: "include" })`, that endpoint *must* be protected by `withAuth` / `isAuthenticated` / `[Authorize]` on the backend. Client-side gating alone is not security.

### ASP.NET Core

```csharp
app.MapGet("/admin", () => Results.Ok(new { ok = true }))
   .RequireAuthorization();
```

Or `[Authorize]` on a controller method. For role-based:

```csharp
.RequireAuthorization(policy => policy.RequireRole("admin"));
```

For permission checks beyond simple role membership, see `authaz-permission-check`.

## Step 3 — Verify

For each protected route:

1. **Anonymous request** — should 302 to login (page) or return 401 (API).
2. **Signed-in request** — should reach the handler.
3. **Wrong role / missing permission** — should return 403 (not 401).

If the protected route loads but auth was supposed to redirect, the matcher likely doesn't include the path. Double-check `middleware.ts` `matcher` (Next) or `app.use("*", ...)` ordering (Hono).

## Anti-patterns

- **Don't use the access token as a database key** to look up "is this user allowed". The token already proves identity; use Authaz's permission check (`/api/v1/authz/check`) for "allowed".
- **Don't mix client-only gating with server returns of protected data.** Either every request is server-validated, or you have a vulnerability.
- **Don't 301-redirect protected pages to login.** Use 302 (or `Results.Challenge()` in .NET) — 301s get cached and break login UX.
- **Don't strip the cookie's `SameSite` or `Secure` flags** to make local dev work — set up `localhost` HTTPS instead.

## References

- `authaz-setup-{nextjs,hono,react,dotnet}` — the underlying wiring
- `authaz-permission-check` — when "logged in" isn't enough
- `references/error-codes.md` — for 401 / 403 troubleshooting
