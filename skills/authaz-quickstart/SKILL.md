---
name: authaz-quickstart
description: Use when the user is starting an Authaz integration from scratch and hasn't named a framework yet. Detects the framework from the project, picks single- vs multi-tenant, and dispatches to the matching authaz-setup-* skill. Triggers on "add login", "add auth", "set up Authaz", "use authaz".
---

# Authaz quickstart router

You are not setting anything up yourself — your job is to figure out which `authaz-setup-*` skill applies and hand off to it.

## Step 1 — Detect the framework

Look at the project root in this order. Stop at the first match.

| Signal | Framework | Hand off to |
|---|---|---|
| `next.config.{js,ts,mjs}` or `"next"` in `package.json` deps | Next.js | `authaz-setup-nextjs` |
| `"hono"` in `package.json` deps | Hono | `authaz-setup-hono` |
| `vite.config.{js,ts}` and `"react"` in deps, no SSR framework | React SPA | `authaz-setup-react` |
| Any `*.csproj` with `Microsoft.NET.Sdk.Web` | ASP.NET Core | `authaz-setup-dotnet` |
| `package.json` with React but also Next/Remix/etc. | the SSR framework wins | (Next.js etc.) |

If multiple match (e.g. a monorepo with both Next.js and Hono), ask which one to set up first. Don't try to do them all in one pass.

If nothing matches, ask the user what stack they're on. The supported stacks are Next.js, Hono, React SPA, and ASP.NET Core. For anything else, point them at the [recipes index](https://authaz.io/docs/recipes/) — they may need a custom integration using `@authaz/sdk` (JS) or `Authaz.Sdk` (.NET) plus standard OIDC middleware.

## Step 2 — Confirm prerequisites

Before handing off, check that the user has these four values from the Authaz Dashboard:

| Value | Where it comes from |
|---|---|
| `clientId` | Dashboard → your application → Auth Flow Configuration |
| `clientSecret` | Same place. Shown **once** at creation — if lost, rotate it |
| `organizationId` | Dashboard → top-level (your Authaz organization). **Required by the JS SDK.** |
| `tenantId` | Dashboard → tenant. For single-tenant apps, use the default tenant Authaz created for your org. For multi-tenant B2B SaaS, this is set per-customer at runtime — but for initial setup, any tenant id will do |

Also confirm that the callback URL the browser will use (e.g., `http://localhost:3000/auth/callback` for Next.js, `http://localhost:5173/auth/callback` for Vite) is in the application's **Allowed callback URLs** list.

If they don't have an Authaz account yet, point them at <https://authaz.io/docs/getting-started>.

The JS SDK defaults to `https://auth.authaz.io` (identity domain) and `https://api.authaz.io` (Management API). The .NET SDK defaults to `https://api.authaz.io`. Override only if the customer set up a custom domain.

## Step 3 — Hand off

Invoke the chosen setup skill. The setup skill handles the single- vs multi-tenant question — it's purely a matter of which `tenantId` the user passes; there are no separate code paths.

After setup is done, suggest the natural next steps:

- Need to enable a different login method? → `authaz-add-provider`
- Need to gate more routes? → `authaz-protect-route`
- Need to check permissions inside a route? → `authaz-permission-check`
- Multi-tenant with tenant-scoped data? → `authaz-multi-tenant`
- Errors during the OAuth round-trip? → `authaz-troubleshoot-oauth`

## Anti-patterns

- Don't set up auth without confirming the framework — every SDK has different conventions and the wrong one creates work to undo
- Don't assume the user has API keys for the Management API; the OAuth/sign-in flow only needs `clientId`, `clientSecret`, `organizationId`, `tenantId`. API keys are separate and only needed for `authaz-management-api`
- Don't make up env var names. The JS SDK reads `AUTHAZ_CLIENT_ID`, `AUTHAZ_CLIENT_SECRET`, `AUTHAZ_ORGANIZATION_ID`, `AUTHAZ_TENANT_ID`. The .NET SDK reads its own config section

## References

- Recipes index: <https://authaz.io/docs/recipes/>
- Architecture: <https://authaz.io/docs/architecture>
- `references/glossary.md` — what organization / app / tenant / user / role mean
