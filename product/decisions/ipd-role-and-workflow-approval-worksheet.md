# IPD Role and Workflow Approval Worksheet

**Status:** Pending hospital, product, clinical and security approval.  
**Prepared:** 2026-11-28  
**Related:** HeapCare.Doc issue #200; ADR-018; ADR-019; `security/persisted-role-permission-matrix.md`.

> **No permissions are approved by this worksheet.** Every IPD permission remains ungranted to all existing tenant roles. The routes must continue to fail closed until this worksheet is completed, reviewed and the resulting role matrix is accepted through a separate implementation change.

## 1. Current technical permission set

The API defines these dedicated permissions. They are intentionally separate from OPD, patient registration, facility management and billing permissions.

| Permission | Operation covered | Current grant state |
|---|---|---|
| `inpatient.read` | Read admission details, list admissions and visible bed-placement history | Not granted |
| `inpatient.admit` | Create an inpatient admission for a patient registered at the selected Hospital | Not granted |
| `inpatient.bed.assign` | Assign the first bed to an active admission | Not granted |
| `inpatient.bed.transfer` | Transfer an active admission to another eligible bed within the same Hospital | Not granted |
| `inpatient.admission.cancel` | Cancel an admission under an approved cancellation policy | Not granted |
| `inpatient.admission.discharge` | Discharge an admission and release its active placement | Not granted |

The current Tenant role resolver contains five persisted tenant roles: HospitalAdmin, FacilityManager, FacilityViewer, PatientRegistrar and ClinicalUser. PlatformOperator is a separate global identity, not a sixth tenant role.

## 2. Role-to-operation approval matrix

Fill each cell only after the relevant hospital owner has explicitly approved or rejected that operation for that role. **Pending means denied.** A role's existing OPD, billing, patient-registration, facility or platform permissions must not be treated as an implied IPD grant.

| Role | Read | Admit | Assign bed | Transfer bed | Cancel | Discharge |
|---|---|---|---|---|---|---|
| HospitalAdmin | Pending — deny | Pending — deny | Pending — deny | Pending — deny | Pending — deny | Pending — deny |
| FacilityManager | Pending — deny | Pending — deny | Pending — deny | Pending — deny | Pending — deny | Pending — deny |
| FacilityViewer | Pending — deny | Pending — deny | Pending — deny | Pending — deny | Pending — deny | Pending — deny |
| PatientRegistrar | Pending — deny | Pending — deny | Pending — deny | Pending — deny | Pending — deny | Pending — deny |
| ClinicalUser | Pending — deny | Pending — deny | Pending — deny | Pending — deny | Pending — deny | Pending — deny |
| PlatformOperator | Deny tenant clinical access by default | Deny | Deny | Deny | Deny | Deny |

For every approved cell, record the permitted organization scope and any constraints below. An operation grant does not itself establish access to all hospitals, departments, patients or historical records.

## 3. Scope approval for each granted operation

For each approved role/permission pair, choose one or more explicit scopes and explain the resource boundary:

| Scope | Meaning to approve | Safeguard |
|---|---|---|
| Tenant-wide | Across the hospital organization | Must not silently expose every site's patient or clinical records |
| Hospital | One specifically assigned physical Hospital | Must not include sibling hospitals under the same Tenant |
| Department | One or more explicitly assigned departments in one Hospital | Must not reveal unplaced admissions or movement history outside the assigned department |
| FacilitySite | Proposed future site scope under ADR-019 | Not implemented; do not grant or assume inheritance |
| Cross-hospital | Not part of the initial IPD increment | Requires a separate patient-sharing and transfer decision |

Record any permitted scope inheritance explicitly. Parent/child site relationships, tenant membership and PlatformOperator status do not automatically grant clinical or patient access.

## 4. Workflow decisions required before implementation is enabled

### Admission

- [ ] Confirm whether an Active admission may be created before a bed is assigned (for example, while awaiting a bed), or whether the operation must be atomic with bed assignment.
- [ ] Confirm the minimum mandatory admission fields.
- [ ] Approve an admission-reason catalogue and decide whether hospital-specific values are configurable.
- [ ] Define how duplicate, cancelled and discharged episodes are handled when a patient returns.

Current technical behavior: the API can create an Active episode for a patient registered at the exact selected Hospital; the newly created episode has no bed placement. This is an implementation capability, not hospital policy approval.

### Bed management and transfer

- [ ] Confirm whether assignment is limited to active/ready beds and define how blocked, occupied, inactive and under-maintenance beds are represented.
- [ ] Decide whether bed reservations are required or whether occupancy begins only on confirmed placement.
- [ ] Define who may override a bed block and how the override is reasoned and audited.
- [ ] Confirm the allowed transfer boundary: same ward, cross-ward and cross-department within one Hospital.
- [ ] Confirm whether cross-hospital transfers are excluded from the first release.

Current technical behavior: assignment and transfer are limited to the same Hospital; each admission can have at most one active placement and each bed at most one active occupant. Transfers preserve prior placement history.

### Cancellation

- [ ] Define the exact point at which cancellation is allowed.
- [ ] Decide whether an admission with an active bed placement can ever be cancelled, or whether a separate reversal/early-discharge workflow is required.
- [ ] Approve mandatory cancellation reasons, actor constraints and any approval/escalation process.
- [ ] Define how cancellation affects bed occupancy and subsequent readmission.

### Discharge

- [ ] Define who can discharge an inpatient and whether a second-person approval is required.
- [ ] Approve mandatory discharge data and reason codes.
- [ ] Define whether discharge requires an active placement, and how a patient admitted without a bed is handled.
- [ ] Define the effective discharge timestamp and any correction/reversal process.
- [ ] Define the hand-off to billing, deposits and settlement as a separate integration decision. No billing action is implied by the ADT event.

### Patient/site sharing and audit

- [ ] Confirm patient discovery and MRN rules across multiple Hospitals under one Tenant.
- [ ] Confirm whether any explicit cross-hospital access grant is permitted and who may approve it.
- [ ] Confirm audit review ownership, retention, legal hold and export controls.
- [ ] Confirm the minimum metadata needed for admission, bed placement, transfer, cancellation and discharge audit records.

## 5. Required evidence before role grants

- Positive and negative PostgreSQL authorization tests for every approved role/operation pair.
- Tenant, hospital, department and (when implemented) site-scope regression tests.
- Tests proving token-supplied role/permission claims cannot elevate privileges.
- Real-Keycloak Local full-stack verification using persisted identities.
- Audit verification for successful, denied, rejected and failed operations.
- Clinical/operational sign-off of lifecycle transitions and reason codes.
- Security/privacy review of the minimum-necessary access model.
- Web workspace validation after API permissions and contracts are approved.

## 6. Approval record

Complete this section without including patient data, credentials or other secrets.

| Approval item | Record |
|---|---|
| Pilot hospital organization | Pending |
| Representative Hospital/site hierarchy | Pending |
| Hospital operations owner | Pending |
| Clinical governance owner | Pending |
| Patient administration / admissions owner | Pending |
| Information security / privacy reviewer | Pending |
| Finance owner for billing hand-off | Pending |
| Decision date and revision | Pending |
| Approved matrix version | Pending |

## 7. Implementation gate

Until this worksheet and the associated decisions are approved:

1. Keep every IPD permission ungranted to existing tenant roles.
2. Keep PlatformOperator separate from tenant clinical authorization.
3. Continue implementing policy-neutral persistence, isolation, audit, concurrency and test coverage.
4. Do not implement cancellation/discharge eligibility or required reason rules by assumption.
5. Do not enable a positive IPD workspace flow in Web.
6. Do not close HeapCare.Doc issue #200 or claim the full IPD workflow is complete.

The approval must be followed by a focused code change, updated source-derived role matrix, PostgreSQL positive/negative authorization tests, Local real-Keycloak acceptance and Web end-to-end verification.
