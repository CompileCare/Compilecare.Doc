# IPD: Admission, Transfer, Discharge and Bed Management

**Status:** Initial ADT scope accepted; admission creation, paginated admission list/detail reads and bed placement/transfer technical increments are merged. Detailed hospital policy and role grants remain under review.  
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
- **Validation:** both PRs passed 12/12 unit tests and 247/247 PostgreSQL integration tests, Release build, EF model-drift guard, PostgreSQL migration rehearsal and Docker Integration Verification.
- **Real-Keycloak negative authorization:** Local PR #46 is merged. Its exact-head full-stack run [#46114200037](https://github.com/HeapCare/HeapCare.Local/actions/runs/46114200037) denied every IPD route for all five persisted tenant roles and platform-only PlatformOperator while also passing nine-service startup, PostgreSQL logical backup/restore, Chromium PKCE and Firefox/WebKit six-role accessibility baseline. Existing IPD permissions remain ungranted. API PR #137's PostgreSQL suite separately covers a PlatformOperator who also has a tenant FacilityViewer membership.

These are automated isolation, concurrency and negative-authorization proofs. They do not approve positive IPD role grants, admission/bed sequencing, cancellation/discharge or any broader clinical workflow.

## Occupied-bed conflict and denied bed-operation audit — merged 2026-11-28

- **Occupied-bed transfer rejection:** API PR #145 assigns two synthetic active admissions to different beds, then attempts to transfer one admission into the other's occupied bed. The service returns 409 and the test verifies both original placements remain active and unchanged. Merged as `b7663ee29f231c55f4775b7a473af71eb05bc80b`.
- **Denied bed-operation audit:** API PR #146 calls bed assignment and bed transfer with a persisted FacilityViewer membership that has no IPD permission. Both requests return 403; PostgreSQL audit records retain the validated tenant, issuer/subject, admission resource ID, endpoint/action, route, denied outcome/status and correlation ID. No bed placement is created. Merged as `39c40bb7ccdb899b23684b18595293c331a79078`.
- **Validation:** both PRs passed 12/12 unit and 249/249 PostgreSQL integration tests, Release build, EF model-drift guard, migration rehearsal and Docker Integration Verification.
- **Scope:** these are test-only safety/audit proofs. They do not grant IPD permissions or decide bed readiness, cancellation/discharge or positive clinical workflows.

## Validation checklist

- [x] Admission service only accepts a patient registered at the selected Hospital (API PR #139; authorization remains permission-gated).
- [x] IPD bed transfer to a sibling Hospital or foreign Tenant returns a non-disclosing 404 and leaves the active placement unchanged (API PR #143).
- [ ] Other cross-Tenant references return a non-disclosing not-found/denied result.
- [x] Hospital-scoped admission-list reads cannot access a sibling Hospital under the same Tenant; sibling requests return a non-disclosing 404 (API PR #141; PostgreSQL integration test).
- [x] Hospital-scoped admission list supports pagination and department visibility by current placement; out-of-scope placement history is omitted (API PR #140; route remains permission-gated).
- [x] Current tenant roles and platform-only PlatformOperator are denied all IPD HTTP routes with real Keycloak-issued tokens; API and Local tests preserve the fail-closed default (API PR #137; Local PR #46).
- [x] An inactive or occupied bed cannot receive a new active placement (API PR #136 inactive-bed test; API PR #145 occupied-bed transfer conflict and unchanged-placement test).
- [x] Concurrent attempts to occupy one bed produce one winner and safe conflicts for others (API PR #136 PostgreSQL service tests).
- [x] Transfer closes the old placement and opens the new placement atomically (API PR #136 PostgreSQL service tests).
- [x] Concurrent transfers cannot leave two current placements for one admission; one succeeds and one conflicts (API PR #144).
- [ ] Cancellation and discharge enforce allowed lifecycle transitions.
- [ ] Discharge releases the bed and preserves historical placement data.
- [x] Denied admission reads preserve validated tenant/actor attribution in metadata-only audit records (API PR #138).
- [x] Denied bed assignment and transfer requests preserve validated tenant/actor, admission resource, route/action and correlation in metadata-only audit (API PR #146).
- [x] Successful admission creation records the created Admission ID rather than its parent Hospital ID as the audit resource (API PR #142).
- [ ] Audit records for all remaining IPD reads/mutations are verified for validated actor attribution without patient/clinical content.
- [x] PostgreSQL migration rehearsal passes on merged API increments, and Local real-Keycloak full-stack negative authorization verification passes (API PRs #139–#146; Local PR #46).

## Open policy decisions

- Admission before bed assignment versus admission and bed assignment together.
- Mandatory admission data and configurable reason catalogue.
- Role-to-permission mapping for each IPD operation.
- Cancellation eligibility and reason requirements.
- Bed reservation, readiness, housekeeping, isolation and maintenance.
- Future cross-Hospital transfer and patient-record visibility.
- IPD billing and discharge-settlement hand-off.
