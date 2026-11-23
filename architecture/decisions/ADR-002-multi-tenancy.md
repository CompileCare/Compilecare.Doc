# ADR-002: Tenant Isolation Strategy

- **Status:** Proposed
- **Date:** 2026-11-23

## Context
CompileCare serves multiple client organizations. Tenant isolation is a security boundary across APIs, data, caches, files, jobs, messages, logs, and analytics.

## Decision
- Tenant is the top-level customer isolation boundary.
- Tenant context is resolved from trusted authentication and server-side configuration.
- Every request and background operation must have an explicit tenant context unless it is a narrowly controlled platform operation.
- All data-access paths enforce tenant scope; cross-tenant access is denied by default.
- Shared-database tenant-scoped storage is a candidate for the first SaaS deployment; dedicated database/deployment options remain supported as a future topology.
- Tenant context must be included in cache keys, file/object paths, message metadata, audit records, and telemetry where safe.
- Automated tests must attempt cross-tenant reads, writes, updates, deletes, and indirect access.

## Open questions
Patient sharing across hospitals within one tenant; doctors/users belonging to multiple tenants; branch-level permissions; tenant provisioning and offboarding; database-per-tenant criteria; support access and audit; retention/export/deletion.
