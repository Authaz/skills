# Authaz skills

Skills for AI coding agents (Claude Code, Copilot, Cursor, Cline, and others) that help you **add Authaz to a new project**. They wrap the canonical recipes at `https://authaz.io/docs/recipes` so your agent picks the right setup for your stack instead of guessing.

The bundle is scoped to onboarding: **setting up Authaz** in a TypeScript/Node or .NET project, and **identifying common issues** during the first integration. It is *not* for working on the Authaz codebase itself.

Supported stacks: Next.js, Hono, React SPA, ASP.NET Core. For anything else, agents can fall back to `@authaz/sdk` (JS) or raw OAuth/JWKS calls per the recipes.

## Install

Install the whole bundle:

```bash
npx skills add authaz/skills
```

Or install just the CLI skill:

```bash
npx skills add authaz/skills --skill authaz-cli
```

Skills install into your agent's local skills directory automatically. See [skills.sh](https://skills.sh) for the supported agents and CLI reference.

## What's in the bundle

Two skills, so your agent's skill list stays readable:

| Skill | Use when |
|---|---|
| `authaz-quickstart` | Anything about adding or using Authaz in an app — signup, framework setup, providers, route protection, permissions, tenants, Management API, OAuth debugging |
| `authaz-cli` | Configuring an Authaz application with the `authaz` CLI — login, OAuth/MFA/branding, declarative YAML apply |

`authaz-quickstart` is a router. The recipes live in `authaz-quickstart/references/` and the agent opens only the one the task needs:

| Reference | Covers |
|---|---|
| `signup.md` | Brand-new to Authaz: sign up, collect credentials |
| `setup-nextjs.md` | Next.js App Router |
| `setup-hono.md` | Hono backend |
| `setup-react.md` | React SPA |
| `setup-dotnet.md` | ASP.NET Core |
| `add-provider.md` | Password / Google / GitHub / magic link / passkey / SAML / M2M |
| `protect-route.md` | Gating a route or endpoint |
| `permission-check.md` | Checking a user's permission |
| `multi-tenant.md` | `tenant_id`, tenant-scoped queries |
| `management-api.md` | Users, roles, tenants, invitations from a backend |
| `troubleshoot-oauth.md` | redirect_uri, PKCE, JWKS, token errors |
| `glossary.md`, `endpoints.md`, `error-codes.md` | Shared lookup tables |

## How the skills are designed

Each skill is short on purpose. It says **when it applies**, **the steps**, **how to verify** the result, and **what not to do** — then points to the canonical recipe at [authaz.io/docs/recipes](https://authaz.io/docs/recipes) for the full code. That keeps the agent grounded on real Authaz APIs and avoids hallucinated endpoints or made-up SDK methods.

If a skill ever drifts from the docs, the docs win — file an issue.

## Versioning

The skills track the Authaz public API. Major API changes will bump the bundle version; minor SDK additions just update the relevant skills.

## License

See [LICENSE](./LICENSE).
