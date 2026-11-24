# Workflow: Patient Registration

**Status:** Draft workflow for validation with hospital users. This is a proposed design, not an approved clinical or identity policy.

## Actors
Reception/front-desk staff; registration administrator; patient (provides details).

## Preconditions
- User is authenticated and authorized for the active tenant and hospital/branch.
- Registration configuration and required fields are available.

## Main flow
1. Patient arrives or registration is initiated.
2. Staff searches the selected hospital's patient registrations using configured identifiers and permitted demographic details.
3. If a likely match is found, staff reviews only permitted details and confirms the correct patient; do not create a duplicate.
4. If no match is found, staff enters the minimum required demographic details.
5. System validates input and presents possible duplicate candidates; candidate matching is advisory and never silently merges records.
6. Staff confirms creation; system creates a tenant-owned patient master and a hospital registration/association.
7. Patient master uses an internal immutable GUID. A human-readable MRN/UHID policy is not finalized; do not invent a number or auto-generate one until the hospital/customer policy is approved.
8. Staff selects the intended department/service and continues to appointment or walk-in check-in.
9. Registration confirmation and next-step details are shown.

## Alternatives and exceptions
- Possible duplicate: show a safe, minimal candidate list; route unresolved matches to authorized review. Never auto-merge.
- Required data missing/invalid: show field-level errors and preserve safe entered data.
- Identifier conflict: return a conflict and resolve through the approved identifier workflow.
- Patient already exists: associate the existing master with the hospital only if the staff member has permission and the access policy allows it.
- Network/service failure: communicate failure; protect against duplicate submission using idempotency or a retry-safe operation design.
- Emergency registration: a temporary/minimal-registration workflow requires a separate approved policy.

## Proposed data boundary
- Patient master is tenant-owned; there is no global cross-tenant patient search/linking.
- Hospital registration/affiliation is a separate relationship, with facility context and optional local identifier.
- Patient master identifiers and hospital-local identifiers must have explicit uniqueness scope and normalization.
- Patient records are not physically deleted through ordinary CRUD; correction, merge, retention and deletion policies require a governed workflow.
- Only minimal demographic information is stored in this registration slice. Clinical history, diagnoses, encounters, orders, prescriptions and billing belong to their owning modules.

## Data created/updated
Patient master, hospital registration/affiliation, optional identifier mapping, registration metadata and audit event. Appointment/check-in records remain in their owning modules.

## Permissions and scope
Patient search, create, associate, update and merge/review are distinct permissions. Access is restricted to the tenant and hospital/department scopes the user is authorized for. A tenant-wide membership alone should not imply unrestricted clinical-data access.

## Audit
Record actor, timestamp, tenant, facility scope, operation, outcome and correlation ID. Avoid patient-sensitive payloads in logs or general audit metadata. Registration/identity changes should be auditable without copying full demographics into the audit event.

## Acceptance criteria (initial)
- Search happens before registration.
- Existing patient can be associated with an authorized hospital without creating a second master.
- Internal IDs are immutable and tenant-scoped.
- Cross-tenant lookup is denied.
- Hospital/department scope is enforced on search and registration.
- Duplicate candidates are advisory; no silent merge.
- Identifier uniqueness conflicts are handled safely.
- Retry does not create duplicate masters or affiliations.
- Registration is auditable.
- Demographic fields and requiredness are validated with pilot hospital users.
