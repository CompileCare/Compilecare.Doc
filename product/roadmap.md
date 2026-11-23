# CompileCare Delivery Roadmap

**Status:** Working delivery plan aligned with the agreed engineering sequence on 2026-11-23. This is a milestone order, not a calendar promise. Exit criteria must pass before a phase is called complete.

CompileCare is a hospital and healthcare information/management platform. This roadmap contains only hospital-system capabilities.

## Phase 1 — Backend foundation

**Status: Completed and merged**

1. API foundation: ASP.NET Core/.NET 10 host, middleware, tenant-context plumbing, health checks, versioned API structure, and initial endpoint foundation.
2. Shared observability: structured logging, tracing, metrics, correlation IDs, and health instrumentation through CompileCare.Observability.

**Evidence:** CompileCare.Api and CompileCare.Observability repositories have merged implementations. The API currently targets .NET 10; use the repository's actual target rather than older .NET 9 references in earlier planning notes.

**Exit:** API foundation builds and its baseline CI checks pass; observability is consumable as a shared package. Completed.

## Phase 2 — Developer platform

**Status: In progress — next active phase**

### 2.1 CompileCare.ApiClient
The initial CompileCare.ApiClient package is merged. It already provides named HttpClient registration, validated base-address/timeout options, injectable access-token, tenant and correlation providers, and outbound header handlers.

Remaining work:
1. Add strongly typed client registration for generated NSwag clients and hand-written clients.
2. Add standard, bounded HTTP resilience with explicit timeouts, rate limiting and circuit breaking. Retries must be disabled for unsafe/non-idempotent methods by default; only safe methods may be retried unless an operation explicitly opts into an idempotency contract.
3. Add a stable client-side error contract for RFC 9457 Problem Details, preserving HTTP status, problem fields, validation errors where present, and correlation ID without logging tokens or patient data.
4. Expand unit tests for typed registration, provider failures, header propagation, safe retries, unsafe-method non-retries, timeouts, circuit behaviour, and error parsing.
5. Publish/version the package only after CI, package inspection, and consumer tests succeed.

### 2.2 API conventions and error contracts
1. Standardize /api/v1 resource routes and HTTP method semantics.
2. Use consistent success responses, 201 Created with Location, 204 No Content, and bounded pagination metadata.
3. Standardize errors as application/problem+json (RFC 9457) with consistent type, title, status, detail, instance, and a non-sensitive traceId/correlation reference.
4. Keep tenant/resource existence responses from leaking cross-tenant data.
5. Document concurrency, validation, conflict, not-found, cancellation, and pagination conventions in OpenAPI and this repository.

### 2.3 Integration testing strategy
1. Keep unit tests for client handlers, options, mapping and pure business rules.
2. Use WebApplicationFactory and PostgreSQL-backed integration tests for API/database behaviour.
3. Test the API client against a controlled in-memory HTTP server for request headers, authentication provider, Problem Details parsing, timeouts, and resilience behaviour.
4. Add contract checks between OpenAPI/API responses and client expectations.
5. Use isolated test databases and synthetic data only; never use patient data.
6. When production EF migrations are introduced in Phase 3, run integration tests against applied migrations rather than relying only on EnsureCreated.

**Phase 2 exit criteria**
- Typed and named API client use cases are covered and documented.
- Safe retry rules are explicit and unsafe requests are never transparently replayed.
- API error responses and client error parsing share one tested contract.
- API and API-client CI suites pass.
- A documented integration strategy is implemented by the relevant test projects.

## Phase 3 — Database and security

**Status: Partially implemented; starts after Phase 2 contracts are stable**

### Implemented foundation
1. PostgreSQL is the active database provider in CompileCare.Api.
2. EF Core persistence and tenant-aware facility entities/query filters are present.
3. Tenant-aware indexes, composite tenant foreign keys, audit timestamps and optimistic concurrency foundations exist.

### Remaining work
1. Add reviewed, versioned EF Core migrations and validate clean install/upgrade paths.
2. Define identity-provider integration using a standards-based OIDC/OAuth 2.0 approach; provider choice remains an explicit decision.
3. Implement authentication, tenant membership validation, and role/permission authorization at API and use-case boundaries.
4. Implement append-only audit events for authentication, authorization, sensitive record access, administrative actions, and material business changes.
5. Add tenant isolation/security tests for reads, writes, updates, deletes, relationship traversal, cache keys, background work, and privileged access.
6. Define backup/restore, retention, encryption, secret-management, and operational recovery controls before production.
7. Obtain qualified privacy, legal, and security reviews for applicable Indian healthcare/data-protection obligations; do not claim compliance from documentation alone.

**Phase 3 exit criteria:** authenticated users can access only tenants, facilities, roles and resources permitted by trusted server-side identity/membership; migrations upgrade safely; audit and security tests pass.

## Phase 4 — Core healthcare domain

**Status: Partially implemented**

### Implemented
1. Hospital hierarchy: Hospital → Department → Ward → Room → Bed.
2. CRUD endpoints and bounded pagination for the hierarchy.
3. Tenant-scoped uniqueness, relationships and persistence constraints.

### Remaining delivery order
1. Tenant and organization onboarding/provisioning, including hospital group/branch scope.
2. Complete facility/unit model and operational metadata needed by real hospitals.
3. Bed lifecycle and availability/occupancy rules (not just an IsActive flag).
4. Healthcare staff, provider profiles, departments/roles, assignments and schedules.
5. Patient registration, patient identifiers/MRN policy, search and duplicate-review workflow.
6. Patient appointment, check-in and queue journey.
7. OPD encounter and clinical documentation foundation.
8. Billing, pharmacy, laboratory and inpatient workflows in the approved MVP sequence.

Every domain workflow requires hospital-user validation, permissions, audit, tenant scope, acceptance criteria, migration and automated tests.

## Phase 5 — UI begins in parallel

**Status: Planned; start when Phase 2 API contracts are stable**

Do not wait for the complete backend. Build the UI incrementally against versioned contracts and mocked/staging APIs:
1. Design system and accessible component library.
2. Login and tenant/hospital selection.
3. Hospital admin dashboard shell, navigation and sidebar.
4. Organization and facility management UI.
5. Staff and role management.
6. Ward/room/bed management dashboard and availability views.
7. Patient registration UI when its API contract and workflow are ready.

The UI must enforce user experience rules, but authorization remains server-side. Unavailable backend capabilities must be clearly represented as pending, not simulated as persisted functionality.

## Release gates and working rules
- Work in the phase order above. Do not skip ahead because a later feature appears easier.
- Keep work in short-lived feature branches and focused pull requests; do not commit directly to main.
- Review root causes and existing architecture before changing implementation.
- Every PR includes the related documentation updates, tests, security/privacy implications, migration/recovery notes, and CI evidence.
- A phase is complete only when its exit criteria are met and reviewed.
- Keep the modular core simple; introduce separately deployed services only when a documented scaling, ownership, security, integration, or resilience need justifies them.
