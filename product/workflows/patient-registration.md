# Workflow: Patient Registration

**Status:** Approved product direction; the patient-master and hospital-registration API foundation is implemented. The hospital MRN policy and assignment API are a separate increment and remain subject to review and pilot configuration.

## Actors

Patient; receptionist/front-desk staff; registration staff; registration supervisor; patient identity administrator; authorized hospital/department staff.

## Preconditions

- User is authenticated and authorized for the active tenant and selected hospital/branch.
- A hospital MRN policy is configured by an authorized HospitalAdmin before staff assign MRNs.
- Cross-hospital discovery is available only to users with its separate permission.

## Main flow

1. Patient arrives or registration is initiated.
2. Staff searches the selected hospital's registrations using configured identifiers and permitted demographic details.
3. If no local candidate is found, the system may offer controlled cross-hospital discovery only when the user has the separate permission. Candidate results expose only minimum identity details; clinical and unrelated hospital information are not disclosed.
4. If a likely match is found, staff verifies identity. An existing tenant-owned patient master may be associated with the selected hospital only when identity is sufficiently established and the user is authorized.
5. If no match is confirmed, staff enters the minimum registration data. Patient name is required; DOB is captured when known or represented as estimated age/age unit. Mobile, email, address, ABHA and Aadhaar are not universal prerequisites. Hospital-specific additional requirements are configurable and separately validated from clinical requirements.
6. System normalizes permitted matching inputs and presents advisory duplicate candidates. Matching is multi-factor and configurable; a shared phone/address or name alone is not proof of identity.
7. Staff confirms creation. The system creates a tenant-owned patient master with immutable internal UUID and a separate hospital registration.
8. After registration, an authorized PatientRegistrar requests MRN assignment through the hospital-scoped identifier API. The service allocates the next value from the hospital's configured policy in the same transaction as identifier persistence and request audit. Repeating the request for the same registration returns the existing MRN.
9. Staff selects the intended department/service and continues to appointment or walk-in check-in. Those records reference the internal Patient ID and remain in their owning modules.
10. Registration confirmation and next-step details are shown.

## Alternatives and exceptions

- **No MRN policy:** the system does not invent a default prefix or silently issue an identifier; an authorized HospitalAdmin must configure the policy first.
- **Possible duplicate:** show a minimal candidate list; route unresolved identity to authorized review. Never silently link or merge.
- **Strong candidate:** prioritize for staff verification; a match score alone does not authorize association.
- **Conflicting identifier:** block assignment and route to an identity-resolution workflow.
- **Existing patient at another hospital:** use separately authorized cross-hospital discovery, verify identity, associate the existing master and assign a local MRN if configured.
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
- Appointments, encounters, prescriptions, diagnoses, billing and clinical documents reference the internal Patient ID and remain in their owning modules.
- Ordinary CRUD does not physically delete patient records. Correction, merge, retention, legal hold, de-identification and deletion are governed workflows.
- MRN policy prefix/width and the issued identifier value are distinct from the immutable Patient GUID. Initial policy replacement and identifier mutation are not supported.

## Permissions and scope

Search, create, associate, update, duplicate review, merge request, merge approval and cross-hospital discovery are distinct capabilities. Default search is the selected hospital's registrations. Tenant membership alone does not grant unrestricted patient access. Cross-hospital candidate search returns minimum identity details; access to patient or clinical information remains subject to hospital/department scope. System administration does not automatically confer clinical merge authority. HospitalAdmin manages the hospital MRN policy; PatientRegistrar assigns and reads identifiers within authorized hospital scope.

## Audit

Record actor, timestamp, tenant, hospital/department scope, operation, outcome, reason/evidence references where relevant, approver for privileged actions and correlation ID. Protect old/new demographic values and identity evidence as sensitive data; do not copy full patient payloads into general logs. MRN value itself is not copied into general audit metadata. Merge and correction records preserve historical patient identifiers and support governed reversal.

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
- Duplicate candidates are advisory; no automatic linking or merge occurs.
- Conflicting identifier assignment is blocked and routed for resolution.
- Guardian relationships are separate, effective-dated records.
- Temporary emergency identifiers are distinct from permanent MRNs and can be reconciled through an audited workflow.
- Merge approval is privileged, auditable and preserves historical references; high-risk cases support two-person approval.
- Retry does not create duplicate masters, registrations or identifiers.
- Normal CRUD cannot physically delete patient records or edit/delete issued MRNs.
- Tenant, hospital and department scope is enforced and tested.
- Retention, consent, legal hold and deployment-specific settings are not represented as universally approved legal policy.
