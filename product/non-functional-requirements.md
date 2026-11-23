# Non-Functional Requirements

**Status:** Draft. Targets are placeholders until workload, client SLAs, hosting, and risk are validated.

## Performance
Define p95/p99 latency targets per workflow and API; expected concurrent users; peak registration/appointment throughput; reporting limits; and load-test scenarios.

## Availability and resilience
Define availability objectives by deployment tier, health/readiness checks, graceful degradation, timeout/retry policies, circuit breakers where appropriate, and dependency failure behavior.

## Security and privacy
Threat model; least privilege; secure secrets; encryption in transit and at rest; tenant isolation; audit; vulnerability/dependency scanning; secure SDLC; incident response. Healthcare and India-specific legal/regulatory obligations require qualified review—this document does not claim compliance.

## Recovery and backup
Define RPO/RTO per client/deployment; backup frequency, encryption, retention, restore tests, disaster recovery, and tenant-level recovery/export needs.

## Scalability
Support horizontal scaling for stateless components; identify bottlenecks and data partitioning strategy only based on evidence. Avoid coupling module scale to unrelated workloads where service extraction is justified.

## Observability
Structured logs, metrics, distributed traces, correlation IDs, service health, alerting, dashboards, and safe redaction. Do not log patient data or secrets unless explicitly approved and protected.

## Maintainability and quality
Architecture boundaries, automated tests, code review, versioned contracts, migration discipline, documentation, and dependency updates.

## Accessibility and usability
Target applicable accessibility standards after selection; support keyboard navigation, readable contrast, responsive UI, and localization (initial language requirements to be decided).

## Interoperability
Evaluate standards and client integrations (FHIR/HL7, imaging, ABDM, insurer/TPA, etc.) based on actual requirements and approval.

## Open items
Set measurable numeric targets before production readiness, per client tier and deployment model.
