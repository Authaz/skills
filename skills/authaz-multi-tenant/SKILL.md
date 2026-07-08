---
name: authaz-multi-tenant
description: Use when working with tenants — reading `tenant_id` from the JWT, scoping queries by tenant, choosing shared-pool vs isolated tenancy, or handling tenant switching. Triggers on "multi-tenant", "tenant_id claim", "scope by tenant", "B2B SaaS", "tenant isolation".
---

# Multi-tenant Authaz

In Authaz, **your customer is a tenant**, not a separate organization. Acme Corp = one tenant inside *your* app; Globex = another tenant in the same app. One application, many tenants, optionally one user across multiple tenants.

If the user wants a separate Authaz **organization** per customer, redirect them — organizations are for *you*, the integrator; tenants are for your customers.

## Step 1 — Pick a tenancy mode

Set at application creation in the Dashboard.

| Mode | Description | Best for |
|---|---|---|
| single-tenant | One user pool, no tenant concept | Consumer apps, internal tools |
| multi-tenant, shared pool (default) | One user pool, users belong to N tenants with per-tenant roles | Most B2B SaaS |
| multi-tenant, isolated | Separate user pool per tenant — a user record cannot exist in two tenants | Hard legal/compliance walls between tenants |

Pick *shared* unless you have a concrete reason for *isolated*. Shared lets the same human switch between work and personal accounts in your app.

## Step 2 — Provision a tenant

Via the Dashboard, or the SDK:

### JS
```ts
const result = await authaz.tenants.create({ name: "Acme Corp" });
if (!result.ok) throw result.error; // or use isOk(result)
const tenantId = result.data.id;
```

### .NET
```csharp
var result = await authaz.Tenants.CreateAsync(new CreateTenantRequest("Acme Corp"));
var tenantId = result.Value?.Id;
```

Store the returned `tenantId` linked to your customer record.

## Step 3 — How `tenantId` flows through Authaz

Non-obvious — read carefully. Three distinct points, same value:

1. **SDK init**: the JS SDK config takes a `tenantId` — scopes the **OAuth flow**. When the browser hits `/api/auth/login`, the authorize URL embeds this id; Authaz signs the user into that tenant.
2. **Issued JWT**: the access token carries a `tenant_id` claim — the tenant the *session* is bound to.
3. **Authorization-check time** (SDK's `authz.check`): pass `tenantId` to scope the check to that tenant's roles/policies.

Don't confuse these.

**Single-tenant apps**: use the default tenant Authaz created for your org; `tenantId` is constant.

**Multi-tenant B2B SaaS apps**: `tenantId` varies per-customer. Common patterns:
- **Subdomain routing**: `acme.your-app.com` / `globex.your-app.com` each look up the right `tenantId` at request time and start login with it.
- **Pre-selection in your UI**: customer enters workspace name on sign-in page; you look up the tenant id and redirect to SDK-driven login configured for that tenant.

Hosted Authaz Sign-In does NOT render a tenant picker for end users by default — your app picks the tenant before initiating the flow.

## Step 4 — Read `tenant_id` from the access token in your backend

### Next.js / Hono / Node (`jose` for decoding)
```ts
import { decodeJwt } from "jose";
const claims = decodeJwt(accessToken) as { sub: string; tenant_id?: string };
const tenantId = claims.tenant_id;
```
For production, verify the signature against JWKS at `${AUTHAZ_IDENTITY_DOMAIN}/.well-known/jwks.json` (defaults to `https://auth.authaz.io/.well-known/jwks.json`) — `decodeJwt` only parses; pair with `jwtVerify`.

If running an `@authaz/next` or `@authaz/hono` handler, `/api/auth/me` returns the parsed user — use that instead of decoding manually.

### React (`@authaz/react`)
```tsx
import { useAuthaz } from "@authaz/react";

const { user } = useAuthaz();
const tenantId = user?.tenantId; // session tenant claim, exposed directly on AuthazUser
```

### .NET
```csharp
var tenantId = User.FindFirst("tenant_id")?.Value;
```
`AddOpenIdConnect` validates the signature against JWKS automatically; reading the claim is safe.

## Step 5 — Use `tenantId` in your data layer

Non-negotiable:
1. **Every query reading tenant data filters on `tenantId`** from the token — not from user input.
2. **Every authorization check passes `tenantId`** to `authz.check`. See `authaz-permission-check`.

```sql
-- good
SELECT * FROM invoices WHERE tenant_id = $1 AND ...;

-- bad: tenant from URL/body
SELECT * FROM invoices WHERE tenant_id = :req.body.tenantId;
```

If your DB supports row-level security (Postgres RLS), use it as defense-in-depth — set the tenant via `SET LOCAL` derived from the token claim.

## Step 6 — Tenant switching

Changing tenants = **new login** with a different SDK `tenantId`. The token is bound to one tenant for its lifetime — no client-side "switch tenant" minting. To switch:
1. Re-initialize the SDK (or its OAuth config) with the new `tenantId`.
2. Send the user to `/api/auth/login` again — re-prompt depends on session state.
3. The new token's `tenant_id` claim reflects the new tenant.

Only swap by re-initiating login with the new `tenantId` — the binding is tamper-proof by design, so editing a client-side session store can't substitute for it.

## Step 7 — Verify tenant isolation

Manual test:
1. Create two tenants: Acme and Globex.
2. Invite the same user (`alice@example.com`) into both with different roles.
3. Sign in as Alice scoped to Acme — token has `tenant_id: acme_id`. Confirm she sees only Acme data.
4. Sign out; sign in scoped to Globex — token has `tenant_id: globex_id`. Confirm she sees only Globex data.
5. Try fetching Globex data with the Acme token (paste the URL with the wrong tenant in a query string). Server should return 403, not 404 — log the attempt.

If step 5 leaks data, your tenant filtering is broken — fix before shipping anything else.

## Anti-patterns

- **Don't accept `tenant_id` from query strings, headers, or request bodies.** Token only.
- **Don't render your own tenant picker on the Authaz Sign-In page.** Pick the tenant in your app before initiating login.
- **Don't cache user → tenant as a single field.** Cache `(user, tenant)` together — tenant is a primary key alongside user.
- **Don't use isolated mode unless you actually need it.** Triples per-customer support cost (more user-record edge cases).
- **Don't rely on the `roles` claim for authoritative authorization.** Use SDK `authz.check` — see `authaz-permission-check`.

## Source of truth

- JS SDK tenant config: `authaz-sdk-js/packages/core/src/client.ts` (`tenantId` field in `AuthazConfig`)
- JS tenants API: `authaz-sdk-js/packages/core/src/management/tenants.ts`
- .NET tenants API: `authaz-sdk-dotnet/Authaz.Sdk/src/Resources/Tenants/`

## References

- `authaz-permission-check`, `authaz-management-api`
