# Persisted tenant role-to-permission matrix

**Status:** Source-derived implementation snapshot, reviewed 2026-11-26.  
**Source of truth for this snapshot:** `CompileCare.Api/src/CompileCare.Api/Security/TenantRolePermissions.cs` and `CompileCare.Api/src/CompileCare.Api/Middleware/TenantContextMiddleware.cs`.  
**Important:** This records what the current API implementation grants. It is not, by itself, product-owner approval of the intended job responsibilities, a clinical privilege model, or proof that every endpoint and resource scope is correctly enforced.

## Trust and evaluation model

For a tenant-scoped request, the API validates the OIDC token, then resolves the validated issuer and subject against an active persisted membership in the selected active tenant. The persisted role is passed to the role-to-permission resolver. The API removes caller-supplied reserved role/permission claims and creates a dedicated `CompileCareTenantMembership` identity containing the derived permission claims. Authorization policies accept permissions only from that dedicated identity.

The `X-Tenant-ID` header selects tenant context; it does not grant access. Hospital and department scopes are separately enforced through organization-scope context and service/persistence checks.

## Current persisted tenant roles

The table below is a readable summary of the permission constants returned by the current resolver. “—” means the resolver currently returns no permission in that capability group.

| Capability | HospitalAdmin | FacilityManager | FacilityViewer | PatientRegistrar | ClinicalUser |
|---|---|---|---|---|---|
| Facility hierarchy | Read, create/update, delete | Read, create/update | Read | Read | Read |
| Membership administration | Manage | — | — | — | — |
| Audit events | Read | — | — | — | — |
| Patient records | Search/read, identity read, register, duplicate review | — | — | Search/read, identity read, register, duplicate review | Identity read |
| Patient identifiers / MRN | Read, assign, policy read/manage | — | — | Read, assign | — |
| Staff profiles and assignments | Read, manage | — | — | — | — |
| Appointment availability | Read, manage | — | — | Read | — |
| Appointments | Read, manage | — | — | Read, manage | — |
| OPD queue | Read, manage | — | — | Read, manage | Read |
| OPD encounters | — | — | — | — | Read, manage, sign, amend |
| Billing catalogue | Manage | — | — | — | — |
| Billing invoices | Read, create/manage | — | — | Read, create/manage | — |
| Billing payments | Record/manage | — | — | Record/manage | — |
| Billing reports | Read | — | — | Read | — |

### Exact implementation details

- `HospitalAdmin`: `facility.read`, `facility.write`, `facility.delete`, `membership.manage`, audit read; patient read/identity-read/register/duplicate-review; patient identifier read/assign/policy-read/policy-manage; staff read/manage; availability read/manage; appointment read/manage; queue read/manage; billing catalogue-manage, invoice-read/invoice-manage, payment-manage and report-read.
- `FacilityManager`: `facility.read` and `facility.write`.
- `FacilityViewer`: `facility.read`.
- `PatientRegistrar`: facility read; patient read/identity-read/register/duplicate-review; identifier read/assign; availability read; appointment read/manage; queue read/manage; invoice read/manage; payment manage; billing report read.
- `ClinicalUser`: patient identity-read; facility read; queue read; OPD encounter read/manage/sign/amend.

The current resolver contains five tenant roles. `PlatformOperator` is a separate global operator identity and authorization policy, not a sixth tenant membership role. Its access must be reviewed independently from this table.

## Endpoint-policy distinctions and known caveats

- The role grant list is not the same thing as endpoint coverage. The [API authorization policy inventory](authorization-policy-inventory.md) records named policies and endpoint assignments.
- A permission at the HTTP boundary does not replace resource-level hospital/department scope validation in application services and persistence.
- Billing catalogue listing currently uses the `BillingInvoiceRead` policy; there is no separate catalogue-read permission in the current policy resolver. Treat this as an explicit design decision to review, not as an implied grant.
- `PatientRegistrar` currently receives duplicate-candidate review and patient identity-read permissions. Confirm that these capabilities match the approved reception workflow and minimum-necessary access requirements.
- `ClinicalUser` currently receives encounter amendment permission in addition to read/manage/sign. Clinical amendment must remain governed by the encounter's signed-state, reason, attribution and audit controls.
- Staff read/manage is currently limited to `HospitalAdmin`; do not broaden it without an approved role design and negative authorization tests.
- UI navigation and visibility are usability controls only. Every API operation must enforce its own authorization and resource scope.

## Regression coverage merged through 2026-11-27

- API PR #90 adds PostgreSQL-backed facility read/create/delete checks for all five persisted tenant roles and verifies denied mutations do not change hospital state. CI and Docker Integration Verification passed.
- API PR #92 adds negative billing coverage for FacilityViewer and ClinicalUser across catalogue, invoice, payment and report reads/writes. It verifies invoice state remains unchanged and the denied operations are attributed in audit metadata. CI and Docker Integration Verification passed.
- API PR #93 adds negative scheduling coverage for FacilityViewer and ClinicalUser across availability listing, slot retrieval and appointment booking. It verifies denied booking leaves the appointment list unchanged and denials are audited. CI and Docker Integration Verification passed.
- API PR #94 adds PostgreSQL-backed staff profile and assignment denials across FacilityManager, FacilityViewer, PatientRegistrar and ClinicalUser. It merged on 2026-11-27 as `e72af18ac6b70aef462d8d554e9f6d66c030cd63` after hosted PR CI, push CI and Docker Integration Verification passed.
- API PR #95 adds PostgreSQL-backed patient identifier/MRN authorization coverage. FacilityManager, FacilityViewer and ClinicalUser are denied identifier-policy read/manage and identifier read/lookup/assignment; PatientRegistrar retains identifier read/assignment but is denied policy read/manage. The tests verify denied operations are audited and leave policy and identifier state unchanged. It merged on 2026-11-27 as `62e73a0577d0dd10e69a1d480290d3c1e3210f30` after hosted PR CI, push CI and Docker Integration Verification passed.

These are focused regression slices, not exhaustive proof of every permission, endpoint, data scope or UI route. Continue with positive/negative coverage for each capability and organization scope, including tenant, hospital and department boundaries.

## Required verification before pilot

1. Product and clinical owners approve the responsibilities and minimum-necessary access for each role, especially patient identity, duplicate review, identifiers, billing payments/reports and encounter amendment.
2. Maintain positive and negative PostgreSQL integration tests for every role/capability combination, including denied operations that must not mutate state.
3. Test tenant-wide, hospital and department scopes, including cross-tenant, cross-hospital and same-hospital cross-department access.
4. Verify that token-supplied role and permission claims cannot elevate any persisted role.
5. Verify audit attribution and metadata-only logging for allowed and denied sensitive operations.
6. Reconcile this document whenever `TenantRolePermissions`, authorization policies, endpoint assignments or role provisioning changes.

This snapshot does not lift the production/pilot HOLD. Authenticated Keycloak acceptance, full authorization regression, operational controls and independent security/privacy review remain release gates.
