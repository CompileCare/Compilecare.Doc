# ADR-015: OPD encounter and clinical documentation boundaries

- **Status:** Proposed for pilot implementation and clinical workflow validation
- **Date:** 2026-11-24
- **Decision owners:** Product and engineering; clinicians, hospital operations, privacy/security and qualified clinical stakeholders must validate before production use

## Context

CompileCare now has tenant-owned patient masters and hospital registrations, staff/provider profiles, appointment scheduling, check-in and a front-desk queue. The next outpatient step is the clinical hand-off: an authorized clinical user reviews the patient and records the care delivered.

An appointment reserves time; a queue entry represents operational waiting/work; an OPD encounter represents a clinical interaction. They must remain separate records. A walk-in can have an encounter without an appointment, and the same patient can have many encounters over time.

The initial increment must not become a full electronic medical record, autonomous decision-support system, prescribing engine or diagnostic-order system. Clinical data is sensitive and needs explicit authorization, attribution, audit and immutable signed-record semantics.

## Decision

1. **Keep OPD encounters as a separate logical capability** within CompileCare.Api's modular monolith. Patient Management owns patient identity/registration; Appointments owns appointments; Front Desk owns queue entries; Clinical/OPD owns encounters, clinical notes and observations.
2. **Create an encounter from a queue entry, not directly from an appointment.** The queue entry must belong to the same tenant, hospital, department and patient registration. It must be in `InProgress`. Enforce at most one encounter per queue entry. The appointment reference, when present, is reached through the queue entry; walk-ins remain valid without an appointment.
3. **Preserve tenant and facility ownership.** Encounter records carry TenantId, HospitalId, DepartmentId and PatientId. Use tenant-aware composite foreign keys to the queue entry, hospital, department and hospital patient registration. Restrictive deletes preserve clinical history. Do not allow cross-tenant or cross-hospital patient association through encounter creation.
4. **Use explicit lifecycle states.** Initial states are `InProgress`, `Signed`, `Closed` and `Cancelled`. Creation starts an encounter in progress. Signing records a timestamp and authenticated actor. A signed encounter is immutable through normal update APIs. Corrections require a separately attributable amendment record linked to the signed encounter; never silently overwrite signed content. Closing is a separate administrative/clinical hand-off action and must not rewrite the signed note.
5. **Start with a bounded OPD note.** Capture structured sections for chief complaint, history, examination, assessment and plan, with explicit length limits and plain text only. Do not embed patient contact data, credentials, billing information or copied telemetry. Do not infer diagnoses, suggest treatment or auto-populate clinical conclusions.
6. **Keep structured observations separate from narrative.** The first increment may record a minimal, explicitly coded set of vital observations with value, unit, observation time and recorder. Do not apply automated triage, severity scoring, clinical alerts or treatment thresholds. The permitted vital codes, units, ranges and requiredness must be validated by the pilot hospital before enabling entry.
7. **Attribute every clinical action.** Record the validated OIDC issuer and subject for the encounter creator, note author, signer and amendment author. A staff profile or provider designation is not an authenticated identity and does not establish credentialing or clinical privilege. The initial pilot may use explicitly provisioned tenant membership roles and hospital/department scopes; this is an administrative authorization assertion, not professional credential verification.
8. **Use explicit permissions and organization scope.** Define separate `encounter.read`, `encounter.manage`, `encounter.sign` and `encounter.amend` permissions. Require matching tenant, hospital and department scope on every operation. Do not grant clinical permissions from caller-supplied claims, queue permissions or provider designation alone. Exact role assignment must be reviewed with the pilot hospital.
9. **Audit clinical data access and mutation.** Audit encounter creation, clinical-record reads, draft updates, signing, amendments and closure with actor, tenant, facility, resource ID, action, outcome and correlation. Do not copy note text, vital values, patient names or other clinical content into general logs, traces or metric labels. Clinical audit access and retention require separate policy.
10. **Keep clinical workflows deliberately narrow.** This increment excludes coded diagnosis catalogues, allergy reconciliation, medication/prescription orders, lab/radiology orders, referrals, procedures, clinical decision support, triage, emergency-department workflow, inpatient notes, discharge summaries, patient portal access, document/image attachments and external interoperability.
11. **No autonomous clinical behavior.** CompileCare records clinician-entered information and workflow state only. It must not diagnose, recommend treatment, rank acuity, prescribe, calculate doses or replace professional judgement.
12. **Do not treat this ADR as production approval.** Before production use, validate note templates, vital code/unit/range policy, author/signer role mapping, amendment and retention policy, consent/privacy notices, access review, audit operations, emergency/break-glass access and applicable Indian legal requirements with qualified stakeholders.

## Initial acceptance criteria

- A patient with a valid hospital registration and an in-progress queue entry can have one OPD encounter created.
- A walk-in queue entry and appointment-linked queue entry both follow the same encounter workflow.
- Tenant, hospital, department, patient-registration and queue-entry mismatches are rejected by database constraints and API authorization.
- Only explicitly authorized and correctly scoped users can read or change encounter data.
- Draft notes can be updated; signed notes cannot be overwritten; amendments preserve the original signed content and identify the author and reason.
- Every clinical read/write/sign/amend action is attributable and audited without exposing clinical text to general telemetry.
- PostgreSQL migration, EF model snapshot, integration tests and Docker verification pass.
- Pilot clinical stakeholders validate the workflow and the permitted fields before production activation.

## Open questions for pilot validation

- Which note sections and templates are mandatory for each OPD specialty?
- Which vital observations are required, optional or not collected, and which units/ranges are approved?
- Which membership roles may author, sign, amend and read notes in each department?
- Is a clinician allowed to sign only encounters assigned to them, or can a department lead sign?
- What is the approved amendment, correction, retention and legal-hold policy?
- Which patient history, allergy and medication summaries must be visible at the encounter?
- What is the operational rule for queue completion relative to encounter signing?
- What consent, privacy notice, emergency access and audit-review process applies?

## Consequences

- Encounter, queue and appointment lifecycles remain independent and can be evolved without overloading appointment or queue records.
- The first clinical slice remains intentionally small and does not imply a complete EMR or credentialed clinical-practice authorization.
- Additional clinical modules must integrate through explicit encounter references and independently governed workflows.
