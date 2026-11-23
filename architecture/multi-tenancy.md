# Multi-Tenancy and Organization Model

**Status:** Proposed; validate with product/commercial stakeholders.

## Proposed hierarchy
Customer/Tenant → Hospital Group (optional) → Hospital → Branch/Campus → Department → Unit/Ward/Room/Bed.

Tenant is the isolation boundary. Access may be scoped to tenant, hospital, branch, department, and operational role.

## Initial strategy
Begin tenant-aware. Shared database with tenant-scoped rows is a candidate for initial SaaS; not finalized. Dedicated database/deployment options may be considered for customers requiring stronger isolation or operational separation.

## Mandatory controls
- Resolve tenant context from trusted authentication/host configuration, not an arbitrary client identifier alone.
- Scope every data-access path by tenant.
- Enforce authorization centrally and test cross-tenant access.
- Tenant-aware unique constraints where identifiers are tenant-local.
- Tenant-aware cache keys, jobs, queues, file paths, and search indexes.
- Audit tenant context for sensitive and administrative actions.
- Prevent tenant scope changes through ordinary update requests.
- Automated cross-tenant read/write isolation tests.
- Carefully scope support/admin access and audit privileged access.

## Data model considerations
Core records carry tenant scope or an unambiguous tenant-owned parent. Hospital/branch scope may be required. Decide whether patient records are shared across a tenant's hospitals.

## Open decisions
Tenant definition/commercial contract; patient-sharing rules; shared DB/schema/dedicated DB options; provisioning/offboarding; data residency/backup/restore/retention/export/deletion; support access model.
