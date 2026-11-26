# Security and Privacy Baseline

**Status:** Initial API identity and facility-authorization controls are implemented; the full production security baseline remains in progress. This document is not a legal opinion or compliance certification.

CompileCare may process sensitive patient and operational information. Controls must be designed, implemented, tested, operated, and independently reviewed before production.

## Implemented API security slice

The current `CompileCare.Api` implementation provides:
- OIDC-compatible JWT Bearer validation of signed tokens, issuer, audience, expiry/lifetime and required subject.
- HTTPS-only authority/discovery configuration outside the explicitly permitted Development opt-in.
- Identity matching by the validated token issuer and `sub`; email is not used as a stable identity key.
- Tenant selection through `X-Tenant-ID`, followed by an exact active membership lookup for `(issuer, subject, tenantId)`. The header is not authentication or proof of tenant access.
- Facility route authorization based on a role loaded from CompileCare persistence. Current facility roles are `HospitalAdmin`, `FacilityManager`, and `FacilityViewer`.
- Server-created membership permission claims. Token-supplied CompileCare role/permission claims are not trusted for authorization.
- RFC 9457 Problem Details for authentication and authorization failures.

The implementation is the first authenticated tenant boundary for the facility API; it is not a complete identity lifecycle or production security programme. Membership provisioning is currently a trusted operational responsibility. There is no self-service tenant onboarding or membership administration API yet.

## Remaining security controls

- Tenant registry, tenant lifecycle, secure membership provisioning, role assignment and periodic access reviews.
- Append-only audit of authentication, authorization, sensitive record access, membership/role changes, exports, administrative actions and privileged support activity.
- Tenant isolation across every data-access path, including relationships, caching, background jobs, message consumers, search, file/object storage, exports and observability.
- Least privilege, resource-level scope restrictions and controlled break-glass/support access.
- TLS in transit, encryption at rest, key and secret rotation, and secure key storage.
- Input validation, output encoding, CSRF protections where applicable, rate limiting, abuse protection and secure session/token handling. API CORS configuration is validated outside Development to accept only explicit HTTPS origins (no wildcard, credentials, path, query or fragment); the Web's enforced CSP separately validates runtime API/OIDC origins. Cross-origin deployments must keep both allowlists aligned with the exact deployed hosts, and same-origin deployments should omit the API CORS allowlist.
- Dependency/container scanning, software supply-chain controls, least-privilege CI and release credentials.
- Backups, restore tests, incident response, disaster recovery and tested operational runbooks. See the [Backup, Restore and Disaster-Recovery Operational Baseline](../operations/backup-restore-disaster-recovery.md) for the protected-data inventory, recovery-objective decisions, isolated restore procedure and drill acceptance evidence.
- Data minimization, purpose limitation, retention, controlled export and verified deletion processes.
- No patient-identifiable information in logs, telemetry, test fixtures or screenshots unless specifically approved and protected.

## India-specific review

Qualified reviewers must assess applicable Indian data-protection and healthcare requirements, including DPDP Act/rules as applicable, ABDM ecosystem policies/agreements, and other obligations. Documentation and technical controls alone do not establish compliance.

## Clinical safety

Software supports clinical workflows. AI or automated outputs must not be represented as clinical decisions without an approved clinical safety process; human review/responsibility must remain explicit.

## Threat model

See [CompileCare initial application threat model](threat-model.md) for the current trust boundaries, abuse cases, implemented controls and residual risks. It is an engineering baseline, not an independent assessment or compliance certification.

## Production security gate

Threat model and data-flow review; complete authentication/authorization and cross-tenant tests; audit review; secrets/dependency scans; backup/restore verification; incident/breach procedures; suitable penetration testing; privacy/legal review; operational approvals; and formal production sign-off.
