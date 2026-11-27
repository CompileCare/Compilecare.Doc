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
4. When opening a queue-linked encounter, the client retrieves the patient's identity through the department-scoped detail route. The API returns the patient only when the caller is assigned to that department and the patient has queue history there.
5. Search and detail reads require the existing patient identity-read permission. Hospital-wide reception search and detail reads remain separate permissioned endpoints.
6. A clinician assigned to another department, or to no matching department, receives no patient results (not-found for an out-of-scope department or patient).

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


## Clinical queue row presentation — 2026-11-25

The clinical queue presentation is extracted into `ClinicalQueueRow` in [CompileCare.Web PR #33](https://github.com/CompileCare/CompileCare.Web/pull/33).

- Rows are named by their operational queue token, not patient identity.
- The API-returned status is announced as a labelled status region.
- The open/resume action has an accessible name that identifies the queue token and distinguishes an in-progress encounter from a completed visit.
- The action is disabled while the selected encounter is opening or the clinical workspace reports a busy operation.
- The row passes the original server-returned `QueueEntry` to the existing `openEncounter` handler. It does not create an encounter, infer its state or change lifecycle transitions.

This is a presentation-only change. The existing identity verification step remains mandatory before clinical note content is displayed or edited. Draft saving, signing, closing, cancellation and append-only amendments retain their existing permission checks and API contracts. Do not add patient names or note content to the clinical queue list, browser storage, general logs or telemetry.

Verification includes keyboard operation, status announcements, desktop/tablet/375px layouts, and busy/opening states. API integration and real Keycloak authorization are verified separately from mocked Web component tests.

## Real Keycloak clinical note lifecycle and signed-note UI acceptance — 2026-11-27

HeapCare.Local PR #38 merged as `fae841e12223e82ff6e7a8c0dfdf571222f744d6`. Full-stack run [#46050151054](https://github.com/HeapCare/HeapCare.Local/actions/runs/46050151054) passed Local tooling validation, nine-service Docker/API/observability startup, persisted-role API acceptance, PostgreSQL logical backup/restore, the real Keycloak Authorization Code + PKCE browser suite and cleanup.

The API acceptance uses the real ClinicalUser token against a separate synthetic patient and in-progress OPD visit. It creates an encounter, saves a draft, signs with persisted signer attribution, verifies a second note update is rejected (409), closes the encounter, appends an amendment, re-reads the encounter, and verifies that the signed original remains unchanged. A PatientRegistrar token is denied encounter read with HTTP 403.

The browser acceptance reopens this second encounter through the ClinicalUser workspace, requires patient identity confirmation, asserts all signed note fields are read-only, verifies Review and sign is not available, and displays the append-only amendment. A separate original encounter remains empty and unsigned, proving the sign action stays disabled until required note fields are entered.

Run #46050151054 checked out API `745085912684d54a2c2f6ca830ade712d76d9e63`, Web `baab2966baeafd8f07e94bf45ab02f8ef3ecc1e9`, and Observability `1a06382e4eeed3e0070a0def2b89b8d11512dd52`. The real-OIDC suite completed 11 passing tests.

**Synthetic-data boundary:** all added note strings are explicit CI placeholders in a disposable database. No actual patient history, examination findings, diagnosis, treatment or care plan is entered. The automation does not approve clinical privileges or make the platform production-ready.
