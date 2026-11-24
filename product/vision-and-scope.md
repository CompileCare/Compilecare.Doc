# Product Vision and Scope

**Status:** Draft v1 — see the [Product Blueprint v1](blueprint-v1.md) for the consolidated product model and delivery sequence.

## Vision

Build a reliable, secure, configurable, and extensible hospital software platform for hospitals in India. Support the hospital journey—from patient registration and appointments through care delivery, diagnostics, pharmacy, billing, admission, discharge, and reporting—with integrated workflows rather than disconnected tools.

## Principles

1. Patient-centred workflows that reduce duplicate entry.
2. Hospital-configurable departments, services, tariffs, roles, and policies.
3. Explicit tenant isolation.
4. Modular, maintainable boundaries.
5. API-first documented contracts.
6. Security/privacy by design: least privilege, auditability, data minimization, secure defaults.
7. Operational resilience: backups, recovery, monitoring, safe deployment.
8. India-ready: currency, time zones, languages, addresses, and workflows; validate legal/standards requirements.
9. Interoperability planning: ABDM/ABHA, FHIR, HL7, DICOM where relevant.
10. Pragmatic delivery: complete a coherent workflow before broadening scope.

## Target customers

Small/mid-sized hospitals, multi-specialty hospitals, hospital groups/chains, and specialty providers that may be supported through configurable workflows.

## Initial scope

- Tenant/customer and hospital organization setup.
- Authentication, users, roles, permissions.
- Patient registration and identity record.
- Appointment scheduling and queue/token flow.
- OPD encounter and clinical-documentation foundation.
- Service catalogue and basic billing/payment when required by the validated pilot.
- Audit trail and operational dashboard.

MVP workflows and acceptance criteria must be validated with hospital stakeholders.

## Not in first MVP

Full microservices decomposition, autonomous clinical decision-making, full PACS/DICOM implementation, complex insurance/TPA automation, HR/procurement/facilities suite, native patient apps, or unverified compliance claims.

## Success measures

Registration-to-appointment time, duplicate-record rate, queue wait time, documentation completion, billing reconciliation, availability/recovery, role adoption and support/defect rates. Pilot targets are to be agreed with participating hospitals.

## Technology and product assumptions

- PostgreSQL and EF Core 10 are the active transactional foundation in CompileCare.Api (see ADR-003); SQL Server is not the current default.
- A shared SaaS tenant model is an initial candidate; dedicated/hybrid deployment may be assessed against customer and operational requirements.
- Tenant may represent a single hospital customer or a hospital group; organization and patient-sharing policy requires validation.
- Clinical workflows and terminology require qualified hospital-user validation.
- Applicable Indian privacy, health-data, ABDM and other obligations require qualified review before production claims or integrations.
