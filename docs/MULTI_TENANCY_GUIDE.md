# Multi-Tenancy Architecture Guide

The RWA Marketplace supports multi-tenancy, allowing a single application instance to serve multiple isolated customers (tenants).

## Data Isolation (Row-Level Security)
We utilize **PostgreSQL Row-Level Security (RLS)** to guarantee strict data segregation.
- The `tenantContextMiddleware` intercepts every incoming request.
- It extracts the `tenant_id` from the JWT token or the `X-Tenant-ID` HTTP header.
- A dedicated Postgres transaction is spawned using `SET LOCAL app.current_tenant_id = '...'`.
- Postgres RLS policies automatically filter all queries (`SELECT`, `UPDATE`, `DELETE`) ensuring Tenant A can never read Tenant B's data.

## Database-per-Tenant vs. Pooled (RLS)
This architecture implements a **Pooled Strategy with RLS**. All tenants share the same physical tables, but are logically isolated by the database engine. This provides the best balance of operational simplicity and high security. If a customer requires extreme isolation (Database-per-Tenant), the application is fully compatible by simply deploying a separate database connection string per customer environment.

## Tenant Onboarding & Management
The `TenantService` handles automated onboarding, config management, and analytics.
- **Performance Isolation:** Tenants on the `standard` plan are limited to 100 API requests per minute. `enterprise` tenants receive higher quotas.
- **Tenant Analytics:** Aggregated usage reports are generated securely using the tenant's execution context.

### Quotas & Custom Enterprise Deals (Issue #900)
Default quotas are defined in `TenantService.PLAN_QUOTAS` (`standard`: 100 req/min, `enterprise`: 1000 req/min). For negotiated B2B deals a **custom per-tenant quota** can be provisioned at onboarding and adjusted later — no direct database edits are ever required.

**At onboarding** — pass `max_api_requests_per_min` to override the plan default:

```js
await tenantService.onboardTenant({
  name: 'Acme Corp',
  billing_plan: 'enterprise',
  max_api_requests_per_min: 5000, // negotiated custom quota (overrides the 1000 default)
});
```

**Post-onboarding adjustment process** (for support/sales teams):
1. The deal owner confirms the new quota and the reason (e.g. "enterprise uplift, ticket SUP-123").
2. An admin calls the admin endpoint with the admin API key (`x-api-key` header):

   ```http
   PUT /api/admin/tenants/{tenantId}/quota
   x-api-key: <ADMIN_API_KEY>
   Content-Type: application/json

   { "maxApiRequestsPerMin": 5000, "reason": "Enterprise uplift, ticket SUP-123" }
   ```

   or calls `tenantService.updateTenantQuota(tenantId, { maxApiRequestsPerMin }, { updatedBy, reason })` directly.
3. The change is applied immediately and audit-logged with the previous quota, new quota, actor, and reason.
4. Quotas must be positive integers; invalid values are rejected with a `400` before touching the database. Unknown tenants return `404`.

The full admin API surface also includes `POST /api/admin/tenants` (onboard, accepts the same custom quota field) and `GET /api/admin/tenants/{tenantId}` (inspect current quota). See `backend/__tests__/tenantQuota.test.js` for executable examples.

## Usage
When making API requests on behalf of a tenant, ensure the authentication token contains the `tenantId` claim, or pass the header:
```http
X-Tenant-ID: tnt_1234567890
```
