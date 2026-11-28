# IPD: Admission, Transfer, Discharge and Bed Management

**Status:** Initial ADT scope accepted; detailed hospital policy remains under review.  
**Related:** [ADR-018: Inpatient ADT and Bed-Management Boundaries](../../architecture/decisions/ADR-018-inpatient-ADT-bed-management-boundaries.md), [ADR-006: Tenant, Organization and Patient Data Boundaries](../../architecture/decisions/ADR-006-tenant-organization-patient-boundaries.md).

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

## Validation checklist

- [ ] Admission only accepts a patient registered at the selected Hospital.
- [ ] Cross-Tenant and cross-Hospital references return a non-disclosing not-found/denied result.
- [ ] Unauthorized roles cannot read or mutate IPD resources.
- [ ] An inactive or occupied bed cannot receive a new active placement.
- [ ] Concurrent attempts to occupy one bed produce one winner and safe conflicts for others.
- [ ] Transfer closes the old placement and opens the new placement atomically.
- [ ] Concurrent transfers cannot leave two current placements for one admission.
- [ ] Cancellation and discharge enforce allowed lifecycle transitions.
- [ ] Discharge releases the bed and preserves historical placement data.
- [ ] Audit records validated actor attribution without patient/clinical content.
- [ ] PostgreSQL migration rehearsal and Local real-Keycloak full-stack verification pass.

## Open policy decisions

- Admission before bed assignment versus admission and bed assignment together.
- Mandatory admission data and configurable reason catalogue.
- Role-to-permission mapping for each IPD operation.
- Cancellation eligibility and reason requirements.
- Bed reservation, readiness, housekeeping, isolation and maintenance.
- Future cross-Hospital transfer and patient-record visibility.
- IPD billing and discharge-settlement hand-off.
