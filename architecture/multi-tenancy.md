# Multi-Tenancy and Organization Model

**Status:** The first tenant boundary is implemented for the facility API. The broader customer, organization and hospital-group model remains a product/architecture decision to validate with commercial and hospital stakeholders.

## Proposed hierarchy
Customer/Tenant → Hospital Group (optional) → Hospital → Branch/Campus → Department → Unit/Ward/Room/Bed.

Tenant is the data-isolation boundary. Access may be scoped further to tenant, hospital, branch, department and operational role.

## Implemented API tenant boundary
- Tenant-scoped facility and platform-context routes require authentication through OIDC-compatible JWT Bearer validation.
- The API validates the token and binds the external principal using the validated issuer and `sub`.
- `X-Tenant-ID` is a tenant selector only. The API requires exactly one valid GUID and verifies an active membership matching `(issuer, subject, tenantId)` in PostgreSQL before setting request tenant context.
- Facility authorization is derived from the membership role stored in HeapCare, not from tenant or permission claims supplied by the caller.
- Current facility roles: `HospitalAdmin`, `FacilityManager`, `FacilityViewer`.
- Tenant-aware query filters, keys, indexes and composite hierarchy relationships support persistence isolation.

The current API slice does not yet provide a tenant registry, tenant self-service onboarding, membership administration, or centralized role-management UI. Membership provisioning must be performed through a trusted operational process. These capabilities are required before normal customer onboarding can be considered complete.

## Mandatory controls
- Resolve tenant context from validated authentication and trusted membership data, not an arbitrary client identifier alone.
- Scope every data-access path by tenant.
- Enforce authorization centrally and test cross-tenant access.
- Tenant-aware unique constraints where identifiers are tenant-local.
- Tenant-aware cache keys, jobs, queues, file paths, exports and search indexes.
- Audit tenant context for sensitive and administrative actions.
- Prevent tenant scope changes through ordinary update requests.
- Automated cross-tenant read/write isolation tests across synchronous and asynchronous paths.
- Carefully scope support/admin access and audit privileged access.

## Data model considerations
Core records carry tenant scope or an unambiguous tenant-owned parent. Hospital/branch scope may be required. Decide whether patient records are shared across a tenant's hospitals. Define tenant activation/suspension, membership provisioning/revocation, and customer offboarding before production.

## Open decisions
Tenant definition/commercial contract; tenant registry and provisioning lifecycle; patient-sharing rules; shared DB/schema/dedicated DB options; data residency/backup/restore/retention/export/deletion; support access model.
