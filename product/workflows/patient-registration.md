# Workflow: Patient Registration

**Status:** Patient master, hospital registration, hospital-scoped MRN policy/assignment and normalized advisory duplicate matching are merged (API PRs #14–#17). Verified phone/email contact capture is designed in ADR-012 but is not implemented. Identity association and merge workflows remain separate work.

## Actors

Patient; receptionist/front-desk staff; registration staff; registration supervisor; patient identity administrator; authorized hospital/department staff.

## Preconditions

- User is authenticated and authorized for the active tenant and selected hospital/branch.
- A hospital MRN policy is configured by an authorized HospitalAdmin before staff assign MRNs.
- Cross-hospital discovery is not included in the initial duplicate-check API; it requires a separately approved permission and workflow.

## Main flow

1. Patient arrives or registration is initiated.
2. Staff searches the selected hospital's registrations using the hospital MRN and other permitted details.
3. Before creating a record, staff may call the hospital-local duplicate-candidate endpoint when an exact date of birth is available. The merged normalized matcher requires exact DOB and either normalized given name, or both normalized middle and family names. It applies Unicode compatibility normalization, invariant uppercase and whitespace collapsing; punctuation and diacritics remain significant. Phone/email are not currently collected or used by this endpoint.
4. If candidates are returned, staff verifies identity using the hospital's approved procedure. The candidate response is advisory only: it does not link, merge, block registration or expose clinical information.
5. If date of birth is unavailable, v1 matching is not evaluated and no name-only search is performed. Staff follows the hospital's manual identity-check procedure. A no-candidates response is not proof that no duplicate exists.
6. If no existing local registration is confirmed for use, staff enters the minimum registration data. Patient name is required; DOB is captured when known or represented as estimated age/age unit. Mobile, email, address, ABHA and Aadhaar are not universal prerequisites. Hospital-specific additional requirements are configurable and separately validated from clinical requirements.
7. Staff confirms creation. The system creates a tenant-owned patient master with immutable internal UUID and a separate hospital registration.
8. After registration, an authorized PatientRegistrar requests MRN assignment through the hospital-scoped identifier API. The service allocates the next value from the hospital's configured policy in the same transaction as identifier persistence and request audit. Repeating the request for the same registration returns the existing MRN.
9. Staff selects the intended department/service and continues to appointment or walk-in check-in. Those records reference the internal Patient ID and remain in their owning modules.
10. Registration confirmation and next-step details are shown.

## Alternatives and exceptions

- **No MRN policy:** the system does not invent a default prefix or silently issue an identifier; an authorized HospitalAdmin must configure the policy first.
- **Possible duplicate:** show a minimal candidate list; route unresolved identity to authorized review. Never silently link or merge.
- **Strong candidate:** prioritize for staff verification; the v1 endpoint does not calculate a confidence score and a candidate is not proof of identity.
- **Conflicting identifier:** block assignment and route to an identity-resolution workflow.
- **Existing patient at another hospital:** cross-hospital discovery and association are future increments requiring a separate permission, identity-verification workflow and audit controls; they are not available through the v1 duplicate endpoint.
- **Patient without phone/email/documents:** allow registration using the configured minimum data.
- **DOB unknown:** capture unknown or estimated age with unit; do not invent an exact DOB.
- **Minor/dependant:** create a separate patient master and a guardian/dependant relationship where required; support multiple guardians and effective-dated authority.
- **Unknown emergency patient:** create a temporary emergency registration with a temporary identifier distinct from permanent MRN; reconcile later through an authorized workflow.
- **Identity conflict:** place the case into manual review; do not silently associate or merge.
- **Concurrent/retried MRN request:** serialize allocation per hospital; return the existing MRN for the same registration, and rely on database uniqueness for different registrations.
- **Network/service failure:** communicate failure and protect against duplicate submission through idempotent/retry-safe operations.

## Data boundaries

- Patient master is tenant-owned; cross-tenant matching/linking is disabled by default.
- Patient master, identifiers, hospital registration, guardian/representative relationships and identity-management cases are separate concepts.
- One tenant-wide internal UUID identifies the master. MRN/UHID is hospital-local/configured-scope. ABHA is an optional external patient-linked reference.
- Phone/email contact methods are designed as separate optional, patient-owned records under ADR-012; they are not fields on the Patient row and are not implemented yet.
- Appointments, encounters, prescriptions, diagnoses, billing and clinical documents reference the internal Patient ID and remain in their owning modules.
- Ordinary CRUD does not physically delete patient records. Correction, merge, retention, legal hold, de-identification and deletion are governed workflows.
- MRN policy prefix/width and the issued identifier value are distinct from the immutable Patient GUID. Initial policy replacement and identifier mutation are not supported.

## Permissions and scope

Search, create, associate, update, duplicate review, merge request, merge approval and cross-hospital discovery are distinct capabilities. The initial duplicate-candidate endpoint requires the dedicated patient.duplicate.review permission and tenant-wide or explicit hospital scope; PatientRegistrar receives this permission in the first increment. Department-only membership cannot use this hospital-wide endpoint. Default search is limited to the selected hospital's registrations. Cross-hospital candidate search requires a separate permission and workflow, and must return minimum identity details. Access to patient or clinical information remains subject to hospital/department permissions. System administration does not automatically confer clinical merge authority. HospitalAdmin manages the hospital MRN policy; PatientRegistrar assigns and reads identifiers within authorized hospital scope.

Contact permissions are not granted by the PatientRegistrar role automatically. The proposed patient.contact.manage, patient.contact.read.masked, patient.contact.read.full and patient.contact.verify.initiate permissions are defined in ADR-012 and require explicit role mapping. Every operation must verify the active patient registration and selected hospital scope. Unmasked contact reads are purpose-limited and audited. Challenge completion proves control of a channel, not patient identity or guardian authority.

## Contact data — designed, not implemented

When contact capture is delivered under ADR-012, phone and email remain optional and are stored in separate protected contact records. Values are masked by default; verification requires a one-time challenge and cannot be set manually by staff. The same family phone/email may be attached to multiple patient records. A shared, unverified, expired, revoked, superseded or guardian-owned contact is not a patient identity signal. The first contact-aware matcher may only add a corroborating annotation to an existing exact-DOB/name candidate; it cannot create a candidate from contact alone. Contact verification is distinct from consent or communication preferences.

## Audit

Record actor, timestamp, tenant, hospital/department scope, operation, outcome, reason/evidence references where relevant, approver for privileged actions and correlation ID. Protect old/new demographic values and identity evidence as sensitive data; do not copy full patient payloads into general logs. MRN value itself is not copied into general audit metadata. Contact values, canonical values, lookup digests and OTP/challenge secrets must not appear in audit metadata or telemetry. Unmasked contact reads and verification lifecycle events require dedicated audit evidence. Merge and correction records preserve historical patient identifiers and support governed reversal.

## Acceptance criteria

- Search happens before create within the selected hospital.
- Cross-hospital discovery requires its separate permission and returns minimum candidate data.
- Existing patient can be associated with an authorized hospital without creating a second master.
- Internal Patient ID is immutable and tenant-scoped.
- MRN policy is explicitly configured per hospital; no universal prefix or numbering format is hardcoded.
- MRN uniqueness is enforced within the configured hospital; it is not the primary key.
- MRN sequence allocation is concurrency-safe and assignment is idempotent per patient registration.
- MRN assignment requires an existing hospital registration and hospital-wide authorization.
- ABHA/Aadhaar/mobile/email are not universal registration prerequisites.
- Hospital-specific registration-required fields are configurable and distinct from clinical requirements.
- Duplicate checks are hospital-local and require exact DOB plus normalized given name or both normalized middle and family names. Family name alone is insufficient.
- Unknown date of birth returns an explicit not-evaluated result; no name-only search is performed.
- Candidate output is bounded, contains minimum identity details only, and never includes clinical data, normalized keys or MRN values.
- Duplicate candidates are advisory; no automatic linking, registration blocking or merge occurs.
- A no-candidate result is not proof that no duplicate exists; hospital manual verification remains necessary.
- Conflicting identifier assignment is blocked and routed for resolution.
- Guardian relationships are separate, effective-dated records.
- Temporary emergency identifiers are distinct from permanent MRNs and can be reconciled through an audited workflow.
- Merge approval is privileged, auditable and preserves historical references; high-risk cases support two-person approval.
- Retry does not create duplicate masters, registrations or identifiers.
- Normal CRUD cannot physically delete patient records or edit/delete issued MRNs.
- Tenant, hospital and department scope is enforced and tested.
- Retention, consent, legal hold and deployment-specific settings are not represented as universally approved legal policy.
