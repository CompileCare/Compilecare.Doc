# Platform tenant lifecycle administration

## Purpose and boundary

The Platform tenants workspace is the global HeapCare control plane for customer tenant registration and lifecycle. It is separate from the hospital/tenant workspace and must never be shown to ordinary tenant HospitalAdmins.

The workspace is visible only when the API confirms that the currently authenticated, validated OIDC issuer/subject has an active persisted `PlatformOperator` record. The browser does not infer platform privileges from token roles, tenant roles or client-side configuration.

## Platform operator discovery

The Web application calls:

`GET /api/v1/platform/operator/status`

The endpoint:
- Requires an authenticated OIDC identity.
- Reads the validated issuer and subject from the server-authenticated principal.
- Queries the persisted PlatformOperators registry.
- Returns only `{ "isPlatformOperator": true|false }` for the current caller.
- Does not accept arbitrary issuer/subject values and does not expose the platform-operator list.

This is a presentation signal only. Every global tenant operation independently requires the persisted PlatformOperator authorization policy.

## Tenant provisioning workflow

1. The platform operator creates a tenant with a unique URL-safe slug and display name.
2. The tenant is persisted in `Provisioning` state. It cannot yet be used for ordinary tenant-scoped API access.
3. The platform operator first creates the intended user in the configured OIDC identity provider, then enters that user's exact OIDC subject. HeapCare creates the tenant membership; it does not create identity-provider users.
4. The platform operator provisions the initial tenant-wide HospitalAdmin through the platform-only initial-administrator endpoint.
5. The platform operator activates the tenant. The API independently verifies that an active tenant-wide HospitalAdmin exists and returns 409 if it does not.
6. The new HospitalAdmin signs in and creates the first hospital and department through tenant-scoped facility APIs.
7. Subsequent users and organization scopes are managed through the tenant Memberships workspace, not through the platform control plane.

Initial-administrator provisioning is idempotent for the same identity and rejects a competing identity. The API enforces the provisioning invariant; the UI sequence is not trusted as a security boundary.

## Lifecycle state machine

| Current state | Allowed next state | UI action |
|---|---|---|
| `Provisioning` | `Active` after initial admin exists | Provision administrator and activate |
| `Provisioning` | `Closed` | Close tenant |
| `Active` | `Suspended` | Suspend tenant |
| `Active` | `Closed` | Close tenant |
| `Suspended` | `Active` | Reactivate tenant |
| `Suspended` | `Closed` | Close tenant |
| `Closed` | None | Terminal; no action |

Suspension blocks tenant-scoped access until reactivation. Closure is terminal and the UI exposes no reactivation action for a Closed tenant. Tenant records are not hard-deleted.

## API routes

All global tenant registry/lifecycle routes except the self-status discovery endpoint require the persisted PlatformOperator policy and do not use `X-Tenant-ID`. The self-status endpoint requires authentication only and returns a boolean for the caller's own identity.

| Operation | Route |
|---|---|
| Current operator self-status | `GET /api/v1/platform/operator/status` |
| List tenants | `GET /api/v1/platform/tenants?page=1&pageSize=50` |
| Create tenant | `POST /api/v1/platform/tenants` |
| Read tenant | `GET /api/v1/platform/tenants/{id}` |
| Provision initial administrator | `POST /api/v1/platform/tenants/{tenantId}/initial-administrator` |
| Activate | `POST /api/v1/platform/tenants/{id}/activate` |
| Suspend | `POST /api/v1/platform/tenants/{id}/suspend` |
| Close | `POST /api/v1/platform/tenants/{id}/close` |

The self-status endpoint is authenticated and self-only; it does not reveal other operators. The tenant list is paginated at 50 entries per page.

## UI behavior and safeguards

- The Platform tenants navigation item and workspace are rendered only after the API self-status response is true.
- If operator status cannot be verified, the platform workspace remains hidden; tenant workspaces continue to follow their own membership permissions.
- New tenants are created in Provisioning state.
- The initial administrator form uses the current validated issuer and requires the target user's exact OIDC subject.
- Activation is attempted only after initial-administrator provisioning succeeds. If provisioning succeeds but activation fails, the same subject can be retried; the API's idempotency protects the workflow.
- Active tenants can be suspended or closed. Suspended tenants can be reactivated or closed. Closed tenants are terminal.
- Suspension and closure require explicit in-context confirmation. Closure is described as permanent.
- All platform operations are audited by the API. No patient data is involved in tenant registry administration.

## Verification status

The platform workspace is merged in [HeapCare.Web PR #36](https://github.com/HeapCare/HeapCare.Web/pull/36), merge commit `54589758db04848d25e2e078f3619c1932cc645a`.

Hosted Web CI runs 45747601458 and 45747606834 passed on final head `426c8dcbc39da1ea6c0d9d1ecf8c4ff07cb416bf`:
- 85 Vitest tests across 9 files.
- All 14 Chromium browser E2E tests.
- TypeScript/Vite production build.
- Docker build and container smoke.

The platform self-status API is merged in [HeapCare.Api PR #57](https://github.com/HeapCare/HeapCare.Api/pull/57), merge commit `916b75a9a8a81e37216d48562b65c23d07682fc1`. Server-side initial-administrator activation enforcement is merged in [HeapCare.Api PR #58](https://github.com/HeapCare/HeapCare.Api/pull/58), merge commit `2d74b412f6b26274381ebd0d0ed461eb1b75ead8`.

A real authenticated local-stack acceptance walkthrough remains required. Use a synthetic persisted platform operator, synthetic OIDC users and synthetic tenants. Verify that ordinary tenant users cannot see the platform workspace or invoke global routes; that provisioning without an initial admin cannot activate; and that Active, Suspended and Closed transitions behave as documented.
