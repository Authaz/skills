---
name: authaz-signup
description: Use when the user is brand-new to Authaz and needs to create their Authaz account before integrating. Walks the customer through Dashboard signup, the auto-provisioned organization, creating their first application via the CLI, collecting the three core credentials (and the organization ID for Management API use), and adding callback URLs. Hand off to authaz-quickstart afterward. Triggers on "I'm new to Authaz", "how do I sign up to Authaz", "create Authaz account", "first time setting up Authaz".
---

# Become a new Authaz customer

This skill is the **front of the funnel**: before the customer can integrate Authaz into their app, they need an Authaz account, an organization, and an application. Signing up and completing first login auto-provisions their **organization**, with them as **Owner** (`DashboardOnboardingService.OnboardIfNeededAsync`). They create their **first application** themselves, in Step 3, via the CLI — takes one command.

If the user already has an Authaz account and just needs to integrate, skip this skill and run `authaz-quickstart` directly.

## Step 1 — Sign up at the Dashboard

Send the user to:

<https://dashboard.authaz.io>

They sign up with email + password (and optionally a social provider if the Dashboard has those enabled). Use their work email — Authaz derives the **organization name** from the local part of the email (e.g., `john.doe@acme.com` → "John Doe", `team@acme.com` → "Team"). They can rename the org later.

> The Dashboard URL is `dashboard.authaz.io` for the hosted product. For self-hosted Authaz, point at the customer's dashboard host instead.

## Step 2 — Verify email

If the Dashboard's signup provider requires it (it does by default), they get a verification email. Click the link, then return to the Dashboard.

If the link expired or the email never arrived, they can request a resend from the sign-in page.

## Step 3 — Create the first application via the CLI

First sign-in auto-provisions the customer's **organization** — they land as **Owner**, org already selected. The application itself they create with one `apply` call. Drive this through `authaz` (see `authaz-cli` for the full reference) rather than the Dashboard:

```bash
dotnet tool install --global Authaz.Cli
authaz login                              # device-flow login against the account just created
authaz whoami                             # confirm identity + org
```

**If this identity belongs to more than one organization** (an existing customer with several orgs — not a brand-new signup, which only ever has the one auto-provisioned org), `authaz whoami`'s "Current organization" is just whatever org was last active on that profile. `authaz apply` creates the new application there **silently, with no prompt to confirm** — it's easy to create the app in the wrong org without noticing. Before applying:

```bash
authaz org list                # see every org this identity can act on
authaz org switch <org-id>     # pick the right one explicitly (omit <org-id> to pick interactively)
authaz whoami                  # re-confirm — "Current organization" should now match
```

If it's not obvious which org the customer wants, ask before running `authaz apply` — creating the application in the wrong org means redoing credential creation and Dashboard callback config too.

Write a minimal application YAML. Only the fields below are needed to get a working single-tenant app — see `authaz-cli`'s Application YAML reference for the full field list (password policy, session limits, invitations, etc.). **`enabled: true` under `settings` is required** — the application stays disabled without it, and it's the one field with no server-side default:

```yaml
# app.yaml
apiVersion: authaz/v1
kind: Application
metadata:
  name: my-app
spec:
  tenancy:
    type: single_tenant
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

```bash
authaz validate --file app.yaml
authaz apply --file app.yaml --yes       # no metadata.id -> creates new; prints the new application id
```

Capture the printed application id — every following command needs it as `--application-id`.

## Step 4 — Mint credentials via the CLI

Each of these is shown **once** — capture immediately:

```bash
authaz credential create --application-id <id> --name "<app-name>-credential" --quiet
# line 1 = clientId, line 2 = clientSecret

authaz apikey create --application-id <id> --name "<app-name>-mgmt-key" --quiet
# only if calling the Management API — prints the plain-text key
```

`tenantId` isn't in the exported YAML or on any CLI command yet — pull it from the Dashboard: **Applications → (app) → Tenants**, copy the default tenant's id. It's optional for single-tenant apps; omit the env var entirely rather than passing an empty string.

`organizationId` (Management API / org-switching only, not basic SDK setup): from the Dashboard URL bar after sign-in, or **Settings → Organization**.

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

The list must contain the **exact** URL the browser sends — trailing slash, port, scheme all count. Add staging and prod URLs ahead of time so deploys don't fail at the IdP.

## Step 6 — Decide single-tenant vs multi-tenant

Their first app was created as **single-tenant**. That's fine for consumer apps and internal tools.

If they're building **B2B SaaS** (each customer = a tenant inside their app), they want **multi-tenant**. Tenancy type is set at application creation and **cannot be changed later**. To go multi-tenant, apply a fresh YAML with no `metadata.id` (creates a new app) and `spec.tenancy.type: multi_tenant` / `mode: shared`:

```bash
authaz apply --file multi-tenant-app.yaml   # no metadata.id -> creates new, doesn't touch the single-tenant one
```

Tenancy is YAML-only, like every other feature-level config — set it via `authaz apply`, not an imperative command. See `authaz-multi-tenant`.

## Step 7 — Hand off

The customer now has everything to start integrating. Invoke `authaz-quickstart` next — it detects the framework and dispatches to the right setup skill.

```
authaz-quickstart
```

Or, if the framework is already known, jump straight to the matching skill:

- Next.js → `authaz-setup-nextjs`
- Hono → `authaz-setup-hono`
- React SPA → `authaz-setup-react`
- ASP.NET Core → `authaz-setup-dotnet`

## Anti-patterns

- **Don't have the customer sign up at `authaz.io`** (the marketing site) when they mean to integrate. The Dashboard is at `dashboard.authaz.io`.
- **Don't try to provision the organization via API before the customer has an account.** It's auto-created on first Dashboard sign-in. The application is created *after* that, by the customer, via `authaz apply` (Step 3) — not auto-created, and not something to provision before the account exists either.
- **Don't lose the `clientSecret`.** It's shown once. If lost, rotate it from the Dashboard — old secret stops working immediately, so coordinate the rotation with the integration timing.
- **Don't paste the system API key into the customer's app.** That key is for the Dashboard's internal use. Customers create their own Management API key in Step 4.
- **Don't pick single- vs multi-tenant without asking.** It's irreversible.
- **Don't assume the "current organization" from `authaz whoami` is the intended one when the identity has multiple orgs.** `authaz apply` targets whatever org is currently active with no confirmation prompt — run `authaz org list` / `authaz org switch` first.

## Source of truth

- Dashboard signup onboarding: `authaz/Authaz.Dashboard/Auth/DashboardOnboardingService.cs` — `OnboardIfNeededAsync`
- Org-name derivation: `DashboardOnboardingService.ExtractOrganizationName`
- Public docs: `authaz-documentation/getting-started.md`

## References

- `authaz-quickstart` — the next step after the customer has their credentials
- `authaz-cli` — change tenancy, set branding, add OAuth providers from the command line
- `references/glossary.md` — organization vs application vs tenant
