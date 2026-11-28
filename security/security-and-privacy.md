# Security and Privacy Baseline

**Status:** Persisted OIDC identity, tenant and organization-scoped authorization, membership administration, platform tenant lifecycle, audit controls and core hospital-workflow security are implemented; the full production security baseline remains in progress. This document is not a legal opinion or compliance certification.

HeapCare may process sensitive patient and operational information. Controls must be designed, implemented, tested, operated, and independently reviewed before production.

## Implemented API security slice

The current `HeapCare.Api` implementation provides:
- OIDC-compatible JWT Bearer validation of signed tokens, issuer, audience, expiry/lifetime and required subject.
- HTTPS-only authority/discovery configuration outside the explicitly permitted Development opt-in.
- Identity matching by the validated token issuer and `sub`; email is not used as a stable identity key.
- Tenant selection through `X-Tenant-ID`, followed by an exact active membership lookup for `(issuer, subject, tenantId)`. The header is not authentication or proof of tenant access.
- Route authorization is derived from persisted tenant membership and organization scopes. Current tenant roles include `HospitalAdmin`, `FacilityManager`, `FacilityViewer`, `PatientRegistrar` and `ClinicalUser`; `PlatformOperator` is a separate platform control-plane identity, not a tenant role.
- Server-created membership permission claims. Token-supplied legacy role/permission claims are not trusted for authorization.
- RFC 9457 Problem Details for authentication and authorization failures.

Tenant registry/lifecycle operations and initial tenant-wide HospitalAdmin provisioning are implemented behind the separate `PlatformOperator` control plane. Tenant membership APIs support scoped list, create, update, deactivate and reactivate operations; tenant-wide HospitalAdmin membership management is authorized independently and constrained by the caller's organization scope. The API stores the identity-provider issuer and subject, but does not create or verify accounts in the external OIDC provider: the user identity must already exist there before a membership can be created. Automated identity-provider joiner/mover/leaver synchronization, federation lifecycle and periodic access reviews remain outside the implemented slice. These capabilities do not constitute a complete identity lifecycle or a production security programme.

## Remaining security controls

- Periodic access reviews, documented joiner/mover/leaver approvals, external identity-provider account creation/deprovisioning and membership synchronization, and formal ownership of privileged/support access.
- Append-only audit of authentication, authorization, sensitive record access, membership/role changes, exports, administrative actions and privileged support activity.
- Tenant isolation across every data-access path, including relationships, caching, background jobs, message consumers, search, file/object storage, exports and observability.
- Least privilege, resource-level scope restrictions and controlled break-glass/support access.
- TLS in transit, encryption at rest, key and secret rotation, and secure key storage.
- Input validation, output encoding, CSRF protections where applicable, rate limiting, abuse protection and secure session/token handling. API CORS configuration accepts only explicit HTTPS origins outside Development. Development is also constrained to explicit HTTP(S) loopback origins (`localhost`, `*.localhost` or loopback IPs); remote hosts, wildcards, credentials, paths, queries and fragments are rejected in every environment. The Web's enforced CSP separately validates runtime API/OIDC origins. Cross-origin deployments must keep both allowlists aligned with the exact deployed hosts, and same-origin deployments should omit the API CORS allowlist.
- Dependency/container scanning, software supply-chain controls, least-privilege CI and release credentials.
- Backups, restore tests, incident response, disaster recovery and tested operational runbooks. The API's `/metrics` route now requires explicit trusted source CIDRs outside Development/Testing and returns 404 to untrusted sources after trusted forwarded-header processing; it remains a network allowlist, not user authentication. The production ingress must also keep it private, and health probes must be limited to approved sources. See the [Production Ingress and Network-Security Acceptance Checklist](../operations/production-ingress-and-network-checklist.md). See the [Backup, Restore and Disaster-Recovery Operational Baseline](../operations/backup-restore-disaster-recovery.md) for the protected-data inventory, recovery-objective decisions, isolated restore procedure and drill acceptance evidence.
- Data minimization, purpose limitation, retention, controlled export and verified deletion processes.
- No patient-identifiable information in logs, telemetry, test fixtures or screenshots unless specifically approved and protected.

## India-specific review

Qualified reviewers must assess applicable Indian data-protection and healthcare requirements, including DPDP Act/rules as applicable, ABDM ecosystem policies/agreements, and other obligations. Documentation and technical controls alone do not establish compliance.

## Clinical safety

Software supports clinical workflows. AI or automated outputs must not be represented as clinical decisions without an approved clinical safety process; human review/responsibility must remain explicit.

## Threat model

See [HeapCare initial application threat model](threat-model.md) for the current trust boundaries, abuse cases, implemented controls and residual risks. It is an engineering baseline, not an independent assessment or compliance certification.

## Production security gate

Threat model and data-flow review; complete authentication/authorization and cross-tenant tests; audit review; secrets/dependency scans; backup/restore verification; incident/breach procedures; suitable penetration testing; privacy/legal review; operational approvals; and formal production sign-off.
