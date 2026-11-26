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
| Scheduling | Date-specific provider availability, appointment booking/cancellation/rescheduling | MVP — next delivery increment |
| Front desk & queue | Walk-ins, arrival, check-in, FIFO tokens and queue worklists | MVP — next delivery increment |
| Clinical/OPD | Encounters, notes, observations, diagnoses, care plan and orders | MVP foundation |
| Revenue cycle | Tariffs, charges, invoices, receipts, payments and refunds | Minimal pilot scope |
| Diagnostics | Laboratory and radiology workflows/results | Later |
| Pharmacy | Medication catalogue, prescription validation, dispensing and stock | Later |
| Inpatient | Admission, transfer, bed occupancy, nursing and discharge | Later |
| Inventory/supply chain | Stores, batches, expiry, procurement and movement | Later |
| Workforce | Staff/provider profiles and assignments; later identity linking, credentialing, rosters, attendance, leave and HR | Initial profile foundation merged; broader capabilities deferred |
| Notifications | Templates, delivery, preferences, consent and retries | Supporting |
| Documents | Metadata, secure storage, access and retention | Supporting |
| Reporting/analytics | Permissioned operational/financial reporting and exports | Later/basic MVP views |
| Integrations | Identity, payments, messaging, FHIR/HL7, ABDM, DICOM | Validate per pilot |
| Audit/compliance | Security/business audit and controlled audit access | Cross-cutting |

### Workforce, credentialing and HR — retained future scope

The initial staff/provider profile and assignment foundation is implemented. The following capabilities remain planned, but are not part of the first staff increment or initial OPD MVP:

1. Workforce identity: verified staff-profile-to-membership linking and employee self-service.
2. Credentialing: qualification and licence evidence, verification, expiry/renewal and facility-specific clinical privileges. Provider designation is descriptive and never grants clinical authority.
3. Workforce operations: rostering, shifts, attendance and leave with explicit policies and approval workflows.
4. HR/finance: compensation, payroll, payslips and payroll-provider integrations; prefer integration before building a full payroll engine.
5. Secure employee records: sensitive identifiers and employment documents behind dedicated encryption, least-privilege access, audit and retention controls.
6. Cross-tenant workforce: multi-hospital sharing only after legal basis, consent, ownership, authorization and tenant-isolation decisions are approved.

**Provisional pilot defaults (revisit before the relevant milestone):** keep the profile to staff code, name, designation and only operationally necessary contact details; allow concurrent assignments only where policy permits and prevent conflicting effective-date combinations; keep create/update/deactivate with HospitalAdmin initially and treat membership linking as a separately restricted operation; prefer a tenant-configurable designation catalogue; require verified identity and an auditable approval before linking a profile to a login; and confirm applicable Indian requirements plus the pilot hospital's retention policy before collecting employment records or sensitive identifiers. These are product-planning assumptions, not legal advice or final approvals.

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
5. Patient master, hospital registration/search, hospital-local MRN policy/assignment and the normalized advisory duplicate-candidate API are merged in API PRs #14–#17. ADR-010 records the first pass; ADR-011 records the merged normalized name matcher. ADR-012 defines the next optional verified-contact design, not yet implemented.
6. **Staff/provider foundation implemented:** tenant-owned profiles, effective-dated hospital/department assignments and scoped staff permissions are merged in CompileCare.Api PR #18, following [ADR-013](../architecture/decisions/ADR-013-staff-and-provider-profile-boundaries.md) and the [staff/provider workflow](workflows/staff-and-provider-management.md). Profile-to-membership linking remains deferred. A profile or provider designation must not itself grant access or establish credentialing/clinical authority.
7. Implement appointments, walk-ins, check-in and queue lifecycle following [ADR-014](../architecture/decisions/ADR-014-appointment-scheduling-and-queue-boundaries.md) and [the appointment/queue workflow](workflows/appointments-and-queue.md). Keep availability, appointment, queue and encounter ownership separate.
8. Complete patient-owned contact capture and verification only under the reviewed data-protection design in [ADR-012](../architecture/decisions/ADR-012-verified-patient-contact-data-and-matching.md). Keep verified contacts as corroboration for existing name/DOB candidates initially; any contact-only candidate rule needs a separate ADR and pilot validation. Cross-hospital discovery, guardian relationships, emergency registration reconciliation and governed identity merge/correction remain separate and gated on approved privacy/security decisions.
9. Implement OPD encounter and follow-up.
10. Add minimal billing only as confirmed by pilot requirements.
11. Deliver matching web experience against versioned API contracts.
12. Validate pilot operations, security, privacy, clinical workflows, backups, observability and release readiness.

13. Deliver the deferred workforce roadmap in dependency order: verified identity linking and employee self-service; credential verification/clinical privileging before granting clinical authority; rostering/attendance/leave; compensation/payroll integrations; secure employee records; and governed cross-tenant workforce sharing last.

Each step gets requirements, workflow, acceptance criteria, module design, tests and a focused PR. Repository restructuring/new repositories are considered only after module ownership and deployment needs are known.

## 13. Governance

Distinguish implemented facts, accepted decisions, proposals, assumptions and open questions. Hospital stakeholders validate workflows; technical owners approve architecture/security. Requirements follow the [Requirements Lifecycle](requirements-lifecycle.md). Update MVP scope, module catalogue, workflows, roadmap, security/NFR documents and API contracts as decisions are approved. Never claim compliance, production readiness or phase completion from merged code or green CI alone.


## 14. Delivery roadmap — implemented foundation, MVP completion and future scope

This section makes the long-term hospital product plan explicit. A capability listed as planned is not an assertion that it is implemented. Current delivery status must be checked against repository code, merged PRs, CI and authenticated local acceptance.

### 14.1 Immediate stabilization and pilot-readiness work

Before expanding the clinical domain, complete the following gates:

1. **Regression baseline:** run the full local Docker stack, API and Web automated suites, authenticated Keycloak workflows, PostgreSQL persistence checks, tenant-isolation negative tests and observability end-to-end checks. Use synthetic data only.
2. **Defect closure:** investigate every failing or skipped test; fix root causes without weakening assertions, authorization or security controls. Record PASS, FAIL, BLOCKED and NOT TESTED distinctly.
3. **Security and operations:** complete audit coverage/retention governance, secret and signing-key lifecycle, encryption review, backup/PITR and restore rehearsal, migration/rollback runbooks, rate/abuse controls, dependency/container scanning and incident response.
4. **Real acceptance:** obtain role-based hospital stakeholder acceptance and qualified clinical, privacy, legal and security review before any production-readiness claim.
5. **Observability:** verify a real CompileCare.Api request produces safely redacted logs, metrics and traces through the shared OpenTelemetry pipeline into the local/target backends. Telemetry must not contain PHI, secrets or high-cardinality patient identifiers.

### 14.2 Web information architecture and application experience

The Web application must evolve from a single workspace composition into a route-based, modular hospital application. The current single-workspace/hash-section implementation is an interim delivery state, not the target information architecture.

**Target principles**
- Use a shared authenticated application shell with independent route-level workspaces and feature-owned components.
- Give each module a stable URL, page heading, navigation identity, loading/empty/error states and focused tests.
- Keep tenant, hospital and applicable department context visible and consistent.
- Build navigation from the authenticated membership's effective permissions and scope. Route guards improve user experience; server-side API authorization remains authoritative.
- Preserve deep links, browser history, refresh behavior, safe session expiry and unauthorized/not-found handling.
- Keep the platform control plane separate from tenant hospital workspaces.
- Avoid a single oversized application component. Decompose incrementally by feature and route; do not perform an untested big-bang rewrite.
- Meet responsive desktop/tablet/mobile requirements and target WCAG 2.2 AA, with evidence before claiming conformance.
- Capture real authenticated screenshots using synthetic data. Mocked component/Playwright screenshots are not proof of local authenticated acceptance.

**Proposed route map** (final route names may be refined during implementation; preserve backwards-compatible redirects where useful):

| Workspace | Proposed route |
|---|---|
| Hospital overview | `/app/overview` |
| Reception and patient search | `/app/reception` |
| Patient registration | `/app/patients/register` |
| Patient record (permission-controlled) | `/app/patients/:patientId` |
| Appointment management | `/app/appointments` |
| Queue operations | `/app/queue` |
| Clinical/OPD | `/app/clinical` |
| Billing | `/app/billing` |
| Facilities | `/app/facilities` |
| Staff and providers | `/app/staff` |
| Membership administration | `/app/memberships` |
| Patient identifier policy | `/app/settings/patient-identifiers` |
| Platform tenant lifecycle | `/platform/tenants` |

The exact module links shown to a user must follow the effective permission/context contract. Never rely on route hiding or client-side checks as an authorization boundary.

**UI implementation sequence**
1. Introduce the shared authenticated shell, route layout, navigation groups, breadcrumbs/page headings and active hospital context.
2. Extract overview and reception/patient registration into independent route workspaces.
3. Extract appointments and queue operations, preserving existing booking, check-in and queue lifecycle behavior.
4. Extract clinical/OPD, preserving identity confirmation, draft/sign/close/amendment safeguards.
5. Extract billing, facilities, staff, memberships and identifier-policy workspaces.
6. Keep platform tenant lifecycle in its separate platform-operator control plane.
7. Add route-level loading/error/forbidden/not-found behavior, keyboard navigation, responsive tests, browser visual regression and authenticated evidence.
8. Remove legacy hash navigation only after redirects/deep links and regression coverage are in place.

This work is a structural and usability improvement to existing capabilities. It does not authorize new clinical or financial features or invented dashboard metrics. See [Enterprise Experience Uplift](design/enterprise-experience-uplift.md).

### 14.3 Future hospital implementation phases — explicitly deferred

The following are retained product scope, not part of the first outpatient MVP. Each requires domain workflows, ownership boundaries, permissions, audit, data lifecycle, integration contracts, tests and hospital validation before implementation.

| Future phase | Scope | Entry dependencies |
|---|---|---|
| IPD / inpatient | Admission, discharge and transfer; bed reservation/occupancy; ward and nursing workspaces; observations, care plans, handover, medication administration, discharge summaries | Facility/bed model, clinical governance, pilot nursing and admission workflows |
| Pharmacy | Formulary/catalogue, prescription validation, dispensing, batch/expiry, stock movements, returns, suppliers and pharmacy settlement | Clinician order lifecycle, pharmacy policy, inventory and financial boundaries |
| LIS | Test catalogue, diagnostic orders, specimen/barcode collection, worklists, processing, result validation, critical-result escalation and reports | Clinical order contracts, lab stakeholder validation, instrument integration requirements |
| RIS/PACS | Imaging orders, scheduling, modality worklists, report approval, DICOM references, image storage/access and retention | Diagnostic order lifecycle, DICOM/PACS architecture, security and storage decisions |
| Inventory and procurement | Stores, requisitions, purchase orders, vendors, receiving, batch/expiry, transfers, stock adjustments, assets and maintenance | Pharmacy/LIS/department consumption requirements and finance integration |
| Advanced revenue cycle | Tax/statutory invoice rules, estimates, discounts/waivers, deposits, refunds/reversals/credit notes, insurance/TPA, preauthorization/claims, gateways, accounting and reconciliation | Pilot finance policy, qualified accounting/tax review, payment and payer contracts |
| Workforce and HR | Verified staff-to-account linking, credential/license evidence and renewal, clinical privileges, rosters, shifts, attendance, leave, employee records, payroll integrations and governed multi-hospital workforce | Identity proofing, clinical governance, HR/legal/privacy decisions |
| Patient engagement | Patient/guardian portal, mobile experience, self-booking, records/instructions access, consent-aware reminders and communication preferences | Patient identity, guardian/consent, security and support model |
| Interoperability | ABDM/ABHA, FHIR/HL7, DICOM, identity, payment, messaging and external hospital systems | Validated use case, partner agreements, applicable approvals, conformance and security testing |
| Analytics and intelligent assistance | Permissioned operational/financial reporting, exports, capacity/wait-time analytics and carefully governed AI-assisted workflows | Data quality, privacy controls, clinical safety review, explainability and human oversight |

**Explicitly deferred until separately justified:** autonomous diagnosis, triage or prescribing; full ERP/HR/payroll engine; broad patient mobile app; multi-region active-active; and microservice extraction without measured deployment, scaling, ownership, security or resilience need.

### 14.4 Recommended dependency order

1. Stabilize and regression-test the current outpatient platform.
2. Deliver the route-based Web application architecture and complete the existing OPD experience.
3. Close pilot security, privacy, audit, observability, backup/recovery and operational readiness gates.
4. Validate the outpatient pilot with hospital stakeholders.
5. Implement IPD and nursing workflows.
6. Implement pharmacy and its inventory boundaries.
7. Implement LIS and RIS/PACS against explicit diagnostic-order contracts.
8. Implement hospital-wide inventory, procurement and asset operations.
9. Extend revenue cycle after finance policy validation.
10. Deliver credentialed workforce/HR capabilities and patient engagement.
11. Add validated interoperability and analytics capabilities.

The sequence may be adjusted by pilot evidence, but dependencies and deferred decisions must remain visible in roadmap updates. Every delivered feature must update its workflow, module catalogue, API contract where applicable, tests and this blueprint. A merged PR or green CI is not, by itself, proof of complete workflow acceptance or production readiness.
