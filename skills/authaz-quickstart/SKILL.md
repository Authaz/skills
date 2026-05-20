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

If nothing matches, ask the user what stack they're on. The supported stacks are Next.js, Hono, React SPA, and ASP.NET Core. For anything else, point them at the [recipes index](https://authaz.io/docs/recipes/) — they may need a custom integration using `@authaz/sdk` (JS) or raw OAuth/JWKS calls.

## Step 2 — Pick single- vs multi-tenant

Ask the user: "Is this a B2B SaaS where each customer is a tenant with their own users, or a single-tenant app (consumer, internal tool)?"

Don't infer from the codebase — many B2B apps look single-tenant until they're not, and getting this wrong forces a rebuild. Ask.

Cache the answer for the rest of the session: pass it to the chosen `authaz-setup-*` skill so it picks the right recipe.

## Step 3 — Confirm prerequisites

Before handing off, check that the user has:

1. **An Authaz account** — sign up at <https://authaz.io> if they don't have one yet (or confirm a self-hosted instance is reachable).
2. **An application** in the Authaz Dashboard for this project. Dashboard → Applications → New application. If they haven't created one yet, the setup skill walks through it — they only need the account ready.
3. **`client_id` + `client_secret`** — shown in the Dashboard under the application's *Auth Flow Configuration*. The secret is shown **once** at creation; if they lost it, rotate.
4. **Their identity domain** — also in the Dashboard, the URL where their hosted Sign-In lives (custom domain or the default subdomain Authaz issued). They'll use it as `AUTHAZ_IDENTITY_DOMAIN`.

If any are missing, point them at <https://authaz.io/docs/getting-started> first.

## Step 4 — Hand off

Invoke the chosen setup skill. Pass through the tenancy mode you established in step 2.

After setup is done, suggest the natural next steps:

- Need to enable a different login method? → `authaz-add-provider`
- Need to gate more routes? → `authaz-protect-route`
- Need to check permissions inside a route? → `authaz-permission-check`
- Multi-tenant with tenant-scoped data? → `authaz-multi-tenant`
- Errors during the OAuth round-trip? → `authaz-troubleshoot-oauth`

## Anti-patterns

- Don't set up auth without confirming the framework — every SDK has different conventions and the wrong one creates work to undo
- Don't skip the single- vs multi-tenant question — it determines the JWT shape, the recipe, and the data model
- Don't assume the user has API keys for the Management API; the OAuth flow only needs `client_id`/`client_secret`

## References

- Recipes index: <https://authaz.io/docs/recipes/>
- Architecture: <https://authaz.io/docs/architecture>
- `references/glossary.md` — what organization / app / tenant / user / role mean
