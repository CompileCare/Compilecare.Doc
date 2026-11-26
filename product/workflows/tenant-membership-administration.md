# Tenant membership administration

## Purpose

The Memberships workspace allows an authorized tenant HospitalAdmin to administer user membership inside the currently selected tenant. It reuses the existing tenant membership API and does not create identity-provider users or change global platform-operator privileges.

The workflow is limited to:
- Listing memberships visible within the administrator's manageable organization scope.
- Creating a membership for an identity that already exists in the configured OIDC identity provider.
- Assigning an allow-listed CompileCare tenant role and one or more organization scopes.
- Updating an existing membership's role and scope assignments.
- Deactivating or reactivating a membership.

Memberships are deactivated rather than deleted so identity and audit history remain attributable.

## Access and authorization

The Web navigation and workspace are shown only when the active API-derived tenant context includes `membership.manage`. The current role catalog grants this permission to HospitalAdmin.

This is presentation gating only. Every request carries the bearer token and selected `X-Tenant-ID`; the API validates the active tenant membership, role and requested organization scopes. The UI never acts as the authorization boundary.

The backend enforces the following rules:
- Only an active HospitalAdmin membership can manage tenant memberships.
- A scoped administrator can list or manage only target memberships whose complete scope set is contained within the administrator's own scope.
- A tenant-wide scope is exclusive and cannot be combined with hospital or department scopes.
- A hospital scope must reference a hospital in the selected tenant.
- A department scope must reference a department in the selected tenant; its hospital is resolved and validated by the API.
- Duplicate scopes and redundant department scopes beneath an already assigned hospital scope are rejected.
- Scope assignment is limited to 25 entries.
- Unknown roles and scope types fail closed.
- Administrators cannot modify or deactivate their own membership.
- The last active tenant-wide HospitalAdmin cannot be demoted, lose its tenant scope or be deactivated.
- Membership creation, role/scope changes and activation lifecycle operations are recorded by the request audit middleware as metadata; request bodies, OIDC subjects and credentials are not copied into audit payloads.

## Identity contract

Membership identity is the exact validated OIDC `issuer` plus `subject` pair within the selected tenant.

The create form:
- Uses the issuer from the currently validated sign-in; it is not editable.
- Requires the target user's exact OIDC subject.
- Does not accept email as an identity key.
- Requires the operator to create the user in the configured identity provider before membership creation. This is an operational precondition: CompileCare stores the membership but does not create or verify identity-provider accounts.

The UI displays the subject because administrators need to distinguish identity records. Access to this identifier is restricted to the membership-management permission.

## Roles

The UI offers only the roles accepted by the API:

| Role | Intended responsibility |
|---|---|
| `HospitalAdmin` | Hospital/tenant administration within assigned scope, including membership management |
| `FacilityManager` | Facility read/write operations within assigned scope |
| `FacilityViewer` | Read-only facility access within assigned scope |
| `PatientRegistrar` | Patient registration and approved reception/appointment/queue/billing workflows |
| `ClinicalUser` | Scoped clinical identity and encounter workflows |

The API remains authoritative for the effective permission set associated with each role. The UI does not accept arbitrary permission strings.

## Organization scopes

| Scope | Meaning |
|---|---|
| `Tenant` | Entire tenant; exclusive and cannot be combined with other scopes |
| `Hospital` | One hospital and its accessible child resources |
| `Department` | One department within its owning hospital |

A tenant-wide administrator can assign tenant, hospital or department scope. A hospital-scoped administrator can assign scopes only within hospitals they manage. A department-scoped administrator can assign department scope only within their assigned scope. The API independently validates every assignment.

## API routes

All tenant membership routes require an active tenant context and the membership-management authorization policy.

| Operation | Route |
|---|---|
| List | `GET /api/v1/tenant/memberships?page=1&pageSize=50` |
| Create | `POST /api/v1/tenant/memberships` |
| Update role/scopes | `PUT /api/v1/tenant/memberships/{membershipId}` |
| Deactivate | `POST /api/v1/tenant/memberships/{membershipId}/deactivate` |
| Reactivate | `POST /api/v1/tenant/memberships/{membershipId}/reactivate` |

The list is paginated at 50 entries per page. The service applies scope containment in SQL before pagination; it does not load the entire tenant membership table into application memory.

## Interaction and failure behavior

- Deactivation and reactivation require an explicit in-context confirmation.
- Self-management controls are disabled in the UI; the API independently rejects self-update and self-deactivation.
- A 409 response may indicate a concurrent change, last-administrator protection or another protected lifecycle rule. The UI asks the administrator to refresh and review the current assignments.
- A 403/404 response is surfaced without substituting a broader list or bypassing scope checks.
- The create form clearly states that the target user must already exist in the configured identity provider, that CompileCare does not create IdP accounts, and that the OIDC subject (not email) is required.
- Scope editing preserves the full scope set in the update request; changing a role does not silently discard existing scopes.

## Platform tenant lifecycle is a separate control plane

Global tenant registry operations are not part of this tenant-scoped workspace. The dedicated platform workspace is implemented in [Platform tenant lifecycle](platform-tenant-lifecycle.md) and merged in [CompileCare.Web PR #36](https://github.com/CompileCare/CompileCare.Web/pull/36). It appears only when the API self-status endpoint confirms the caller's persisted PlatformOperator identity. Tenant HospitalAdmin roles never grant platform-operator access.

The platform workspace can create tenants in Provisioning, provision the initial tenant-wide HospitalAdmin and activate the tenant, suspend active tenants, reactivate suspended tenants and permanently close tenants. The API enforces the state-transition matrix and rejects activation until an active tenant-wide HospitalAdmin exists. Global operations remain protected by the PlatformOperator policy on every request.

## Verification status

The Web implementation is merged in [CompileCare.Web PR #35](https://github.com/CompileCare/CompileCare.Web/pull/35), merge commit `b8e4a4ec8864884f5e71dc070a319a5f4df07185`.

Hosted Web CI runs 45745881098 and 45746024118 passed on final head `2d3ce43014222da08e4e30d25bc4d4bee4ff7916`:
- 75 Vitest tests across 8 test files.
- All 14 Chromium browser E2E tests.
- TypeScript/Vite production build.
- Docker build and container smoke.

The component tests cover membership listing, department-scope creation, role/scope updates, explicit lifecycle confirmation and self-management controls. API client tests verify tenant headers, no-store reads, request methods and payloads.

A real authenticated local-stack acceptance walkthrough remains required. Use synthetic OIDC identities and synthetic tenant/hospital/department records; verify tenant-wide, hospital-scoped and department-scoped administrators, self-management rejection and last-active-tenant-wide-HospitalAdmin protection. Also verify the platform operator control plane is hidden for ordinary tenant users and that activation is rejected until initial-admin provisioning completes.
