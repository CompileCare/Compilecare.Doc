# ADR-007: Reuse HeapCare Shared Observability

- **Status:** Accepted architectural direction; package hardening remains open
- **Date:** 2026-11-24
- **Decision owners:** HeapCare Engineering
- **Related:** [Product Blueprint v1](../../product/blueprint-v1.md), [ADR-001](ADR-001-distributed-architecture.md)
- **Review trigger:** Adding a host/service, changing exporters, changing package compatibility, or introducing sensitive telemetry.

## Context

HeapCare already has shared NuGet packages in **HeapCare.Observability** and **HeapCare.Observability.AspNetCore**. The current **HeapCare.Api** references both and uses the shared host logging, OpenTelemetry and health-check registration extensions.

Duplicating telemetry bootstrap in individual APIs/services would create inconsistent correlation, configuration, exporters, redaction and operational behaviour. Replacing the library without an evidenced gap would discard existing investment and create parallel implementations.

## Decision

1. The existing **HeapCare.Observability** and **HeapCare.Observability.AspNetCore** packages are the canonical shared observability foundation for HeapCare .NET hosts.
2. New .NET hosts must evaluate and reuse these packages before adding an independent logging, tracing, metrics or health-check stack.
3. Hosts may add domain-specific instruments, safe enrichers, dependency checks and exporter configuration through supported extension points, but must not create a competing duplicate pipeline.
4. Shared package changes belong in the Observability repository and require focused tests, compatibility validation, versioning, release notes and consuming-application validation.
5. The API's .NET 10 target and the shared library's .NET 9 target are not, by themselves, a reason to replace the library. Validate target-framework/dependency compatibility and make the smallest supported change if required.
6. Configuration remains explicit and environment-driven. Do not commit endpoints containing credentials, tokens or secret values.
7. Logs, traces, metric dimensions, health details and exception messages must not contain patient names, MRNs, diagnoses, clinical notes, bearer tokens, credentials, raw request/response bodies or other PHI/secrets.
8. Use only the minimum safe tenant/correlation metadata needed for diagnosis. Never use patient or other high-cardinality identifiers as metric labels.
9. Telemetry is not the authoritative clinical or financial audit store. Domain/security audit events use approved audit persistence and access policy.
10. Independently deployed services identify themselves consistently and propagate trace/correlation context and trusted tenant context where safe and required.

## Consequences

Services share instrumentation and operational conventions. Observability remains a first-class asset and is not removed as a repository-cleanup exercise. Compatibility, beta package review, meaningful tests, exporter defaults, redaction, supply-chain permissions and stable publishing remain explicit engineering work.

## Validation required

- Build and test each package on its supported target framework(s).
- Verify package consumption from the .NET 10 API and a representative worker/host.
- Verify logs, traces, metrics, health endpoints, correlation propagation, exporter-failure behaviour and redaction.
- Confirm no PHI/secrets reach logs, traces, metric labels, health responses or dashboards.
- Validate NuGet versioning, permissions, provenance and release workflow.
