---
name: authaz-signup
description: Use when the user is brand-new to Authaz and needs to create their Authaz account before integrating. Walks the customer through Dashboard signup, the auto-provisioned organization, creating their first application via the CLI, collecting the three core credentials (and the organization ID for Management API use), and adding callback URLs. Hand off to authaz-quickstart afterward. Triggers on "I'm new to Authaz", "how do I sign up to Authaz", "create Authaz account", "first time setting up Authaz".
---

# Become a new Authaz customer

Front of the funnel: before integrating, the customer needs an Authaz account, an organization, and an application. Signup + first login auto-provisions their **organization** (customer becomes **Owner**, via `DashboardOnboardingService.OnboardIfNeededAsync`). They create their **first application** themselves in Step 3, via the CLI, in one command.

Already has an account and just needs to integrate? Skip this skill, run `authaz-quickstart` directly.

## Step 1 — Sign up at the Dashboard

- URL: <https://dashboard.authaz.io> (self-hosted: use the customer's dashboard host instead)
- Sign up with email + password (or social provider, if enabled).
- Use work email — org name is derived from the local part (`john.doe@acme.com` → "John Doe", `team@acme.com` → "Team"). Renameable later.

## Step 2 — Verify email

- Verification email required by default. Click link, return to Dashboard.
- Expired/missing link → resend from the sign-in page.

## Step 3 — Create the first application via the CLI

First sign-in auto-provisions the **organization** (customer = Owner, org already selected). Use `authaz` CLI (full reference: `authaz-cli`), not the Dashboard:

```bash
dotnet tool install --global Authaz.Cli
authaz login                              # device-flow login against the account just created
authaz whoami                             # confirm identity + org
```

**Confirmed in practice:**
- `authaz login` prints a device-authorization URL + code and blocks until approval. No specific browser required — any browser already signed in to the Dashboard account works, including one driven by automation (e.g. Playwright). Navigate to the URL, click **Approve**; CLI unblocks within seconds.
- `dotnet tool install` is idempotent — re-running reports "already installed" rather than erroring. Tool lands in `~/.dotnet/tools`, which may not be on `PATH` — add it for the session if `authaz` isn't found.
- The very first `authaz login` occasionally fails mid-flow with `Error: 'e' is an invalid start of a value` right after "Waiting for authorization..." — this is a transient 502 (identity backend returning HTML instead of JSON), not a CLI/credentials problem. Re-run `authaz login` for a fresh device code; second attempt succeeds.

**Multi-org identities** (existing customer with several orgs — not a fresh signup, which only ever has one auto-provisioned org): `authaz whoami`'s "Current organization" is just whatever was last active on that profile. `authaz apply` creates the app there **silently, no confirmation prompt** — easy to create it in the wrong org. Before applying:

```bash
authaz org list                # see every org this identity can act on
authaz org switch <org-id>     # pick the right one explicitly (omit <org-id> to pick interactively)
authaz whoami                  # re-confirm — "Current organization" should now match
```

If unclear which org, ask before `authaz apply` — wrong org means redoing credential creation and Dashboard callback config too.

**Before writing the YAML, ask whether the app is single-tenant or multi-tenant** — don't default silently. Tenancy is set at creation, irreversible (see Step 6). Consumer apps/internal tools → usually single-tenant; B2B SaaS with per-customer isolated data/users → multi-tenant.

Minimal single-tenant app YAML (full field list — password policy, session limits, invitations, etc. — in `authaz-cli`'s Application YAML reference). **`enabled: true` under `settings` is required** — app stays disabled without it; the one field with no server-side default:

```yaml
# app.yaml
apiVersion: authaz/v1
kind: Application
metadata:
  name: my-app
spec:
  tenancy:
    type: single_tenant       # or multi_tenant — whichever the customer confirmed above
  authentication:
    providers:
      emailPassword:
        enabled: true
    signup:
      enabled: true
    settings:
      redirectUris:
        - http://localhost:3000/auth/callback
      enabled: true
```

For `multi_tenant`, also set `mode: shared` under `spec.tenancy` — see `authaz-multi-tenant` for the full multi-tenant YAML shape and tenant scoping in app code.

```bash
authaz validate --file app.yaml
authaz apply --file app.yaml --yes       # no metadata.id -> creates new; prints the new application id
```

Capture the printed application id — every following command needs it as `--application-id`.

## Step 4 — Mint credentials via the CLI

Each shown **once** — capture immediately:

```bash
authaz credential create --application-id <id> --name "<app-name>-credential" --quiet
# line 1 = clientId, line 2 = clientSecret

authaz apikey create --application-id <id> --name "<app-name>-mgmt-key" --quiet
# only if calling the Management API — prints the plain-text key
```

| Credential | Source |
|---|---|
| `clientId` / `clientSecret` | `authaz credential create` output above |
| Management API key | `authaz apikey create` output above |
| `tenantId` (optional, single-tenant) | Not in exported YAML or any CLI command — Dashboard: **Applications → (app) → Tenants**, copy default tenant id. Omit env var entirely rather than passing empty string. |
| `organizationId` (Management API / org-switching only) | Dashboard URL bar after sign-in, or **Settings → Organization** |

**Confirmed in practice:** a brand-new single-tenant app's Tenants page can show "No tenants found" — no default tenant auto-provisioned in that case. Not an error; just skip the `tenantId` env var for SDK setup.

## Step 5 — Add more callback URLs (staging, prod)

Step 3's `app.yaml` already registered the dev callback URL. Add each new environment's URL to the same `spec.settings.redirectUris` list, then re-apply:

```bash
authaz validate --file app.yaml
authaz apply --file app.yaml --application-id <id>
```

| Stack | Add these |
|---|---|
| Next.js (dev) | `http://localhost:3000/auth/callback` |
| Hono alone (dev) | `http://localhost:3000/auth/callback` |
| Vite + React (dev) | `http://localhost:5173/auth/callback` |
| ASP.NET Core (dev, OIDC middleware) | `https://localhost:5001/signin-oidc` |
| Any stack (prod) | The production URL — exact match, including scheme and port |

URL must be **exact** — trailing slash, port, scheme all count. Add staging/prod URLs ahead of time so deploys don't fail at the IdP.

## Step 6 — Single-tenant vs multi-tenant, if it changes later

Tenancy type is set at creation and **cannot be changed later**. If the customer picked single-tenant in Step 3 but later turns out to need **B2B SaaS** (each customer = a tenant inside their app), they need **multi-tenant** in a *new* application — not a migration. Apply a fresh YAML with no `metadata.id` and `spec.tenancy.type: multi_tenant` / `mode: shared`:

```bash
authaz apply --file multi-tenant-app.yaml   # no metadata.id -> creates new, doesn't touch the single-tenant one
```

Tenancy is YAML-only, like every feature-level config — set via `authaz apply`, not an imperative command. See `authaz-multi-tenant`.

## Step 7 — Hand off

Customer now has everything to start integrating. Invoke `authaz-quickstart` next — it detects the framework and dispatches to the right setup skill.

```
authaz-quickstart
```

Or, if framework already known, jump straight to the matching skill:

| Framework | Skill |
|---|---|
| Next.js | `authaz-setup-nextjs` |
| Hono | `authaz-setup-hono` |
| React SPA | `authaz-setup-react` |
| ASP.NET Core | `authaz-setup-dotnet` |

## Anti-patterns

- **Don't sign up at `authaz.io`** (marketing site) when they mean to integrate — Dashboard is at `dashboard.authaz.io`.
- **Don't provision the organization via API before the account exists.** It's auto-created on first Dashboard sign-in. The application is created *after*, by the customer, via `authaz apply` (Step 3) — not auto-created, not pre-provisionable.
- **Don't lose the `clientSecret`.** Shown once. If lost, rotate from Dashboard — old secret stops working immediately, so coordinate rotation with integration timing.
- **Don't paste the system API key into the customer's app.** That key is for the Dashboard's internal use. Customers create their own Management API key in Step 4.
- **Don't pick single- vs multi-tenant without asking.** Irreversible.
- **Don't assume `authaz whoami`'s "current organization" is correct when the identity has multiple orgs.** `authaz apply` targets whatever org is active, no confirmation prompt — run `authaz org list` / `authaz org switch` first.

## Source of truth

- Dashboard signup onboarding: `authaz/Authaz.Dashboard/Auth/DashboardOnboardingService.cs` — `OnboardIfNeededAsync`
- Org-name derivation: `DashboardOnboardingService.ExtractOrganizationName`
- Public docs: `authaz-documentation/getting-started.md`

## References

- `authaz-quickstart` — the next step after the customer has their credentials
- `authaz-cli` — change tenancy, set branding, add OAuth providers from the command line
- `references/glossary.md` — organization vs application vs tenant
