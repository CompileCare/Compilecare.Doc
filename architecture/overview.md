# Architecture Overview

**Status:** Proposed; decisions must be recorded and approved.

## Initial approach
Start as a **modular monolith**: one deployable backend with clear module boundaries. Avoid premature microservices. Extract services only for demonstrated scaling, deployment, ownership, or integration needs.

## Proposed technology baseline
| Layer | Proposal | Status |
|---|---|---|
| Web UI | React + TypeScript | Proposed |
| API | ASP.NET Core / .NET | Proposed |
| Persistence | Relational SQL; SQL Server candidate | ADR required |
| Data access | EF Core; SQL/Dapper when justified | Proposed |
| API contracts | Versioned REST + OpenAPI | Proposed |
| Authentication | OIDC/OAuth-compatible provider | ADR required |
| Cache | CompileCare.Caching | Existing repo; audit required |
| Service communication | CompileCare.ApiClient | Existing repo; audit required |
| Observability | CompileCare.Observability + OpenTelemetry | Existing foundation |
| Background jobs | Queue/job abstraction; evaluate by use case | Proposed |
| Packaging | Docker | Proposed |
| CI | GitHub Actions | Existing in one repo; standardize |

## Logical layers
Presentation/API; Application (use cases/orchestration); Domain (rules/entities/value objects); Infrastructure (database/providers/messaging/storage); Modules (Patients, Appointments, Clinical, Billing, Pharmacy, Laboratory, IPD); Building blocks (tenant context, audit, errors, identity, time, IDs).

## Module boundaries
Modules own their rules and data. Cross-module interactions use explicit contracts or domain events, not another module's internals. Shared code stays small and stable.

## API principles
Version intentionally; publish OpenAPI; consistent validation/errors; authorize at endpoint and use-case boundaries; validate tenant scope on every data access; idempotency for retry-sensitive operations; pagination/filtering; no internal exception leakage.

## Data principles
Relational integrity; explicit tenant ownership/indexing; audit important events; migrations; no production schema drift; consistent time handling; define correction/retention rules rather than casually hard-deleting clinical/financial records.

## Deployment
Local, CI/test, staging, production. Reproducible Docker-based deployments. SaaS vs dedicated deployment remains open; design must not assume only one topology.

## Quality attributes
Security, privacy, tenant isolation, availability, recovery, performance, maintainability, observability, interoperability, accessibility.
