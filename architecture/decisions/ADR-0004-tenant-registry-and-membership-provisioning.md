# ADR-0004: Tenant registry and trusted membership provisioning

**Status:** Accepted — implementation merged; operational acceptance remains in progress  
**Decision date:** 2026-11-23  
**Last reviewed:** 2026-11-26  
**Scope:** CompileCare.Api and CompileCare.Web — hospital-management platform

## Context

A tenant is the customer/contract boundary and may own one or more hospitals. Tenant selection through `X-Tenant-ID` is only a selector; it is not an authorization credential. Every tenant-scoped request must resolve the exact validated OIDC issuer/subject to an active membership in an active tenant.

Tenant lifecycle and membership provisioning must be centrally controlled. A tenant role must never grant global platform administration, and a tenant header must never make an unregistered or inactive tenant valid.

## Decision

1. **Persist a global tenant registry.** Each tenant has a generated UUID, normalized unique slug, bounded display name, lifecycle status and UTC timestamps. The slug is an operational identifier, not an authentication credential.
2. **Make tenant lifecycle authoritative.** Tenant-scoped requests require the selected tenant to exist and be `Active` before membership authorization. Provisioning, Suspended and Closed tenants cannot be used for ordinary tenant-scoped access. Closed is terminal; tenants are closed rather than hard-deleted after provisioning.
3. **Bind memberships to validated identity.** Membership identity is the exact OIDC `issuer` + `subject` + tenant UUID tuple. Memberships reference the tenant registry, roles are allow-listed, and unknown roles fail closed.
4. **Separate platform operators from tenant roles.** A persisted `platform_operators` record is keyed by validated issuer/subject. The PlatformOperator authorization policy queries this registry; token role/permission claims and tenant HospitalAdmin roles cannot grant platform privileges.
5. **Restrict platform bootstrap.** The one-time platform-operator bootstrap endpoint is mapped only in Development and requires the configured one-time secret. Production platform-operator provisioning remains a trusted operational responsibility; it is not a tenant-user self-service operation.
6. **Protect global tenant operations.** Platform tenant create/list/get and lifecycle transition endpoints require the separate PlatformOperator policy. Initial administrator provisioning is also platform-operator-only.
7. **Provision the first tenant administrator transactionally.** A tenant is created in `Provisioning`; exactly one initial tenant-wide HospitalAdmin can be provisioned while it remains in that state. The API enforces this prerequisite: Provisioning → Active returns 409 unless an active tenant-wide HospitalAdmin membership already exists. The initial administrator operation locks the tenant, is idempotent for the same initial identity and rejects competing initial administrators.
8. **Constrain tenant membership administration.** Tenant membership list/create/update/deactivate/reactivate routes require an active tenant context and `membership.manage`. The service accepts only HospitalAdmin callers, constrains target membership scopes to the caller's own scope, rejects self-modification and protects the last active tenant-wide HospitalAdmin. Tenant-wide scope is exclusive; hospital and department scopes are validated against tenant-owned records.
9. **Audit metadata only.** Membership provisioning, role/scope changes and lifecycle actions use the request audit pipeline. Audit records contain actor/resource/outcome metadata, not credentials, bearer tokens, patient payloads or raw request bodies.
10. **Do not cache membership decisions without invalidation.** Authorization continues to resolve from persisted tenant/membership state; a future cache requires a designed and tested invalidation strategy.

## Tenant lifecycle

The supported transitions are:

| Current state | Allowed next state |
|---|---|
| `Provisioning` | `Active`, `Closed` |
| `Active` | `Suspended`, `Closed` |
| `Suspended` | `Active`, `Closed` |
| `Closed` | None |

A transition that is not in this table returns a conflict. A suspended or closed tenant cannot be used for ordinary tenant-scoped requests.

## API surface

### Platform-operator routes

| Operation | Route |
|---|---|
| Current operator status (self only) | `GET /api/v1/platform/operator/status` | Returns only `isPlatformOperator` for the caller's validated identity; no subject lookup input |
| Create tenant | `POST /api/v1/platform/tenants` |
| List tenants | `GET /api/v1/platform/tenants` |
| Read tenant | `GET /api/v1/platform/tenants/{id}` |
| Provision initial administrator | `POST /api/v1/platform/tenants/{tenantId}/initial-administrator` |
| Activate | `POST /api/v1/platform/tenants/{id}/activate` |
| Suspend | `POST /api/v1/platform/tenants/{id}/suspend` |
| Close | `POST /api/v1/platform/tenants/{id}/close` |

All tenant registry/lifecycle routes above require the persisted PlatformOperator authorization policy. The self-status route requires authentication and returns only the caller's own operator boolean. A tenant HospitalAdmin is not a platform operator.

### Tenant membership routes

| Operation | Route |
|---|---|
| List memberships | `GET /api/v1/tenant/memberships` |
| Create membership | `POST /api/v1/tenant/memberships` |
| Update role/scopes | `PUT /api/v1/tenant/memberships/{membershipId}` |
| Deactivate | `POST /api/v1/tenant/memberships/{membershipId}/deactivate` |
| Reactivate | `POST /api/v1/tenant/memberships/{membershipId}/reactivate` |

These routes require an active tenant membership with `membership.manage`; the service further requires HospitalAdmin and enforces organization-scope containment.

## Provisioning sequence

1. A trusted platform operator creates a tenant in `Provisioning` state.
2. The platform operator provisions the initial administrator using the exact configured OIDC issuer and target user's subject.
3. The API rejects activation with 409 until an active tenant-wide HospitalAdmin membership exists; the Web platform workspace provisions that membership first and activates the tenant only after provisioning succeeds.
4. The HospitalAdmin signs in through OIDC and creates the first hospital and department through tenant-scoped APIs.
5. Subsequent users are added through tenant membership administration. Their roles and scopes are checked by the API on every request.

Provisioning does not implicitly create hospitals, departments, staff or patient records.

## Implementation and verification

The following API implementation is merged:
- Tenant registry, slug validation, lifecycle transitions and active-tenant enforcement.
- Persisted platform-operator registry and separate authorization policy.
- Development-only one-time platform-operator bootstrap.
- Authenticated platform-operator self-status endpoint (API PR #57, merge commit `916b75a9a8a81e37216d48562b65c23d07682fc1`).
- Platform-only tenant create/list/get and lifecycle routes.
- Server-enforced initial tenant-wide HospitalAdmin prerequisite before activation (API PR #58, merge commit `2d74b412f6b26274381ebd0d0ed461eb1b75ead8`).
- Platform tenant lifecycle workspace in [CompileCare.Web PR #36](https://github.com/CompileCare/CompileCare.Web/pull/36), merge commit `54589758db04848d25e2e078f3619c1932cc645a`.
- Tenant-scoped membership administration with role/scope validation, self-management rejection, last-admin protection and metadata audit.
- PostgreSQL integration coverage for platform-only access, lifecycle transitions, terminal closure, invalid/duplicate slugs, initial-admin concurrency, scoped membership operations and audit behavior.

The tenant membership administration UI is merged in [CompileCare.Web PR #35](https://github.com/CompileCare/CompileCare.Web/pull/35), merge commit `b8e4a4ec8864884f5e71dc070a319a5f4df07185`. It supports tenant-scoped membership list/create/update/deactivate/reactivate for authorized HospitalAdmins. Its workflow is documented in [Tenant membership administration](../../product/workflows/tenant-membership-administration.md).

The trusted platform-operator self-status API is merged in CompileCare.Api PR #57. The global platform tenant lifecycle UI is merged in [CompileCare.Web PR #36](https://github.com/CompileCare/CompileCare.Web/pull/36), merge commit `54589758db04848d25e2e078f3619c1932cc645a`. It shows the global workspace only after the API confirms the caller's persisted platform-operator identity. Hosted Vitest, production build, all 14 Chromium E2E tests and Docker smoke passed. Every global operation remains protected by the PlatformOperator API policy; the UI is not an authorization boundary.

## Remaining operational acceptance

Implementation and hosted PostgreSQL integration tests do not constitute production readiness. Before pilot activation:
- Exercise platform bootstrap/provisioning and tenant activation against the real local Keycloak + API + PostgreSQL Docker stack.
- Verify active, suspended and closed tenant behavior using authenticated browser/API requests.
- Verify initial-admin concurrency and membership role/scope changes using synthetic identities.
- Complete tenant closure, retention/export/deletion and customer offboarding policy.
- Complete backup/restore and production migration rehearsal, incident response and access-review procedures.
- Perform qualified privacy/legal review for applicable Indian requirements; do not claim regulatory compliance from these implementation decisions alone.

## Consequences

The registry prevents membership rows from authorizing access to unknown or inactive tenants. Platform privileges are separate from customer tenant roles. Tenant administrators can manage memberships only within their assigned organization scope, and protected administrator invariants prevent accidental lockout. Operational onboarding and tenant offboarding remain controlled platform responsibilities.

## Open questions

- Who is the designated production platform operator and what is the approved identity onboarding/revocation procedure?
- What are the tenant closure, retention, export and deletion policies?
- Can a customer tenant represent a hospital group, a single hospital, or both? How are branches scoped?
- Which customer onboarding steps are required before tenant activation beyond the initial administrator?
