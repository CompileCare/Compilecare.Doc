# CompileCare Product Blueprint v1

**Status:** Draft for product and hospital-user validation  
**Version:** 1.0-draft  
**Date:** 2026-11-24

> This is the product-first reference for scope, users, workflows, capabilities and delivery order. It records recommendations and open questions; it is not an approved clinical specification, legal opinion, or compliance certification. Hospital workflows and applicable obligations must be validated with qualified stakeholders.

## 1. Product intent

CompileCare is an India-focused hospital information and management platform for healthcare organizations. It should connect administrative and clinical workflows around the patient journey, rather than become a collection of disconnected CRUD screens.

The platform should work for a single hospital or clinic and be able to grow to a customer group operating multiple hospitals, campuses, branches and departments without redesigning tenant ownership or authorization.

**Product outcomes**
- Reduce repeated patient data entry and prevent avoidable duplicate records.
- Coordinate appointments, walk-ins, arrivals, queues and care hand-offs.
- Give clinical users a coherent encounter workflow while preserving professional judgement.
- Connect orders, results, prescriptions and revenue workflows with explicit ownership and status.
- Give operators reliable facility and operational visibility.
- Protect patient and organizational information through least privilege, tenant isolation, audit and controlled access.
- Support integration and deployment options without imposing unnecessary distributed-system complexity.

**Initial market posture:** India-first localization; candidate customers include clinics, small/mid-sized hospitals, multi-specialty hospitals and hospital groups. The first release should prove a focused outpatient journey. Segment, specialty, hosting and commercial assumptions require pilot discovery.

## 2. Product principles

1. **Workflow-first:** organize around patients and staff, not repositories or tables.
2. **Patient-centred:** search before create; preserve identity history; never silently merge records.
3. **Secure by default:** validate identity, tenant membership, organizational scope and permissions at trusted server boundaries.
4. **Configurable, not bespoke:** prefer explicit policies and configuration over customer forks.
5. **Clinical safety:** support healthcare professionals; do not autonomously diagnose, prescribe or replace judgement.
6. **Auditable:** sensitive access and material clinical, financial and administrative actions are attributable.
7. **Interoperable:** use documented contracts and standards when relevant and validated.
8. **Operationally supportable:** migrations, backups, restore, monitoring, incident response and safe release are product requirements.
9. **Incremental delivery:** deliver a coherent vertical slice before broad module coverage.
10. **Architecture follows product needs:** repositories and separately deployed services are implementation choices, not product boundaries.

## 3. Users and access model

These are personas, not a finalized role matrix. A person may hold multiple scoped roles. Authorization combines role permissions with tenant, hospital, branch, department, resource and workflow scope where required.

| Persona | Responsibilities | Typical scope |
|---|---|---|
| Platform operator | Tenant lifecycle, trusted onboarding, operational support | Explicit platform scope only |
| Customer/group administrator | Organization policy and hospital oversight | Tenant and assigned group/sites |
| Hospital administrator | Facility setup, staffing and operations | Assigned hospitals |
| Reception/registration | Patient search, registration, appointments, check-in | Assigned branches and functions |
| Doctor/consultant | Review history, document encounters, diagnoses and orders | Assigned patients/encounters |
| Nurse/triage staff | Vitals, triage and nursing workflows | Assigned clinical areas |
| Lab/radiology staff | Samples, diagnostic worklists, results and reports | Authorized diagnostic worklists |
| Pharmacist | Prescription review, dispensing and stock | Pharmacy and permitted prescriptions |
| Cashier/billing | Estimates, invoices, payments, receipts and refunds | Financial functions |
| Insurance/TPA staff | Eligibility, preauthorization and claims | Explicitly assigned financial data |
| Store/inventory staff | Stock, batches, expiry, purchases and issues | Assigned stores |
| Auditor/compliance | Authorized audit and compliance review | Purpose-limited read-only access |
| Patient/guardian | View permitted personal records, appointments and instructions | Patient-authorized self/guardian scope |

**Authorization rules**
- Tenant roles never confer platform-operator privileges.
- UI visibility and caller-supplied role claims are not security controls.
- Deny by default; apply least privilege and organization/resource-level scope.
- Sensitive clinical, financial, export and administrative actions require explicit permission and audit.
- Break-glass, delegation, guardianship, temporary staff and emergency access need separately approved workflows.

## 4. Tenant and organization model

**Proposed hierarchy:**

**CompileCare Platform → Tenant/Customer → optional Hospital Group → Hospital → Branch/Campus → Department → Service/Unit → Ward/Room/Bed**

A tenant represents one customer relationship and is the top-level data-isolation boundary. It may operate one hospital or a group. Branches and departments are not separate tenants merely because they are separate sites. A group is optional. Distinguish the legal/operational hospital, physical campus, organizational department and operational ward/room/bed; allow approved customer variations through configuration.

### Recommended patient data boundary

- Patient master identity is tenant-owned by default, not globally shared across CompileCare customers.
- A patient may have registrations/affiliations at multiple hospitals or branches within that tenant, governed by explicit access policy.
- Encounters, orders, prescriptions, billing records and audit retain tenant and originating facility context.
- Cross-tenant matching/linking/sharing is disabled by default and requires a separately approved legal, consent, identity and technical model.
- Patient identifiers (UHID/MRN) follow a documented tenant policy. Facility aliases may be supported, but uniqueness/search/display rules must be explicit.
- Duplicate detection may suggest a match. Merge is privileged, reviewable, reversible where possible and audited; never automatic/silent.
- Same-tenant membership alone does not grant every staff member access to all clinical data.

This recommended boundary requires validation against patient mobility, group operations, consent, guardianship, emergency registration and retention before final schema approval. See [ADR-006](../architecture/decisions/ADR-006-tenant-organization-patient-boundaries.md) and proposed [ADR-008: Patient identity and registration boundary](../architecture/decisions/ADR-008-patient-identity-registration-boundary.md).

## 5. Patient and hospital workflows

### First vertical slice: outpatient journey

1. Pre-registration or arrival.
2. Patient search by configured identifiers and permitted demographic details.
3. Identity resolution and authorized duplicate review.
4. Registration with minimum required demographics/contact details and tenant-scoped identifier.
5. Appointment or walk-in selection: hospital/branch, service/department, provider if applicable, and date/time or queue.
6. Check-in and queue/token creation; handle cancellation, rescheduling, no-show and late arrival.
7. Triage/vitals by authorized staff.
8. OPD encounter: clinician review of permitted history, notes, diagnoses and care plan.
9. Orders for diagnostics, medicines, procedures or referrals using explicit lifecycles.
10. Minimal billing/receipt/payment if required by the pilot workflow.
11. Disposition: complete, follow-up, referral or later admission workflow.
12. Approved patient communication through a configured, consent-aware channel.

Each step needs actors, preconditions, alternate paths, permissions, state transitions, audit, failure/retry behaviour and acceptance tests. Clinical terminology, templates and triage protocols require hospital-user validation.

### Future workflows

- **IPD:** admission → bed search/reservation → admission → ward/bed assignment → nursing/medical care → orders/medication administration → transfer → discharge planning → settlement → discharge summary → follow-up.
- **Laboratory:** order → collection → specimen identification → processing → result validation → critical-result escalation → clinician review.
- **Radiology:** order → scheduling → modality/worklist → imaging/PACS reference → report → clinician review.
- **Pharmacy:** prescription → validation → dispensing → stock/batch movement → reconciliation/return.

These require dedicated lifecycle, status, authorization, traceability and failure requirements; they are not simply CRUD extensions.

## 6. Capability map and module responsibilities

Modules are logical domain boundaries, not automatic instructions to create repositories, databases or deployables.

| Module/capability | Owns | Initial priority |
|---|---|---|
| Platform & tenant registry | Tenant lifecycle, trusted provisioning, platform operations | Foundation |
| Organization & facilities | Group/hospital/branch/department/unit/facility resources | Extend existing slice |
| Identity & access | OIDC principal binding, membership, roles, permissions, scope | Foundation |
| Patient management | Identity, demographics, identifiers, registration, duplicate review | MVP |
| Scheduling | Provider/service availability, appointments, cancellation/reschedule | MVP |
| Front desk & queue | Arrival, check-in, tokens, queue and worklists | MVP |
| Clinical/OPD | Encounters, notes, observations, diagnoses, care plan and orders | MVP foundation |
| Revenue cycle | Tariffs, charges, invoices, receipts, payments and refunds | Minimal pilot scope |
| Diagnostics | Laboratory and radiology workflows/results | Later |
| Pharmacy | Medication catalogue, prescription validation, dispensing and stock | Later |
| Inpatient | Admission, transfer, bed occupancy, nursing and discharge | Later |
| Inventory/supply chain | Stores, batches, expiry, procurement and movement | Later |
| Workforce | Staff/provider profiles, schedules, rosters and credentials | As needed |
| Notifications | Templates, delivery, preferences, consent and retries | Supporting |
| Documents | Metadata, secure storage, access and retention | Supporting |
| Reporting/analytics | Permissioned operational/financial reporting and exports | Later/basic MVP views |
| Integrations | Identity, payments, messaging, FHIR/HL7, ABDM, DICOM | Validate per pilot |
| Audit/compliance | Security/business audit and controlled audit access | Cross-cutting |

Each module owns its rules and write operations. Modules use explicit application contracts or versioned events, not another module's internals or direct table writes. One PostgreSQL deployment may serve the initial modular monolith, but data ownership, schema ownership and transaction boundaries remain explicit. Do not share domain entities through a common library.

## 7. MVP definition

**Objective:** deliver a secure, tenant-aware outpatient journey for a pilot customer, from trusted organization setup to a completed OPD visit and follow-up.

**Stage A — platform and hospital setup**
- Tenant registry/lifecycle and restricted, idempotent tenant plus initial membership provisioning.
- Hospital/group/branch/department setup from the validated hierarchy.
- User membership, roles, permissions and organization scope.
- Audit for provisioning, role changes and privileged actions.

**Stage B — patient and outpatient operations**
- Patient registration, search, identifier policy and duplicate review.
- Appointment and walk-in booking.
- Check-in, queue/token lifecycle and operational worklist.
- OPD encounter with approved minimal clinical fields, vitals/observations, diagnosis, notes and follow-up.
- Minimal service catalogue and billing/receipt only if required to complete the pilot.

**Stage C — pilot readiness**
- Role-specific admin, reception and clinical screens.
- Critical workflow, tenant isolation and security integration tests.
- Audit review and controlled audit access.
- Backup/restore rehearsal; deployment and rollback runbooks.
- Observability dashboards, actionable alerts, redaction and performance tests.
- Hospital stakeholder acceptance.

**Not in the first MVP:** full IPD/nursing/medication administration/surgery; full LIS/RIS/PACS; pharmacy inventory/procurement; advanced insurance/TPA; ERP/HR/payroll; native patient app and broad portal; autonomous AI diagnosis/triage/prescribing; multi-region active-active; microservice extraction without measured need; ABDM/ABHA integrations before requirements, permissions and agreements are validated.

## 8. Architecture and technology direction

- Initial hospital core: **modular monolith hosted in CompileCare.Api**, with explicit bounded modules.
- Keep a selective distributed-service path; extract only when deployment, scaling, ownership, security isolation, integration or resilience justifies the cost.
- Use REST/OpenAPI for synchronous workflows. Add asynchronous messaging only where it improves decoupling/reliability and can be operated safely.
- Avoid circular dependencies, shared-table writes and business logic in transport endpoints.
- Existing implementation facts: API uses ASP.NET Core/.NET 10, PostgreSQL and EF Core 10, OIDC/JWT with database-backed tenant membership, versioned REST conventions, and CompileCare.ApiClient.
- React + TypeScript is the proposed web direction, subject to product/UI confirmation. Mobile is deferred until a validated use case and API/auth contract exist.
- No mandatory cache or message broker for the MVP. Add them only against measurable requirements and tenant-safe semantics.

### Shared Observability is a platform dependency

Reuse **CompileCare.Observability** and **CompileCare.Observability.AspNetCore** as the canonical shared instrumentation/bootstrap library. Do not recreate Serilog/OpenTelemetry setup, exporters or health-check wiring independently in each host. Hosts and modules may add domain-specific instruments and dependency checks through supported extension points, but must not create a competing duplicate pipeline.

Compatibility, framework targets, beta dependency review, package tests, exporter defaults, redaction and release permissions are hardening tasks—not reasons to replace the library. Validate package compatibility and make the smallest required change through the Observability repository.

No patient names, MRNs, diagnoses, clinical notes, tokens, credentials, raw request/response bodies or other PHI/secrets in logs, traces, metric labels, health details, exceptions or dashboards. Avoid patient/high-cardinality identifiers as metric dimensions. Telemetry is not the authoritative clinical or financial audit store.

See [ADR-007](../architecture/decisions/ADR-007-shared-observability-reuse.md).

## 9. Security, privacy, clinical safety and operations

These are requirements, not claims of production readiness or legal compliance.

- OIDC/OAuth-compatible authentication; reviewed token lifecycle, MFA and account recovery.
- Server-side authorization combining tenant membership with hospital/branch/department/resource scope.
- Isolation across API, relationships, caches, documents, jobs, messages, search, exports, reporting and support access.
- Data minimization, purpose limitation, consent/guardian policy where applicable, correction, retention, export and deletion governance.
- Auditable sensitive-record access and privileged operations.
- Encryption in transit/at rest, managed secrets, key rotation and secure backups.
- Input validation, safe errors, rate/abuse limits, idempotency and secure session handling.
- Dependency/container scanning, least-privilege CI, protected branches and reviewed releases.
- Clinical safety review for triage/templates/decision support; no autonomous clinical decisions in MVP.
- Define availability, RPO/RTO, performance, retention, support and incident-response targets with pilot customers.
- Qualified privacy/legal/security reviewers must assess applicable Indian obligations (including relevant DPDP and ABDM requirements) before production claims or integrations. This document is not legal advice or certification.

Use shared observability for service health, logs, metrics, traces and correlation. Dashboards/alerts should cover availability, dependency health, latency, failures, saturation and critical workflow outcomes with safe dimensions and redaction.

## 10. Success measures and MVP acceptance

Baseline and agree targets with pilot hospitals:
- Registration time and duplicate-record rate.
- Appointment success, cancellations/no-shows and queue accuracy.
- Arrival-to-consultation wait time.
- OPD completion and required documentation completeness.
- Billing/receipt reconciliation when in scope.
- Authorization denials, cross-tenant test coverage and audit completeness.
- Availability, p95/p99 latency, error rate and recovery evidence.
- User adoption, support volume and workflow defect rate.

MVP acceptance requires: documented tenant provisioning; authenticated organization-scoped access; demonstrable registration/search/appointment/check-in/queue/OPD/follow-up; negative cross-tenant tests; approved audit evidence; passing build/tests/format/security checks/CI; rehearsed migrations, backup/restore and rollback; hospital acceptance; and completed privacy/security/clinical/legal review for the pilot use.

## 11. Open product decisions

| Decision | Recommended starting point | Must validate |
|---|---|---|
| First customer segment | Small/mid-sized hospital OPD | Pilot discovery |
| Tenant definition | One customer relationship; single hospital or group | Commercial/legal model |
| Patient master | Tenant-owned; facility associations within tenant | Mobility, sharing, consent |
| MRN/UHID | Tenant-level default; facility aliases if needed | Customer identity policy |
| First workflow | OPD registration to follow-up | Front desk and clinician workshops |
| Billing in MVP | Minimal only if pilot needs it | Cashier/finance workflow |
| Identity provider | OIDC with trusted membership | Provider, MFA, SSO, lifecycle |
| Platform operator | Separate trusted control-plane identity | Operating model |
| Languages | English baseline, localized readiness | Hindi/Marathi and pilot needs |
| Deployment | Shared SaaS candidate; dedicated option later | Customer/data/hosting needs |
| Clinical templates | Configurable, controlled defaults | Qualified clinicians |
| Integrations | Contract-first, selected per pilot | ABDM/FHIR/HL7/payments/messaging |

## 12. Execution sequence

1. Validate this blueprint and decisions that affect data ownership and clinical workflow.
2. Approve tenant/organization/patient boundary and shared observability ADRs.
3. Implement tenant registry, lifecycle and trusted provisioning in CompileCare.Api.

**Implementation status (2026-11-24):** The global tenant registry and lifecycle controls are merged in CompileCare.Api PR #12. Trusted initial tenant membership provisioning and tenant/hospital/department organization scopes are merged in PR #13. The facility hierarchy, PostgreSQL/EF Core persistence, OIDC/JWT membership authorization and metadata-only audit are also implemented. These provide a platform foundation, not a complete hospital workflow.
4. Membership/role provisioning and organization-scope authorization are implemented; customer onboarding and production operations require further validation.
5. Patient master, hospital registration/search and hospital-local MRN policy/assignment are merged in patient master PR #14 and MRN PR #15. The conservative advisory duplicate-candidate API is the current implementation increment; see ADR-010.
6. Implement richer identity matching, authorized cross-hospital discovery, guardian relationships, emergency registration reconciliation and governed identity merge/correction.
7. Implement appointments, check-in and queue lifecycle.
8. Implement OPD encounter and follow-up.
9. Add minimal billing only as confirmed by pilot requirements.
10. Deliver matching web experience against versioned API contracts.
11. Validate pilot operations, security, privacy, clinical workflows, backups, observability and release readiness.

Each step gets requirements, workflow, acceptance criteria, module design, tests and a focused PR. Repository restructuring/new repositories are considered only after module ownership and deployment needs are known.

## 13. Governance

Distinguish implemented facts, accepted decisions, proposals, assumptions and open questions. Hospital stakeholders validate workflows; technical owners approve architecture/security. Requirements follow the [Requirements Lifecycle](requirements-lifecycle.md). Update MVP scope, module catalogue, workflows, roadmap, security/NFR documents and API contracts as decisions are approved. Never claim compliance, production readiness or phase completion from merged code or green CI alone.
