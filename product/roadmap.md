# CompileCare Delivery Roadmap

**Status:** Working product-first delivery plan, updated 2026-11-24. This is a milestone order, not a calendar promise. Exit criteria must pass before a phase is called complete.

CompileCare is a hospital and healthcare information/management platform. This roadmap contains only hospital-system capabilities. The [Product Blueprint v1](blueprint-v1.md) is the source for product scope and patient workflows.

## Phase 1 — Backend foundation

**Status: Completed and merged**

1. API foundation: ASP.NET Core/.NET 10 host, middleware, tenant-context plumbing, health checks, versioned API structure and initial endpoint foundation.
2. Shared observability: structured logging, tracing, metrics, correlation IDs and health instrumentation through CompileCare.Observability.

**Evidence:** CompileCare.Api and CompileCare.Observability repositories have merged implementations. The API currently targets .NET 10 and consumes the shared Observability packages.

## Phase 2 — Developer platform

**Status: Completed and merged — initial developer-platform baseline**

CompileCare.ApiClient provides named/typed client registration, validated options, injectable access-token/tenant/correlation providers, outbound handlers, bounded safe-method resilience and RFC 9457 Problem Details parsing. API conventions include versioned routes, bounded pagination, consistent errors and correlation IDs.

**Follow-up release hardening:** publish/inspect the 0.2.0 package and validate it from a consuming application; expand OpenAPI response annotations and contract validation as modules are introduced.

## Phase 3 — Database, identity and security

**Status: Foundation milestone completed — persistence, OIDC/tenant RBAC, request audit, tenant registry/lifecycle, trusted membership provisioning and organization scopes are implemented and merged. Production operational hardening remains.**

### Implemented foundation

1. PostgreSQL is the active provider in CompileCare.Api; EF Core 10 is the persistence and migration mechanism.
2. Tenant-aware facility persistence, query filters, scoped uniqueness, composite tenant foreign keys, restrictive deletes, audit timestamps and concurrency foundations are implemented.
3. The InitialFacilityHierarchy migration and model snapshot cover Hospital → Department → Ward → Room → Bed (API PR #9).
4. Integration fixtures apply versioned migrations to an isolated _test database; CI checks model drift.
5. OIDC/JWT Bearer authentication validates signature, issuer, audience, lifetime, expiry and subject. The validated issuer and OIDC sub are used as the external identity key.
6. Tenant-scoped facility routes require X-Tenant-ID as a selector and resolve an active membership matching the validated (issuer, subject, tenantId) tuple. The header alone grants no access.
7. Facility permissions are derived from the database membership role; caller-supplied role/permission claims cannot elevate privileges.
8. Current facility roles are HospitalAdmin, FacilityManager and FacilityViewer (API PR #10).
9. Append-only tenant-scoped request audit events, EF/PostgreSQL protections and PostgreSQL-backed tests are merged (API PR #11). The current implementation avoids patient payloads and sensitive request data in those events.
10. Global tenant registry and lifecycle are implemented and merged (API [PR #12](https://github.com/CompileCare/CompileCare.Api/pull/12)). Tenants have unique normalized slugs and Provisioning/Active/Suspended/Closed states; new tenants start in Provisioning, and only persisted platform operators can perform registry actions.
11. Platform-operator access is resolved from a persisted, active OIDC issuer/subject record; external role/permission claims do not grant platform access. Platform registry requests use the existing append-only audit middleware. The first platform operator is bootstrapped out-of-band by an authorised database administrator; no public operator self-enrolment endpoint exists.
12. The tenant-registry migration backfills existing tenant IDs from membership, facility and audit records before adding tenant foreign keys, preserving existing tenant allocations. CI passed the EF model-drift check, Release build, PostgreSQL integration tests and unit tests on the merged implementation.
13. Trusted initial administrator provisioning and tenant/hospital/department membership scopes are implemented, including scoped membership administration and protections against self-modification and removing the last active tenant-wide HospitalAdmin (API [PR #13](https://github.com/CompileCare/CompileCare.Api/pull/13)).

### Remaining work — recommended order

1. Validate product and architecture decisions for tenant/customer/organization and patient data boundaries against pilot-customer and hospital workflows.
2. Extend audit coverage for authentication/authorization outcomes, membership and role changes, sensitive record access, exports, privileged support and material clinical/business actions. Define controlled audit query/export, retention and operational access policies.
3. Extend tenant isolation and authorization tests to background jobs, messaging, caches, file/object storage, exports, search, support tools and privileged operations.
4. Rehearse migration install/upgrade and production migration deployment. Complete backup/restore automation and recovery drills; define signing-key rotation, secrets, encryption, retention/deletion, incident response and access reviews.
5. Complete threat modelling, suitable penetration testing and qualified privacy/legal review for applicable Indian requirements. Do not claim regulatory compliance from implementation or documentation alone.

**Phase 3 exit criteria:** reviewed migration install/upgrade paths; authenticated identity bound to active trusted tenant membership; organization-scope authorization at relevant tenant entry points; append-only audit coverage for the approved sensitive/privileged actions; cross-tenant tests for synchronous and asynchronous paths; and reviewed production operational controls. Core security and membership capabilities are implemented; operational and cross-cutting validation remains before calling this phase production-ready.

## Phase 4 — Core healthcare domain

**Status: Partially implemented**

### Implemented
1. Initial Hospital → Department → Ward → Room → Bed hierarchy.
2. CRUD endpoints and bounded pagination for the facility hierarchy.
3. Tenant-scoped uniqueness, relationships and persistence constraints.

### Remaining delivery order
1. Complete tenant and organization onboarding/provisioning, including trusted initial membership provisioning and the validated hospital group/branch model. Tenant registry and lifecycle are complete; group/hospital/branch onboarding remains.
2. Staff/provider profiles, organizational assignments, schedules and scoped permissions.
3. Patient registration, patient identifier/MRN policy, search and duplicate-review workflow.
4. Patient appointments, walk-in, check-in and queue journey.
5. OPD encounter and clinical documentation foundation.
6. Minimal billing only if required by the validated pilot journey.
7. Pharmacy, diagnostics, inpatient, inventory and advanced revenue workflows in the approved roadmap sequence.

Each domain workflow requires hospital-user validation, permissions, audit, tenant scope, acceptance criteria, migration and automated tests.

## Phase 5 — User experience

**Status: Product discovery and API-contract preparation; implementation follows blueprint validation**

Start with personas, workflow prototypes, design-system/accessibility requirements and versioned API contracts. Then deliver the responsive admin/reception/clinical experience incrementally as each backend workflow is specified. Avoid building screens against assumed patient, tenant or clinical contracts. Mobile remains deferred until validated use cases and identity/API contracts justify it.

## Release gates and working rules

- Product workflow and ownership precede module/repository implementation.
- Work in short-lived branches and focused PRs; do not commit directly to main.
- Review root causes and existing architecture before changing implementation.
- Every PR includes related docs, tests, security/privacy implications, migration/recovery notes and CI evidence.
- A phase is complete only when its exit criteria are met and reviewed.
- Keep the modular core simple; introduce separately deployed services only when documented needs justify them.
