# CompileCare Delivery Roadmap

**Status:** Working product-first delivery plan, updated 2026-11-24. This is a milestone order, not a calendar promise. Exit criteria must pass before a phase is called complete.

CompileCare is a hospital and healthcare information/management platform. This roadmap contains only hospital-system capabilities. The [Product Blueprint v1](blueprint-v1.md) is the source for product scope and patient workflows.

## Phase 1 — Backend foundation

**Status: Completed and merged**

1. API foundation: ASP.NET Core/.NET 10 host, middleware, tenant-context plumbing, health checks, versioned API structure and initial endpoint foundation.
2. Shared observability: structured logging, tracing, metrics, correlation IDs and health instrumentation through CompileCare.Observability.

**Evidence:** CompileCare.Api and CompileCare.Observability repositories have merged implementations. The API currently targets .NET 10 and consumes the shared Observability packages.

## Phase 2 — Developer platform

**Status: Completed and merged — shared client, database operations and caching baseline**

CompileCare.ApiClient provides named/typed client registration, validated options, injectable access-token/tenant/correlation providers, outbound handlers, bounded safe-method resilience and RFC 9457 Problem Details parsing. The correlation header handler validates provider values and replaces untrusted/unsafe caller values.

CompileCare.Database establishes PostgreSQL/EF migration ownership, deployment/release, backup/restore and operations runbooks. CompileCare.Caching has a .NET 10 HybridCache package with optional Redis, validated configuration, explicit tenant/global cache-key namespaces and a tested sample (Caching PR #2; restore/build/test/pack passed).

**Follow-up release hardening:** publish/inspect packages and validate them from consuming applications; expand OpenAPI response annotations and contract validation as modules are introduced. Database backup automation and production recovery drills remain.

## Phase 3 — Database, identity and security

**Status: Foundation milestone completed — persistence, OIDC/tenant RBAC, request audit, tenant registry/lifecycle, trusted membership provisioning and organization scopes are implemented and merged. Production operational hardening remains.**

### Implemented foundation

1. PostgreSQL is the active provider in CompileCare.Api; EF Core 10 is the persistence and migration mechanism.
2. Tenant-aware facility persistence, query filters, scoped uniqueness, composite tenant foreign keys, restrictive deletes, audit timestamps and concurrency foundations are implemented.
3. The InitialFacilityHierarchy migration and model snapshot cover Hospital → Department → Ward → Room → Bed.
4. Integration fixtures apply versioned migrations to an isolated test database; CI checks model drift.
5. OIDC/JWT Bearer authentication validates signature, issuer, audience, lifetime, expiry and subject. The validated issuer and OIDC subject are used as the external identity key.
6. Tenant-scoped routes resolve an active membership matching the validated identity and tenant; a tenant header alone grants no access.
7. Facility permissions are derived from persisted membership roles; caller-supplied role/permission claims cannot elevate privileges.
8. Append-only tenant-scoped request audit events and PostgreSQL-backed tests are merged.
9. Global tenant registry and lifecycle are implemented. Tenants have unique normalized slugs and Provisioning/Active/Suspended/Closed states.
10. Trusted initial administrator provisioning and tenant/hospital/department membership scopes are implemented, including protections against self-modification and removing the last active tenant-wide HospitalAdmin (API PR #13).

### Remaining work — recommended order

1. Extend audit coverage for authentication/authorization outcomes, membership and role changes, sensitive record access, exports, privileged support and material clinical/business actions. Define controlled audit query/export, retention and operational access policies.
2. Extend tenant isolation and authorization tests to background jobs, messaging, caches, file/object storage, exports, search, support tools and privileged operations.
3. Rehearse migration install/upgrade and production migration deployment. Complete backup/restore automation and recovery drills; define signing-key rotation, secrets, encryption, retention/deletion, incident response and access reviews.
4. Complete threat modelling, suitable penetration testing and qualified privacy/legal review for applicable Indian requirements. Do not claim regulatory compliance from implementation or documentation alone.

**Phase 3 exit criteria:** reviewed migration install/upgrade paths; authenticated identity bound to active trusted tenant membership; organization-scope authorization at relevant tenant entry points; append-only audit coverage for approved sensitive/privileged actions; cross-tenant tests for synchronous and asynchronous paths; and reviewed production operational controls. Core security and membership capabilities are implemented; operational and cross-cutting validation remains before calling this phase production-ready.

## Phase 4 — Core healthcare domain

**Status: Partially implemented**

### Implemented

1. Hospital → Department → Ward → Room → Bed hierarchy with CRUD, bounded pagination, tenant-scoped relationships and persistence constraints.
2. Tenant-owned patient master and hospital registration, with hospital-scoped search, authorization and PostgreSQL-backed tests (API PR #14).
3. Hospital-configured MRN policy and concurrency-safe, idempotent MRN assignment/lookup (API PR #15). Patient GUID remains the internal identity; MRN is hospital-local.
4. Required patient registration and MRN workflows are documented in ADR-008/009 and the patient-registration workflow.

### Remaining delivery order

1. Complete the validated hospital group/branch onboarding model; tenant registry, lifecycle, trusted initial membership provisioning and organization scopes are implemented, while customer-specific onboarding and operational controls remain.
2. Staff/provider profiles, organizational assignments, schedules and scoped permissions.
3. **Patient identity — current increment.** Patient master, hospital registration, hospital-local MRN assignment and the initial advisory duplicate-candidate API are merged. The normalized multi-signal matching enhancement is being implemented in API PR #17 (see ADR-011). Next deliver richer demographic/contact inputs only after their data-protection model is approved, then separately authorized cross-hospital discovery, guardian relationships, emergency reconciliation, identifier lifecycle and governed patient association/merge workflows.
4. Patient appointments, walk-in, check-in and queue journey.
5. OPD encounter and clinical documentation foundation.
6. Minimal billing only if required by the validated pilot journey.
7. Pharmacy, diagnostics, inpatient, inventory and advanced revenue workflows in the approved roadmap sequence.

Each domain workflow requires hospital-user validation, permissions, audit, tenant scope, acceptance criteria, migration and automated tests.

## Phase 5 — User experience

**Status: Product discovery and API-contract preparation; implementation follows backend workflow stability**

Start with personas, workflow prototypes, design-system/accessibility requirements and versioned API contracts. Then deliver the responsive admin/reception/clinical experience incrementally as each backend workflow is specified. Avoid building screens against assumed patient, tenant or clinical contracts. Mobile remains deferred until validated use cases and identity/API contracts justify it.

## Release gates and working rules

- Product workflow and ownership precede module/repository implementation.
- Work in short-lived branches and focused PRs; do not commit directly to main.
- Review root causes and existing architecture before changing implementation.
- Every PR includes related docs, tests, security/privacy implications, migration/recovery notes and CI evidence.
- A phase is complete only when its exit criteria are met and reviewed.
- Keep the modular core simple; introduce separately deployed services only when documented needs justify them.
