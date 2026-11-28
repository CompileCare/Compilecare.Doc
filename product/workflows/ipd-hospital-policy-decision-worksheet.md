# IPD Hospital Policy Decision Worksheet

**Status:** Draft for hospital/operations review — not an approved clinical or operational policy.  
**Date:** 2026-11-28  
**Related:** HeapCare.Doc issue #200; [ADR-018: Inpatient ADT and Bed-Management Boundaries](../../architecture/decisions/ADR-018-inpatient-ADT-bed-management-boundaries.md); [IPD workflow](inpatient-admission-transfer-discharge-and-bed-management.md); [facility-site model proposal](../../architecture/decisions/ADR-019-facility-site-model-proposal.md).

## Purpose and approval boundary

The API now contains a policy-neutral technical foundation for admission creation, admission list/detail, first-bed assignment and same-hospital bed transfer. It also has PostgreSQL coverage for tenant/hospital isolation, department-history privacy, occupancy conflicts, concurrent transfers, and successful/denied/audit-failure transaction behavior.

The next lifecycle slice cannot be completed safely by inferring hospital policy from the current database shape. This worksheet makes the outstanding decisions explicit. It is a discussion aid only:

- No option below is approved by this document.
- No role permission is granted by this document.
- A hospital administrator, operational owner and appropriately qualified clinical stakeholder must validate the workflow and reason requirements.
- Until approval is recorded in an accepted ADR, all six IPD permissions remain ungranted to tenant roles and PlatformOperator receives no implicit tenant clinical access.

## 1. Admission and bed-assignment sequence

**Decision required:** Can an inpatient episode be created before a physical bed is assigned?

| Option | Meaning | Product and engineering impact |
|---|---|---|
| A — Separate steps | Hospital accepts the inpatient admission first. The episode may be active without a current bed placement while the patient awaits allocation. | Matches the current API/schema shape. Requires the workspace to distinguish “admitted, awaiting placement” from “placed”. Bed occupancy begins only when a placement is recorded. |
| B — Atomic admission and placement | Admission is accepted only when an eligible bed can be assigned in the same operation. | Requires an atomic combined command and failure handling when the destination bed is unavailable. Does not represent accepted admissions waiting for a bed without another workflow. |
| C — Explicit pre-admission queue | A patient may wait for a bed, but that state is not yet an inpatient admission. A separate accepted-admission action creates the inpatient episode. | Requires a distinct waiting/request entity and lifecycle, rather than treating a waiting request as an admission. |

**Recommended discussion baseline (not approved):** Option A is the smallest extension of the existing model and represents a real “accepted admission awaiting placement” state without pretending a bed is occupied. Confirm whether the hospital recognizes that state as an admission under its own policy before retaining it.

Record the chosen option, the event that legally/operationally starts an admission, who may perform it, and whether an admission without a placement appears in operational lists and reports.

## 2. Admission data and reason codes

**Decisions required:**

| Data point | Decision to record | Guardrail |
|---|---|---|
| Patient identity | Confirm that the existing Patient and HospitalPatientRegistration are sufficient. | Do not create a second patient identity for IPD. |
| Hospital | Confirm the selected Hospital is the admitting facility. | Do not admit against a registration at a sibling Hospital or another Tenant. |
| Admission effective time | Confirm whether server-recorded acceptance time is sufficient or a separately authorized effective time is required. | The API currently generates UTC time; clients do not set lifecycle timestamps. |
| Admission reason | Decide whether a controlled administrative reason code is mandatory, optional or deferred. | Do not place diagnosis, clinical narrative or sensitive details in general audit metadata. |
| Responsible provider | Decide whether an admitting/attending provider is required at admission or may be assigned later. | A staff profile or designation alone does not establish verified clinical privilege. |
| Referral/source | Decide whether referral source or transfer-in information is needed for the initial administrative workflow. | Keep the initial data set minimal and operationally necessary. |

Avoid free-form clinical notes in the ADT command. Clinical documentation, orders and discharge summaries remain separately scoped and are not introduced through this workflow.

## 3. Cancellation versus discharge

**Decision required:** Define the point after which an accepted admission can no longer be cancelled and must instead be discharged.

| Option | Cancellation | Discharge |
|---|---|---|
| A — Before first placement | Cancellation is allowed only while the admission has never had a bed placement. After placement, use discharge. | Discharge closes the active episode and, if present, its current placement. |
| B — Before physical arrival | Cancellation is allowed until a hospital-defined arrival/acceptance event, even if a bed was provisionally assigned. | Discharge applies after physical admission. Requires an explicit arrival/occupancy event not currently modeled. |
| C — Hospital-configured stages | The hospital defines named milestones and cancellation windows. | Each milestone has a distinct, approved transition and audit event. |

**Recommended discussion baseline (not approved):** Option A is the clearest fit for the existing model because it has no separate arrival or bed-reservation state. It must not be adopted if the hospital assigns beds before physical arrival and needs to cancel those assignments without treating them as an inpatient discharge.

For each transition, approve:
- permitted source status and destination status;
- whether a bed placement may be active or may exist in history;
- whether a reason is mandatory and whether reason codes are controlled;
- who may initiate the operation and who may approve exceptions;
- whether the transition can be reversed, and by which separate audited action;
- required effective time and whether backdating is permitted.

A cancellation or discharge must never physically delete admission or placement history.

## 4. Bed assignment, readiness and reservation

The current model distinguishes whether a bed record is active from whether it has an active inpatient occupant. It does not represent clean/ready, maintenance, isolation, blocked or reserved states.

**Decisions required:**
1. Does a bed placement mean actual occupancy, or can it reserve a bed before arrival?
2. Must a bed be marked ready/clean before assignment?
3. Which role may block/unblock a bed for cleaning, maintenance or isolation?
4. Is a bed allowed to become inactive while occupied, and what transition is required first?
5. Are transfers restricted to currently active beds, or are additional readiness/clinical constraints required?

**Recommended discussion baseline (not approved):** Treat the existing active placement as actual occupancy and defer reservation, housekeeping and maintenance workflows to a separate capability. Do not overload Bed.IsActive with several operational states. If pre-arrival reservation is required, model it explicitly so reserved capacity is not reported as occupied.

## 5. Six-role IPD permission matrix

Permissions are separate and already defined in the Application layer:

- inpatient.read
- inpatient.admit
- inpatient.bed.assign
- inpatient.bed.transfer
- inpatient.admission.cancel
- inpatient.admission.discharge

The following matrix intentionally contains no grants. Hospital owners must approve the exact positive and negative mapping, including whether read access includes placement history.

| Role | Read | Admit | Assign bed | Transfer bed | Cancel | Discharge |
|---|---|---|---|---|---|---|
| PlatformOperator | Pending; default deny | Pending; default deny | Pending; default deny | Pending; default deny | Pending; default deny | Pending; default deny |
| HospitalAdmin | Pending approval | Pending approval | Pending approval | Pending approval | Pending approval | Pending approval |
| FacilityManager | Pending approval | Pending approval | Pending approval | Pending approval | Pending approval | Pending approval |
| FacilityViewer | Pending approval | Pending approval | Pending approval | Pending approval | Pending approval | Pending approval |
| PatientRegistrar | Pending approval | Pending approval | Pending approval | Pending approval | Pending approval | Pending approval |
| ClinicalUser | Pending approval | Pending approval | Pending approval | Pending approval | Pending approval | Pending approval |

**Scope must be approved separately from role.** For each positive grant, specify whether it is Tenant-wide, Hospital-wide or Department-only, and which resource operations are permitted at that scope. A Tenant-wide membership must not silently become permission to inspect every hospital's patient/clinical records. Do not infer cross-hospital access from a shared customer Tenant. PlatformOperator is a global platform identity and must not gain tenant clinical access by virtue of that role.

## 6. Site and organization boundaries that affect IPD

The accepted customer boundary is Tenant: one Tenant may represent a hospital organization operating one or multiple hospitals, branches/campuses and clinics. Hospital remains the physical-hospital record in the current API.

Before IPD is enabled across multiple sites, decide:
- whether a branch is a separately managed site, a child campus of a Hospital or a reporting/location label;
- whether a clinic is a first-class site or a department/service inside a Hospital;
- whether a site can provide IPD and therefore own beds, wards and inpatient admissions;
- whether patient registration and MRN uniqueness are Hospital-specific, site-specific or organization-wide;
- whether patient discovery across hospitals is allowed, and for which roles and purposes;
- how staff assignment/temporary coverage and billing/reporting/audit boundaries work across sites.

Until these are accepted, retain the existing same-Hospital admission and bed-transfer constraints. Do not introduce a HospitalGroup entity or widen patient visibility as a shortcut.

## 7. Billing and clinical boundaries

The initial ADT slice must not automatically create invoices, deposits, payments, refunds, packages or insurance claims. Hospital owners may define a future hand-off event, but it belongs in a separately reviewed billing workflow.

The following remain outside the first IPD ADT increment: medication orders/MAR, nursing notes, observations/vitals, clinical orders, diagnostics, pharmacy, clinical discharge summaries, patient portal, payer/TPA/ABDM integration and clinical decision support. Each requires separate domain, privacy, role, safety and acceptance review.

## 8. Approval and implementation sequence

1. Record answers for sections 1–6, including named approvers and the effective date of the decision.
2. Update ADR-018 with the accepted policy; do not treat a comment or proposed default as approval.
3. Update the role/permission matrix and API authorization tests. Keep every unapproved capability denied.
4. Implement cancellation/discharge service transitions and actor-attributed audit events only after their eligibility/reason rules are accepted.
5. Add PostgreSQL tests for duplicate/concurrent lifecycle transitions, placement closure, history retention, audit success and audit failure rollback.
6. Verify the exact API main SHA in the real-Keycloak Local stack before enabling the corresponding Web workflow.
7. Build the Web IPD workspace only against accepted API contracts and permissions; complete role-based workflow and accessibility regression.

## Current engineering state

The merged API already supports guarded admission creation, paginated admission list, admission detail, first-bed assignment and same-Hospital transfer. The Local integration verifies IPD routes remain denied for existing tenant roles and platform-only PlatformOperator. Automated PostgreSQL tests cover the technical invariants listed in ADR-018. The missing work is lifecycle policy and implementation, approved positive role grants, the full positive real-Keycloak workflow and the Web IPD workspace.

This worksheet does not change any runtime behavior, schema, migration, authorization grant or production configuration.
