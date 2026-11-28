# ADR-013: Staff and provider profile boundaries

- **Status:** Accepted for the initial profile/assignment slice; operationalized in API/Web; production hospital validation pending
- **Date:** 2026-11-24
- **Decision owners:** Product and engineering; hospital operations and clinical stakeholders must validate the workflow before production use

## Context

HeapCare already models authenticated external identities, tenant memberships, tenant roles and hospital/department scopes. Staff and provider records represent people and their organizational/workforce context; they must not become a second identity or authorization system.

A staff member may work at more than one hospital or department, may hold multiple assignments over time, and may have a provider designation in addition to a non-clinical workforce role. Employment status, organizational assignment, application access and clinical credentialing are related but distinct concepts.

## Decision

1. **Separate person/workforce profile from authentication.** A staff profile is a tenant-owned business record. It is not an OIDC principal, login credential, tenant membership or role grant.
2. **Keep authorization authoritative in the existing access model.** Only validated OIDC identity plus an active tenant membership, scoped role grants and server-side authorization can permit access. If a future workflow links a profile to a membership, it must be an explicit, same-tenant, auditable association; the profile itself must never grant permissions.
3. **Model assignments explicitly.** Hospital and department assignments belong to a tenant and reference the existing facility hierarchy. Assignments carry effective-from/effective-to dates and an explicit status. Validate that department assignments belong to the selected hospital and tenant. Do not infer access scope solely from an employment assignment.
4. **Treat provider designation as a workforce classification, not credential verification.** A provider profile may record operational specialty/designation metadata, but must not imply that a professional licence, registration, qualification or scope of practice has been verified.
5. **Preserve history.** Deactivate or end-date profiles and assignments instead of physically deleting records that have operational history. Prevent overlapping or contradictory active assignments where the business rule requires it; make changes auditable.
6. **Minimize sensitive data.** The initial slice excludes government identifiers, payroll/bank details, biometrics, background checks, credential-document storage and other high-risk employment data. Add such data only through a separately reviewed design with purpose, access, retention, encryption and audit controls.
7. **Use the modular monolith.** Implement workforce/staff-provider capability within the existing HeapCare.Api modular architecture and tenant-aware persistence. Do not introduce a separate service or identity provider for this capability.

## Initial scope

- Create and maintain a tenant-owned staff profile with minimal operational fields.
- List/search profiles within the caller's authorized tenant and organization scope, with bounded pagination.
- Record and manage hospital/department assignments with effective dates.
- Support provider designation metadata without representing credential verification.
- Deactivate/reactivate profiles and end-date assignments with audit history.

## Explicitly out of scope

- Authentication/account creation, password management, OIDC user provisioning or role-grant administration.
- Payroll, attendance, rostering, leave, compensation or performance management.
- Credential verification, licensing checks, privileging, clinical scope-of-practice decisions or credential document vaults.
- Government identity numbers, bank details, biometrics and background-check data.
- Profile-to-membership linking until the identity-verification evidence and audit workflow is approved.
- Cross-tenant staff sharing or automatic profile matching.

## Security and data boundaries

- Every profile and assignment is tenant-owned and protected by tenant query filters, tenant-aware foreign keys and organization-scope authorization.
- A caller cannot create or assign a profile outside their authorized hospital/department scope.
- Profile-to-membership linking requires an explicit verified identity association; do not match by name or email alone.
- Do not accept caller-supplied tenant IDs, roles or permissions as proof of authorization.
- Sensitive reads and all material create/update/deactivate/link/assignment changes must be attributable through the approved audit mechanism.
- Do not emit personal workforce data in logs, traces, metrics, exception messages or health details.

## Consequences

The design avoids conflating an employee record with an account or permission grant and supports people with multiple, time-bounded assignments. It requires careful organization-scope checks and audit coverage on every write path. Credentialing and HR systems remain separate integrations or future capabilities.

## Open questions for hospital validation

- Which minimal profile fields are operationally necessary at pilot launch?
- Is a person allowed multiple concurrent active assignments at the same facility?
- Which roles can create, update, link and deactivate staff profiles?
- Should provider specialty/designation use a controlled tenant catalogue?
- What is the approved process and evidence for verifying a profile-to-membership link?
- Which local retention and employment-data requirements apply to the pilot?

## Related documents

- [Staff and provider management workflow](../../product/workflows/staff-and-provider-management.md)
- [Tenant and organization boundaries](ADR-006-tenant-organization-patient-boundaries.md)
- [Verified patient contact data](ADR-012-verified-patient-contact-data-and-matching.md)
- [Product blueprint](../../product/blueprint-v1.md)
- [Delivery roadmap](../../product/roadmap.md)

## Implementation record — 2026-11-28

The initial scope above is implemented in HeapCare.Api and HeapCare.Web:

- API endpoints: `POST/GET /api/v1/staff`, `GET/PUT /api/v1/staff/{staffId}`, profile deactivate/reactivate, assignment create and assignment end-date.
- Distinct persisted grants: `staff.read` and `staff.manage`.
- Tenant-aware profile/assignment queries, hospital/department scope enforcement, bounded paging, duplicate staff-code protection and effective-dated assignment history.
- API integration tests cover HospitalAdmin management, department scope, explicit denials for roles without grants, cross-tenant isolation and denied-access audit records.
- Web workspace and component tests cover staff-directory search/display, initial profile/assignment creation and confirmation before deactivation.

This implementation does not include login/account provisioning, profile-to-membership linking, credential verification, licensing, privileging, payroll, roster, HR records or cross-tenant sharing. The hospital/clinical decision owners must validate the operational fields, role mapping and assignment rules before production activation.

