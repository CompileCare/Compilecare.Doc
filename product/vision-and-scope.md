# Product Vision and Scope

**Status:** Draft v1

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
- Tenant/customer and hospital organization setup
- Authentication, users, roles, permissions
- Patient registration and identity record
- Appointment scheduling and queue/token flow
- OPD encounter and clinical-documentation foundation
- Service catalogue and basic billing/payment
- Audit trail and operational dashboard

MVP workflows and acceptance criteria must be validated with hospital stakeholders.

## Not in first MVP
Full microservices decomposition, autonomous clinical decision-making, full PACS/DICOM implementation, complex insurance/TPA automation, HR/procurement/facilities suite, native patient apps, or unverified compliance claims.

## Success measures (to define with pilot hospitals)
Registration-to-appointment time, duplicate-record rate, queue wait time, documentation completion, billing reconciliation, availability/recovery, role adoption, and support/defect rates.

## Assumptions/open questions
- SQL Server is a candidate; licensing, hosting, operational cost, and requirements need evaluation/ADR.
- SaaS shared deployment and dedicated hospital deployment may both be desired; MVP model remains open.
- Tenant may represent customer/hospital group with hospitals/branches below it; validate.
- Clinical workflows and terminology require qualified hospital-user validation.
- Applicable Indian privacy, health-data, ABDM, and other obligations require qualified review before production.
