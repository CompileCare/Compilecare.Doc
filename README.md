# CompileCare Documentation

CompileCare is an India-focused hospital information and management platform for hospitals, clinics, and healthcare provider organizations. This repository is the product, architecture, workflow, and engineering documentation source of truth.

> **Status:** Foundation documentation — draft v1. Product scope and several architecture decisions remain proposals until reviewed and approved with hospital stakeholders and technical owners.

## Start here
- [Product Blueprint v1](product/blueprint-v1.md) — product intent, user roles, patient journey, capability map, organization/tenant model, MVP and execution sequence.
- [Product vision and scope](product/vision-and-scope.md)
- [MVP scope and non-goals](product/mvp-scope.md)
- [Glossary](product/glossary.md)
- [Hospital module catalogue](product/module-catalogue.md)
- [Delivery roadmap and current implementation status](product/roadmap.md)
- [Requirements lifecycle](product/requirements-lifecycle.md)
- [Patient registration workflow](product/workflows/patient-registration.md)
- [Non-functional requirements](product/non-functional-requirements.md)
- [Architecture overview](architecture/overview.md)
- [API conventions and error contracts](architecture/api-conventions.md)
- [Tenant model](architecture/multi-tenancy.md)
- [ADR-006: Tenant, organization and patient data boundaries](architecture/decisions/ADR-006-tenant-organization-patient-boundaries.md)
- [ADR-007: Reuse CompileCare shared observability](architecture/decisions/ADR-007-shared-observability-reuse.md)
- [Architecture Decision Records](architecture/decisions/)
- [Security and privacy principles](security/security-and-privacy.md)
- [Integration testing strategy](engineering/integration-testing-strategy.md)
- [Engineering standards](engineering/standards.md)
- [Definition of Done](engineering/definition-of-done.md)
- [Existing repository assessment](engineering/repository-assessment.md)

## Delivery status

1. **Phase 1 — Backend foundation:** API foundation and shared observability are implemented and merged.
2. **Phase 2 — Developer platform:** ApiClient foundation and resilience/error contracts are implemented and merged. Database operations ownership, migration, backup/restore and environment runbooks are also established in CompileCare.Database (PR #1). Package publication and consuming-application validation, plus platform-specific operational automation, remain release tasks.
3. **Phase 3 — Database and security:** PostgreSQL/EF Core 10, OIDC/JWT tenant membership, facility RBAC, append-only request audit, global tenant registry/lifecycle, trusted initial membership provisioning and organization-scope authorization are implemented and merged in CompileCare.Api (PRs #12–#13). Broader asynchronous/cross-tenant tests, audit querying/retention policy and production operational controls remain.
4. **Phase 4 — Core healthcare domain:** Initial hospital hierarchy CRUD and pagination are implemented. Organization onboarding, staff, patient registration, appointments, queue and OPD remain.
5. **Product-first definition:** The Product Blueprint remains the source of truth. Validate open decisions affecting tenant, patient and clinical data boundaries before expanding into patient-facing and clinical domains.

A merged repository or passing build does not imply that a complete hospital workflow or production readiness has been delivered.

## Documentation governance

Important decisions must be recorded as ADRs. Documents distinguish implemented facts, accepted decisions, proposals, assumptions and open questions. Update affected requirements, workflows, API contracts and tests when scope or behaviour changes.

CompileCare is exclusively a hospital and healthcare systems product. Unrelated products and repositories are outside this documentation and delivery scope.

## Contributing

Use short-lived branches, focused commits, reviewable pull requests, automated tests and documentation updates. Follow [Engineering Standards](engineering/standards.md) and [Definition of Done](engineering/definition-of-done.md).
