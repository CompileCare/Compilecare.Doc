# ADR-001: Distributed Architecture with a Modular Core

- **Status:** Accepted for the initial delivery architecture; future service extraction remains evidence-driven
- **Date:** 2026-11-24
- **Decision owner:** CompileCare Product & Engineering
- **Review trigger:** When product scope, deployment constraints, module ownership, scaling, security or resilience requirements materially change

## Context

CompileCare must support multiple hospitals, branches, modules, integrations and potentially different deployment models. Independent scaling and deployment may be needed over time, but premature service fragmentation increases operational cost, latency, failure modes and development complexity.

The product blueprint, not the existing repository layout, determines the business capabilities and their ownership. A capability does not require its own repository or deployment.

## Decision

Adopt a **distributed-system target architecture**, implemented pragmatically:

- The initial hospital core runs as a **modular monolith** with strict internal boundaries and one primary application deployment.
- Modules are organized by bounded capability and interact through explicit application contracts and internal events, not another module's internals.
- Independently deployable services may be introduced only when a documented need for independent scaling, deployment, security isolation, integration isolation, ownership or resilience justifies the operational cost.
- Each independently deployed service owns its data and schema. Avoid shared-database coupling between separate services.
- Use REST/OpenAPI for synchronous APIs and versioned event contracts for asynchronous communication where useful.
- Design for timeouts, bounded retries/backoff, idempotency, correlation/trace context, observability and graceful degradation.
- Tenant identity and authorization context must be propagated securely; consumers validate it rather than blindly trusting message payloads.
- Shared libraries must be focused and stable. Do not share domain entities across module/service boundaries.
- Existing repositories may be retained, consolidated, reworked, archived or replaced to support the approved product architecture.

## Alternatives

1. Microservices for every capability immediately — rejected for initial delivery because it adds operational complexity before there is evidence of need.
2. A single monolith with no explicit boundaries or extraction path — rejected because it creates coupling and limits future deployment options.
3. Modular core with selective independent services — accepted.

## Consequences

- CompileCare can deliver the first hospital workflow without operating a distributed system prematurely.
- Module boundaries, service ownership, contracts and tenant isolation must be enforced and tested.
- Deployment, monitoring, secrets, messaging and incident operations become more complex as services are extracted.
- Repository boundaries remain an implementation decision and may change independently of product capabilities.

## Follow-up

Define data ownership, module dependencies, deployment topology, event broker and extraction criteria as concrete product capabilities require them. See [Product Blueprint v1](../../product/blueprint-v1.md).
