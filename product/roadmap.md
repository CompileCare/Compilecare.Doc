# CompileCare Delivery Roadmap

**Status:** Working product-first delivery plan, updated 2026-11-25. This is a milestone order, not a calendar promise. Exit criteria must pass before a phase is called complete.

CompileCare is a hospital and healthcare information/management platform. This roadmap contains only hospital-system capabilities. The [Product Blueprint v1](blueprint-v1.md) is the source for product scope and patient workflows.

## Phase 1 — Backend foundation

**Status: Completed and merged**

1. API foundation: ASP.NET Core/.NET 10 host, middleware, tenant-context plumbing, health checks, versioned API structure and initial endpoint foundation.
2. Shared observability: structured logging, tracing, metrics, correlation IDs and health instrumentation through CompileCare.Observability.

**Evidence:** CompileCare.Api and CompileCare.Observability repositories have merged implementations. The API currently targets .NET 10 and consumes the shared Observability packages.

## Phase 2 — Developer platform

**Status: Completed and merged — shared client, database operations and caching baseline**

CompileCare.ApiClient provides named/typed client registration, validated options, injectable access-token/tenant/correlation providers, outbound handlers, bounded safe-method resilience and RFC 9457 Problem Details parsing. The correlation header handler validates provider values and replaces untrusted/unsafe caller values.

CompileCare.Database establishes PostgreSQL/EF migration ownership, deployment/release, backup/restore and operations runbooks. CompileCare.Caching has a .NET 10 HybridCache package with optional Redis, validated configuration, explicit tenant/global cache-key namespaces and a tested sample (Caching PR #2; restore/build/test/pack passed).

**Follow-up release hardening:** publish/inspect packages and validate them from consuming applications; expand OpenAPI response annotations and contract validation as modules are introduced. Database backup automation and production recovery drills remain.

## Phase 3 — Database, identity and security

**Status: Foundation and identity-context milestones implemented and merged — persistence, OIDC/tenant RBAC, request audit, tenant registry/lifecycle, trusted membership provisioning, organization scopes and current-user tenant discovery are implemented. Audit and production operational hardening continue in parallel with Phase 5.**

### Implemented foundation

1. PostgreSQL is the active provider in CompileCare.Api; EF Core 10 is the persistence and migration mechanism.
2. Tenant-aware facility persistence, query filters, scoped uniqueness, composite tenant foreign keys, restrictive deletes, audit timestamps and concurrency foundations are implemented.
3. The InitialFacilityHierarchy migration and model snapshot cover Hospital → Department → Ward → Room → Bed.
4. Integration fixtures apply versioned migrations to an isolated test database; CI checks model drift.
5. OIDC/JWT Bearer authentication validates signature, issuer, audience, lifetime, expiry and subject. The validated issuer and OIDC subject are used as the external identity key.
6. Tenant-scoped routes resolve an active membership matching the validated identity and tenant; a tenant header alone grants no access.
7. Facility permissions are derived from persisted membership roles; caller-supplied role/permission claims cannot elevate privileges.
8. Append-only tenant-scoped request audit events and PostgreSQL-backed tests are merged.
9. Global tenant registry and lifecycle are implemented. Tenants have unique normalized slugs and Provisioning/Active/Suspended/Closed states.
10. Trusted initial administrator provisioning and tenant/hospital/department membership scopes are implemented, including protections against self-modification and removing the last active tenant-wide HospitalAdmin (API PR #13).

### Remaining work — recommended order

1. **Audit hardening:** API PR #24 (merge commit `700785110f26ce17f2aee5dc456ec8965a37668f`) covers anonymous access denial and malformed tenant-context rejection. API PR #25 (merge commit `ffb5c128932de12356b9e907aab5f8414477c7c0`) covers privileged initial-administrator provisioning. API PR #26 (merge commit `dbf0f0cc3af5da90b29255fbca37fbb419b78134`) adds audited current-user tenant-context discovery by validated issuer/subject, with active membership and tenant filtering. Continue with membership role/scope updates, sensitive record access, exports, privileged support and material clinical/business actions. Define controlled audit query/export, retention and operational access policies.
2. Extend tenant isolation and authorization tests to background jobs, messaging, caches, file/object storage, exports, search, support tools and privileged operations.
3. Rehearse migration install/upgrade and production migration deployment. Complete backup/restore automation and recovery drills; define signing-key rotation, secrets, encryption, retention/deletion, incident response and access reviews.
4. Complete threat modelling, suitable penetration testing and qualified privacy/legal review for applicable Indian requirements. Do not claim regulatory compliance from implementation or documentation alone.

**Phase 3 exit criteria:** reviewed migration install/upgrade paths; authenticated identity bound to active trusted tenant membership; organization-scope authorization at relevant tenant entry points; append-only audit coverage for approved sensitive/privileged actions; cross-tenant tests for synchronous and asynchronous paths; and reviewed production operational controls. Core security and membership capabilities are implemented; operational and cross-cutting validation remains before calling this phase production-ready.

## Phase 4 — Core healthcare domain

**Status: Partially implemented — patient identity matching and the initial staff/provider management foundation are implemented and merged; patient contact enrichment is designed separately.**

### Implemented

1. Hospital → Department → Ward → Room → Bed hierarchy with CRUD, bounded pagination, tenant-scoped relationships and persistence constraints.
2. Tenant-owned patient master and hospital registration, with hospital-scoped search, authorization and PostgreSQL-backed tests (API PR #14).
3. Hospital-configured MRN policy and concurrency-safe, idempotent MRN assignment/lookup (API PR #15). Patient GUID remains the internal identity; MRN is hospital-local.
4. Required patient registration and MRN workflows are documented in ADR-008/009 and the patient-registration workflow.

### Remaining delivery order

1. Complete the validated hospital group/branch onboarding model; tenant registry, lifecycle, trusted initial membership provisioning and organization scopes are implemented, while customer-specific onboarding and operational controls remain.
2. **Patient identity — normalized candidate matching completed; contact design is next.** Patient master, hospital registration, hospital-local MRN and advisory duplicate matching are merged through API PRs #14–#17. The normalized matcher is on API main at c603dc602565b69f25b24b4738e45f24c4090362; post-merge build, EF model-drift check, 12 unit tests and 58 PostgreSQL integration tests passed. Patient-owned phone/email contact lifecycle, protected storage, verification, permissions and matching limits are designed in ADR-012, merged through documentation PR #19. No contact API/schema/runtime implementation is included. Implementation remains gated on approved security, privacy, key-management and retention decisions; contact corroboration may only be considered later for existing name/DOB candidates. Cross-hospital discovery, guardian relationships, emergency reconciliation and governed identity workflows remain separate.
3. **Staff and provider management — initial foundation merged:** tenant-owned profiles, effective-dated hospital/department assignments, bounded staff search and scoped staff permissions are implemented in CompileCare.Api PR #18. Follow [ADR-013](../architecture/decisions/ADR-013-staff-and-provider-profile-boundaries.md) and the [staff/provider workflow](workflows/staff-and-provider-management.md). Profile-to-membership linking, credential verification and clinical privileging are deliberately not included; profiles and designations do not independently grant access or clinical authority.
4. **Appointments, walk-ins, check-in and queue — implemented and merged:** CompileCare.Api PR #20 delivers appointment rescheduling with persisted lineage, idempotent check-in, walk-in registration, department/day queue tokens and queue lifecycle transitions. CI, EF model snapshot validation and PostgreSQL Docker integration verification passed before squash merge (commit `c968d9441affaa21070063081b2bc785c848e547`). Follow [ADR-014](../architecture/decisions/ADR-014-appointment-scheduling-and-queue-boundaries.md) and the [appointments and queue workflow](workflows/appointments-and-queue.md).
5. Patient-owned contact storage and verification following [ADR-012](../architecture/decisions/ADR-012-verified-patient-contact-data-and-matching.md), only after security/privacy/provider/key-management and retention decisions are approved. Initially allow only corroboration of existing exact-DOB/name candidates.
6. **OPD encounter and clinical documentation — implemented and merged:** CompileCare.Api PR #21 adds queue-linked encounters, scoped ClinicalUser permissions, draft notes, signed-note immutability, append-only amendments and a bounded lifecycle. CI, EF model-drift validation, PostgreSQL integration tests and Docker Integration Verification passed before squash merge (commit `723a107c0de40e95cf28f1c12a772f1ab2a75c73`). Follow [ADR-015](../architecture/decisions/ADR-015-opd-encounter-and-clinical-documentation-boundaries.md) and the [OPD encounter workflow](workflows/opd-encounter-and-clinical-documentation.md). Production activation remains gated on clinician, privacy, role and retention validation.
7. **Minimal OPD billing — API and Web pilot foundation implemented and merged:** CompileCare.Api PR #23 delivered the initial catalogue, visit-linked invoice, payments, receipts and daily collection report. API PR #39 added explicit `Issued`, `PartiallyPaid` and `Paid` settlement states with a PostgreSQL migration; API PR #40 added scoped invoice-by-visit lookup; API PR #41 added regression coverage for duplicate issuance, department isolation and concurrent payment overage. CompileCare.Web PR #10 delivers the permission-driven billing workspace for visit selection, invoice issue/resume, payment capture, receipt retrieval, catalogue administration and daily collection. CI and Docker verification passed. Statutory tax, discounts/waivers, refunds/credit notes, insurance/TPA, gateway/accounting integrations, deposits/advances and advanced cashier reconciliation remain deferred pending pilot decisions. Follow [OPD billing foundation](workflows/opd-billing-foundation.md).
8. Pharmacy, diagnostics, inpatient, inventory and advanced revenue workflows in the approved roadmap sequence.

### Deferred workforce, credentialing and HR roadmap — retained for later

These capabilities are intentionally outside the initial staff-profile increment and the first outpatient MVP. They remain planned product areas and must not be dropped from future planning:

1. **Workforce identity:** approved staff-profile-to-membership linking and the employee self-service foundation. A profile is not an account; linking requires a verified identity and an attributable approval trail.
2. **Credentialing and clinical privileging:** qualification and credential evidence, issuing-body verification, licence validity/expiry, renewal and facility-specific privileges. Provider designation alone never authorizes clinical practice; clinical permissions must depend on verified credentials and active privileges.
3. **Workforce operations:** rosters, shifts, attendance, leave and related approval workflows, designed as explicit workforce capabilities rather than extra staff-profile fields.
4. **HR and finance:** compensation, payroll, payslips and payroll integration. Prefer integration with established payroll providers before building a full payroll engine unless pilot requirements justify otherwise.
5. **Secure employee records:** sensitive identifiers and employment documents in a dedicated, encrypted and access-restricted employee-record/document capability, with purpose limitation, audit and retention controls. Do not expose these fields in ordinary staff APIs or telemetry.
6. **Cross-tenant workforce:** governed multi-hospital/cross-tenant staff sharing and assignments only after ownership, consent, legal basis, authorization, audit and tenant-isolation rules are explicitly approved.

**Provisional pilot defaults to revisit with the hospital:** collect only operationally necessary profile fields (staff code, name, designation and only the contact details required for the workflow); permit concurrent assignments only where explicitly allowed by policy and validate conflicting effective dates; keep profile create/update/deactivate permissions with HospitalAdmin initially and make account linking a separately restricted action; prefer a tenant-configurable designation catalogue; require verified identity plus an auditable approval process before membership linking; and confirm applicable Indian legal requirements and the pilot hospital's retention policy before collecting employment records or sensitive identifiers. These are planning defaults, not legal advice or final product approvals.

**Next engineering hardening increment:** extend and verify audit and tenant/organization-scope coverage for clinical record access and material clinical actions. This is a Phase 3 production-readiness gate and can proceed while billing requirements remain unvalidated.

Each domain workflow requires hospital-user validation, permissions, audit, tenant scope, acceptance criteria, migration and automated tests.

## Phase 5 — User experience

**Status: In progress — authenticated reception, patient registration/MRN, appointment booking, walk-in arrival, queue operations, OPD clinical workspace, permission-driven feature gating, OIDC silent renewal and the initial billing workspace are implemented and merged.**

The first UX foundation increment is documented in [Reception and clinical workspace UX](workflows/reception-and-clinical-workspace-ux.md). It maps reception, queue and clinical tasks to the implemented patient, appointment, queue and OPD contracts; defines screen responsibilities, access/data boundaries, accessibility and failure states; and keeps billing, patient self-service and unimplemented clinical capabilities out of the first slice.

The local developer environment is maintained in `CompileCare.Local`. Its setup and troubleshooting are documented in [Local development setup](../engineering/local-development-setup.md). The first-run platform bootstrap and hospital onboarding increments are under implementation: API PR #32 adds the Development-only one-time persisted platform-operator bootstrap; Local PR #4 restores explicit profile/email OIDC scopes and corrects sibling build contexts; Web PR #8 adds the guided first tenant, initial administrator, activation, hospital and department workflow. These changes must pass CI/Docker verification and then be exercised on the developer Mac before local acceptance is declared complete.

The organization has a dedicated `CompileCare.Web` repository, separate from the public-facing `CompileCare.Site`. The authenticated React + TypeScript workspace shell is implemented and merged (Web PR #1, merge commit `af37ecba2d8224474964f9052b394cb9dca6fd6b`). The first reception vertical slice is also implemented and merged (Web PR #2, merge commit `c83a5583d81be91e5f123d836aef5f8d51cad067`): it consumes the current-user tenant-context contract, allows explicit tenant and API-filtered hospital selection, and supports hospital-scoped patient name search with pagination. The API context contract is merged in CompileCare.Api PR #26. Patient results remain in component memory, requests use no-store, and the API revalidates every permission and scope. Patient registration with a hospital-local duplicate-candidate review and MRN assignment is merged in [CompileCare.Web PR #3](https://github.com/CompileCare/CompileCare.Web/pull/3), with paired documentation in [Compilecare.Doc PR #30](https://github.com/CompileCare/Compilecare.Doc/pull/30). The UI uses the existing patient registration, duplicate-candidate and identifier APIs; it does not add contact capture, patient merge or cross-hospital discovery. The appointment and arrival vertical slice is merged in [CompileCare.Web PR #4](https://github.com/CompileCare/CompileCare.Web/pull/4), with paired documentation in [Compilecare.Doc PR #31](https://github.com/CompileCare/Compilecare.Doc/pull/31). It delivers selectable patient results, server-published availability and booking, walk-in registration, check-in confirmation with server-issued queue token, and department queue visibility. The queue operations UI is merged in [CompileCare.Web PR #5](https://github.com/CompileCare/CompileCare.Web/pull/5), with paired workflow and screenshot evidence documentation in [Compilecare.Doc PR #32](https://github.com/CompileCare/Compilecare.Doc/pull/32). It provides authorized call, recall, start, complete, cancel and no-show actions using the server-side queue transition contract. The OPD clinical workspace is merged in CompileCare.Web PR #9. It provides clinical queue discovery, queue-linked encounter creation/resume, draft note entry, signing, append-only amendments, cancellation and closure using the existing clinical APIs. API PRs #35–#38 added the related Reception/HospitalAdmin permission corrections, department-scoped patient search and detail reads, and server-derived tenant permission discovery. Web PR #9 uses those permission identifiers for module/action visibility and enables OIDC silent session renewal; the API remains the authorization boundary. ClinicalUser receives patient.identity.read to verify only the patient linked to an encounter without granting general patient search, plus scoped facility/queue read permissions to discover departments and operational tokens; it does not receive queue mutation permissions. Follow the [CompileCare experience quality bar](design/experience-quality-bar.md) for visual hierarchy, accessibility, responsive behavior and operational feedback. Continue Phase 3 audit/tenant-scope and production-operational hardening in parallel as a release-readiness track. Mobile remains deferred until validated use cases and identity/API contracts justify it.

### Latest Phase 5 delivery and next increment — 2026-11-25

The initial OPD vertical slice now includes reception and patient registration/MRN, appointments and queue operations, department-scoped clinical identity/encounter access, permission-driven clinical/reception modules, OIDC session renewal and the OPD billing workspace. Current delivery evidence: CompileCare.Api PRs #35–#41, CompileCare.Web PRs #9–#10, and paired workflow documentation. API PR #41 adds PostgreSQL-backed duplicate-invoice, cross-department invoice-read and concurrent-payment regression tests; CI #753 and Docker Integration Verification #156 passed on the same commit.

**Next recommended increment:** complete an end-to-end operational verification pass across Reception → Appointment/Walk-in → Queue → Clinical Encounter → Billing → Receipt, then implement payment/receipt history so a cashier can re-open and reprint an existing receipt after the initial payment response. Do not start GST/tax, refunds, credit notes, TPA, gateway or accounting integration until pilot policy decisions are recorded. Phase 3 audit, tenant-scope, migration/backup and production operational hardening remains a parallel release-readiness track.

## Release gates and working rules

- Product workflow and ownership precede module/repository implementation.
- Work in short-lived branches and focused PRs; do not commit directly to main.
- Review root causes and existing architecture before changing implementation.
- Every PR includes related docs, tests, security/privacy implications, migration/recovery notes and CI evidence.
- A phase is complete only when its exit criteria are met and reviewed.
- Keep the modular core simple; introduce separately deployed services only when documented needs justify them.
