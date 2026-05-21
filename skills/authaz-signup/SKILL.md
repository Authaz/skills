---
name: authaz-signup
description: Use when the user is brand-new to Authaz and needs to create their Authaz account before integrating. Walks the customer through Dashboard signup, the auto-onboarded organization/application/tenant, collecting the four credentials, and adding callback URLs. Hand off to authaz-quickstart afterward. Triggers on "I'm new to Authaz", "how do I sign up to Authaz", "create Authaz account", "first time setting up Authaz".
---

# Become a new Authaz customer

This skill is the **front of the funnel**: before the customer can integrate Authaz into their app, they need an Authaz account, an organization, and an application. The good news is that the Dashboard creates the organization, first application, and first tenant **automatically on first login** (`DashboardOnboardingService.OnboardIfNeededAsync`).

If the user already has an Authaz account and just needs to integrate, skip this skill and run `authaz-quickstart` directly.

## Step 1 — Sign up at the Dashboard

Send the user to:

<https://dashboard.authaz.io>

They sign up with email + password (and optionally a social provider if the Dashboard has those enabled). Use their work email — Authaz derives the **organization name** from the local part of the email (e.g., `john.doe@acme.com` → "John Doe", `team@acme.com` → "Team"). They can rename the org later.

> The Dashboard URL is `dashboard.authaz.io` for the hosted product. For self-hosted Authaz, point at the customer's dashboard host instead.

## Step 2 — Verify email

If the Dashboard's signup provider requires it (it does by default), they get a verification email. Click the link, then return to the Dashboard.

If the link expired or the email never arrived, they can request a resend from the sign-in page.

## Step 3 — Auto-onboarding (happens on first sign-in)

On first sign-in, the Dashboard provisions everything the customer needs to start integrating:

1. **An organization** — your Authaz tenant. The customer is the **Owner**.
2. **A first application** — named `{OrgName} App`, single-tenant. This is what they'll integrate against.
3. **A default tenant** — bound to the organization (used as `tenantId` in SDK config).
4. **A system API key** — provisioned for Dashboard-internal use. The customer doesn't see it directly; they'll create their own Management API key in Step 5 if needed.
5. **Owner-role grants** — on the new org, the new app, and the new tenant.

The customer lands in the Dashboard already wired up. No "first-run wizard" — just navigate to the application they were given and pull the credentials.

## Step 4 — Take inventory in the Dashboard

Walk the customer through finding each value in the UI. You'll need all four:

| Value | Where to find it |
|---|---|
| `clientId` | Dashboard → Applications → (their app) → **Auth Flow Configuration** |
| `clientSecret` | Same page. Shown **once** at creation — if they didn't capture it, rotate from the same page |
| `organizationId` | Dashboard → top-level (the URL bar contains it after sign-in), or **Settings → Organization** |
| `tenantId` | Dashboard → Applications → (their app) → **Tenants**. The default tenant is preselected; copy its id |

If they plan to call the Management API (admin operations from their backend), they also need:

| Value | Where to find it |
|---|---|
| Management API key | Dashboard → **API Keys** → Create. Shown **once**. Scope to the permissions actually needed |

## Step 5 — Add callback URLs

In the Dashboard, **Applications → (app) → Auth Flow Configuration → Allowed callback URLs**, add the URL the browser will hit after sign-in:

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

If they're building **B2B SaaS** (each customer = a tenant inside their app), they want **multi-tenant**. Tenancy type is set at application creation and **cannot be changed later**. To go multi-tenant:

- Easiest path: create a second application via Dashboard → New application → choose **multi-tenant** → **shared pool** (default). Use this as their integration app and ignore the auto-created single-tenant one.
- Or use the CLI: `authaz tenancy set --type multi_tenant --mode shared` (only works *before* anyone has signed into the app — see `authaz-cli`).

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
- **Don't try to provision the organization or application via API before the customer has an account.** Both are auto-created on first Dashboard sign-in. The Management API is for *post-account* operations.
- **Don't lose the `clientSecret`.** It's shown once. If lost, rotate it from the Dashboard — old secret stops working immediately, so coordinate the rotation with the integration timing.
- **Don't paste the system API key into the customer's app.** That key is for the Dashboard's internal use. Customers create their own Management API key in Step 4.
- **Don't pick single- vs multi-tenant without asking.** It's irreversible.

## Source of truth

- Dashboard signup onboarding: `authaz/Authaz.Dashboard/Auth/DashboardOnboardingService.cs` — `OnboardIfNeededAsync`
- Org-name derivation: `DashboardOnboardingService.ExtractOrganizationName`
- Public docs: `authaz-documentation/getting-started.md`

## References

- `authaz-quickstart` — the next step after the customer has their credentials
- `authaz-cli` — change tenancy, set branding, add OAuth providers from the command line
- `references/glossary.md` — organization vs application vs tenant
