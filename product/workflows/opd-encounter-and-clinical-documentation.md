# Workflow: OPD encounter and clinical documentation

**Status:** Proposed for pilot validation  
**Related decision:** [ADR-015 — OPD encounter and clinical documentation boundaries](../../architecture/decisions/ADR-015-opd-encounter-and-clinical-documentation-boundaries.md)

## Purpose

Record a clinician-led outpatient interaction after front-desk check-in and queue hand-off. The encounter is a clinical record, not an appointment or queue status.

## Actors

- Reception / queue operator
- Clinician or other explicitly authorized clinical user
- Hospital administrator / authorized clinical access administrator
- Authorized auditor

## Preconditions

- Tenant and hospital are active.
- Patient has a valid registration at the selected hospital.
- The patient has a queue entry for the same tenant, hospital, department and registration.
- The queue entry is in `InProgress`.
- The acting user has an explicitly provisioned membership permission and matching organization scope.
- Pilot role mapping and clinical documentation policy have been approved.

## A. Start an OPD encounter

1. The clinical user opens the department's in-progress queue.
2. The user verifies the patient using the hospital registration and queue context.
3. The API validates tenant, hospital, department, patient and queue-entry relationships.
4. The API creates one encounter for the queue entry and records the authenticated issuer/subject and start time.
5. A database uniqueness constraint prevents a second encounter from being created for the same queue entry.
6. The encounter begins in `InProgress`; the queue entry remains independently `InProgress`.

Appointment-linked check-ins and walk-ins use the same workflow. The encounter does not require an appointment.

## B. Record a draft note

1. The authorized clinical user records the approved note sections: chief complaint, history, examination, assessment and plan.
2. The API enforces field limits and stores only the clinical content needed for the encounter.
3. If enabled by the pilot policy, the user records coded vital observations separately with value, unit, observation time and recorder.
4. Draft content may be updated by authorized users while the encounter remains in progress.
5. The API records attributable audit events for reads and writes without copying clinical content into general telemetry.

The first increment does not generate diagnoses, treatment suggestions, triage scores, prescriptions, referrals or diagnostic orders.

## C. Sign the encounter

1. The authorized signer reviews the note and available observations.
2. The API verifies `encounter.sign`, tenant/hospital/department scope and the encounter lifecycle.
3. The API records signer issuer/subject and signing timestamp.
4. The encounter becomes `Signed`; normal update endpoints reject changes to signed content.
5. The queue entry remains independently in progress until an authorized queue operator completes the operational hand-off.

## D. Amend a signed encounter

1. An authorized user with `encounter.amend` supplies an amendment reason and correction/addendum.
2. The API creates an append-only amendment linked to the signed encounter, recording author issuer/subject and timestamp.
3. The original signed note remains unchanged and is returned as the authoritative original plus its amendment history.
4. Amendment access and mutation are audited.

## E. Close or cancel

- Closing is a separate state transition after the clinical workflow is complete; it does not rewrite the signed note.
- Cancellation is allowed only for an encounter that has not been signed and requires an explicit reason.
- Queue completion remains a separate front-desk/operational action; signing an encounter does not silently complete or reorder the queue.

## F. Department-scoped clinical patient discovery

1. A clinician searches within the department assigned to their active tenant membership using the hospital's department-scoped patient lookup endpoint.
2. The API verifies the requested hospital and department against the caller's persisted organization scope; a department ID supplied by the caller does not grant access.
3. Search results include only patients with a hospital registration and at least one queue entry in that same hospital and assigned department. Registration at the hospital alone does not expose the patient to every department.
4. Results use bounded pagination and the existing patient identity-read permission. Hospital-wide reception search remains a separate permission and endpoint.
5. A clinician assigned to another department, or to no matching department, receives no patient results (not-found for an out-of-scope department).

The initial implementation is intended for clinical continuity through department queue history. It does not enable cross-hospital patient discovery or grant access to clinical notes by itself; OPD encounter permissions and encounter department checks remain independently enforced.

## Security and privacy

- All reads and writes are tenant-, hospital- and department-scoped.
- Clinical permissions are independent from appointment and queue permissions.
- Record the validated identity issuer and subject for each clinical author and signer. A staff designation is not credential verification.
- Audit clinical record access and material actions. Keep note text, vital values and patient identifiers out of general logs, traces and metric labels.
- Signed content is immutable through normal APIs; amendments are append-only.
- Production activation is blocked on pilot validation of roles, note templates, observation policy, amendment/retention rules, privacy notices, emergency access and applicable legal requirements.

## Out of scope

Diagnosis catalogues and coding, allergy/medication reconciliation, prescriptions, lab/radiology orders, referrals, procedures, clinical decision support, triage, emergency and inpatient workflows, discharge summaries, patient portal access, attachments and external interoperability.
