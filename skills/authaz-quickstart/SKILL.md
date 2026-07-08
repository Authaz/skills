---
name: authaz-quickstart
description: Use when the user wants to add Authaz to their project but hasn't named a framework yet. Detects the framework and dispatches to the matching authaz-setup-* skill. Triggers on "add login", "add auth", "set up Authaz", "use authaz".
---

# Authaz quickstart — framework dispatch

Job: detect the right `authaz-setup-*` skill and hand off. Do **not** set anything up yourself.

No Authaz account yet? Hand off to `authaz-signup` first (Dashboard signup + collecting `clientId`/`clientSecret`/`tenantId`), then return here.

## Step 1 — Detect the framework

Check the project root in this order; stop at first match.

| Signal | Framework | Hand off to |
|---|---|---|
| `next.config.{js,ts,mjs}` or `"next"` in `package.json` deps | Next.js | `authaz-setup-nextjs` |
| `"hono"` in `package.json` deps | Hono | `authaz-setup-hono` |
| `vite.config.{js,ts}` + `"react"` in deps, no SSR framework | React SPA | `authaz-setup-react` |
| Any `*.csproj` with `Microsoft.NET.Sdk.Web` | ASP.NET Core | `authaz-setup-dotnet` |
| React + an SSR framework (Next/Remix/etc.) | The SSR framework wins | (e.g. Next.js) |

- Multiple matches (e.g. monorepo with Next.js *and* Hono) → ask which to set up first; don't do them all in one pass.
- No match → ask the user their stack. Supported: Next.js, Hono, React SPA, ASP.NET Core. Anything else → custom integration via `@authaz/sdk` (JS) or `Authaz.Sdk` (.NET) + standard OIDC middleware.

## Step 2 — Hand off

Invoke the chosen setup skill.

- SDK integration code doesn't branch on single- vs multi-tenant — every stack's setup passes the same optional `tenantId` value in config regardless.
- The app's tenancy type is a Dashboard-level choice made at application creation and cannot be changed later (see `authaz-signup` Step 6).
- The setup skill takes it from there.

## After setup — suggest next steps

| Need | Skill |
|---|---|
| Different login method (Google, magic link, …) | `authaz-add-provider` |
| More routes to gate | `authaz-protect-route` |
| Runtime permission checks | `authaz-permission-check` |
| Tenant-scoped queries / B2B SaaS | `authaz-multi-tenant` |
| OAuth round-trip failing | `authaz-troubleshoot-oauth` |
| Configure providers/branding/etc. from CLI | `authaz-cli` |

## Anti-patterns

- Don't pick a setup skill without confirming the framework — every SDK has different conventions.
- Don't ask the user for credentials before checking they have an Authaz account; hand off to `authaz-signup` instead.

## References

- `references/glossary.md` — what organization / app / tenant / user / role mean
- `references/endpoints.md` — hosted defaults and SDK sub-clients
