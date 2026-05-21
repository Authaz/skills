# Proposal: cut cold-start onboarding from ~7 min to under 3 min

**Status:** draft
**Author:** Rodrigo (with Claude)
**Date:** 2026-05-21
**Goal:** new Authaz customer goes from "I want to add auth to my app" to "I'm signed in to my own app" in under 5 minutes, consistently — even on first contact with the product.

## Why we're not there today

The Authaz skill bundle is tuned for the AI agent's part of the flow. The agent can write all the integration code in seconds. The bottleneck is the **human-driven Dashboard navigation** between the agent's steps:

| Step | Time | Owner |
|---|---|---|
| Sign up at `dashboard.authaz.io` | 60-90s | Customer (human, email verification) |
| Find `clientId` in Dashboard → copy | ~20s | Customer (manual UI) |
| Find `clientSecret` → copy (shown once) | ~20s | Customer (manual UI) |
| Find `organizationId` → copy | ~20s | Customer (manual UI) |
| Find `tenantId` → copy | ~20s | Customer (manual UI) |
| Add `http://localhost:3000/auth/callback` to Allowed callback URLs | ~30s | Customer (manual UI) |
| `pnpm add @authaz/next @authaz/react` | ~30s | Agent + npm |
| Write the 5 source files | ~10s | Agent |
| Run `pnpm dev` + browser login | 60-90s | Customer (test) |
| **Total (cold start)** | **~5-7 min** | |

The agent collapses the writing-code step to essentially zero, but the customer still spends 2-3 minutes navigating and pasting. The skills can't remove that — it's outside their surface area.

## The three changes that hit the goal

### A. `authaz init <stack>` CLI command (high leverage)

Today the CLI can authenticate, apply YAML, and configure providers — but it can't **scaffold an integration**. Add an `init` subcommand that does the whole hand-off in one invocation:

```bash
$ authaz init
✓ Detected: Next.js 15 App Router (next.config.js + "next" in deps)
✓ Opening browser for device-code login...
  (user clicks "Allow" on Authaz)
✓ Authenticated as alice@acme.com — using app "Acme App" (default)
✓ Wrote .env.local
✓ Wrote src/app/api/auth/[...authaz]/route.ts
✓ Wrote src/app/auth/callback/page.tsx
✓ Updated src/app/layout.tsx (added AuthazProvider)
✓ Wrote src/app/dashboard/page.tsx
✓ Added http://localhost:3000/auth/callback to Allowed callback URLs
✓ Installed @authaz/next @authaz/react

Next: pnpm dev → open http://localhost:3000 → click Login.
```

**What it does:**

1. Detect the framework from `package.json` / `*.csproj` (same heuristic as `authaz-quickstart`).
2. Device-flow login (`authaz login --app-id` is already there — `init` runs it if no creds exist).
3. Fetch the four credentials via the Management API using the user's bearer token: `clientId` + `clientSecret` from the application's Auth Flow Configuration; `organizationId` + `tenantId` from the user's context.
4. Write the boilerplate files **inlined from the skill bundle** (`@authaz/skills` could be a NuGet/npm dep the CLI pulls, or templates embedded in the CLI binary).
5. Add the dev callback URL to the application via the same Management API.
6. Install the SDK packages (`pnpm add` / `npm install` / `dotnet add package` based on stack).

**Cost to build:** moderate. The CLI already does device flow + Management API calls + YAML mutation. `init` is template assembly + a `POST /applications/{id}/auth-flow` to append the callback URL + shelling out to the package manager. ~1-2 weeks of work; mostly boilerplate templates per stack.

**Supported stacks at v1:** Next.js, Hono, React SPA, ASP.NET Core (matching the skill bundle).

### B. "Copy `.env` for {stack}" Dashboard button (low effort, broad reach)

Some customers won't install a CLI. For them, the Dashboard should offer the same shortcut:

> **On the Application page → Auth Flow Configuration:**
>
> [Copy `.env` for ▾ Next.js / Hono / React / .NET]

Clicking copies a ready-to-paste snippet:

```
AUTHAZ_CLIENT_ID=app_01abc...
AUTHAZ_CLIENT_SECRET=cs_01...
AUTHAZ_ORGANIZATION_ID=org_01...
AUTHAZ_TENANT_ID=ten_01...
```

(For .NET, the snippet uses `appsettings.json` shape; for everything else, dotenv.)

**UX note:** the `clientSecret` is shown-once. The button can produce the snippet *only* until the customer leaves the page or rotates the secret — after that, it's masked like everywhere else. The "shown once" property is preserved.

**Cost to build:** small. It's a server-side endpoint that returns the snippet (with `clientSecret` populated only when the secret is still in the cached single-view window) + a button. ~2-3 days.

### C. Default callback URLs on new applications (one line, huge UX)

When the Dashboard auto-creates the first application during signup (`DashboardOnboardingService`), seed `Allowed callback URLs` with the common dev URLs:

```
http://localhost:3000/auth/callback        (Next.js / generic)
http://localhost:5173/auth/callback        (Vite + React)
https://localhost:5001/signin-oidc         (ASP.NET Core)
```

The customer can remove what they don't need. But for first-run, every common dev port works without a manual add. This removes ~30s of friction and a common cause of "it doesn't work" tickets (`invalid_request` from a missing callback).

**Cost to build:** tiny. Append three URLs to the auto-created application in `DashboardOnboardingService.OnboardIfNeededAsync` (step 2 / "Create the organization's application"). ~1 hour of work + a test.

## Combined impact

With A + B + C deployed, the cold-start path looks like:

| Step | Time |
|---|---|
| Sign up | 60-90s |
| Either: `authaz init` (terminal) **OR** click "Copy .env" + paste | 20-30s |
| Run dev server + browser login | 60-90s |
| **Total (cold start)** | **~2.5-3 min** |

Well under 5 minutes, consistently.

For warm-start (existing customer adding auth to a new app), `authaz init` alone takes them from "I have my project open" to "I'm signed in" in about **60 seconds**.

## Why all three, not just one

- **A (`authaz init`)** serves devs comfortable with a CLI — the same audience that uses `npx create-next-app`. It's the lowest-friction path *if they're willing to install one tool*.
- **B (Copy .env button)** serves devs who'd rather click than install — broader audience, lower commitment.
- **C (default callback URLs)** is a tiny no-cost change that removes the most common "I followed the docs and it doesn't work" failure. It pays off regardless of A or B.

Together they eliminate every manual nav-and-paste step in the cold-start path.

## Open questions

1. **Where do `authaz init`'s file templates live?** Two options:
   - **Embed in the CLI binary.** Pro: no extra fetch, works offline. Con: templates rot with SDK versions; CLI ships frequently.
   - **Fetch from `@authaz/skills` at runtime.** Pro: single source of truth with the skill bundle. Con: needs network; adds a dependency.

   Recommendation: embed for v1 (offline reliability matters more), revisit later.

2. **Should `authaz init` overwrite existing files?** Default: prompt per-file with a unified diff (Spectre.Console can do this). `--force` skips prompts. `--dry-run` shows what would change.

3. **What about TypeScript vs JavaScript?** All current SDK examples are TS. `authaz init` should detect TS via `tsconfig.json` presence; if missing in a Node project, default to TS (since the SDK types matter) but offer `--js` to write JS instead.

4. **Should the Dashboard's "Copy .env" know about the CLI?** Could include a "or run `authaz init`" hint underneath. Cross-promotion without forcing a tool choice.

5. **Self-hosted Authaz?** `authaz init` needs to honor the customer's `--api-url` / `AUTHAZ_API_URL`. The Dashboard "Copy .env" button can hard-code the customer's own host (it knows where it's running). Both should already work — verify.

## Out of scope for this proposal

- A `create-authaz-app` scaffolder that creates a *whole new project* from a template (à la `create-next-app`). That's a different surface and serves a different segment (people starting from scratch). Worth doing eventually, but not the highest-leverage move for the customers we have now.
- IDE plugins. Same reasoning — `authaz init` and the Dashboard button cover the agent-driven flow this proposal targets.

## Recommendation

Ship **C** this week (1 hour). Start **B** next week (2-3 days). Plan **A** as a 1-2 week project after that.

The skill bundle stays as-is once these land — the skills are correct and pinned to SDK versions. They become the **fallback** for stacks the CLI doesn't yet support, not the primary path.
