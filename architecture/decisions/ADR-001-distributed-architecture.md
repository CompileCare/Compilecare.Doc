# ADR-001: Distributed Architecture with a Modular Core

- **Status:** Proposed
- **Date:** 2026-11-23

## Context
CompileCare is a client-based healthcare platform expected to support multiple hospitals, branches, modules, integrations, and deployment models. Distributed deployment and independent scaling may be needed, but early fragmentation increases operational cost, latency, failure modes, and development complexity.

## Decision
Adopt a **distributed-system target architecture**, implemented pragmatically:

- The hospital core initially runs as a **modular monolith** with strict module boundaries and a single primary deployment.
- Independently deployable services are permitted for capabilities with clear isolation, scaling, integration, security, or operational reasons (for example, OTP, notifications, document processing, or external integrations).
- Modules communicate through explicit contracts. In-process application interfaces/domain events may be used inside the core; asynchronous messaging is preferred for cross-service workflows where decoupling and resilience matter.
- Each service owns its data and schema. Avoid shared-database coupling between independently deployed services.
- Use REST/OpenAPI for synchronous APIs and versioned event contracts for asynchronous communication.
- Design for timeouts, retries with backoff, idempotency, correlation IDs, observability, and graceful degradation.
- Tenant identity and authorization context must be propagated securely; consumers must validate it rather than blindly trust message payloads.

## Alternatives
1. Microservices for every module immediately — rejected for initial delivery due to unnecessary complexity.
2. A single monolith with no extraction path — rejected because it constrains future independent deployment and scaling.
3. Distributed modular core with selective services — chosen as the proposed target.

## Consequences
- We get a distributed-system architecture and deployment flexibility without requiring every domain to be a microservice from day one.
- Module boundaries, service ownership, contracts, and tenant isolation must be enforced and tested.
- Deployment, monitoring, secrets, messaging, and incident operations become more complex as services are extracted.

## Next validation
Confirm module/service boundaries, deployment model, event broker, service discovery, and data ownership in later ADRs before implementation.
