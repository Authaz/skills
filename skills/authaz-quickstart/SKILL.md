---
name: authaz-quickstart
description: Use for anything involving adding or using Authaz in an application — signing up for an Authaz account, adding login to Next.js / Hono / React / ASP.NET Core, enabling a provider (password, Google, GitHub, magic link, passkey, SAML, M2M), protecting a route, checking a permission, working with tenants, calling the Management API, or debugging an OAuth failure. Routes to the right recipe in references/. Triggers on "add login", "add auth", "set up Authaz", "use authaz", "sign up to Authaz", "protect this route", "check permission", "tenant_id", "management API", "redirect_uri mismatch", "invalid_grant", "authaz callback error".
---

# Authaz — entry point

The one skill for integrating Authaz. Every recipe lives in `references/`; open the file the task needs and follow it. Read only that one — these are full recipes, not summaries.

> **Scope**: informational. Commands that mutate the customer's Authaz tenant — Dashboard actions, `POST/PATCH /api/v1/...`, `authaz apply` — get surfaced for the developer to run, not executed for them. Editing the developer's own source (SDK wiring, middleware, handlers, env placeholders) is in scope.

## Route the request

| The user wants to… | Read |
|---|---|
| Create an Authaz account, get `clientId` / `clientSecret` / `tenantId` | `references/signup.md` |
| Add login, framework not named yet | Detect it (below), then the matching setup file |
| Add Authaz to Next.js (App Router) | `references/setup-nextjs.md` |
| Add Authaz to a Hono backend | `references/setup-hono.md` |
| Add Authaz to a React SPA | `references/setup-react.md` |
| Add Authaz to ASP.NET Core | `references/setup-dotnet.md` |
| Enable a provider — password, Google, GitHub, magic link, passkey, SAML, M2M | `references/add-provider.md` |
| Gate a route or endpoint | `references/protect-route.md` |
| Check whether a user is allowed to do something | `references/permission-check.md` |
| Read `tenant_id`, scope queries by tenant, B2B SaaS | `references/multi-tenant.md` |
| Manage users / roles / tenants / invitations from a backend | `references/management-api.md` |
| Debug redirect_uri, PKCE, JWKS, or token errors | `references/troubleshoot-oauth.md` |
| Configure an application from the command line | the separate `authaz-cli` skill |

No Authaz account yet? Start at `references/signup.md` whatever they asked for — every other recipe needs the credentials it produces.

## Detecting the framework

Check the project root in this order; stop at first match.

| Signal | Framework | Read |
|---|---|---|
| `next.config.{js,ts,mjs}` or `"next"` in `package.json` deps | Next.js | `references/setup-nextjs.md` |
| `"hono"` in `package.json` deps | Hono | `references/setup-hono.md` |
| `vite.config.{js,ts}` + `"react"` in deps, no SSR framework | React SPA | `references/setup-react.md` |
| Any `*.csproj` with `Microsoft.NET.Sdk.Web` | ASP.NET Core | `references/setup-dotnet.md` |
| React + an SSR framework (Next/Remix/etc.) | The SSR framework wins | (e.g. Next.js) |

- Multiple matches (e.g. monorepo with Next.js *and* Hono) → ask which to set up first; don't do them all in one pass.
- No match → ask the user their stack. Supported: Next.js, Hono, React SPA, ASP.NET Core. Anything else → custom integration via `@authaz/sdk` (JS) or `Authaz.Sdk` (.NET) + standard OIDC middleware.
- SDK integration code doesn't branch on single- vs multi-tenant — every stack passes the same optional `tenantId` in config.
- The app's tenancy type is a Dashboard-level choice made at application creation and cannot be changed later (see `references/signup.md` Step 6).

## After setup — suggest next steps

| Need | Read |
|---|---|
| Different login method (Google, magic link, …) | `references/add-provider.md` |
| More routes to gate | `references/protect-route.md` |
| Runtime permission checks | `references/permission-check.md` |
| Tenant-scoped queries / B2B SaaS | `references/multi-tenant.md` |
| OAuth round-trip failing | `references/troubleshoot-oauth.md` |
| Configure providers/branding/etc. from CLI | the `authaz-cli` skill |

## Anti-patterns

- Don't write setup code from memory — open the reference file first. Every SDK has different conventions and the recipes are the grounded source.
- Don't pick a setup file without confirming the framework.
- Don't ask for credentials before checking the user has an Authaz account; `references/signup.md` covers that.
- Don't read every reference up front. Read the one the task needs.

## Shared lookups

- `references/glossary.md` — what organization / app / tenant / user / role mean
- `references/endpoints.md` — hosted defaults, endpoint URLs, SDK sub-clients
- `references/error-codes.md` — error shapes and retry guidance
