---
name: authaz-quickstart
description: Use when the user wants to add Authaz to their project but hasn't named a framework yet. Detects the framework and dispatches to the matching authaz-setup-* skill. Triggers on "add login", "add auth", "set up Authaz", "use authaz".
---

# Authaz quickstart — framework dispatch

Your only job here is to figure out which `authaz-setup-*` skill applies and hand off. You are **not** setting anything up yourself.

If the user doesn't have an Authaz account yet, hand off to `authaz-signup` first — it covers signing up at the Dashboard and collecting the four credentials (`clientId`, `clientSecret`, `organizationId`, `tenantId`). Then come back here.

## Step 1 — Detect the framework

Look at the project root in this order. Stop at the first match.

| Signal | Framework | Hand off to |
|---|---|---|
| `next.config.{js,ts,mjs}` or `"next"` in `package.json` deps | Next.js | `authaz-setup-nextjs` |
| `"hono"` in `package.json` deps | Hono | `authaz-setup-hono` |
| `vite.config.{js,ts}` + `"react"` in deps, no SSR framework | React SPA | `authaz-setup-react` |
| Any `*.csproj` with `Microsoft.NET.Sdk.Web` | ASP.NET Core | `authaz-setup-dotnet` |
| React + an SSR framework (Next/Remix/etc.) | The SSR framework wins | (e.g. Next.js) |

If multiple match (e.g. a monorepo with Next.js *and* Hono), ask which one to set up first — don't try to do them all in one pass.

If nothing matches, ask the user what stack they're on. The supported stacks are Next.js, Hono, React SPA, and ASP.NET Core. For anything else, they need a custom integration using `@authaz/sdk` (JS) or `Authaz.Sdk` (.NET) + standard OIDC middleware.

## Step 2 — Hand off

Invoke the chosen setup skill. There are no separate code paths for single- vs multi-tenant — that's purely the `tenantId` value the customer passes. The setup skill takes it from there.

## After setup

Suggest the natural next steps:

- Different login method (Google, magic link, …)? → `authaz-add-provider`
- More routes to gate? → `authaz-protect-route`
- Runtime permission checks? → `authaz-permission-check`
- Tenant-scoped queries / B2B SaaS? → `authaz-multi-tenant`
- OAuth round-trip is failing? → `authaz-troubleshoot-oauth`
- Configure providers/branding/etc. from the CLI? → `authaz-cli`

## Anti-patterns

- Don't pick a setup skill without confirming the framework — every SDK has different conventions.
- Don't ask the user for credentials before checking they have an Authaz account; hand off to `authaz-signup` instead.

## References

- `references/glossary.md` — what organization / app / tenant / user / role mean
- `references/endpoints.md` — hosted defaults and SDK sub-clients
