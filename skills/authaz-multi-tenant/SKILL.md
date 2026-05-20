---
name: authaz-multi-tenant
description: Use when working with tenants — reading `tenant_id` from the JWT, scoping queries by tenant, choosing shared-pool vs isolated tenancy, or handling tenant switching. Triggers on "multi-tenant", "tenant_id claim", "scope by tenant", "B2B SaaS", "tenant isolation".
---

# Multi-tenant Authaz

In Authaz, **your customer is a tenant**, not a separate organization. Acme Corp = one tenant inside *your* application; Globex = another tenant in the same app. One application, many tenants, optionally one user across multiple tenants.

If the user is asking about creating a separate Authaz **organization** per customer, redirect them — that's not how Authaz multi-tenancy is designed. Organizations are for *you*, the integrator.

## Step 1 — Pick a tenancy mode

Set at application creation; **not changeable later**.

| Mode | Description | Best for |
|---|---|---|
| `single_tenant` | One user pool, no tenant concept | Consumer apps, internal tools |
| `multi_tenant` + `shared` (default for multi-tenant) | One user pool, users belong to N tenants with per-tenant roles | Most B2B SaaS |
| `multi_tenant` + `isolated` | Separate user pool per tenant — a user record literally cannot exist in two tenants | When tenants have hard legal/compliance walls |

Pick `shared` unless you have a concrete reason for `isolated`. Shared lets the same human switch between their work and personal accounts in your app.

## Step 2 — Provision a tenant

```http
POST https://api.authaz.io/api/v1/tenants
X-API-Key: sk_live_…

{ "name": "Acme Corp" }
```

The response includes `tenant_id`. Store it linked to your customer record.

Invite the customer's first user:

```http
POST https://api.authaz.io/api/v1/tenants/{tenantId}/invitations
X-API-Key: sk_live_…

{ "email": "founder@acme.com", "roles": ["admin"] }
```

The invitee gets an email; clicking the link drops them into your Authaz Sign-In already scoped to that tenant.

## Step 3 — Read `tenant_id` in your app

The tenant binding lives in the access token's `tenant_id` claim. Authaz Sign-In handles the picker UI — your app **never renders one**. The user picks a tenant during sign-in; the chosen tenant is in the issued token.

### Next.js / Hono / Node

```ts
import { decodeJwt } from "jose";
const claims = decodeJwt(accessToken) as { sub: string; tenant_id?: string; };
const tenantId = claims.tenant_id;
```

For production, also verify the signature against the JWKS at `${AUTHAZ_IDENTITY_DOMAIN}/universal/.well-known/jwks/${client_id}.json`. `decodeJwt` only parses; pair with `jwtVerify`.

### React (with `@authaz/react`)

```tsx
const { user } = useUser();
const tenantId = user?.tenantId;
```

### .NET

```csharp
var tenantId = User.FindFirst("tenant_id")?.Value;
```

`AddOpenIdConnect` already validates the signature against the JWKS; reading the claim is safe.

## Step 4 — Use `tenant_id` in your data layer

Two non-negotiable rules:

1. **Every query that reads tenant data filters on `tenant_id`** from the token. Not from user input.
2. **Every authorization check passes `tenant_id`** to `/api/v1/authz/check`. See `authaz-permission-check`.

```sql
-- good
SELECT * FROM invoices WHERE tenant_id = $1 AND ...;

-- bad: tenant from URL/body
SELECT * FROM invoices WHERE tenant_id = :req.body.tenantId;
```

If your DB supports row-level security (Postgres RLS), use it as defense-in-depth — set the tenant from a SET LOCAL or session var derived from the token, and have RLS reject queries that try to escape.

## Step 5 — Tenant switching

A user changing tenants = **new login**. The token is bound to one `tenant_id` for its lifetime. To switch:

1. Send the user to `/api/auth/login` (or `/signin-oidc` in .NET).
2. Authaz Sign-In presents the picker again (it remembers the previous selection but lets them change).
3. New token issued with the new `tenant_id`.

Don't try to swap tenants by editing your session store. The whole point of binding to the token is that it's tamper-proof.

## Step 6 — Verify tenant isolation

Manual test:

1. Create two tenants: Acme and Globex.
2. Invite the same user (`alice@example.com`) into both with different roles.
3. Sign in as Alice; pick Acme — token has `tenant_id: acme_id`. Confirm she sees only Acme data.
4. Sign out; sign in again; pick Globex. Token has `tenant_id: globex_id`. Confirm she sees only Globex data.
5. Try to fetch Globex data with the Acme token (e.g., paste the URL with the wrong tenant in a query string). Server should return 403, not 404 — log the attempt.

If step 5 leaks data, your tenant filtering is broken. Fix it before shipping anything else.

## Anti-patterns

- **Don't render your own tenant picker.** Authaz Sign-In does it; integrating yours fights the SDK.
- **Don't accept `tenant_id` from query strings, headers, or request bodies.** Token only.
- **Don't cache user → tenant in your session.** Cache `(user, tenant)` together — tenant is the primary key alongside user.
- **Don't use `isolated` mode unless you actually need it.** It triples the per-customer support cost (more user-record edge cases).
- **Don't rely on the `roles` claim for authoritative authorization.** Use `/api/v1/authz/check` — see `authaz-permission-check`.

## References

- Multi-tenancy: <https://authaz.io/docs/multi-tenancy/overview>
- Architecture (tenant binding & isolation): <https://authaz.io/docs/architecture>
- `authaz-permission-check`, `authaz-management-api`
- `references/glossary.md` — organization vs application vs tenant
