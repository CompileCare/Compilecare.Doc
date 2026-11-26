# CompileCare Delivery Roadmap

**Status:** Working product-first delivery plan, updated 2026-11-26. Phase 5 includes a formal enterprise experience uplift, tracked separately from feature delivery. The 2026-11-25 local full-stack acceptance run is recorded in [Local stack acceptance](qa/local-stack-acceptance-2026-11-25.md). This is a milestone order, not a calendar promise. Exit criteria must pass before a phase is called complete.

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

**Follow-up release hardening:** publish/inspect packages and validate them from consuming applications; expand OpenAPI response annotations and contract validation as modules are introduced. Provider-native scheduled backups/PITR and production recovery drills remain; the repository now includes guarded logical backup/restore helpers and PostgreSQL 16 Docker verification.

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

1. **Audit and tenant-isolation hardening:** API PR #24 (merge commit `700785110f26ce17f2aee5dc456ec8965a37668f`) covers anonymous access denial and malformed tenant-context rejection. API PR #25 (merge commit `ffb5c128932de12356b9e907aab5f8414477c7c0`) covers privileged initial-administrator provisioning. API PR #26 (merge commit `dbf0f0cc3af5da90b29255fbca37fbb419b78134`) adds audited current-user tenant-context discovery by validated issuer/subject, with active membership and tenant filtering. API PR #44 verifies metadata-only audits for clinical and billing actions; PR #45 verifies denied cross-department clinical reads; PR #46 verifies cross-tenant patient-read denial and audit attribution; PR #47 verifies cross-tenant billing confidentiality; PR #48 verifies OPD encounter confidentiality; PR #49 verifies appointment and queue isolation; and PR #50 verifies staff-profile and assignment isolation. API PR #51 adds patient identifier/MRN confidentiality coverage (present on API main at `222c5569fa3019ee62b865cb14e3f3c2fad6978e`; CI #798/#799 and Docker #166 passed). API PR #53 adds hospital-local duplicate-candidate isolation and audit coverage (present on API main at `8dadb4cb1865769d67d4d5e6a81309364c543301`; CI #806/#807 and Docker #169 passed). API PR #54 adds PostgreSQL integration coverage for tenant-membership role/scope updates, deactivation/reactivation and rejected self-demotion, asserting metadata-only audit attribution and unchanged persisted administrator state; it was merged on 2026-11-26 after hosted CI and Docker/PostgreSQL verification passed. API PR #55 addresses local acceptance finding SEC-002 by validating hospital visibility before patient search and department collection listing, with PostgreSQL regression coverage for tenant-wide foreign-hospital and out-of-scope requests; it was merged on 2026-11-26 after .NET CI and Docker Integration Verification passed (runs 45739052864 and 45739052930). API PR #61 and #62 add cross-hospital and same-hospital cross-department OPD denial audit coverage; API PR #64 adds audit attribution for department-scoped patient search/detail denials; API PR #65 adds audit attribution for department-scoped invoice and payment-history denials; API PR #66 adds same-hospital cross-department isolation coverage for all ten appointment/queue operations, including availability, booking, read, cancel, reschedule, check-in, queue listing/transition and walk-in registration, with rejected audit attribution and resource identifiers. API PR #67 adds PostgreSQL failure injection proving patient registration and its hospital registration roll back with HTTP 503 when audit-event persistence fails; CI and Docker verification passed. API PR #68 adds an endpoint-registration guard asserting every authenticated route carries tenant, user or platform-operator audit metadata, including metadata inherited from route groups; Release build, EF model-drift, integration tests and Docker verification passed. API PR #69 strengthens that guard by asserting route discovery is non-empty, preventing a false-positive pass when no endpoint sources are exposed to the test host; Release build and Docker verification passed. Continue with additional sensitive record access and material clinical/business actions. Controlled tenant-wide audit query/export is implemented in API PRs #71/#72; define retention/deletion, export monitoring, operational access reviews and privacy/legal approval before customer activation.
2. Continue tenant isolation and authorization tests across the synchronous routes and persistence paths that exist today. Require the same tenant/organization binding, metadata-only audit and cross-tenant regression tests when background jobs, messaging consumers, cache integrations, file/object storage, export pipelines, search indexes or support tooling are introduced; do not create those subsystems solely to satisfy this checklist.
3. A disposable PostgreSQL 16 CI rehearsal now validates base-to-upgrade, fresh install and idempotent migration-script reapplication (API PR #73); production migration rehearsal against representative data and an approved release process remain. Provider-neutral logical backup/restore helpers and Docker verification are implemented (Database PR #4); provider-native physical backups/PITR, encrypted off-site storage, retention, defined RPO/RTO and scheduled restore drills remain. Define signing-key rotation, production secrets, encryption, retention/deletion, incident response and access reviews.
4. The initial application threat model is recorded in [Threat model](../security/threat-model.md). Validate it with the pilot hospital, complete suitable independent penetration testing and qualified privacy/legal review for applicable Indian requirements. Do not claim regulatory compliance from implementation or documentation alone.

**Phase 3 exit criteria:** reviewed migration install/upgrade paths; authenticated identity bound to active trusted tenant membership; organization-scope authorization at relevant tenant entry points; append-only audit coverage for approved sensitive/privileged actions; cross-tenant tests for synchronous and asynchronous paths; and reviewed production operational controls. Core security and membership capabilities are implemented; operational and cross-cutting validation remains before calling this phase production-ready.

## Phase 4 — Core healthcare domain

**Status: Partially implemented — patient identity matching and the initial staff/provider management foundation are implemented and merged; patient contact enrichment is designed separately.**

### Implemented

1. Hospital → Department → Ward → Room → Bed hierarchy with CRUD, bounded pagination, tenant-scoped relationships and persistence constraints. Backend CRUD and the permission-aware Web administration workspace are implemented and merged. Web PR #34 merged as `c8a5a5ec9a31e85cab05d60d05e8852616319be3`; final hosted CI runs 45744960504 and 45744951178 passed 67 Vitest tests, all 14 Chromium E2E tests, production build and Docker smoke. Authenticated local-stack acceptance remains pending.
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
7. **Minimal OPD billing — API and Web pilot foundation implemented and merged:** CompileCare.Api PR #23 delivered the initial catalogue, visit-linked invoice, payments, receipts and daily collection report. API PR #39 added explicit `Issued`, `PartiallyPaid` and `Paid` settlement states with a PostgreSQL migration; API PR #40 added scoped invoice-by-visit lookup; API PR #41 added regression coverage for duplicate issuance, department isolation and concurrent payment overage; API PR #43 added department-scoped payment history for receipt recovery. Web PRs #10–#11 deliver the permission-driven billing workspace, payment history, receipt re-open and print. Web PR #12–#13 add browser E2E coverage for billing recovery and the full walk-in → clinical → billing journey. API PR #44 verifies metadata-only audit events for clinical/billing actions and confirms no clinical note or payment reference is stored. API CI #767/Docker #158, API CI #771/Docker #159, Web CI #154, Web CI #168 and Web CI #175 passed. Statutory tax, discounts/waivers, refunds/credit notes, insurance/TPA, gateway/accounting integrations, deposits/advances and advanced cashier reconciliation remain deferred pending pilot decisions. Follow [OPD billing foundation](workflows/opd-billing-foundation.md).
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

**Next engineering hardening increment:** extend audit and tenant/organization-scope verification to additional sensitive clinical record reads and material clinical/business actions. API PR #61 covers cross-hospital OPD encounter denials; PR #62 covers same-hospital cross-department encounter read/update/sign/create denials; PR #64 covers department-scoped patient list/detail denials; PR #65 covers department-scoped invoice and payment-history denials; PR #66 covers same-hospital cross-department appointment and queue operations. PR #68 verifies all authenticated routes are included in the audit middleware contract. These tests validate actor/tenant audit attribution. This remains a Phase 3 production-readiness gate; controlled audit query/export, retention and operational access policies are still outstanding. Proposed guardrails and implementation gates are documented in [ADR-016: Audit review access, export and retention governance](../architecture/decisions/ADR-016-audit-review-access-and-retention-governance.md); hospital/compliance validation is required before customer-facing audit browsing or export is implemented.

Each domain workflow requires hospital-user validation, permissions, audit, tenant scope, acceptance criteria, migration and automated tests.

## Phase 5 — User experience

**Status: In progress — authenticated reception, patient registration/MRN, appointment booking, walk-in arrival, queue operations, OPD clinical workspace, permission-driven feature gating, OIDC silent renewal and the initial billing workspace are implemented and merged. Browser E2E now covers billing recovery, the full walk-in-to-billing path, appointment booking/check-in and successful OIDC silent renewal.**

The first UX foundation increment is documented in [Reception and clinical workspace UX](workflows/reception-and-clinical-workspace-ux.md). It maps reception, queue and clinical tasks to the implemented patient, appointment, queue and OPD contracts; defines screen responsibilities, access/data boundaries, accessibility and failure states. Billing is now included in the delivered OPD slice; patient self-service and unimplemented clinical capabilities remain deferred.

The local developer environment is maintained in `CompileCare.Local`. Its setup and troubleshooting are documented in [Local development setup](../engineering/local-development-setup.md). The first-run platform bootstrap and hospital onboarding changes are merged: API PR #32 adds the Development-only one-time persisted platform-operator bootstrap; Local PR #4 restores explicit profile/email OIDC scopes and corrects sibling build contexts; Web PR #8 adds the guided first tenant, initial administrator, activation, hospital and department workflow. Their CI/PR work is merged; the remaining acceptance item is hands-on Docker/OIDC smoke validation in the developer Mac environment, which cannot be confirmed by GitHub CI alone.

The organization has a dedicated `CompileCare.Web` repository, separate from the public-facing `CompileCare.Site`. The authenticated React + TypeScript workspace shell is implemented and merged (Web PR #1, merge commit `af37ecba2d8224474964f9052b394cb9dca6fd6b`). The first reception vertical slice is also implemented and merged (Web PR #2, merge commit `c83a5583d81be91e5f123d836aef5f8d51cad067`): it consumes the current-user tenant-context contract, allows explicit tenant and API-filtered hospital selection, and supports hospital-scoped patient name search with pagination. The API context contract is merged in CompileCare.Api PR #26. Patient results remain in component memory, requests use no-store, and the API revalidates every permission and scope. Patient registration with a hospital-local duplicate-candidate review and MRN assignment is merged in [CompileCare.Web PR #3](https://github.com/CompileCare/CompileCare.Web/pull/3), with paired documentation in [Compilecare.Doc PR #30](https://github.com/CompileCare/Compilecare.Doc/pull/30). The UI uses the existing patient registration, duplicate-candidate and identifier APIs; it does not add contact capture, patient merge or cross-hospital discovery. The appointment and arrival vertical slice is merged in [CompileCare.Web PR #4](https://github.com/CompileCare/CompileCare.Web/pull/4), with paired documentation in [Compilecare.Doc PR #31](https://github.com/CompileCare/Compilecare.Doc/pull/31). It delivers selectable patient results, server-published availability and booking, walk-in registration, check-in confirmation with server-issued queue token, and department queue visibility. The queue operations UI is merged in [CompileCare.Web PR #5](https://github.com/CompileCare/CompileCare.Web/pull/5), with paired workflow and screenshot evidence documentation in [Compilecare.Doc PR #32](https://github.com/CompileCare/Compilecare.Doc/pull/32). It provides authorized call, recall, start, complete, cancel and no-show actions using the server-side queue transition contract. The OPD clinical workspace is merged in CompileCare.Web PR #9. It provides clinical queue discovery, queue-linked encounter creation/resume, draft note entry, signing, append-only amendments, cancellation and closure using the existing clinical APIs. API PRs #35–#38 added the related Reception/HospitalAdmin permission corrections, department-scoped patient search and detail reads, and server-derived tenant permission discovery. Web PR #9 uses those permission identifiers for module/action visibility and enables OIDC silent session renewal; the API remains the authorization boundary. ClinicalUser receives patient.identity.read to verify only the patient linked to an encounter without granting general patient search, plus scoped facility/queue read permissions to discover departments and operational tokens; it does not receive queue mutation permissions. Follow the [CompileCare experience quality bar](design/experience-quality-bar.md) for visual hierarchy, accessibility, responsive behavior and operational feedback. Continue Phase 3 audit/tenant-scope and production-operational hardening in parallel as a release-readiness track. Mobile remains deferred until validated use cases and identity/API contracts justify it.

### Latest Phase 5 delivery and next increment — 2026-11-25

The initial OPD vertical slice now includes reception and patient registration/MRN, appointments and queue operations, department-scoped clinical identity/encounter access, permission-driven feature gating, OIDC session renewal and the OPD billing workspace. Current delivery evidence: CompileCare.Api PRs #35–#53 and CompileCare.Web PRs #9–#26, with paired workflow documentation. API PR #44 verifies metadata-only audit for patient identity reads, encounter creation/note/sign/close and payment capture while excluding note text/payment references (CI #771, Docker #159); PR #45 verifies denied cross-department clinical reads are audited (CI #775, Docker #160); PR #46 verifies cross-tenant patient-read denial and audit attribution (CI #779, Docker #161); PR #47 verifies cross-tenant billing confidentiality (CI #783, Docker #162); PR #48 verifies OPD encounter confidentiality (CI #786/#787, Docker #163); PR #49 verifies appointment and queue isolation (CI #790/#791, Docker #164); PR #50 verifies staff-profile and assignment isolation (CI #795/#796, Docker #165); PR #51 verifies patient identifier/MRN confidentiality (CI #798/#799, Docker #166); and PR #53 verifies tenant isolation in duplicate-candidate search (CI #806/#807, Docker #169). Web PR #12 tests billing recovery/permission gating (CI #168); PR #13 tests the walk-in → queue → clinical → billing journey (CI #175); PR #14 tests appointment booking/check-in (CI #181); PRs #15–#18 cover successful OIDC renewal, slot-conflict recovery and re-authentication after renewal failure/expiry (CI #185, #191, #195 and #199); PR #19 adds visible keyboard focus, reduced-motion support and narrow-viewport overflow checks (CI #210). Web PRs #20–#26 extend keyboard-only Chromium coverage across reception search, clinical encounter entry, billing visit/invoice/payment/receipt workflows, appointment booking/check-in, queue call/start and patient registration/MRN assignment. The final relevant browser checks passed on Web CI #214, #218, #246, #247, #234, #254 and #257. The cross-tenant MRN and duplicate-search tests are present on API main at commits `222c5569fa3019ee62b865cb14e3f3c2fad6978e` and `8dadb4cb1865769d67d4d5e6a81309364c543301`; the keyboard patient-registration test is present on Web main at `4dc3a991eafe625d80bae887886283aa722fef94`.

**Next recommended increment:** perform the remaining authenticated-workspace accessibility review across Reception, Appointments, Queue, Clinical and Billing: verify end-to-end keyboard completion, focus order, dialog behavior, semantic labels/live announcements, contrast and screen-reader output at desktop, tablet and mobile sizes. PRs #19–#26 provide an automated and keyboard-navigation baseline, not a complete WCAG 2.2 AA conformance claim. In parallel, continue Phase 3 operational hardening: migration deployment/rehearsal, backup/restore automation and recovery drills, signing-key rotation, secret management, encryption, retention/deletion, incident response and access reviews. Require equivalent tenant/organization binding, audit and isolation tests when background jobs, exports, caches or other asynchronous integrations are introduced; do not create those subsystems solely to satisfy a checklist. Exercise the local Docker/OIDC onboarding path on the developer Mac before declaring environment acceptance; GitHub CI cannot replace this environment-specific smoke test. Keep staff/provider UI gated on hospital-user workflow validation; do not invent workforce policy. Do not start GST/tax, refunds, credit notes, TPA, gateway or accounting integration until pilot policy decisions are recorded.


## Active release-readiness work — status 2026-11-26

The local acceptance findings and follow-up PRs have been rechecked. The GitHub Actions runner-allocation issue that previously produced `runner_id=0` and zero-step jobs is resolved. All 11 CompileCare repositories are now public, hosted jobs receive runners and execute actual tests and Docker verification, and issue [CompileCare.Api #56](https://github.com/CompileCare/CompileCare.Api/issues/56) is closed as completed.

### Merged after executed verification

- **CompileCare.Api PR #54 — membership audit coverage:** merged after the PostgreSQL integration suite passed. It verifies membership role/scope updates, deactivation/reactivation, rejected self-demotion and metadata-only audit attribution.
- **CompileCare.Api PR #55 — inaccessible hospital collection contract:** merged as `07fb9cea2dc6f706fc5c1a6fff67850f585b79d9`. CI run 45739052864 passed restore, Release build, EF model-drift validation and unit/integration tests; Docker Integration Verification run 45739052930 passed.
- **CompileCare.Database PR #2 — logical backup/restore:** merged as `97bafc795ddae67168c88462d35ecc9fcfc30e27` after PostgreSQL Docker backup/restore verification run 45738452765 passed. This remains logical backup/restore tooling, not physical backup/PITR or a production recovery drill.
- **CompileCare.Caching PR #5 — tenant cache isolation tests:** merged after the tenant/global namespace isolation workflow passed.
- **CompileCare.Local PR #5 — local Npgsql GSSAPI warning:** merged; the connection-string setting remains scoped to the local Docker configuration.
- **CompileCare.Local PR #7 — Keycloak silent renewal:** merged as `6fc76effa728a36529ddcc25eaf051cab8412068`. Local CI run 45738522025 passed realm validation, Docker Keycloak import smoke test and Compose validation. The workflow also tests reconciliation of an existing client with a stale redirect list without deleting its realm volume/users.
- **CompileCare.Local PR #6 — observability overlay:** merged after Docker runtime smoke runs 45740981547 and 45740976578 passed. The smoke started Collector, Prometheus, Jaeger, Elasticsearch and Kibana, sent synthetic OTLP metrics/traces plus a synthetic log, and verified queryability/readiness. A real CompileCare.Api exporter-to-backend check remains local post-merge acceptance.
- **Compilecare.Doc PR #50:** merged as `dffe467a98854b9d688ac75c45bd1a15fba69faf`, paired with Local PR #5.
- **CompileCare.Web PR #27 — cross-platform OPD browser test fixes:** merged after browser E2E and Docker smoke verification passed.
- **CompileCare.Web PR #28 — local Playwright report configuration:** merged after Web CI passed.
- **CompileCare.Web PR #35 — tenant membership administration:** merged as `b8e4a4ec8864884f5e71dc070a319a5f4df07185`.
- **CompileCare.Web PR #36 — platform tenant lifecycle workspace:** merged as `54589758db04848d25e2e078f3619c1932cc645a`. Hosted CI runs 45747601458 and 45747606834 passed 85 Vitest tests across 9 files, 14 Chromium E2E tests, production build and Docker smoke. The workspace supports platform-only tenant creation, initial HospitalAdmin provisioning, activation, suspension, reactivation and terminal closure. It is visible only to persisted platform operators; the API independently authorizes every global operation.

### Completed Web experience and facility PR stack

All eight Web implementation PRs #29–#36 have been merged into `main` in dependency order, using merge commits to preserve the stacked visual/experience branch history; the tenant and platform administration slices were then merged on top:

- **Web PR #29 — enterprise visual foundation:** merged as `37f30508a23add4a8af0918307a524dd9f9f4bdb`; hosted CI 45738737945 passed Vitest, build, Chromium E2E and Docker smoke.
- **Web PR #30 — Reception patient selection:** merged as `f0e548737ac05a1886640be742ed64a32f892e38`; hosted CI 45738814930 passed.
- **Web PR #31 — appointment slot selection:** merged as `0c10cafacb43183b47bc2f18ea0deadf8a90a95e`; hosted CI 45738824899 passed.
- **Web PR #32 — queue-row accessibility:** merged as `68a8c9798a29df34031e2e13d7857550f887546b`; hosted CI 45738835427 passed.
- **Web PR #33 — clinical queue-row accessibility:** merged as `23cf6dcf503fc4e781e9b343d244ea9665566e29`; hosted CI 45739127780 passed, including the OPD journey browser test and Docker smoke.
- **Web PR #34 — permission-aware Hospital → Department → Ward → Room → Bed workspace:** merged as `c8a5a5ec9a31e85cab05d60d05e8852616319be3`. Final CI runs 45744960504 and 45744951178 passed 67 Vitest tests, 14 Chromium E2E tests, production build and Docker smoke on head `b36c7115003384eee8f809511a0b6808b36e76f2`.
- **Web PR #35 — tenant membership administration:** merged as `b8e4a4ec8864884f5e71dc070a319a5f4df07185` after 75 Vitest tests, 14 Chromium E2E tests, production build and Docker smoke passed.
- **Web PR #36 — platform tenant lifecycle workspace:** merged as `54589758db04848d25e2e078f3619c1932cc645a` after 85 Vitest tests, 14 Chromium E2E tests, production build and Docker smoke passed.

### Antigravity acceptance findings — disposition

- **AUTH-005:** code/configuration correction merged in Local PR #7; hosted Keycloak Docker reconciliation validation passed. Repeat the full local sign-in and token-renewal acceptance on the developer Mac after pulling the merge.
- **SEC-002:** corrected and merged in API PR #55; PostgreSQL integration and Docker verification passed. Repeat the two real-stack acceptance cases on the updated local stack.
- **Mocked macOS Playwright failures:** corrected and merged in Web PR #27 using deterministic native-select selection and platform-specific Select All behavior. Hosted Linux browser E2E passed; this is not a claim that a separate macOS runner was executed.
- **Visual foundation regressions:** PR #29 now tests skip-link-first focus and mobile navigation width; hosted browser and Docker checks passed. Authenticated screenshot evidence remains outstanding.
- **Clinical queue accessible-name mismatch:** corrected in PR #33's OPD E2E selector; hosted browser and Docker checks passed.

The eight Not Run scenarios describe the original tested build, not the current main branch. Facility hierarchy CRUD, tenant membership administration, platform tenant lifecycle, staff/provider UI, MRN policy/search UI and appointment availability/list/reschedule/cancel UI are now implemented and merged. Authenticated local-stack acceptance remains pending. Remaining unimplemented/deferred areas include inpatient ADT/bed occupancy/MAR/discharge; LIS/RIS/PACS and pharmacy; inventory; notifications and patient portal; HR/payroll; and ABDM/TPA/insurance integrations. Deliver these in dependency order only after product, security, privacy and operational requirements are recorded. Do not implement patient contact verification, clinical credentialing, payroll, insurance or other sensitive subsystems merely to eliminate the Not Run count.

## Release gates and working rules

- Product workflow and ownership precede module/repository implementation.
- Work in short-lived branches and focused PRs; do not commit directly to main.
- Review root causes and existing architecture before changing implementation.
- Every PR includes related docs, tests, security/privacy implications, migration/recovery notes and CI evidence.
- A phase is complete only when its exit criteria are met and reviewed.
- Keep the modular core simple; introduce separately deployed services only when documented needs justify them.


## Phase 5 experience quality uplift — new priority

**Status: In progress — PRs #27–#36 are merged. The shared visual foundation, Reception, appointment, queue, clinical accessibility, facility hierarchy, tenant membership administration and platform tenant lifecycle slices are now on Web main. Authenticated real-application screenshot evidence remains a post-merge product acceptance item, not an open PR or merge blocker.** The 2026-11-25 local baseline remains 48/50 executed checks (96%), with AUTH-005 and SEC-002 corrected in merged PRs #7 and #55, and eight scenarios historically marked Not Run in that original build; facility hierarchy, tenant membership and platform tenant lifecycle are now implemented, while six deferred scenario groups remain unimplemented. Public-repository Actions runner allocation is working again. This is an improvement to the quality of the existing hospital workspace, not a change to the hospital product scope or backend authorization model.

The current authenticated UI has delivered the first operational workflows, but feature completeness alone is not the product quality bar. The workspace must evolve into a polished, cohesive, enterprise-grade hospital application that feels consistent across Reception, Appointments, Queue, Clinical, Billing, Facilities, Memberships and the separate Platform tenants control plane. The first shell/design-system slice in [CompileCare.Web PR #29](https://github.com/CompileCare/CompileCare.Web/pull/29) includes shared visual tokens, permission-driven active navigation, a patient-registration entry when authorized, a keyboard skip-to-content link, and an explicit active-hospital/workspace heading. Hosted CI passed and the visual foundation merged in `37f30508a23add4a8af0918307a524dd9f9f4bdb`. Real authenticated screenshot evidence remains pending as defined by the screenshot-evidence guide, now tracked as post-merge product acceptance. The facility hierarchy workspace merged in [PR #34](https://github.com/CompileCare/CompileCare.Web/pull/34), tenant Memberships in [PR #35](https://github.com/CompileCare/CompileCare.Web/pull/35), and the platform tenant lifecycle workspace in [PR #36](https://github.com/CompileCare/CompileCare.Web/pull/36). Their hosted CI/Docker checks passed; local authenticated acceptance and screenshot evidence remain required. See [Facility management](workflows/facility-management.md), [Tenant membership administration](workflows/tenant-membership-administration.md), [Platform tenant lifecycle](workflows/platform-tenant-lifecycle.md), [Enterprise experience uplift](design/enterprise-experience-uplift.md) and the existing [experience quality bar](design/experience-quality-bar.md).

### Delivery sequence

1. **Experience audit and baseline:** review every existing authenticated route and workflow at desktop, tablet and narrow viewports; record information hierarchy, density, navigation, responsive behavior, accessibility and consistency issues. Use real application captures with synthetic/local data only; do not fabricate screenshots or introduce mock business data.
2. **Design foundation:** establish shared design tokens for typography, color, spacing, radii, elevation, focus and semantic status; consolidate reusable page headers, buttons, form controls, cards/panels, tables/lists, status chips, alerts, empty/loading/error states and confirmation patterns.
3. **Application shell:** refine navigation hierarchy, active tenant/hospital/department context, page title and action regions, session controls, breadcrumbs where useful, responsive navigation and consistent content width. Preserve permission-driven module visibility and API-authoritative authorization.
4. **Module-by-module refinement:** improve Reception/patient search and registration, Appointments, Queue, Clinical and Billing in focused vertical slices. Prioritize high-frequency staff tasks, readable operational density, predictable keyboard interaction and clear recovery from API errors. The merged appointment-slot refinement in [CompileCare.Web PR #31](https://github.com/CompileCare/CompileCare.Web/pull/31) extracts a reusable selection grid, exposes selected/available/unavailable state, formats times in the provider session timezone and prevents changes during in-flight or confirmed booking state. The merged queue-row refinement in [CompileCare.Web PR #32](https://github.com/CompileCare/CompileCare.Web/pull/32) extracts the row presentation, announces the server-returned status and labels the action group while preserving the existing state/action/permission matrix and required cancellation reason. The merged Clinical UI increment in [CompileCare.Web PR #33](https://github.com/CompileCare/CompileCare.Web/pull/33) extracts the clinical queue row, announces the server-returned clinical queue state and labels the open/resume action by token; identity confirmation and all clinical note lifecycle operations remain unchanged. Facility hierarchy administration is delivered by merged PR #34.
5. **Visual and accessibility verification:** validate desktop, tablet and 375px layouts; complete keyboard-only workflows; review focus order, semantic labels, contrast, status announcements and reduced-motion behavior. Add targeted component/browser tests and attach real screenshots to the paired documentation where safe.

### Design and engineering constraints

- Do not trade clinical clarity, data minimization, accessibility or speed for decoration.
- Reuse existing API contracts and server-returned state; do not invent dashboards, KPIs, clinical data, permissions or workflows.
- The API remains authoritative for identity, tenant/facility scope, validation and business transitions.
- Do not place patient/clinical data in URLs, telemetry, browser storage or screenshots used outside the approved synthetic/local test context.
- Keep visual primitives shared and feature code modular; avoid a wholesale rewrite of the large application component while workflow coverage is still being established.
- Prefer small, reviewable PRs with paired documentation, tests and before/after evidence from the running application.

### Exit criteria for the uplift

The existing workflows share a consistent design language; core tasks are discoverable and efficient; layouts are usable at the supported desktop/tablet/narrow widths; keyboard-only completion and focus behavior are verified; loading, empty, validation, conflict, forbidden and server-error states are coherent; WCAG 2.2 AA is treated as a target for review rather than an unverified compliance claim; and the complete test suite/build/Docker smoke checks pass before each UI slice is merged.

The uplift is incremental. It does not block ongoing Phase 3 security/operational hardening, and it does not authorize new hospital capabilities or changes to authentication and authorization.


## Local acceptance evidence — 2026-11-25

The full Docker Compose acceptance run is documented in [Local stack acceptance](qa/local-stack-acceptance-2026-11-25.md). It is a local QA result, not production/pilot sign-off. The two failed real-stack cases were corrected in API PR #55 (SEC-002, merged after CI/Docker verification) and Local PR #7 (AUTH-005, merged after Keycloak Docker validation). The three macOS-only mocked browser-test failures were addressed in Web PR #27 (merged after hosted browser/Docker verification). Facility hierarchy CRUD UI is merged in Web PR #34, tenant membership administration UI in Web PR #35 and the platform tenant lifecycle workspace in Web PR #36; authenticated local-stack acceptance remains pending for all three. The remaining Not Run scenarios remain planned or unimplemented scope and must not be represented as test failures or completed features. Repeat the local full-stack acceptance on the updated merged commits before pilot sign-off.



### Tenant access administration — 2026-11-26

Tenant-scoped membership administration is implemented and merged in [CompileCare.Web PR #35](https://github.com/CompileCare/CompileCare.Web/pull/35), merge commit `b8e4a4ec8864884f5e71dc070a319a5f4df07185`. The workspace is permission-driven by `membership.manage`, uses the validated OIDC issuer and exact subject, supports role and multi-scope assignment, and provides confirmed deactivate/reactivate operations. The API remains authoritative for scope containment, self-management rejection, last-tenant-wide-HospitalAdmin protection and metadata-only audit.

The workflow is documented in [Tenant membership administration](workflows/tenant-membership-administration.md). ADR-0004 now records the implemented tenant registry, separate persisted platform-operator authorization and membership lifecycle rules.

**Completed:** [CompileCare.Web PR #36](https://github.com/CompileCare/CompileCare.Web/pull/36) merged as `54589758db04848d25e2e078f3619c1932cc645a`. The global platform tenant lifecycle workspace uses the merged self-status endpoint from API PR #57 and the activation guard from PR #58. It is visible only to persisted platform operators; all global operations remain protected by the PlatformOperator API policy. **Next:** authenticated local Keycloak/API/PostgreSQL acceptance for tenant creation, initial administrator provisioning, activation, suspension, reactivation and terminal closure.


### Scheduling concurrency and cross-hospital audit — 2026-11-26

Appointment scheduling now maps PostgreSQL exclusion/unique conflicts and deadlocks (SQLSTATE 23P01, 23505 and 40P01) to HTTP 409 rather than leaking an internal-server error. Concurrent booking keeps exactly one successful reservation; the competing caller must refresh availability and choose again.

- [CompileCare.Api PR #60](https://github.com/CompileCare/CompileCare.Api/pull/60) merged as `b31832ef8a416373532bcfe46b9e171dcc9890e7`. Hosted runs 45748869864 and 45748869951 passed Release build, EF model-drift, 12 unit tests, 97 PostgreSQL integration tests and Docker verification.
- [CompileCare.Api PR #61](https://github.com/CompileCare/CompileCare.Api/pull/61) merged as `ac8ef92aa8063321bc07f838c4e512945f8f2182`. The cross-hospital OPD denial test verifies five unauthorized operations return 404, emit rejected audit metadata with validated actor/tenant attribution, and leave the owner's clinical note unchanged. Hosted runs 45749260863 and 45749260926 passed the same Release/model-drift/unit/integration/Docker gates.
- [CompileCare.Api PR #62](https://github.com/CompileCare/CompileCare.Api/pull/62) merged as `ceac287c3d03d92fadc408a4c20f76cb4e223b9e`. Department-scoped ClinicalUser isolation now verifies five same-hospital, cross-department encounter operations return 404, are audited with validated actor/tenant attribution, and cannot mutate or sign the owning department's note. Hosted CI runs 45749980410 and 45749962698 passed Release build, EF model-drift, 12 unit tests and 98 PostgreSQL integration tests; Docker run 45749980403 passed.

The appointment and cross-hospital security workflows are documented in [Appointments and queue](workflows/appointments-and-queue.md) and [ADR-014](../architecture/decisions/ADR-014-appointment-scheduling-and-queue-boundaries.md). Authenticated local Keycloak/API/PostgreSQL acceptance remains required before pilot sign-off.

- [CompileCare.Api PR #64](https://github.com/CompileCare/CompileCare.Api/pull/64) merged as `8b51c3e0058b5daa5215ed5d4234fbc554078ca2`. Department-scoped PatientRegistrar and ClinicalUser denials for hospital-wide patient search, cross-department patient detail and search now assert rejected audit metadata with validated issuer/subject, requested tenant and 404 status. Hosted CI runs 45750443111 and 45750425883 passed Release build, EF model-drift, 12 unit tests and 98 PostgreSQL integration tests; Docker run 45750443058 passed.

- [CompileCare.Api PR #65](https://github.com/CompileCare/CompileCare.Api/pull/65) merged as `7660a2761b2a638c8e42461a4c3a6c86ab073073`. The department-scoped billing integration test now asserts that unauthorized visit-invoice and payment-history reads return 404 and produce rejected Billing audit records with validated actor/tenant attribution. Hosted CI runs 45750909327 and 45750890352 passed Release build, EF model-drift, 12 unit tests and 98 PostgreSQL integration tests; Docker run 45750909307 passed.


## Reception MRN search update — 2026-11-26

CompileCare.Web PR #37 is merged as `150a61a20e824f3e82f4450bc76e536a390417fd`. Reception now supports exact hospital-scoped MRN lookup alongside patient-name search. The MRN mode is visible only to memberships with `patient.identifier.read`; lookup uses the selected tenant and hospital, disables caching/credential forwarding through the existing API client, and does not persist patient search terms or results in browser storage. Switching tenant/hospital or losing identifier-read permission clears the MRN mode and search state. Hosted Web `build-test` runs 45752346956 and 45752342891 passed on head `a9a44eaf0d1f38d6336fc23dc2478bf050b77acd`, including Vitest, Chromium E2E, production build and Docker smoke.

**Follow-up:** MRN policy administration was delivered in Web PR #38; appointment availability/list/reschedule/cancel workflows were delivered in API PR #70 and Web PR #39; staff/provider management was delivered in Web PR #40. The remaining Phase 5 work is cross-module usability polish and authenticated local acceptance with synthetic records. Desktop/tablet/mobile screenshots and keyboard/accessibility evidence remain outstanding; hosted CI is not a substitute.


## Hospital MRN policy administration — 2026-11-26

CompileCare.Web PR #38 merged as `23e84efaaa02ed61d18696fd8a9318db2df1862b`. The selected-hospital MRN policy workspace reads the existing policy and allows one-time initial configuration using a prefix and sequence length. Access is permission-gated by `patient.identifier.policy.read` and `patient.identifier.policy.manage`; creation is exposed only to policy managers. The UI reflects the API's immutable-after-creation contract and uses tenant/hospital-scoped, no-store requests. Hosted Web run 45753221432 passed Vitest, production build, Chromium E2E and Docker smoke; run 45753227661 also passed. Local authenticated acceptance remains outstanding.

**Delivered:** appointment management in API PR #70 and Web PR #39, followed by staff/provider management in Web PR #40. Continue using API-enforced permissions and synthetic data. Do not implement patient contact verification or clinical credential/privileging without the required security/product decisions. Authenticated local acceptance remains outstanding.


## Appointment management workspace — 2026-11-26

**API listing foundation completed:** [CompileCare.Api PR #70](https://github.com/CompileCare/CompileCare.Api/pull/70) merged as `47f8f038603d4c03a923b0a712e24467b6808f5d`. Added paginated appointment listing scoped by tenant, hospital, department and availability-session local date, with validated status filtering and bounded pagination. The endpoint uses the existing `appointment.read` authorization policy and returns scheduling metadata without patient names or contact data. Hosted Release build, EF model-drift, unit/integration tests and Docker verification passed (runs 45754993339 and 45754993230).

**Web workspace completed:** [CompileCare.Web PR #39](https://github.com/CompileCare/CompileCare.Web/pull/39) merged as `13f56f80cb7cf66a373a862fbe0152024ce26c63`. Hospital administrators can publish provider availability only for active providers assigned to the selected hospital/department on the chosen local date. The appointment workspace supports date/department/status filtering, pagination, explicit two-step cancellation and rescheduling that retains the original patient and API lineage. Appointment times are rendered using the hospital session time zone. Hosted Vitest, production build, Chromium E2E and Docker smoke passed on head `251365ffcea7351847192b7adf45b896e179fff1` (run 45755338598).

**Delivered:** staff/provider directory and effective-dated assignment management are complete in Web PR #40. The remaining acceptance pass must cover staff profile lifecycle, department assignments, availability publishing, appointment listing, department isolation, cancellation, reschedule lineage and queue check-in using synthetic records. Hosted CI is not local acceptance.


## Staff and provider management workspace — 2026-11-26

[CompileCare.Web PR #40](https://github.com/CompileCare/CompileCare.Web/pull/40) merged as `3d54ab958ab8129f526a690a25d4ce56bc17cf5a`. The tenant-scoped staff workspace supports staff profile creation and updates, selected-hospital directory search, effective-dated department assignment creation/end, and profile deactivation/reactivation. Deactivation closes open-ended active assignments through the API. UI access is gated by `staff.read` and mutations by `staff.manage`; provider designation is descriptive and does not imply credential verification or clinical privileging. Hosted Vitest, TypeScript/production build, Chromium E2E and Docker smoke passed (runs 45755742307 and 45755757928).

**Current Phase 5 follow-up:** the workspace now includes reception/MRN, scheduling, workforce, OPD, billing, facility administration, membership management, platform tenant lifecycle and the hospital operations overview (Web PR #42). Focus next on authenticated end-to-end acceptance, responsive visual evidence, keyboard/accessibility review and cross-module workflow polish. Credential verification/privileging and patient contact verification remain separate security/product approval gates.


## Workspace visual refresh — 2026-11-26

[CompileCare.Web PR #41](https://github.com/CompileCare/CompileCare.Web/pull/41) merged as `c0cc89c06a847f4a93ce46b20e120199a048dc34`. Refreshed the shared design tokens, typography, surfaces, focus-visible treatment, responsive desktop/tablet/mobile navigation, sticky workspace shell and reduced-motion behavior. The change is CSS-only and retains module authorization and workflow behavior. Hosted Vitest, production build, Chromium E2E and Docker smoke passed (runs 45756076512 and 45756064645). Visual acceptance screenshots at desktop/tablet/mobile sizes and keyboard accessibility review remain outstanding on the local authenticated environment.

**Delivered:** the hospital operations overview is complete in Web PR #42. Continue with authenticated local acceptance, responsive visual evidence, keyboard/accessibility review and cross-module workflow polish. Operational hardening now includes controlled audit query/export, migration rehearsal CI and logical backup/restore verification; production retention, key rotation, secrets/encryption, provider-native physical backup/PITR and recovery sign-off remain. The initial OPD pilot is feature-rich but not production-approved until local acceptance and security/operational sign-off are complete.


## Hospital operations overview — 2026-11-26

[CompileCare.Web PR #42](https://github.com/CompileCare/CompileCare.Web/pull/42) merged as `3243b658067b94f3a462d35527c58bff64653648`. Added a permission-aware hospital overview showing selected-date appointment totals and queue-entry totals scoped to the selected department, plus hospital-wide daily collection and payment count in the configured hospital time zone. Quick links are exposed only for the corresponding module permissions; the dashboard does not display patient names, MRNs or other PHI. Requests are guarded against stale results when tenant, hospital, department or date changes. Hosted Vitest, production build, Chromium E2E and Docker smoke passed on head `b138ee3d99f76caa43adde575e12f809e64f4bf8` (run 45756983905).

**Next priorities:** finish local end-to-end acceptance and visual/accessibility evidence; then harden operational controls (audit retention/export governance, key rotation, secrets/encryption, migration rehearsal and tested physical backup/recovery). After these controls are underway, sequence remaining hospital modules such as IPD, pharmacy, LIS, RIS/PACS, inventory and integrations. The platform is not production-approved until security, privacy, local acceptance and recovery sign-offs are complete.


## Controlled audit query and export — 2026-11-26

[CompileCare.Api PR #71](https://github.com/CompileCare/CompileCare.Api/pull/71) merged as `da0880979b25cc12a2ccfd18419a6368baa1d2de`. Added tenant-wide, permission-gated audit event listing and CSV export. Queries require a validated tenant context and tenant-wide organization scope; hospital-scoped administrators are denied tenant-wide audit access. Listing supports bounded UTC date ranges (maximum 90 days), pagination (maximum 100 rows/page), and action/resource filters. CSV export is bounded to 31 days and 5,000 rows and neutralizes spreadsheet formula prefixes. Exported fields are audit metadata only; request/response bodies and clinical payloads are excluded. PostgreSQL integration tests cover query/export, tenant isolation, tenant-wide scope authorization, date limits and formula-injection handling. Hosted Release CI and Docker integration verification passed on the merged commit.

**Operational follow-up:** audit retention, authorized access review, export access logging/monitoring, retention enforcement and privacy/legal review remain required before production approval. The query/export capability does not itself implement retention or a production backup/recovery process.


## Audit export hardening — 2026-11-26

[CompileCare.Api PR #72](https://github.com/CompileCare/CompileCare.Api/pull/72) merged as `0fc8cfee33290986c798bf0285dc6cf6861a9422`. CSV export now neutralizes formula values even when preceded by whitespace and prefixes leading tab/CR/LF control characters. PostgreSQL integration coverage confirms a control-character-prefixed actor subject is neutralized and that the export endpoint itself is recorded as an `ExportAuditEvents` audit event. Release CI and Docker integration verification passed. This supplements the bounded tenant-wide audit query/export delivered in PR #71; retention policy, access review and production governance remain outstanding.


## PostgreSQL migration rehearsal and integration-test isolation — 2026-11-26

[CompileCare.Api PR #73](https://github.com/CompileCare/CompileCare.Api/pull/73) merged as `e95e23dee5a08fe6805ae64da1c8cdefda759597`. Added a PostgreSQL 16 CI rehearsal that applies the base revision to an isolated upgrade database, then applies the PR revision's idempotent migration script to both the upgraded database and a fresh database. The script is applied twice to verify idempotency, and the complete EF migration histories must match. The rehearsal caught and corrected a missing SQL statement terminator in the audit-trigger migration; the trigger behavior is unchanged. Integration tests now run serially because several fixtures reset the same shared test database. Release CI, Docker integration verification and migration rehearsal passed. This is a disposable CI check, not a production-data migration rehearsal or release approval.


## Local development credential bootstrap hardening — 2026-11-26

[CompileCare.Local PR #8](https://github.com/CompileCare/CompileCare.Local/pull/8) merged as `6784962a9120546952329e9f63d92aeee9c6bd4c`. Fresh macOS and Windows local setups now generate independent random 256-bit Keycloak administrator and PostgreSQL passwords in the ignored `.env`; the macOS `.env` is restricted to owner read/write. Compose requires an explicit PostgreSQL password and no longer falls back to a checked-in weak value. Existing `.env` credentials are not silently rotated because persistent PostgreSQL/Keycloak volumes retain their own credentials; legacy values are warned/rejected with guidance for coordinated rotation. Local CI passed shell syntax, PowerShell parsing, Keycloak Docker import/reconciliation, Compose validation and observability Docker smoke tests. This is local-development hardening, not a production secrets-management or key-rotation solution.


## Logical backup and restore verification — 2026-11-26

[CompileCare.Database PR #4](https://github.com/CompileCare/CompileCare.Database/pull/4) merged as `245c42fcc4e19a51a391b7203087de88a144cf3d`. The existing PostgreSQL 16 Docker backup/restore suite now additionally verifies the archive's SHA-256 sidecar, manifest format/source database metadata and mode `0600` on the archive and both sidecars. The test also exercises successful restore, source/target overwrite guards and corrupted-archive rejection. The Docker verification workflow passed. This validates provider-neutral logical backup helpers only; production physical backups/PITR, encryption/key ownership, off-site storage, retention, RPO/RTO and scheduled restore drills remain open.


## Keyboard skip-link verification — 2026-11-26

[CompileCare.Web PR #43](https://github.com/CompileCare/CompileCare.Web/pull/43) merged as `73764e1bb14ce5de6a84cced5e9072603b796e73`. The Chromium accessibility suite now verifies the skip link is the first keyboard stop, has a visible focus indicator, and activates with Enter to move focus to the main workspace landmark. Vitest, production build, Chromium E2E and Docker smoke passed (Web CI runs `45762291461` and `45762285239`). This is a targeted keyboard-navigation regression test, not a complete WCAG 2.2 AA audit; authenticated local-stack keyboard/screen-reader review and real responsive screenshots remain outstanding.


## Weekly synthetic logical backup/restore regression — 2026-11-26

[CompileCare.Database PR #5](https://github.com/CompileCare/CompileCare.Database/pull/5) merged as `5e53afec375549f81fe05b9f6c8a9db733a24297`. The existing PostgreSQL 16 Docker logical backup/restore verification now also runs every Monday at 03:17 UTC, in addition to relevant PRs, pushes to `main` and manual dispatch. It uses disposable PostgreSQL and synthetic probe rows to validate archive/checksum/manifest, isolated restore, overwrite guards and corrupted-archive rejection. This is scheduled CI regression coverage only; it does not access production, prove provider-native physical backup/PITR or off-site storage, or replace a production restore drill. Provider selection, encryption/key ownership, retention, RPO/RTO and accountable recovery drills remain open.


## Local infrastructure smoke helper — 2026-11-26

[CompileCare.Local PR #9](https://github.com/CompileCare/CompileCare.Local/pull/9) merged as `d9e52044051b3fadb15e8ed3f679f0346ad31952`. Added `bash local/scripts/verify-stack.sh` to verify the four Compose services are running, PostgreSQL readiness, Web/API/OpenAPI reachability and Keycloak OIDC discovery/issuer consistency. Local CI passed shell syntax validation, Keycloak Docker import/reconciliation, Compose validation and the observability Docker smoke pipeline. The script does not perform real user authentication, tenant/RBAC checks or patient workflow acceptance; it must be run against the developer's local Docker stack before local environment acceptance can be closed.


## Audit permission spoofing regression — 2026-11-26

[CompileCare.Api PR #75](https://github.com/CompileCare/CompileCare.Api/pull/75) merged as `649d32636b9bc4bd5114dc08ca27974e50ce4d71`. PostgreSQL integration tests verify PatientRegistrar, ClinicalUser, FacilityManager and FacilityViewer memberships cannot list or export tenant audit events, even when a signed test token carries forged audit.read, tenant-role and standard role claims. This protects the rule that CompileCare permissions are derived from persisted membership, not caller-supplied token role/permission claims. Release CI and Docker integration verification passed.


## Product delivery extension — UI architecture, pilot gates and deferred hospital modules (2026-11-26)

This section complements the milestone checklist above. It distinguishes current implementation, remaining acceptance work and intentionally deferred product scope. Do not mark a milestone complete from PR count or hosted CI alone.

### Immediate priority: stabilization and acceptance
- [x] Merge Web response-security headers with Docker-verified assertions while retaining `X-Frame-Options: DENY` (Web PR #46). Content-Security-Policy remains deferred until deployment-specific OIDC/API origins are allow-listed.
- [ ] Run the merged one-click local Docker launcher on the developer Mac and record service health.
- [ ] Complete authenticated Keycloak/API/PostgreSQL regression with synthetic tenants, users, roles and patient records.
- [ ] Verify cross-tenant and cross-hospital denial using persisted membership and changed resource identifiers.
- [ ] Verify real API logs, metrics and traces reach Elasticsearch/Kibana, Prometheus and Jaeger through the shared Collector; inspect redaction and export errors.
- [ ] Capture authenticated desktop/tablet/mobile screenshots and keyboard/accessibility evidence.
- [ ] Repeat local acceptance after fixes; record PASS/FAIL/BLOCKED/NOT TESTED and retain all skipped scenarios.
- [ ] Complete backup/PITR and restore, migration/rollback, secrets/key rotation, audit retention, security and pilot acceptance gates.

### Web experience architecture milestone
**Goal:** move from the interim single Workspace/hash-section composition to independent route-level module workspaces under a shared authenticated shell.

- [ ] Introduce route-based module layouts and stable URLs; retain safe redirects/deep links from existing hash URLs during migration.
- [ ] Extract overview, reception and patient registration into independent route workspaces.
- [ ] Extract appointments and queue workspaces without changing their API lifecycle semantics.
- [ ] Extract clinical/OPD, retaining patient identity confirmation, signing, closing and append-only amendment safeguards.
- [ ] Extract billing, facilities, staff, memberships and patient identifier policy.
- [ ] Keep platform tenant lifecycle in a separate platform-operator control plane.
- [ ] Use permission-aware navigation and user-friendly route guards while retaining API-side authorization as the only security boundary.
- [ ] Add route-level loading, empty, error, forbidden and not-found states; verify refresh, browser history, session expiry and deep links.
- [ ] Verify desktop/tablet/mobile, keyboard-only navigation, screen-reader semantics and WCAG 2.2 AA target criteria.
- [ ] Capture real authenticated screenshots with synthetic data; mocked tests are not local acceptance evidence.

Implementation must be staged in focused Web PRs. Do not combine this with backend/domain changes or a big-bang rewrite.

### Deferred implementation phases — retained scope

| Phase | Planned module | Major capabilities | Status |
|---|---|---|---|
| 6 | IPD / inpatient | ADT, admission, bed reservation/occupancy, ward/nursing, observations, care plans, handover, medication administration, discharge | Deferred |
| 7 | Pharmacy | Formulary, prescription validation, dispensing, batches/expiry, stock, returns and supplier workflows | Deferred |
| 8 | LIS | Diagnostic test catalogue/orders, specimen collection/barcodes, worklists, processing, result validation, critical-result escalation | Deferred |
| 9 | RIS/PACS | Imaging orders/scheduling, modality worklists, DICOM, reports, image storage/access and retention | Deferred |
| 10 | Inventory/procurement | Stores, requisitions, purchase orders, vendors, receipts, stock movement, assets and maintenance | Deferred |
| 11 | Advanced revenue cycle | Statutory invoice/tax rules, deposits, discounts, refunds, credit notes, insurance/TPA, claims, payment/accounting integrations and reconciliation | Deferred |
| 12 | Workforce/HR | Verified account linking, credentialing, clinical privileges, rostering, attendance, leave, employee records and payroll integrations | Partially scaffolded; extended capabilities deferred |
| 13 | Patient engagement/interoperability | Patient/guardian portal and mobile, self-booking, consent-aware communications, ABDM/ABHA, FHIR/HL7, DICOM and external integrations | Deferred |
| 14 | Advanced analytics/intelligent assistance | Permissioned operational/financial analytics, exports and governed AI-assisted workflows with human oversight | Deferred |

### Deferred scope guardrails
- Patient contact verification, identity merge/correction, guardian and emergency reconciliation require approved privacy, identity, consent, retention and security design.
- Clinical credentialing and privileges require verified evidence and qualified clinical governance; a staff designation is not authority.
- Advanced billing, GST/statutory treatment, refunds, insurance and claims require pilot finance policy and qualified review.
- ABDM/ABHA, FHIR/HL7 and DICOM integrations require validated use cases, partner agreements, approvals and conformance/security testing.
- Autonomous diagnosis, triage or prescribing is not in scope.
- Full ERP/payroll, broad mobile delivery, multi-region active-active and microservice extraction remain gated by validated demand and architecture decisions.

### Dependency order
Stabilize and validate the outpatient journey → route-based Web architecture and OPD completion → security/operations/pilot gates → IPD → pharmacy → LIS/RIS-PACS → inventory/procurement → advanced revenue cycle → workforce/patient engagement → interoperability/analytics. Reorder only with documented pilot evidence and explicit dependency review.

Every completed capability must update the blueprint, module catalogue, workflow documentation, API contracts, tests and release evidence. Keep future work visible as Deferred rather than silently removing it from scope.


## Stabilization evidence update — 2026-11-26 (post full-platform regression)

This is an evidence update to the original Antigravity report. It records code/CI remediation separately from developer-machine acceptance. A green hosted workflow does not prove that the user's nine-container stack or authenticated browser session has been revalidated.

### Defects addressed in code and merged

| QA item | Remediation merged | Verification | Remaining acceptance |
|---|---|---|---|
| API-001 — malformed requests returning 500 | API PR #76 preserves the 4xx status carried by `BadHttpRequestException` and adds malformed audit-query/empty-body integration cases | Release CI, PostgreSQL integration, Docker test and migration rehearsal passed | Re-run the live local API request cases |
| API-002 — concurrent patient booking deadlock | API PR #77 adds a transaction-scoped tenant/patient advisory lock before the overlap check; the exclusion constraint remains the final guard; the race test now runs eight concurrent requests | Release CI, PostgreSQL integration, Docker test and migration rehearsal passed | Re-run repeated booking race in the local full stack |
| WEB-001 — wrong default workspace | Web PR #45 makes Reception the default for staff with patient-search access | Vitest, TypeScript/build, Chromium E2E and Docker smoke passed | Authenticated local browser visual confirmation |
| WEB-002 — 390px horizontal overflow | Web PR #45 adds narrow-screen layout constraints and an authenticated 390×844 Playwright regression | Chromium E2E and Web CI passed | Confirm the actual Mac/browser viewport and capture screenshots |
| WEB-003 — cancellation contrast | Web PR #45 adds high-contrast destructive styling and a component assertion | Vitest and Web CI passed | Visual confirmation in the local authenticated workspace |
| OBS-001 — broken API OTLP traces/metrics | Observability PR #4 fixes the exporter to OTLP/gRPC and publishes package 1.0.1; API PR #78 consumes it and adds Npgsql instrumentation; Local PR #11 configures port 4317 and Local PR #13 runs the complete nine-service Compose stack and `verify-stack.sh` in hosted CI | Observability, API and Local CI passed. Local PR #13 verified all nine services, API traces in Jaeger including a `postgresql` operation, and ASP.NET Core request metrics through the OTLP Collector exporter | **Hosted integration verified.** Still run the merged launcher and `verify-stack.sh` on the developer Mac and retain local evidence; this is separate from authenticated browser acceptance |
| OBS-002 — missing Kibana data view | Local PR #12 adds idempotent `CompileCare Logs` provisioning for `compilecare-logs-*` with `@timestamp` and wires it into stack verification | Python syntax and Docker observability CI, including Kibana data-view API assertion, passed | Verify the view and Discover experience in the local Kibana instance |

### Security and operations — current state

- Web PR #46 adds response headers (`nosniff`, Referrer-Policy, `X-Frame-Options: DENY`, CORP and Permissions-Policy); Docker smoke asserts them. This is defence-in-depth, not a complete browser security policy.
- SEC-001 code-level hardening is merged in API PR #79: per-subject/per-IP rate limiting, RFC 9457 429 responses with Retry-After, HSTS outside Development, trusted forwarded-header processing and the Kestrel banner disabled. Configure ReverseProxy:KnownProxies with the exact trusted ingress proxy IP addresses before deployment. The built-in limiter is per API process/instance, not a fleet-global quota; a multi-replica deployment needs gateway/shared quota enforcement for strict global limits. Tune the defaults against measured hospital traffic.
- SEC-002 is partially addressed: Web PR #47 commits npm's generated lockfile, switches CI and Docker to npm ci, and makes npm audit blocking. The final PR CI passed and the dependency audit reported zero vulnerabilities for that lockfile. CSP remains deferred until exact production Keycloak/API origins are known and the enforced policy is browser-tested. Do not claim complete browser security or supply-chain compliance from the dependency audit alone.
- The previous NU1902 on the OTLP exporter version 1.12.0 was addressed by the Observability 1.0.1 update to exporter 1.15.3.
- GAP-001 (Audit Review UI) remains outstanding. API audit search/export capability does not equal a usable, permission-controlled Audit Review workspace.

### Acceptance gates still open

- [ ] Execute the merged `CompileCare.Local` macOS one-click startup against current `main`, without deleting persistent Docker volumes.
- [x] Run the full `verify-stack.sh` against all nine services in hosted Docker CI; Local PR #13 verified API-to-Jaeger traces including an Npgsql `postgresql` operation, plus Collector-fed ASP.NET Core request metrics.
- [ ] Run the merged `verify-stack.sh` on the developer Mac and retain the local Jaeger/Collector evidence; hosted CI does not replace the local environment check.
- [ ] Repeat authenticated Keycloak/API/PostgreSQL user journeys for at least two synthetic tenants, including membership revocation and cross-tenant negative tests.
- [ ] Re-run the eight-way concurrent appointment booking scenario repeatedly against PostgreSQL.
- [ ] Capture authenticated desktop/tablet/390px screenshots, console/network output and keyboard/accessibility evidence.
- [ ] Complete Audit Review UI requirements, permission model, search/export UX and workflow acceptance before implementing it.
- [ ] Resolve production CSP origins/policy and test the enforced policy; complete secret rotation, backup/PITR/restore, audit retention and production ingress configuration.
- [ ] Configure exact ReverseProxy:KnownProxies ingress addresses and validate forwarded scheme/client-IP behavior in the target deployment; load-test and tune per-instance rate limits, and add gateway/shared quotas if multi-replica global limits are required.
- [ ] Complete hospital pilot acceptance, security/privacy review and applicable legal/regulatory assessment.

### Release decision

The code-level remediation above reduces the original defect set, and OBS-001 now has hosted full-stack evidence from Local PR #13 (real API traces, Npgsql database operation and Collector-fed request metrics). The platform remains **CONDITIONAL FAIL / HOLD NEW FEATURE WORK** until developer-Mac execution, authenticated multi-tenant workflows, actual UI visual/accessibility acceptance and remaining security/operational gates are evidenced. Hosted CI does not replace those environment-specific checks. Keep new hospital modules deferred while these gates remain open.


### Security and reproducible-build follow-up — 2026-11-26

The subsequent remediation is now merged:
- API PR #79 (merge commit 403b633) adds the rate limiter (600/minute per authenticated subject, 120/minute per anonymous IP by default), 429 ProblemDetails/Retry-After, trusted forwarded-header processing with a one-hop/symmetric-header policy, HSTS outside Development and disables the Kestrel server banner. Its final hosted checks passed 12 unit tests, 117 PostgreSQL integration tests, Docker verification and migration rehearsal.
- Web PR #47 (merge commit 0732352) commits the npm-generated v3 lockfile and uses npm ci in both Web CI and the Docker build. The PR's blocking npm audit, Vitest, TypeScript/production build, Chromium E2E and Docker/header smoke all passed. The generating CI audit reported zero vulnerabilities.

These changes close the original API rate-limit code gap and the Web lockfile/ENOLOCK auditability gap at repository/CI level. They do not close deployment acceptance: operators must set the exact trusted proxy IPs, calibrate per-instance rate limits, define a shared quota strategy if strict multi-replica limits are required, and complete CSP policy testing for the actual production OIDC/API origins.


### Full nine-service hosted Docker evidence — 2026-11-26

Local PR #13 merged as `bc609fc67f9656e444bc486a47fa047999bd33fa`. Its PR CI run `45785200869` passed both jobs:
- Full nine-service Compose job: all nine services running; PostgreSQL readiness; Web/API readiness and OpenAPI; Keycloak OIDC discovery; Kibana Logs data view provisioning; API metrics endpoint.
- Real application telemetry: Jaeger contained CompileCare.Api operations including the Npgsql `postgresql` database operation; ASP.NET Core request-duration metrics reached the OpenTelemetry Collector Prometheus exporter.
- The isolated observability pipeline, Keycloak import/reconciliation, configuration and shell validation also passed.
- The uniquely named CI Compose project, volumes and generated local secrets were removed successfully.

This closes the **hosted integration verification** portion of OBS-001. It does not prove the developer's Mac installation, authenticated sign-in/tenant workflows, actual production ingress configuration, or browser visual/accessibility acceptance. Those remain explicit gates.
