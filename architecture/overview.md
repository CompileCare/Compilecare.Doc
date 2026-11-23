# Architecture Overview

**Status:** Proposed; decisions must be recorded and approved.

## Target architecture
CompileCare is intended to be a **distributed system**: domain capabilities may run as independently deployable services and communicate through explicit APIs and asynchronous events. However, the first hospital-core implementation should use a **modular monolith** (one primary deployable) with strict internal boundaries. This is an implementation strategy—not a rejection of distributed architecture.

Extract a capability into a service when independent scaling, deployment, ownership, security boundary, integration isolation, or resilience needs justify the operational cost. See [ADR-001](decisions/ADR-001-distributed-architecture.md).

## Proposed technology baseline
| Layer | Proposal | Status |
|---|---|---|
| Web UI | React + TypeScript | Proposed |
| API | ASP.NET Core / .NET 10 | Implemented in CompileCare.Api |
| Persistence | PostgreSQL + EF Core 10 | Implemented foundation; migrations remain pending |
| Data access | EF Core 10 | Implemented foundation |
| API contracts | Versioned REST + OpenAPI | Proposed |
| Authentication | OIDC/OAuth 2.0-compatible identity provider | Phase 3; provider decision pending |
| Cache | CompileCare.Caching | Existing repo; audit required |
| Service communication | CompileCare.ApiClient | Existing repo; audit required |
| Observability | CompileCare.Observability + OpenTelemetry | Existing foundation |
| Async messaging | Broker abstraction; broker choice TBD | Proposed |
| Background jobs | Queue/job abstraction; evaluate by use case | Proposed |
| Packaging | Docker | Proposed |
| CI | GitHub Actions | Existing in one repo; standardize |

## Logical layers and bounded contexts
Presentation/API; Application (use cases/orchestration); Domain (rules/entities/value objects); Infrastructure (database/providers/messaging/storage). Initial modules include Patients, Appointments, Clinical, Billing, Pharmacy, Laboratory, and IPD. Building blocks include tenant context, audit, errors, identity, time, and IDs.

## Service and module boundaries
- Each module/service owns its business rules and data/schema.
- Modules inside the core interact through explicit interfaces/contracts and internal events, not another module's internals.
- Independently deployed services communicate via APIs or versioned events.
- Avoid runtime shared-database coupling between independently deployed services.
- Shared libraries should remain small and stable; do not share domain entities across service boundaries.

## Distributed-system principles
- Synchronous calls only when the user workflow requires an immediate response; avoid deep call chains.
- Use asynchronous messaging for suitable workflows, with idempotent consumers and versioned contracts.
- Apply timeouts, bounded retries/backoff, circuit breakers where appropriate, and graceful degradation.
- Correlation/trace context and safe tenant context propagation across requests and messages.
- Design for partial failure; never assume a multi-service workflow is one ACID transaction. Use compensating actions/sagas when warranted.
- Secrets, authentication, authorization, and tenant isolation are enforced at each service boundary.

## API principles
Version intentionally; publish OpenAPI; consistent validation/errors; authorize at endpoint and use-case boundaries; validate tenant scope on every data access; idempotency for retry-sensitive operations; pagination/filtering; no internal exception leakage.

## Data principles
Relational integrity; explicit tenant ownership/indexing; audit important events; migrations; no production schema drift; consistent time handling; define correction/retention rules rather than casually hard-deleting clinical/financial records. Service-owned data; shared DB instance may be used initially, but ownership boundaries must be enforced.

## Deployment
Local, CI/test, staging, production. Reproducible Docker-based deployments. SaaS vs dedicated deployment remains open. A client may eventually use shared SaaS, dedicated services/data, or hybrid deployment subject to approved requirements.

## Quality attributes
Security, privacy, tenant isolation, availability, recovery, performance, maintainability, observability, interoperability, accessibility.
