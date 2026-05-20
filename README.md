# Authaz skills

Skills for AI coding agents (Claude Code, Copilot, Cursor, Cline, and others) that integrate [Authaz](https://authaz.io) into your application. They wrap the canonical recipes at `https://authaz.io/docs/recipes` and give your agent the structure it needs to pick the right approach for your stack — instead of guessing.

These skills are for **app developers integrating Authaz**, not for working on the Authaz codebase itself.

## Install

Install the whole bundle:

```bash
npx skills add authaz/skills
```

Or pick a single skill:

```bash
npx skills add authaz/skills --skill authaz-setup-nextjs
```

Skills install into your agent's local skills directory automatically. See [skills.sh](https://skills.sh) for the supported agents and CLI reference.

## What's in the bundle

| Skill | Use when |
|---|---|
| `authaz-quickstart` | Starting a new Authaz integration; not sure which other skill to pick |
| `authaz-setup-nextjs` | Adding Authaz to a Next.js app (App Router) |
| `authaz-setup-hono` | Adding Authaz to a Hono backend |
| `authaz-setup-react` | Adding Authaz to a React SPA |
| `authaz-setup-dotnet` | Adding Authaz to an ASP.NET Core app |
| `authaz-add-provider` | Enabling password / Google / GitHub / magic link / M2M |
| `authaz-protect-route` | Gating a route or endpoint behind authentication |
| `authaz-permission-check` | Calling the authorization API to check a user's permission |
| `authaz-multi-tenant` | Reading `tenant_id` from the JWT, scoping queries by tenant |
| `authaz-management-api` | Using the SDK to manage users, roles, tenants, invitations |
| `authaz-cli-yaml` | Configuring Authaz declaratively with the `authaz` CLI |
| `authaz-troubleshoot-oauth` | Debugging redirect_uri, PKCE, JWKS, or token errors |

The `references/` directory at the repo root holds shared lookup tables (endpoints, error codes, glossary) the skills point your agent at. Install the whole bundle if you want the references available alongside individual skills.

## How the skills are designed

Each skill is short on purpose. It says **when it applies**, **the steps**, **how to verify** the result, and **what not to do** — then points to the canonical recipe at [authaz.io/docs/recipes](https://authaz.io/docs/recipes) for the full code. That keeps the agent grounded on real Authaz APIs and avoids hallucinated endpoints or made-up SDK methods.

If a skill ever drifts from the docs, the docs win — file an issue.

## Versioning

The skills track the Authaz public API. Major API changes will bump the bundle version; minor SDK additions just update the relevant skills.

## License

See [LICENSE](./LICENSE).
