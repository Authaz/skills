---
name: authaz-setup-nextjs
description: Use when adding Authaz authentication to a Next.js 14+/15 App Router app. Single-shot — writes every required file end to end using `@authaz/next` + `@authaz/react`. Triggers on "add Authaz to Next.js", "set up authentication in Next.js", "@authaz/next".
---

# Set up Authaz in a Next.js app — single shot

> **Last verified:** `@authaz/next` + `@authaz/react` v2.1.0 (2026-05-21). If the installed SDK exports don't match (function names, props, env var contract), trust the SDK source over this skill and report the drift.

Use this skill when the user wants Authaz wired into a Next.js App Router app in one pass. The code below is taken verbatim from `authaz-sdk-js/examples/nextjs` — keep it that way. Do not improvise function names, props, or env vars from memory.

If the project is **Pages Router only** (no `app/` directory), stop and tell the user that `@authaz/next` targets the App Router. Offer to add an `app/` directory alongside the existing `pages/`.

## What the user must provide before you start

Three values from the Authaz Dashboard:

| Env var | Where it comes from |
|---|---|
| `AUTHAZ_CLIENT_ID` | Dashboard → your application → Auth Flow Configuration |
| `AUTHAZ_CLIENT_SECRET` | Same place. Shown once at creation — if lost, rotate it |
| `AUTHAZ_TENANT_ID` | Dashboard → tenant. For single-tenant apps, use the default tenant Authaz created for your org |

Plus: in the Authaz Dashboard, add `http://localhost:3000/auth/callback` to the application's **Allowed callback URLs**. Add the production URL when you deploy.

If any of these are missing, stop and ask for them — they cannot be inferred.

The SDK defaults `authazDomain` and `authazIdentityDomain` to `https://auth.authaz.io` and `apiDomain` to `https://api.authaz.com`. Override only if the customer has a custom domain — add `authazDomain` / `apiDomain` options to the handler call in that case.

## Step 1 — Install

```bash
pnpm add @authaz/next @authaz/react
```

(Use `npm install` / `yarn add` if the project isn't on pnpm — match the project's package manager.)

If the project doesn't have Tailwind yet, the example uses it. Skip the Tailwind bits if the user already has a styling solution.

## Step 2 — Write `.env.local`

```
AUTHAZ_CLIENT_ID=your_client_id
AUTHAZ_CLIENT_SECRET=your_client_secret
AUTHAZ_TENANT_ID=your_tenant_id
```

Make sure `.env.local` is in `.gitignore` (Next.js's default `.gitignore` already covers it).

## Step 3 — Write the source files

Paths below assume `src/app/`. If the project has no `src/` dir, use `app/`/`components/` at the root instead — check before writing.

### `src/app/api/auth/[...authaz]/route.ts`

```ts
import { createAuthazHandler } from "@authaz/next";

export const { GET, POST } = createAuthazHandler({
  clientId: process.env.AUTHAZ_CLIENT_ID!,
  clientSecret: process.env.AUTHAZ_CLIENT_SECRET!,
  tenantId: process.env.AUTHAZ_TENANT_ID!,
  afterLoginUrl: "/dashboard",
  afterLogoutUrl: "/",
  debug: process.env.NODE_ENV === "development",
});
```

### `src/app/auth/callback/page.tsx`

```tsx
"use client";

import { Suspense, useEffect } from "react";
import { useSearchParams } from "next/navigation";

const CallbackContent = (): React.ReactNode => {
  const searchParams = useSearchParams();

  useEffect(() => {
    const form = document.createElement("form");
    form.method = "POST";
    form.action = "/api/auth/callback";

    const params = ["code", "state", "error", "error_description"];
    params.forEach((param) => {
      const value = searchParams.get(param);
      if (value) {
        const input = document.createElement("input");
        input.type = "hidden";
        input.name = param;
        input.value = value;
        form.appendChild(input);
      }
    });

    document.body.appendChild(form);
    form.submit();
  }, [searchParams]);

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <div className="text-center">
        <div className="inline-block h-8 w-8 animate-spin rounded-full border-4 border-solid border-blue-600 border-r-transparent" />
        <p className="mt-4 text-gray-600">Completing login...</p>
      </div>
    </div>
  );
};

const CallbackPage = (): React.ReactNode => {
  return (
    <Suspense
      fallback={
        <div className="min-h-screen flex items-center justify-center bg-gray-50">
          <div className="text-center">
            <div className="inline-block h-8 w-8 animate-spin rounded-full border-4 border-solid border-blue-600 border-r-transparent" />
            <p className="mt-4 text-gray-600">Loading...</p>
          </div>
        </div>
      }
    >
      <CallbackContent />
    </Suspense>
  );
};

export default CallbackPage;
```

This page is the bridge between Authaz's GET-redirect and the handler's POST endpoint. **Do not make `/auth/callback` an API route** — the OAuth response arrives via browser redirect and needs the client-side POST to attach the code.

### `src/app/layout.tsx`

If `app/layout.tsx` already exists, only add the `AuthazProvider` wrapper — keep the user's existing markup and metadata.

```tsx
import type { Metadata } from "next";
import { AuthazProvider } from "@authaz/react";

export const metadata: Metadata = {
  title: "Authaz Next.js App",
  description: "Next.js app with Authaz authentication",
};

const RootLayout = ({ children }: { children: React.ReactNode }): React.ReactNode => {
  return (
    <html lang="en">
      <body className="min-h-screen bg-gray-50 text-gray-900 antialiased">
        <AuthazProvider basePath="/api/auth" autoRefresh={true}>
          {children}
        </AuthazProvider>
      </body>
    </html>
  );
};

export default RootLayout;
```

### `src/app/dashboard/page.tsx` — a protected page

```tsx
"use client";

import { useRequireAuth, useAuthaz } from "@authaz/react";

const DashboardPage = (): React.ReactNode => {
  useRequireAuth();
  const { user } = useAuthaz();

  return (
    <main className="max-w-7xl mx-auto px-4 py-12">
      <h1 className="text-3xl font-bold mb-4">Dashboard</h1>
      <p>Signed in as <strong>{user?.email}</strong></p>
    </main>
  );
};

export default DashboardPage;
```

`useRequireAuth()` redirects unauthenticated visitors to `/api/auth/login`. Use it in any client component that should be authenticated-only. For gating many routes at once via middleware instead of per-page, use `authaz-protect-route`.

### Optional — `src/components/Navbar.tsx` for login/logout buttons

```tsx
"use client";

import Link from "next/link";
import { useAuthaz } from "@authaz/react";

export const Navbar = (): React.ReactNode => {
  const { isAuthenticated, isLoading, user, login, logout } = useAuthaz();

  return (
    <nav className="bg-white shadow-sm border-b border-gray-200">
      <div className="max-w-7xl mx-auto px-4 flex justify-between h-16 items-center">
        <Link href="/" className="text-xl font-semibold">Authaz Demo</Link>
        {isLoading ? null : isAuthenticated ? (
          <div className="flex items-center gap-4">
            <span className="text-sm text-gray-600">{user?.email}</span>
            <button onClick={() => logout()} className="px-3 py-1.5 rounded bg-gray-100 hover:bg-gray-200">
              Logout
            </button>
          </div>
        ) : (
          <button onClick={() => login()} className="px-3 py-1.5 rounded bg-blue-600 text-white hover:bg-blue-700">
            Login
          </button>
        )}
      </div>
    </nav>
  );
};
```

`login()` / `logout()` from `useAuthaz()` are the canonical client-side handles for triggering the flow — use them instead of constructing the URLs by hand.

## Step 4 — Run and verify

```bash
pnpm dev
```

Then in the browser:

1. `http://localhost:3000` — loads, **Login** button visible.
2. Click **Login** → redirects to `https://auth.authaz.io/...` (or the custom identity domain if configured) → complete the flow.
3. Lands back on `/dashboard` showing the signed-in user's email.
4. Click **Logout** → cleared session, back at `/`.

If you don't have a Navbar, navigate manually:
- `http://localhost:3000/api/auth/login` triggers the flow (GET works, it's a redirect).
- `http://localhost:3000/dashboard` directly tests the guard.
- Logout is **POST-only** (same CSRF protection as callback) — typing `/api/auth/logout` in the address bar sends GET and 405s. Use the Navbar's `logout()` button, or `fetch("/api/auth/logout", { method: "POST" })` from the console.

## When something fails

Almost every failure on the first run is one of:

1. **`invalid_redirect_uri`** on the Authaz page — the URL the browser used isn't on the application's allowed callback list. Add the exact URL (with port, with/without trailing slash matching the request) in the Dashboard.
2. **`invalid_grant`** on the callback — usually a double-POST (React StrictMode mounting twice). The example callback page is structured to avoid this; if you customized it, restore the `useEffect` body.
3. **No login button / `useAuthaz` is undefined** — the `AuthazProvider` is missing or wraps too narrowly. It must wrap *all* components that call `useAuthaz()` or `useRequireAuth()`.
4. **Cookie not set** — running over plain `http` in production? The session cookie is `Secure`. Use HTTPS in production. In dev, `localhost` is exempt.

For anything else, hand off to `authaz-troubleshoot-oauth`.

## Production checklist

When you move beyond `localhost`:

- [ ] **Add the prod callback URL** in the Dashboard's Allowed callback URLs: `https://your-app.com/auth/callback` (exact match, including scheme).
- [ ] **Move `AUTHAZ_*` env vars to your hosting platform's secret store** (Vercel project envs, AWS Secrets Manager, etc.). Don't commit `.env.local`.
- [ ] **HTTPS only.** The SDK's session cookie is `Secure` — `http://your-app.com` will fail silently with empty `user`.
- [ ] **Behind a reverse proxy or CDN?** Ensure `X-Forwarded-Proto: https` reaches Next.js (Vercel does this; raw Cloudflare in front of a bare server often doesn't).
- [ ] **Custom identity domain?** Set `authazDomain` (and `authazIdentityDomain` if needed) on `createAuthazHandler` to your domain — and add the same custom domain in the Dashboard.
- [ ] **Drop debug.** `debug: process.env.NODE_ENV === "development"` already disables verbose logs in prod, but double-check nothing logs the access token.

## Anti-patterns

- **Never put `AUTHAZ_CLIENT_SECRET` in a `NEXT_PUBLIC_*` var.** It's server-only.
- **Never store tokens in `localStorage`.** The SDK uses HttpOnly cookies; that's the design.
- **`/auth/callback` is a page, not an API route.** The POST is to `/api/auth/callback`, which is the handler — both live under different paths.
- **Don't add `AUTHAZ_IDENTITY_DOMAIN` unless the customer has a custom identity domain.** The SDK defaults to `https://auth.authaz.io` — let it.
- **Don't construct OAuth URLs by hand.** Use `login()` / `logout()` from `useAuthaz()` on the client and `/api/auth/login` etc. on the server.

## Source of truth

The code above is lifted from `authaz-sdk-js/examples/nextjs/`. If the SDK ships a new major version, re-check the example before trusting this skill's snippets. The file layout there is:

```
src/
├── app/
│   ├── api/auth/[...authaz]/route.ts
│   ├── auth/callback/page.tsx
│   ├── dashboard/page.tsx
│   ├── layout.tsx
│   └── page.tsx
└── components/
    ├── Navbar.tsx
    └── UserProfile.tsx
```

## References

- Real example: `authaz-sdk-js/examples/nextjs/`
- SDK source: `authaz-sdk-js/packages/next/src/index.tsx`
- `authaz-troubleshoot-oauth` — failure diagnostics
- `authaz-multi-tenant` — tenant scoping in business logic
- `authaz-permission-check` — runtime authorization checks
