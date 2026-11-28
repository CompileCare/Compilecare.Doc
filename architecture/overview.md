# Architecture Overview

**Status:** Initial delivery architecture accepted; detailed module boundaries and deployment choices remain subject to product requirements and ADR review.

The [Product Blueprint v1](../product/blueprint-v1.md) is the product-first reference for capabilities, workflows, users, MVP and delivery order. Architecture follows those product boundaries, not the current repository inventory.

## Target and implementation architecture

HeapCare has a distributed-system target, but the first hospital core is implemented as a **modular monolith** with explicit internal boundaries and one primary application deployment. Extract a capability into an independently deployed service only when scaling, deployment, ownership, security isolation, integration or resilience needs justify the operational cost. See [ADR-001](decisions/ADR-001-distributed-architecture.md).

## Current technology baseline

| Layer | Direction | Status |
|---|---|---|
| Web UI | React + TypeScript | Proposed; confirm through product/UI discovery |
| API | ASP.NET Core / .NET 10 | Implemented in HeapCare.Api |
| Persistence | PostgreSQL + EF Core 10 | Implemented; initial facility hierarchy migrations are merged |
| API contracts | Versioned REST + OpenAPI | Existing conventions; evolve by module |
| Authentication | OIDC/JWT with database-backed tenant membership | First validation/authorization slice implemented; identity provider lifecycle and provisioning remain |
| API client | HeapCare.ApiClient | Reusable foundation implemented; publication/consumer validation remains |
| Observability | HeapCare.Observability + HeapCare.Observability.AspNetCore | Existing shared packages consumed by HeapCare.Api; retain and reuse |
| Cache | No mandatory cache dependency for the MVP | Introduce only for measured need and tenant-safe semantics |
| Async messaging | Select per validated workflow | Broker choice not yet made |
| Background jobs | Select per validated workflow | No framework mandated today |
| Packaging/deployment | Docker/containerized environments | Existing foundation; hardening and rehearsal remain |
| CI | GitHub Actions | Existing workflows; standardize quality/security gates |

PostgreSQL and EF Core 10 are the current implemented transactional baseline. Revisit only with documented workload, customer, licensing, hosting or operational evidence; do not treat SQL Server as an active default for this foundation.

## Shared observability

The existing HeapCare.Observability and HeapCare.Observability.AspNetCore packages are the canonical shared telemetry/bootstrap foundation for HeapCare .NET hosts. Reuse them for logging, tracing, metrics and health-check wiring rather than creating duplicate pipelines. Package compatibility and security hardening remain review items. See [ADR-007](decisions/ADR-007-shared-observability-reuse.md).

## Logical layers and bounded contexts

Presentation/API; Application (use cases/orchestration); Domain (rules/entities/value objects); Infrastructure (database/providers/messaging/storage). Initial capabilities include platform/tenant, organization/facilities, identity/access, patients, scheduling, front desk/queue, clinical/OPD and revenue cycle. Diagnostics, pharmacy, inpatient, inventory and integrations are later capability areas. The blueprint defines module responsibilities and their proposed delivery order.

## Service and module boundaries

- Each module owns its business rules and write operations.
- Modules inside the core interact through explicit interfaces/contracts and internal events, not another module's internals or direct table writes.
- Independently deployed services communicate via APIs or versioned events.
- Avoid runtime shared-database coupling between independently deployed services.
- Shared libraries remain focused and stable; do not share domain entities across service boundaries.
- A module does not automatically require a separate repository, database or deployable.

## Distributed-system principles

- Synchronous calls only when the user workflow requires an immediate response; avoid deep call chains.
- Use asynchronous messaging only for workflows where decoupling and resilience justify its operational needs.
- Apply timeouts, bounded retries/backoff, circuit breakers where appropriate and graceful degradation.
- Propagate correlation/trace context and validated tenant context across requests and messages.
- Design for partial failure; do not assume a multi-service workflow is one ACID transaction. Use compensating actions/sagas where warranted.
- Enforce secrets, authentication, authorization and tenant isolation at every service boundary.

## API principles

Version intentionally; publish OpenAPI; use consistent validation/errors; authorize at endpoint and use-case boundaries; validate tenant/facility scope on every data access; use idempotency for retry-sensitive operations; paginate/filter collections; never leak internal exception details.

## Data principles

Enforce relational integrity and explicit tenant ownership/indexing. Audit important events. Use versioned migrations and prevent production schema drift. Use consistent time handling. Define correction, retention and deletion rules rather than casually hard-deleting clinical/financial records. A shared database deployment may serve the initial modular monolith, but module data ownership and transaction boundaries remain explicit.

## Deployment and quality attributes

Support local, CI/test, staging and production environments with reproducible containerized deployments. Shared SaaS is an initial candidate; dedicated/hybrid topology remains open pending customer requirements. Security, privacy, tenant isolation, availability, recovery, performance, maintainability, observability, interoperability and accessibility are design requirements.
