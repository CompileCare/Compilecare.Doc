# Workflow: Patient Registration

**Status:** Draft workflow for validation with hospital users.

## Actors
Reception/front-desk staff; registration administrator; patient (provides details).

## Preconditions
- User is authenticated and authorized for the active tenant/hospital/branch.
- Registration configuration and required fields are available.

## Main flow
1. Patient arrives or registration is initiated.
2. Staff searches by configured identifiers and demographic details to find an existing record.
3. If a likely match is found, staff reviews permitted details and confirms the correct record; avoid creating a duplicate.
4. If no match is found, staff enters required demographic/contact details.
5. System validates input and checks duplicate candidates.
6. System assigns a unique patient identifier (UHID/MRN according to hospital configuration).
7. Patient record is saved with tenant and hospital/branch scope.
8. Staff selects the intended department/service and continues to appointment or walk-in check-in.
9. Registration confirmation and next-step details are shown.

## Alternatives and exceptions
- Possible duplicate: stop or warn and route to authorized merge/review; never silently merge.
- Required data missing/invalid: show field-level errors and preserve safe entered data.
- Identifier conflict: retry or resolve using approved workflow.
- Patient record already exists: use existing record, subject to permissions and tenant-sharing policy.
- Network/service failure: communicate failure; avoid duplicate submission with idempotency where appropriate.
- Emergency registration: any temporary/minimal-registration workflow requires a separate approved policy.

## Data created/updated
Patient demographic record, identifier mapping, registration metadata, audit event, and optional appointment/check-in records in their owning modules.

## Permissions and scope
Create/search/update permissions are separately defined. Search and patient visibility must respect tenant, hospital, branch, and approved patient-sharing rules.

## Audit
Record actor, timestamp, tenant, facility scope, operation, and outcome. Avoid putting unnecessary patient-sensitive data in logs.

## Acceptance criteria (initial)
- Existing patients can be found without creating a duplicate.
- New patient identifiers are unique within the defined policy.
- Cross-tenant patient lookup is denied.
- Unauthorized users cannot register or edit patients.
- Validation errors are clear and do not leak sensitive details.
- Successful registration is auditable.
- Retry behavior does not create duplicate records.
