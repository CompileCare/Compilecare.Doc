# Platform tenant lifecycle — local acceptance

**Status:** Pending developer-machine Docker acceptance  
**Implementation:** CompileCare.Web PR #36, merge commit `54589758db04848d25e2e078f3619c1932cc645a`  
**API dependencies:** CompileCare.Api PR #57 (platform self-status) and PR #58 (initial-administrator activation guard), both merged.

## Purpose

Verify the platform control plane against the real local Keycloak, CompileCare API and PostgreSQL stack. Hosted CI verifies compilation, component behavior, browser E2E and Docker image/container smoke; it does not prove the developer's local OIDC realm, client configuration, seeded identities or persisted platform-operator record are correct.

Use synthetic identities and tenant data only. Do not use production credentials, patient data or real customer records.

## Preconditions

- Pull the latest merged `main` branches for CompileCare.Api, CompileCare.Web, CompileCare.Local and Compilecare.Doc.
- Start the documented local Docker Compose stack and verify API, Web, Keycloak and PostgreSQL are healthy.
- Configure the Web OIDC client to use the local Keycloak realm and the API audience.
- Create a synthetic PlatformOperator through the approved Development-only bootstrap flow, or use an already provisioned synthetic operator.
- Create a separate synthetic ordinary HospitalAdmin identity and a test tenant with no active membership for that identity.
- Record the exact API, Web and Local commit SHAs used during the test.

## Acceptance scenarios

| ID | Scenario | Expected result |
|---|---|---|
| PLAT-001 | Sign in as a persisted PlatformOperator | `GET /api/v1/platform/operator/status` returns `isPlatformOperator: true`; Platform tenants navigation appears |
| PLAT-002 | Sign in as an ordinary tenant HospitalAdmin | Self-status returns false; Platform tenants navigation and workspace are absent |
| PLAT-003 | Call platform tenant list/create as ordinary HospitalAdmin | API rejects the request; no global tenant data is returned |
| PLAT-004 | Create a tenant with a unique valid slug | Tenant is created in `Provisioning` state |
| PLAT-005 | Create tenant with duplicate or invalid slug | API returns a validation/conflict response; no duplicate tenant is created |
| PLAT-006 | Attempt to activate Provisioning tenant before initial administrator exists | API returns 409 Conflict; tenant remains Provisioning |
| PLAT-007 | Create the target identity in Keycloak, then provision its exact OIDC subject | Initial tenant-wide HospitalAdmin membership is created |
| PLAT-008 | Retry provisioning with the same issuer/subject | Operation is idempotent and does not create duplicate membership |
| PLAT-009 | Attempt competing initial administrator provisioning | API rejects the competing identity |
| PLAT-010 | Activate after initial administrator provisioning | Tenant becomes Active and tenant-scoped access is allowed only to valid active memberships |
| PLAT-011 | Suspend an Active tenant | Explicit UI confirmation is required; tenant becomes Suspended and tenant-scoped API access is denied |
| PLAT-012 | Reactivate a Suspended tenant | Tenant becomes Active and valid tenant-scoped access resumes |
| PLAT-013 | Close a Provisioning, Active or Suspended tenant | Explicit terminal confirmation is required; tenant becomes Closed |
| PLAT-014 | Attempt to reactivate a Closed tenant | API rejects the transition; UI offers no reactivation action |
| PLAT-015 | Inspect audit records for create/provision/activate/suspend/close | Actor/resource/outcome metadata is present; secrets, tokens and raw request bodies are absent |

## Execution notes

- For each API scenario, record method, route, HTTP status, safe response fields and correlation ID.
- Verify global platform requests omit `X-Tenant-ID`; tenant-scoped requests must carry the selected tenant header.
- Verify the browser does not render platform controls while self-status is loading or if status lookup fails.
- Verify a failed activation after successful provisioning can be retried with the same identity.
- Confirm no tenant is activated solely because the UI attempted the sequence; the API must enforce the initial-admin invariant.
- Confirm closure is terminal and records are not hard-deleted.

## Evidence to attach

- Sanitized request/response evidence for PLAT-001, PLAT-003, PLAT-006, PLAT-010, PLAT-011 and PLAT-014.
- Browser screenshots at desktop (1440×900), tablet (1024×768) and narrow (390×844) where meaningful, following [UI screenshot evidence](../design/ui-screenshot-evidence.md).
- Docker Compose health output and relevant service logs, with secrets and tokens redacted.
- Database evidence showing tenant status and membership state using synthetic IDs only.
- A short run summary listing passed/failed/not-run scenarios and all tested commit SHAs.

## Exit criteria

All applicable scenarios pass on the real local stack; no cross-tenant or ordinary-tenant platform access is possible; tenant activation is blocked without the initial tenant-wide HospitalAdmin; lifecycle transitions match the documented state machine; audit records contain metadata only; and the evidence is reviewed before pilot sign-off.
