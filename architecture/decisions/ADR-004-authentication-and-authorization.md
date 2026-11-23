# ADR-004: Identity, Authentication, and Authorization

- **Status:** Proposed
- **Date:** 2026-11-23

## Context
Healthcare workflows require secure user identity, role and scope-based access, auditability, and potentially client-specific identity providers.

## Decision
- Prefer standards-based OIDC/OAuth 2.0-compatible identity.
- Authorization combines roles/permissions with tenant, hospital, branch, department, and workflow scope as applicable.
- Enforce authorization in API and use-case boundaries; UI visibility is not security.
- Support MFA policy and secure session/token lifecycle according to deployment and risk.
- Sensitive and privileged access is auditable.
- Emergency/break-glass access, if supported, must be explicitly designed, time-limited, justified, and audited.

## Open questions
Identity provider, user lifecycle, MFA requirements, SSO, doctor multi-tenant identity, break-glass workflows, and client-managed identity.
