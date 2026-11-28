# Staff and provider management workflow

**Status:** Implemented operational profile and assignment foundation in HeapCare.Api and HeapCare.Web. Hospital-user validation of operational policy is still required before production use.  
**Related decision:** [ADR-013 — Staff and provider profile boundaries](../../architecture/decisions/ADR-013-staff-and-provider-profile-boundaries.md)

## Purpose

Maintain the workforce profiles and facility assignments needed to operate a hospital while keeping staff records, login identities, access grants and clinical credentialing separate.

## Actors

- Hospital administrator or delegated workforce administrator
- Department administrator, within an explicitly granted department scope
- Staff member or provider (self-service is not part of the initial slice)
- Authorized auditor

## Preconditions

- The tenant is active and the hospital/department hierarchy exists.
- The acting user has a validated OIDC identity and active tenant membership.
- The acting membership has the required staff-management permission in the relevant organization scope.
- A staff profile is tenant-owned; any membership link references an existing membership in the same tenant.

## Implemented operational workflow

1. **Create profile and first assignment:** an authorized administrator records the minimum approved operational details and the initial hospital/department assignment in one transaction. The API derives tenant context from the authenticated request and validates facility references and organization scope.
2. **Record provider designation (optional):** an authorized user records an operational provider designation or specialty from an approved vocabulary. The UI and API clearly distinguish this metadata from verified professional credentials or clinical privileges.
3. **Maintain changes:** authorized administrators update permitted profile fields, add or end-date assignments, or deactivate/reactivate profiles. Changes retain history and generate attributable audit events.
4. **Review:** users with `staff.read` can list/search and retrieve profiles only within their authorized tenant and organization scope. Search uses bounded pagination (maximum 100 records per page) and does not expose payroll, attendance, credential documents or other sensitive HR data.

## Alternate and failure paths

- Reject unknown, inactive or cross-tenant hospital/department references.
- Reject assignment to a department that does not belong to the selected hospital.
- Reject attempts to manage a profile or assignment outside the caller's authorized scope.
- Reject membership links to another tenant, ambiguous identities or an already-conflicting link; do not auto-link by name/email.
- Reject invalid effective-date ranges and preserve historical assignments rather than silently overwriting them.
- Return a safe not-found/forbidden response according to the API's established resource-disclosure policy.
- Record denied privileged attempts using the approved security audit policy without logging profile payloads.

## Implemented API and workspace boundary

The current implementation exposes these tenant-scoped API operations under `/api/v1/staff`:

- `POST /` — create a profile and its initial hospital/optional-department assignment.
- `GET /` and `GET /{staffId}` — bounded list/search and scoped detail read.
- `PUT /{staffId}` — update the permitted profile fields.
- `POST /{staffId}/deactivate` and `/reactivate` — lifecycle controls.
- `POST /{staffId}/assignments` and `POST /{staffId}/assignments/{assignmentId}/end` — add and end-date assignments.

The authenticated HeapCare Web workspace provides staff search, profile creation/edit, activation/deactivation, department assignment and assignment end-date workflows. The API remains authoritative; UI visibility is not an authorization check.

## Authorization and audit

- Read/list is protected by `staff.read`; profile and assignment mutations are protected by `staff.manage`. Profile-to-membership linking is not implemented in this slice.
- Tenant and organization scope are checked on every request and every referenced resource; UI filtering is not an authorization boundary.
- Profiles and assignments do not grant facility access. Existing tenant membership and role-grant logic remains authoritative. Creating or editing a staff profile does not provision an OIDC user, change a tenant role, or grant clinical permissions.
- Audit profile creation, updates, activation/deactivation, assignment changes and membership linking/unlinking. Audit records must identify actor, tenant, target, action and timestamp without duplicating unnecessary personal data.

## Implemented acceptance evidence

- API integration tests exercise tenant-scoped profile creation and retrieval, bounded search, profile update and lifecycle operations.
- Scope-aware profile/assignment reads and mutations reject out-of-tenant or out-of-organization access.
- Hospital and department assignments enforce tenant and hierarchy integrity and retain effective-date history.
- The current implementation does not link profiles to login memberships. Any future linking workflow must be explicit, same-tenant, authorized, evidence-backed and auditable; a profile alone must never grant access.
- Provider designation is explicitly non-credentialing metadata.
- Deactivation preserves profile and assignment history.
- PostgreSQL-backed integration tests and the hosted API test workflow are the source of evidence for tenant isolation, organization-scope authorization, invalid references, pagination and lifecycle behavior.
- Web workspace tests/build/Chromium E2E and API Release build/integration/Docker verification are required for follow-on changes.
- The workflow and API contract are validated with pilot hospital stakeholders before production use.

## Deferred

Profile-to-membership linking, payroll, attendance, rostering, leave, compensation, employee self-service, credential verification, licensing/privileging, sensitive identifiers, document vaults and cross-tenant staff sharing require separate product and security decisions.
