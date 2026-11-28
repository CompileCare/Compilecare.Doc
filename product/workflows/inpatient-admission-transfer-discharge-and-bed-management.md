# IPD: Admission, Transfer, Discharge and Bed Management

**Status:** Initial ADT technical foundation is merged: admission creation/list/detail, first-bed assignment and same-hospital transfer. Hospital isolation, transfer concurrency and real-Keycloak negative authorization regressions are also verified. All IPD role grants remain closed; hospital policy, cancellation/discharge and the positive Web workflow are not complete.  
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

## Transfer isolation and concurrency verification — merged 2026-11-28

- **Cross-Hospital and cross-Tenant transfer targets:** API PR #143 adds PostgreSQL-backed service coverage for an attempt to transfer an admission to a bed in a sibling Hospital under the same Tenant and to a bed in a different Tenant. Both operations return non-disclosing 404 responses. The original placement remains active and unchanged. PR #143 merged as `32e7b56b7969d0b80b8194b3973949ff4865259a`; exact-head Release build/test, Docker Integration Verification and the additional API build/test workflow passed (12/12 unit tests and 247/247 PostgreSQL integration tests).
- **Concurrent transfers:** API PR #144 adds a deterministic PostgreSQL race regression for two transfers beginning from the same active placement. Exactly one operation succeeds, the competing request returns 409, and persisted history contains the closed original placement and one active destination placement. Its test-only PostgreSQL trigger is removed in a `finally` block. PR #144 merged as `c718f0ccd8816158d6740faef5a27656da89dc0a`; exact-head Release build/test, Docker Integration Verification and the additional API build/test workflow passed (12/12 unit tests and 247/247 PostgreSQL integration tests).
- **Real-Keycloak negative authorization:** Local PR #46 merged as `19929c3ec999f3f38d5dd9aa6661916247e74a49`; full-stack run [#46114200037](https://github.com/HeapCare/HeapCare.Local/actions/runs/46114200037) passed the nine-service stack and verified that HospitalAdmin, FacilityManager, FacilityViewer, PatientRegistrar, ClinicalUser and a platform-only PlatformOperator are denied all five current IPD routes with HTTP 403. This is negative-access verification only. It grants no permission and does not prove a positive admission-to-discharge workflow.
- **Role approval:** Docs PR #213 merged as `41a84472d31541cfe2850b5e83bd2b97a1b2d114`, adding the [IPD role and workflow approval worksheet](../decisions/ipd-role-and-workflow-approval-worksheet.md). Each IPD permission remains ungranted pending explicit hospital, clinical, product and security approval.

These changes are isolation and regression evidence only. No schema, migration, runtime behavior, role grant or clinical policy was added.

## Validation checklist

- [x] Admission service only accepts a patient registered at the selected Hospital (API PR #139; authorization remains permission-gated).
- [x] Cross-Hospital and cross-Tenant bed-transfer targets are denied with non-disclosing 404 responses, leaving the original placement unchanged (API PR #143).
- [x] Hospital-scoped admission-list reads cannot access a sibling Hospital under the same Tenant; sibling requests return a non-disclosing 404 (API PR #141; PostgreSQL integration test).
- [x] Hospital-scoped admission list supports pagination and department visibility by current placement; out-of-scope placement history is omitted (API PR #140; route remains permission-gated).
- [x] Existing roles and platform-only PlatformOperator are denied all five current IPD routes in real-Keycloak Local integration (Local PR #46; negative authorization only, no positive role grants).
- [x] Inactive beds are rejected before a placement is created (API PR #136 service regression).
- [ ] Transfer into an occupied bed returns conflict and leaves both existing placements unchanged (API PR #145 validation pending at this documentation checkpoint).
- [x] Concurrent attempts to occupy one bed produce one winner and safe conflicts for others (API PR #136 PostgreSQL regression).
- [x] Transfer closes the old placement and opens the new placement atomically (API PR #136).
- [x] Concurrent transfers cannot leave two current placements for one admission (API PR #144).
- [ ] Cancellation and discharge enforce approved lifecycle transitions.
- [ ] Discharge releases the bed and preserves historical placement data.
- [x] Denied admission reads preserve validated tenant/actor attribution in metadata-only audit records (API PR #138).
- [x] Successful admission creation records the created Admission ID rather than its parent Hospital ID as the audit resource (API PR #142).
- [ ] Audit records for denied bed assignment/transfer HTTP operations are verified for validated actor attribution without patient/clinical content (API PR #146 validation pending at this documentation checkpoint).
- [x] PostgreSQL migration rehearsal passed for the schema-bearing IPD persistence increment; Local real-Keycloak full-stack negative authorization verification passed (Local PR #46).

## Open policy decisions

- Admission before bed assignment versus admission and bed assignment together.
- Mandatory admission data and configurable reason catalogue.
- Role-to-permission mapping for each IPD operation.
- Cancellation eligibility and reason requirements.
- Bed reservation, readiness, housekeeping, isolation and maintenance.
- Future cross-Hospital transfer and patient-record visibility.
- IPD billing and discharge-settlement hand-off.
