# HeapCare Documentation

> **Brand migration status:** The GitHub organization and all 11 repository slugs now use HeapCare. Source namespaces, project paths and package IDs have been migrated in the repositories where implementation exists. Legacy database identifiers, EF migration history, Keycloak realm/client/audience IDs and persistent Docker volume names remain intentionally unchanged pending separately tested compatibility migrations.

HeapCare is an India-focused hospital information and management platform for hospitals, clinics, and healthcare provider organizations. This repository is the product, architecture, workflow, and engineering documentation source of truth.

> **Status:** Foundation documentation — draft v1. Product scope and several architecture decisions remain proposals until reviewed and approved with hospital stakeholders and technical owners.

## Start here
- [Product Blueprint v1](product/blueprint-v1.md) — product intent, user roles, patient journey, capability map, organization/tenant model, MVP and execution sequence.
- [Product vision and scope](product/vision-and-scope.md)
- [MVP scope and non-goals](product/mvp-scope.md)
- [Glossary](product/glossary.md)
- [Hospital module catalogue](product/module-catalogue.md)
- [Delivery roadmap and current implementation status](product/roadmap.md)
- [ADR-017: HeapCare brand migration](architecture/decisions/ADR-017-heapcare-brand-migration.md) — staged identity migration, technical contracts, dependencies and verification gates.
- [Production identity and infrastructure cutover inventory (2026-11-28)](operations/heapcare-production-cutover-inventory-2026-11-28.md) — repository-backed evidence, preserved local compatibility identifiers, unknown production settings and gated cutover sequence.
- [HeapCare migration report (2026-11-27)](architecture/migrations/heapcare-migration-report-2026-11-27.md) — merged repository work, CI evidence, compatibility decisions and remaining closure checklist.
- [HeapCare post-migration full-platform QA report (2026-11-27)](qa/post-migration-full-platform-qa-2026-11-27.md) — full nine-service live startup, 11-repo builds/tests, 27/27 E2E, 24/24 security, observability and defect-remediation evidence.
- [Full-platform E2E validation — 2026-11-26](qa/e2e-validation-2026-11-26.md)
- [Requirements lifecycle](product/requirements-lifecycle.md)
- [Patient registration workflow](product/workflows/patient-registration.md)
- [Staff and provider management workflow](product/workflows/staff-and-provider-management.md)
- [Appointments, walk-ins, check-in and queue workflow](product/workflows/appointments-and-queue.md)
- [Reception and clinical workspace UX foundation](product/workflows/reception-and-clinical-workspace-ux.md)
- [Enterprise experience uplift](product/design/enterprise-experience-uplift.md) — visual quality direction, design-system foundations and the Phase 5 refinement sequence.
- [OPD billing foundation and deferred scope](product/workflows/opd-billing-foundation.md)
- [ADR-014: Appointment scheduling and queue boundaries](architecture/decisions/ADR-014-appointment-scheduling-and-queue-boundaries.md)
- [ADR-013: Staff and provider profile boundaries](architecture/decisions/ADR-013-staff-and-provider-profile-boundaries.md)
- [ADR-012: Verified patient contact data](architecture/decisions/ADR-012-verified-patient-contact-data-and-matching.md)
- [Non-functional requirements](product/non-functional-requirements.md)
- [Architecture overview](architecture/overview.md)
- [API conventions and error contracts](architecture/api-conventions.md)
- [Tenant model](architecture/multi-tenancy.md)
- [ADR-006: Tenant, organization and patient data boundaries](architecture/decisions/ADR-006-tenant-organization-patient-boundaries.md)
- [ADR-007: Reuse HeapCare shared observability](architecture/decisions/ADR-007-shared-observability-reuse.md)
- [Architecture Decision Records](architecture/decisions/)
- [Security and privacy principles](security/security-and-privacy.md)
- [Integration testing strategy](engineering/integration-testing-strategy.md)
- [Engineering standards](engineering/standards.md)
- [Definition of Done](engineering/definition-of-done.md)
- [Existing repository assessment](engineering/repository-assessment.md)
- [Local development setup and troubleshooting](engineering/local-development-setup.md) — Docker Compose, Keycloak/OIDC, private NuGet restore, observed issues, fixes and remaining local acceptance checks.

## Delivery status

1. **Phase 1 — Backend foundation:** API foundation and shared observability are implemented and merged.
2. **Phase 2 — Developer platform:** ApiClient foundation and resilience/error contracts are implemented and merged. Database operations ownership, migration, backup/restore and environment runbooks are also established in HeapCare.Database (PR #1). Package publication and consuming-application validation, plus platform-specific operational automation, remain release tasks.
3. **Phase 3 — Database and security:** PostgreSQL/EF Core 10, OIDC/JWT tenant membership, facility RBAC, append-only request audit, global tenant registry/lifecycle, trusted initial membership provisioning and organization-scope authorization are implemented and merged in HeapCare.Api (PRs #12–#13). Broader asynchronous/cross-tenant tests, audit querying/retention policy and production operational controls remain.
4. **Phase 4 — Core healthcare domain:** Hospital hierarchy, patient master/registration, hospital-local MRN, advisory duplicate detection and normalized multi-signal matching are implemented. The initial tenant-scoped staff/provider profile and assignment foundation is also implemented and merged (API PR #18; design/workflow in documentation PR #21). Verified patient-contact capture is designed but not implemented (documentation PR #19). Next delivery is appointments, walk-ins, check-in and queue following ADR-014; patient-contact implementation remains gated on its separate security/privacy decisions, followed by OPD. Broader workforce identity, credentialing, rostering, attendance, leave, payroll, secure employee records and cross-tenant workforce sharing are explicitly retained as later milestones.
5. **Phase 4 — OPD billing:** Minimal baseline approved for implementation: service catalogue, visit-linked invoices, payments, receipts and daily collection reporting. Tax, discounts, refunds, insurance, payment gateway and accounting integrations remain explicitly deferred pending pilot decisions; see `product/workflows/opd-billing-foundation.md`.
6. **Phase 5 — User experience:** Reception and clinical workspace UX foundation is specified in `product/workflows/reception-and-clinical-workspace-ux.md`; the next implementation increment is the authenticated shell and API-backed patient lookup in `HeapCare.Web`, separate from the public `HeapCare.Site`.
6. **Product-first definition:** The Product Blueprint remains the source of truth. Validate open decisions affecting tenant, patient and clinical data boundaries before expanding into patient-facing and clinical domains.

A merged repository or passing build does not imply that a complete hospital workflow or production readiness has been delivered.

## Documentation governance

Important decisions must be recorded as ADRs. Documents distinguish implemented facts, accepted decisions, proposals, assumptions and open questions. Update affected requirements, workflows, API contracts and tests when scope or behaviour changes.

HeapCare is exclusively a hospital and healthcare systems product. Unrelated products and repositories are outside this documentation and delivery scope.

## Contributing

Use short-lived branches, focused commits, reviewable pull requests, automated tests and documentation updates. Follow [Engineering Standards](engineering/standards.md) and [Definition of Done](engineering/definition-of-done.md).

## Operations and security

- [Backup, Restore and Disaster-Recovery Operational Baseline](operations/backup-restore-disaster-recovery.md)
- [Production Ingress and Network-Security Acceptance Checklist](operations/production-ingress-and-network-checklist.md)
- [Security and Privacy](security/security-and-privacy.md)
- [Application Threat Model](security/threat-model.md)
