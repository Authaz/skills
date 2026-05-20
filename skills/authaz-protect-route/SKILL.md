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

### Next.js (App Router) — client component

```tsx
"use client";
import { useRequireAuth, useAuthaz } from "@authaz/react";

export default function AdminPage() {
  useRequireAuth();
  const { user } = useAuthaz();
  return <h1>Admin — {user?.email}</h1>;
}
```

`useRequireAuth()` redirects unauthenticated visitors through `/api/auth/login`.

For server-side checks (route handlers), call the handler's `/api/auth/me` from the request to confirm the session, or read the session cookie via `next/headers` and decode the access token. Don't expect a `requireUser` helper in `@authaz/next` — there isn't a stable one beyond the React hooks.

### Hono

The handler already exposes `/api/auth/me`. Use that as the auth check for downstream routes:

```ts
import type { Context, Next } from "hono";

const requireAuth = async (c: Context, next: Next) => {
  const me = await app.fetch(
    new Request(`http://localhost:${process.env.PORT || 3000}/api/auth/me`, {
      headers: { cookie: c.req.header("cookie") || "" },
    })
  );
  if (!me.ok) return c.json({ error: "Unauthorized" }, 401);
  return next();
};

app.use("/api/protected/*", requireAuth);
app.get("/api/protected/hello", (c) => c.json({ message: "hi" }));
```

### React SPA — `beforeLoad` (TanStack Router)

```tsx
import { createFileRoute, redirect } from "@tanstack/react-router";

export const Route = createFileRoute("/admin")({
  beforeLoad: async () => {
    const response = await fetch("/api/auth/me", { credentials: "include" });
    if (!response.ok) {
      throw redirect({ to: "/api/auth/login" });
    }
  },
  component: AdminPage,
});
```

If your data loads via `fetch("/api/admin", { credentials: "include" })`, that endpoint *must* be protected on the backend. Client-side gating alone is not security.

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

- **Don't use the access token as a database key** to look up "is this user allowed". The token already proves identity; use the SDK's permission check (`authaz.authz.check` in JS, `authaz.Authorization.Permissions.CheckAsync` in .NET) for "allowed".
- **Don't mix client-only gating with server returns of protected data.** Either every request is server-validated, or you have a vulnerability.
- **Don't 301-redirect protected pages to login.** Use 302 (or `Results.Challenge()` in .NET) — 301s get cached and break login UX.
- **Don't strip the cookie's `SameSite` or `Secure` flags** to make local dev work — set up `localhost` HTTPS instead.

## References

- `authaz-setup-{nextjs,hono,react,dotnet}` — the underlying wiring
- `authaz-permission-check` — when "logged in" isn't enough
- `references/error-codes.md` — for 401 / 403 troubleshooting
