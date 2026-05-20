# Authaz glossary

The terms Authaz uses. If a developer asks "what's the difference between an organization and a tenant?", this is the answer.

## Organization

The top-level container in Authaz. **One per Authaz customer account.** Each organization has its own cryptographic isolation — keys, signing material, and encrypted data are partitioned per-org and never cross.

When integrating Authaz as an app developer, you usually have **one organization** (yours) and create **applications** under it. Customers of *your* product become **tenants** inside *your* application.

## Application

A logical product or service that authenticates users. Has a `client_id`, `client_secret`, allowed redirect URIs, configured providers, and (optionally) tenants.

You'll have one application per environment: `my-app-staging`, `my-app-production`. Don't reuse one application across environments — the JWKS, allowed callbacks, and provider configs are per-application.

Created via Dashboard or `POST /api/v1/applications`.

**Tenancy type**: `single_tenant` or `multi_tenant`. Set at creation, not changeable later.

## Tenant

A scope inside a multi-tenant application. **Your customer is a tenant.** Acme Corp = one tenant; Globex Corp = another tenant in the same application.

Two tenancy modes:

- **Shared pool** (default for `multi_tenant`): one user pool across all tenants. A single user can belong to multiple tenants with different roles in each. Best for B2B SaaS.
- **Isolated**: each tenant has its own user pool. A user in tenant A literally cannot exist in tenant B. Best when tenants have legal/compliance walls between them.

The token's `tenant_id` claim binds the session to one tenant. Switching tenants = new login.

## User

An end user of your application. Has an Authaz `sub` (user ID), one or more verified emails, optional MFA factors, and roles in any tenants they belong to.

In a single-tenant app, a user is just a user. In a multi-tenant app, the **same user** can be a member of many tenants — the membership is the (user, tenant) pair, with its own roles.

## Role

A named bundle of permissions. Defined per-application; can be scoped per-tenant in multi-tenant apps.

A user is **assigned** a role in a tenant: `(user, tenant, role)`. The role grants its permissions to the user *within that tenant only*.

## Permission

A single capability, formatted `resource:action`. Examples: `users:read`, `invoices:create`, `settings:update`. **The action comes after the resource.**

Permissions are checked via the authorization API — they're not stored in the JWT (roles are). The permission check resolves the user's roles in the relevant tenant and answers `allowed: true | false`.

## API key

An application-issued credential (`X-API-Key: sk_live_…`) used for backend-to-Authaz calls (Management API, permission checks). API keys have their own permission scope, separate from a user's roles.

API keys are not for end-user authentication — use the OAuth flow for that.

## M2M credentials (machine-to-machine)

OAuth 2 client credentials grant. A backend service authenticates as itself (not as a user) and gets an access token. Used for service-to-service calls into your application that's protected by Authaz.

## Authaz Sign-In

The hosted login UI at your identity domain (e.g., `auth.your-app.com` for a custom domain, or the default subdomain Authaz issued you). The only Authaz screen end users see. Handles the entire OAuth 2.1 + PKCE flow: provider selection, password / social / magic-link, MFA, tenant picker, account recovery.

You don't build this — you redirect to it.

## Management API vs auth flow

- **Auth flow** = `https://<your-identity-domain>/...` (e.g., `https://auth.your-app.com/...`) — what end users hit (browser redirects, OAuth, JWKS).
- **Management API** = `https://api.authaz.io/...` (hosted) — what your backend hits (create users, assign roles, check permissions).

Different hostnames, different auth (cookies vs API key), different audiences. Don't confuse them.

## Zeratul

The internal Zanzibar-style authorization engine that powers `/api/v1/authz/check`. As an integrator you don't talk to it directly; the Management API is the supported interface.

## JWT claims that matter

See `references/endpoints.md` for the full list. The two you'll actually use:

- `sub` — the user's Authaz ID. Use as your foreign key.
- `tenant_id` — present in multi-tenant only. Use to scope every query and every authorization check.
