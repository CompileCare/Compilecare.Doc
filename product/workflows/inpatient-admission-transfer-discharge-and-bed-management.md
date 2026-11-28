# IPD: Admission, Transfer, Discharge and Bed Management

**Status:** Initial IPD ADT technical foundation is merged: admission creation/list/detail, bed assignment and same-hospital transfer. Hospital/Tenant/department read isolation, transfer concurrency, occupied-bed conflict handling, successful/denied bed-operation audit, audit-failure rollback and real-Keycloak negative authorization are verified. All IPD role grants remain closed; hospital policy, cancellation/discharge and the positive Web workflow are incomplete.  
**Related:** [ADR-018: Inpatient ADT and Bed-Management Boundaries](../../architecture/decisions/ADR-018-inpatient-ADT-bed-management-boundaries.md), [ADR-006: Tenant, Organization and Patient Data Boundaries](../../architecture/decisions/ADR-006-tenant-organization-patient-boundaries.md).

## Implementation checkpoint — 2026-11-28

The following technical slices are merged to `HeapCare.Api/main`:

- **Persistence:** PR #134 adds tenant-owned inpatient admissions and bed-placement history with hospital registration constraints, same-hospital facility relationships and filtered uniqueness for active admissions/placements/bed occupancy.
- **History and contracts:** PR #135 adds physical-delete guards and dedicated, ungranted IPD permission names.
- **Bed service:** PR #136 implements admission read, first-bed assignment and same-hospital transfer with retained placement history, department scope checks and PostgreSQL concurrency protection.
- **HTTP read/placement routes:** PR #137 exposes admission read, bed assignment and transfer behind dedicated permissions. Existing roles remain denied.
- **Denied-read audit evidence:** PR #138 verifies that a denied IPD read records the validated Tenant and OIDC actor with outcome, resource ID and correlation metadata.
- **Admission creation:** PR #139 adds `POST /api/v1/hospitals/{hospitalId}/ipd/admissions`. The service requires an existing patient registration at the exact Hospital and tenant-wide or hospital-wide scope, creates an Active episode with a server-generated UTC timestamp and no bed placement, and returns a conflict for duplicate/concurrent active admissions.
- **Admission list:** PR #140 adds `GET /api/v1/hospitals/{hospitalId}/ipd/admissions` with bounded pagination and optional case-insensitive status filtering. Department-only visibility is based on current bed placement, and returned placement history is restricted to assigned departments.

PR #139 exact-head validation passed: 12 unit tests, 240 PostgreSQL integration tests, Release build, EF Core model-drift guard, PostgreSQL migration rehearsal and Docker Integration Verification.

**Access remains closed by default:** no tenant role or PlatformOperator has been granted IPD permissions. The six-role positive/negative permission mapping and hospital workflow approval remain open. These merged technical increments do not establish that admission-before-bed-assignment is the final hospital policy; that workflow decision remains subject to hospital review.

## Purpose

Provide the first inpatient administrative workflow for a hospital: create and manage an inpatient episode, track bed occupancy, move a patient between beds and discharge the patient while retaining a trustworthy history.

The initial release is limited to admission, bed placement, transfer, cancellation and discharge. It does not create or manage clinical orders, medication administration, nursing observations or discharge summaries.

## Actors and access

The first release requires dedicated IPD capabilities for:
- Reading an admission and its placement history.
- Creating an admission.
- Assigning a bed or transferring an existing admission.
- Cancelling an admission where policy permits.
- Discharging an admission.

The exact mapping to PlatformOperator, HospitalAdmin, FacilityManager, FacilityViewer, PatientRegistrar and ClinicalUser must be approved before the routes are exposed. A role must not receive IPD authority merely because it can manage OPD, billing or facility master data.

## Workflow outline

### 1. Confirm patient and hospital registration

- Select the active Tenant and Hospital context through the authenticated workspace.
- Find the existing patient and confirm that a HospitalPatientRegistration exists for that Hospital.
- Do not create a duplicate patient master as part of admission.
- Do not expose a patient registration from another Hospital or Tenant through a manipulated identifier.

### 2. Create an inpatient episode

- Create an admission under the same Tenant and Hospital as the verified hospital registration.
- Record the validated actor, timestamps and approved administrative admission details.
- Admission status transitions are controlled by the service; clients cannot directly set status or timestamps.
- Whether an episode can be created before a bed is assigned remains an open policy decision.

### 3. Assign a bed

When a bed is assigned:
- Resolve Bed → Room → Ward → Department → Hospital and verify every relationship belongs to the same Tenant and Hospital as the admission.
- Confirm that the bed is active and has no current occupant.
- Prevent two admissions from acquiring the same bed, including concurrent requests.
- Preserve a placement record with start time and actor attribution.

### 4. Transfer an inpatient

- Read the active admission and current placement.
- Validate the destination bed and all its parent facility relationships.
- Reject a destination that is inactive, occupied, outside the admission's Hospital or outside the caller's authorized scope.
- Close the old placement and open the new placement in one transaction.
- Preserve the old placement for history; do not rewrite it to make it appear that the patient was always in the new location.

### 5. Cancel or discharge

- Cancellation and discharge are separate operations with distinct transition rules.
- Discharge ends the current placement and inpatient episode; historical admission and placement records remain.
- A discharged/cancelled admission cannot be assigned a new bed or transferred.
- Exact eligibility, reason requirements, discharge timing and billing hand-off remain policy decisions.

## Data invariants

- One patient registration at the selected Hospital is required.
- An admission belongs to one Tenant and one Hospital in the initial increment.
- One bed has at most one active placement.
- One admission has at most one active placement.
- Transfers are atomic and preserve previous placements.
- Discharge releases the current bed.
- No cross-Hospital or cross-Tenant references are accepted.
- Reads and mutations enforce explicit facility/department scope.
- Mutations preserve actor attribution and append-only audit metadata.
- Records cannot be physically deleted to hide an admission or movement.

## Audit and privacy

Audit the operation type, outcome, Tenant, Hospital, admission/placement identifiers, correlation ID, validated issuer/subject and occurrence time. Do not include patient names, clinical notes, diagnosis text, medication details, contact information or other unnecessary personal/clinical data in audit metadata, logs or traces.

Use synthetic patient identities in automated tests and sanitized evidence only.

## Explicitly not in this workflow

Medication reconciliation, prescriptions, MAR, nursing notes, vitals/observations, clinical orders, investigations, diagnostics, pharmacy, housekeeping, inventory, patient portal, insurance/TPA/ABDM, invoices, deposits and discharge summaries.

## Hospital isolation and audit hardening — merged 2026-11-28

- **Hospital-scoped list isolation:** API PR #141 adds PostgreSQL integration coverage with two Hospital records under one Tenant and separate synthetic registrations/admissions. A Hospital-scoped member can list only its own Hospital's admissions; requesting a sibling Hospital's list returns a non-disclosing 404. A member scoped to the sibling Hospital sees only that Hospital's admission. Merged as `57a8489b77ee72e0e34cca89401f75736785090f`.
- **Created Admission audit resource:** API PR #142 adds a request-local audit resource ID override. On successful admission creation, the generated Admission ID is used as the audit resource ID instead of the parent Hospital ID from the create route; existing route-derived IDs remain the fallback. Merged as `22a6dae84b5a8a7c6b549975a321507d308f8b0a`.
- **Validation:** PR #141 passed 12/12 unit and 245/245 PostgreSQL integration tests plus Release build, EF model-drift guard, migration rehearsal and Docker Integration Verification. PR #142 passed 12/12 unit and 246/246 PostgreSQL integration tests plus the same build/model/migration/Docker gates.
- **Scope:** These are isolation and audit hardening increments only. They add no permission grants, schema, migration, billing behavior or clinical workflow.

## Bed-transfer isolation and concurrency — merged 2026-11-28

- **Cross-Hospital and cross-Tenant bed-transfer denial:** API PR #143 seeds a current admission in one Hospital, a sibling Hospital under the same Tenant and a Hospital under a different Tenant. Transfers to both out-of-scope destinations return non-disclosing 404; the original placement remains the only active placement with no end timestamp or end reason. Merged as `32e7b56b7969d0b80b8194b3973949ff4865259a`.
- **Concurrent transfer conflict:** API PR #144 synchronizes two transfers from the same active placement using a test-only PostgreSQL trigger. Exactly one succeeds, the competing request returns 409, and the final history contains one closed original placement plus one active destination. The test trigger/function are removed in `finally`. Merged as `c718f0ccd8816158d6740faef5a27656da89dc0a`.
- **Occupied destination bed conflict:** API PR #145 seeds two active admissions in the same Hospital with separate occupied beds and attempts to transfer the first admission into the second patient's bed. The service returns 409 and both original active placements remain unchanged. Merged as `b7663ee29f231c55f4775b7a473af71eb05bc80b`; exact-head validation passed Release build, model-drift guard, 12/12 unit tests, 249/249 PostgreSQL integration tests, PostgreSQL migration rehearsal and Docker Integration Verification.
- **Validation:** both PRs passed 12/12 unit tests and 247/247 PostgreSQL integration tests, Release build, EF model-drift guard, PostgreSQL migration rehearsal and Docker Integration Verification.
- **Denied bed-operation audit attribution:** API PR #146 adds a PostgreSQL-backed HTTP regression for denied bed assignment and bed transfer. It verifies 403 responses, validated tenant and OIDC actor attribution, action/route/admission resource metadata, denied outcome/status/correlation IDs and no placement side effects. Merged as `39c40bb7ccdb899b23684b18595293c331a79078`; exact-head Release build, 12/12 unit tests, 249/249 PostgreSQL integration tests, the additional API build/test and Docker Integration Verification passed.
- **Real-Keycloak negative authorization:** Local PR #46 is merged. Its exact-head full-stack run [#46114200037](https://github.com/HeapCare/HeapCare.Local/actions/runs/46114200037) denied every IPD route for all five persisted tenant roles and platform-only PlatformOperator while also passing nine-service startup, PostgreSQL logical backup/restore, Chromium PKCE and Firefox/WebKit six-role accessibility baseline. Existing IPD permissions remain ungranted. API PR #137's PostgreSQL suite separately covers a PlatformOperator who also has a tenant FacilityViewer membership.

These are automated isolation, concurrency and negative-authorization proofs. They do not approve positive IPD role grants, admission/bed sequencing, cancellation/discharge or any broader clinical workflow.

## Admission-detail, department-history and successful-audit verification — merged 2026-11-28

- **Admission-detail isolation:** API PR #147 adds PostgreSQL-backed tests proving a Hospital-scoped member cannot read a sibling Hospital's admission detail, even when requesting the actual Hospital ID, and a Tenant-wide member in another Tenant cannot read the foreign admission. Both paths return non-disclosing 404 without an admission DTO. Merged as `031dfab96a6e7f98dc34fd613b36dade4809c004`; exact-head Release build/test, Docker Integration Verification and the additional build/test workflow passed.
- **Department-level placement-history privacy:** API PR #149 creates an admission in Department A, transfers it to Department B under Hospital-scoped service access, then reads it with a Department B-only membership. The reader sees the current Department B placement but not the historical Department A movement. Merged as `8c9da9b48aac855f51f26a0be9f9c419b243185b`; exact-head Release build, EF model-drift guard, 12/12 unit tests, 253/253 PostgreSQL integration tests, Docker Integration Verification and additional build/test passed.
- **Successful bed-operation audit transaction:** API PR #148 invokes the real metadata-only AuditEventMiddleware/service transaction with a synthetic, explicitly scoped test actor. It verifies successful first-bed assignment and same-Hospital transfer each commit the placement/history and audit event together, with validated Tenant/OIDC issuer/subject, endpoint action, IPD resource type, admission ID, route, successful outcome/status and correlation ID. Merged as `74c0a607e3fcb1f1cd4e983822bb690b6684b95c`; exact-head Release build, EF model-drift guard, 12/12 unit tests, 252/252 PostgreSQL integration tests, Docker Integration Verification and additional build/test passed. No role grant or HTTP authorization policy was added.
- **Fresh Local current-main verification:** Local PR #48 merged as `668cee45f46d9313d968ad64b0385f43ae3c797d`. Full-stack run [#46116355864](https://github.com/HeapCare/HeapCare.Local/actions/runs/46116355864) passed with API SHA `39c40bb7ccdb899b23684b18595293c331a79078` (API main including PRs #143–#146), Web SHA `afb61cfbd77aab837ed65dcb30043519c27a5684` and Observability SHA `84bc2ec84de9a1378ac376491d6f6e38dfdcabb6`. It passed nine-service startup, IPD schema assertions, real-Keycloak negative authorization for all five tenant roles and platform-only PlatformOperator across all five IPD routes, OTel/Npgsql/Collector verification, PostgreSQL logical backup/restore of 28 tables with matching row counts, 11/11 Chromium OIDC tests, 6/6 Firefox and 6/6 Playwright WebKit authenticated accessibility tests, and cleanup. This is negative authorization and Linux CI evidence only, not positive clinical workflow approval or manual accessibility certification.

These PRs add test coverage only. They do not change runtime business behavior, database schema/migrations, IPD permission grants or hospital policy. All six IPD permissions remain ungranted.
## IPD audit-failure rollback — merged 2026-11-28

API PR #150 adds PostgreSQL failure-injection regressions against the real `AuditEventMiddleware` and InpatientService:
- A bed assignment that succeeds in the service but encounters an audit-table insert failure returns 503; the new bed placement and audit event are both rolled back.
- A transfer that succeeds in the service but encounters an audit-table insert failure returns 503; the original active placement remains active, no destination placement persists and no partial movement history is written.
- The test-only PostgreSQL trigger/function are removed in `finally`; the injected failures are expected test conditions, not production errors.

PR #150 merged as `4b07eb11c5063288751e1e8a507ce9dc31aafda3`. Exact-head validation passed Release build, EF model-drift guard, 12/12 unit tests, 257/257 PostgreSQL integration tests, Docker Integration Verification and the additional build/test workflow.

This completes regression coverage for both successful and failed IPD bed-operation audit transactions. It does not grant any IPD role permission or change runtime workflow policy.
## Validation checklist

- [x] Admission service only accepts a patient registered at the selected Hospital (API PR #139; authorization remains permission-gated).
- [x] IPD bed transfer to a sibling Hospital or foreign Tenant returns a non-disclosing 404 and leaves the active placement unchanged (API PR #143).
- [x] Admission-detail reads across a sibling Hospital and across Tenants return non-disclosing 404 responses (API PR #147). Other cross-Tenant resource-reference paths remain separately gated and must be tested as those workflows are introduced.
- [x] Hospital-scoped admission-list reads cannot access a sibling Hospital under the same Tenant; sibling requests return a non-disclosing 404 (API PR #141; PostgreSQL integration test).
- [x] Hospital-scoped admission list supports pagination and department visibility by current placement; out-of-scope placement history is omitted (API PR #140; route remains permission-gated).
- [x] Current tenant roles and platform-only PlatformOperator are denied all IPD HTTP routes with real Keycloak-issued tokens; API and Local tests preserve the fail-closed default (API PR #137; Local PR #46).
- [x] Inactive beds are rejected before creating a placement (API PR #136 PostgreSQL service test).
- [x] Transfer into an occupied bed returns 409 and leaves both existing admissions' placements unchanged (API PR #145 PostgreSQL integration test).
- [x] Concurrent attempts to occupy one bed produce one winner and safe conflicts for others (API PR #136 PostgreSQL service tests).
- [x] Transfer closes the old placement and opens the new placement atomically (API PR #136 PostgreSQL service tests).
- [x] Concurrent transfers cannot leave two current placements for one admission; one succeeds and one conflicts (API PR #144).
- [ ] Cancellation and discharge enforce allowed lifecycle transitions.
- [ ] Discharge releases the bed and preserves historical placement data.
- [x] Denied admission reads preserve validated tenant/actor attribution in metadata-only audit records (API PR #138).
- [x] Successful admission creation records the created Admission ID rather than its parent Hospital ID as the audit resource (API PR #142).
- [x] Denied admission read, bed assignment and bed transfer HTTP requests are verified for validated Tenant/actor and metadata-only audit attribution (API PRs #138 and #146).
- [x] Successful first-bed assignment and same-Hospital bed transfer commit their business placement changes with metadata-only audit events containing validated actor, Tenant, admission resource ID, route, outcome/status and correlation ID (API PR #148).
- [x] If audit persistence fails during bed assignment, the request returns 503 and leaves no placement; if audit persistence fails during transfer, the original active placement and history remain unchanged (API PR #150).
- [x] Successful admission creation records the generated Admission ID as its audit resource (API PR #142); the full positive HTTP/OIDC admission workflow remains gated by approved role grants.
- [x] PostgreSQL migration rehearsal passed for schema-bearing IPD increments. Local real-Keycloak full-stack negative authorization passed and was re-run in Local PR #48 against API main after API PRs #143–#146; five current IPD routes remain denied for all persisted tenant roles and platform-only PlatformOperator.

## Open policy decisions

- Admission before bed assignment versus admission and bed assignment together.
- Mandatory admission data and configurable reason catalogue.
- Role-to-permission mapping for each IPD operation.
- Cancellation eligibility and reason requirements.
- Bed reservation, readiness, housekeeping, isolation and maintenance.
- Future cross-Hospital transfer and patient-record visibility.
- IPD billing and discharge-settlement hand-off.
