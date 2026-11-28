# ADR-018: Inpatient ADT and Bed-Management Boundaries

- **Status:** Accepted initial delivery scope; detailed admission policy and role matrix remain open.
- **Date:** 2026-11-28
- **Decision owners:** HeapCare Product & Engineering; hospital operations and clinical stakeholders validate workflows.
- **Related:** [ADR-006: Tenant, Organization and Patient Data Boundaries](ADR-006-tenant-organization-patient-boundaries.md), [ADR-015: OPD Encounter and Clinical Documentation Boundaries](ADR-015-opd-encounter-and-clinical-documentation-boundaries.md), [IPD admission, transfer, discharge and bed-management workflow](../../product/workflows/inpatient-admission-transfer-discharge-and-bed-management.md).

## Context

The current platform has a tenant-scoped Hospital → Department → Ward → Room → Bed hierarchy, hospital patient registrations, staff assignments, OPD encounters and audited role-scoped APIs. It does not yet provide an inpatient admission or bed-occupancy workflow.

The first inpatient increment must establish a reliable administrative ADT foundation without inventing nursing or clinical policies.

## Accepted initial scope

The first IPD increment is limited to:

1. Admission of an existing patient registered at the selected hospital.
2. An inpatient episode with explicit lifecycle and validated actor attribution.
3. Bed placement and current occupancy, where the hospital has confirmed a bed assignment.
4. Transfer between eligible beds/wards within the same hospital, preserving placement history.
5. Discharge and permitted admission cancellation, preserving the episode and movement history.
6. Hospital-scoped lists and details with tenant, hospital and department/site authorization.
7. Append-only audit evidence for admission, placement, transfer, cancellation and discharge.

This is an administrative ADT and bed-management foundation. It is not a clinical decision-support or nursing system.

## Data and security invariants

- Reuse the existing Tenant-owned Patient and HospitalPatientRegistration. Do not create a second patient identity for IPD.
- An admission and all its bed placements belong to one Tenant and one originating Hospital in the initial increment.
- A patient must have a valid registration at the selected Hospital before an admission can be created.
- The bed hierarchy must resolve to the same Tenant and Hospital as the admission. A bed from another Tenant or Hospital must never be accepted by changing an ID in the request.
- A bed may have at most one active occupant at a time.
- An admission may have at most one current bed placement at a time.
- A transfer closes the prior placement and opens the new placement atomically. On conflict or failure, neither side of the transfer is persisted.
- Discharge releases the active bed placement while preserving the admission and prior placements.
- Concurrent admission/transfer attempts must be protected by PostgreSQL constraints and transactional service logic; an application-only availability check is insufficient.
- Admission and placement records are tenant-owned, concurrency-protected and not physically deleted after clinical/operational use.
- Every mutating operation records the validated issuer/subject actor and a metadata-only audit event. Sensitive clinical content must not be copied into audit metadata or telemetry.
- Cross-hospital transfer, cross-hospital patient visibility and cross-Tenant sharing are not part of the initial increment.

## Explicit exclusions

The initial increment does not implement:
- Medication orders, prescription, dispensing or medication administration records (MAR).
- Nursing notes, observations/vitals, clinical assessments or nursing care plans.
- Clinical orders, procedure plans or clinical decision support.
- Clinical discharge summaries or discharge prescriptions.
- Pharmacy, laboratory, radiology, PACS, inventory or procurement.
- Patient/guardian portal, payer/TPA/ABDM exchange or insurance workflows.
- Automatic invoice, deposit, payment, refund or statutory billing changes.
- Bed reservation, housekeeping, cleaning/readiness, isolation, maintenance or bed-blocking policy unless separately approved.

These capabilities require separate domain workflows, clinical/safety review, data policies, permission matrices and acceptance criteria.

## Authorization boundary

Use dedicated IPD permissions for reading admissions, admitting, assigning/transferring beds, cancelling and discharging. Do not reuse OPD encounter or billing permissions as a shortcut. The exact mapping of these permissions to the six persisted tenant roles must be reviewed before routes are enabled for production users.

All reads and writes must preserve active Tenant membership, facility/department scope, fail-closed resource lookup, metadata-only audit attribution, cross-Tenant denial and cross-Hospital denial.

## Engineering acceptance

- Application/domain contracts express permitted lifecycle transitions; clients cannot set arbitrary status values.
- Composite Tenant/Hospital/Department/Bed relationships prevent cross-scope references.
- Database constraints prevent duplicate active bed occupancy and duplicate active placement for an admission.
- PostgreSQL integration tests cover admit, list/detail, assignment, transfer, cancellation, discharge, invalid transitions, inactive beds, wrong-hospital beds, wrong-tenant resources and concurrency races.
- Fresh database migration, upgrade migration and idempotent migration-script verification pass.
- Unit tests, API authorization tests, audit verification and Local real-Keycloak integration run before merge.
- Web workflows are added only after API contracts and permissions are accepted.

## Open policy questions

1. Can an admission be created before a bed is assigned (for example, while the patient is awaiting a bed), or must admission and bed assignment occur together?
2. What patient and admission fields are mandatory, and which admission reasons are configurable?
3. Which roles can admit, assign/transfer beds, cancel, discharge, override a blocked bed and read admission details?
4. Is cancellation allowed only before physical admission, or is a distinct early-discharge/reversal workflow required?
5. What are the hospital's bed reservation, readiness, housekeeping, isolation and maintenance policies?
6. Should future transfers between hospitals within the same Tenant be supported, and what record-sharing/authorization rules would apply?
7. What is the billing hand-off for deposits, IPD charges, package rates and discharge settlement?

Do not implement policies for these open questions by assumption. Continue with the policy-neutral design and test scaffolding while obtaining the necessary decisions.
