# CompileCare initial application threat model

**Status:** Initial engineering threat model — 2026-11-26  
**Owners:** Engineering, product and security; independent security/privacy/legal review remains required  
**Scope:** Current CompileCare hospital workspace and API: React/TypeScript browser client, OIDC identity provider, .NET API, PostgreSQL persistence, audit pipeline, OpenTelemetry pipeline, local Docker Compose environment and GitHub Actions/package supply chain.

This document records the current trust boundaries, material threats, implemented safeguards and unresolved risks. It is a working engineering baseline, not a formal risk acceptance, penetration-test report, privacy impact assessment or compliance certification. Threat likelihood and impact must be re-evaluated with the pilot hospital, hosting provider and qualified reviewers.

## 1. System context and sensitive assets

### Main data flows

1. A hospital user opens the browser workspace and authenticates with the configured OIDC provider using Authorization Code flow with PKCE.
2. The browser obtains an access token and calls the API over HTTPS in production. The browser supplies a tenant selection header; that header is only a requested context, not proof of access.
3. The API validates the token signature, issuer, audience, lifetime and subject. For tenant-scoped endpoints it resolves an active membership by validated issuer, subject and requested tenant, loads persisted role and organization scopes, then derives permission claims server-side.
4. API services enforce resource and organization boundaries and use EF Core/PostgreSQL for tenant-aware persistence.
5. The API emits operational telemetry to the configured OpenTelemetry pipeline. Audit events are persisted separately as metadata-only, append-only records.
6. GitHub Actions builds and tests the application, obtains packages from the configured GitHub Packages feed, and uses disposable databases/containers for CI verification.

### Assets and classification

| Asset | Classification / impact if exposed or altered |
|---|---|
| Patient identity, MRN, registration and demographics | Sensitive personal data; cross-tenant disclosure or unauthorized alteration is high impact |
| OPD notes, encounter status, amendments and clinical workflow | Sensitive health/clinical data; unauthorized access or tampering is high impact |
| Appointments, queues, provider/staff profiles and assignments | Sensitive operational and personal data |
| Invoices, payments, receipts and collection summaries | Sensitive financial and personal data |
| OIDC access/refresh tokens, issuer/subject and session state | Authentication secrets; theft can enable account impersonation |
| Tenant memberships, roles, organization scopes and platform-operator state | Authorization control plane; tampering can enable privilege escalation |
| Audit events and exports | Sensitive security/accountability metadata; tampering or disclosure undermines investigations |
| Database credentials, package credentials, signing/identity keys and backups | Secrets and recovery assets; compromise can enable system/data access |
| Logs, traces, metrics, CI artifacts and screenshots | Potential secondary disclosure channel; must not contain patient payloads or credentials |

## 2. Trust boundaries

- **TB-1 — User device/browser:** outside CompileCare's control. Browser extensions, local malware, XSS and compromised dependencies can access browser-held state.
- **TB-2 — Browser to identity provider:** OIDC redirect/token exchange. Issuer and signing-key trust must be pinned to the configured authority and validated by the API.
- **TB-3 — Browser to API:** bearer-token and tenant-context boundary. Client-supplied IDs, headers and UI permission visibility are untrusted inputs.
- **TB-4 — API to PostgreSQL:** trusted service identity boundary. Tenant predicates, foreign keys, uniqueness and transaction behavior must be enforced server-side.
- **TB-5 — API to telemetry/audit systems:** operational data boundary. Logs/traces/metrics and audit records have different purposes and access policies; telemetry must not become a copy of clinical data.
- **TB-6 — CI/build/package supply chain:** source, Actions workflows, third-party actions, package feeds and build secrets can influence shipped artifacts.
- **TB-7 — Local developer environment:** Docker, local Keycloak, PostgreSQL volumes, .env and package credentials. Local credentials/data must never be reused in production.

## 3. Existing security controls

These controls are present in the current code or verified hosted workflows; they are not a claim that every threat is fully mitigated.

- OIDC JWT validation requires signed tokens, validates issuer, audience, lifetime, signing key and a single non-empty subject. HTTP identity metadata is permitted only in Development with explicit opt-in.
- The API derives the issuer from the validated security token. Authentication-failure logs intentionally do not include token contents.
- Tenant-scoped requests require exactly one non-empty tenant GUID header, then an active persisted membership matching validated issuer, subject and tenant. A missing membership or missing organization scope is denied.
- Token-supplied CompileCare role/permission claims are removed before the API adds server-derived claims from the persisted membership role. UI feature gating is not the authorization boundary.
- Tenant-aware persistence includes query filters, tenant-aware relationships/uniqueness and restrictive delete behavior; cross-tenant, cross-hospital and cross-department tests cover patient, identifier, appointment, queue, OPD, billing and staff workflows.
- Audit records are metadata-only and append-only. Successful business writes and their audit event share a transaction; failure to persist the success audit rolls back the business operation. Endpoint registration tests require audit metadata on authenticated routes.
- Audit listing/export is bounded to validated tenant-wide membership scope. Hospital/department-scoped memberships are denied. Query/export bounds are 90 days/100 rows and 31 days/5,000 rows respectively; CSV formula/control-character values are neutralized and export requests are audited.
- The web client uses OIDC Authorization Code flow with PKCE and session-scoped browser storage. Patient workflow data is intended to remain in component memory and API requests use no-store; this must continue to be verified as new modules are added.
- The Web Nginx container now emits an enforced Content-Security-Policy from runtime-configured API/OIDC origins. Startup rejects missing, malformed, wildcard, path-bearing and (unless explicitly opted into for local development) HTTP origins. The policy blocks objects and framing, limits scripts to the Web origin, and does not allow `unsafe-eval` or inline scripts. The current UI retains `style-src 'unsafe-inline'` for compatibility; CSP still requires validation against the exact production OIDC/API deployment.
- Local development generates independent random Keycloak/PostgreSQL credentials for fresh setups, keeps .env/package credentials out of source control and binds service ports to loopback. This is not a production secrets-management design.
- PostgreSQL migration rehearsal runs against disposable PostgreSQL 16 databases for upgrade and fresh-install paths. Logical backup/restore helper tests validate archive/checksum/manifest and guarded restore behavior. Weekly CI repeats the synthetic logical backup/restore regression.

## 4. Threat scenarios and residual risk

| ID | Threat scenario | Existing safeguards | Residual risk / required action |
|---|---|---|---|
| TM-01 | Attacker changes X-Tenant-ID or guesses another tenant's resource IDs to read or alter records | Header is only a selector; API requires active issuer/subject/tenant membership; tenant filters and resource authorization; cross-tenant tests | Continue adding isolation tests for every new endpoint, export, cache, background job, file store and integration. Verify query plans and tenant constraints on production-like data. |
| TM-02 | Hospital user substitutes another department's patient, appointment, queue or encounter identifier | Persisted organization scopes and resource-level authorization; department-isolation regression coverage | Complete end-to-end local acceptance for all role/scope combinations; test future modules and all negative paths. |
| TM-03 | XSS, malicious browser extension or compromised dependency steals browser-held OIDC tokens | PKCE, validated token lifetime/signature/issuer/audience, session-scoped rather than persistent local storage, silent renewal; enforced Nginx CSP with explicit runtime API/OIDC origins, no inline scripts or `unsafe-eval`, and startup validation of origin configuration | **High residual concern:** sessionStorage remains accessible to same-origin JavaScript; current CSP retains `style-src 'unsafe-inline'`. Test the policy against exact production OIDC/API origins and all real authentication/session flows, review output encoding and script injection surfaces, dependency integrity/scanning, token lifetime/refresh-token behavior and whether a BFF/HttpOnly-cookie architecture is warranted. Complete a focused independent web application security review. |
| TM-04 | Forged role/permission claims or a tenant administrator attempts privilege escalation | API discards reserved role/permission claims from token identities and creates permissions from persisted membership roles; platform operator uses a separate authorization requirement | Review role granularity and privileged provisioning. Add explicit tests for every role and negative permission combination as new capabilities are introduced. |
| TM-05 | Compromised or misused platform operator accesses tenant data or changes tenant lifecycle | Separate platform-operator policy and bootstrap controls; platform actions are audited; tenant-scoped audit query does not grant platform-wide access | Define operator roster, MFA/conditional access, time-bounded emergency access, independent review, alerting and break-glass procedures. Verify platform routes and support tooling under a formal threat review. |
| TM-06 | Attacker or privileged process alters/deletes audit history, or business write succeeds without an audit record | Application and PostgreSQL append-only protections; shared transaction; failure-injection rollback test; route metadata guard | Define retention, archival, legal hold, backup expiry and access-review policy. Monitor audit pipeline failure and unusual export activity. |
| TM-07 | Patient/clinical data leaks through logs, traces, metrics, URLs, browser storage, screenshots or support bundles | Audit excludes bodies/clinical payloads; auth failures do not log token contents; UI no-store and component-memory design; synthetic CI data | Review every instrumentation/enrichment path, exception payload, reverse-proxy log, screenshot and support export. Establish automated PHI/secret scanning and a documented redaction test suite. |
| TM-08 | Tenant-wide audit reader bulk-exports actor/resource metadata or uses CSV formula payloads against spreadsheet users | Dedicated server permission plus tenant-wide scope; hospital-scoped memberships denied; 31-day/5,000-row export cap; CSV neutralization; export itself audited | **Policy remains provisional:** obtain a named pilot reader roster and approved role mapping. Add export monitoring/review, retention and legal-hold controls before production. Do not expand to department-scoped or asynchronous bulk export without trusted scope attribution and approval. |
| TM-09 | Database/package credential compromise or backup theft exposes all tenant data | Local secrets are ignored and generated on fresh setup; package credentials use BuildKit secrets; local ports bind to loopback; backup helper uses restrictive file modes and checksums | Production secret manager, rotation, least-privilege database identities, TLS, encryption-at-rest, backup encryption/key ownership and off-site access controls are not yet approved/verified. |
| TM-10 | Unbounded queries, exports or request floods exhaust API/database resources | Page-size and date-range caps; export row cap; bounded migration and test workflows | Rate limiting, per-tenant quotas, request timeouts, query-cost monitoring and abuse response require verification and policy. Do not assume pagination alone prevents denial of service. |
| TM-11 | Malicious dependency, GitHub Action, package or CI credential compromises release artifacts | CI permissions are restricted where configured; package access is scoped; builds/tests run in GitHub Actions | Complete organization-wide secret/dependency scanning review, action pinning policy, SBOM, artifact provenance/signing and release credential separation. Verify every public repository's historical exposure status. |
| TM-12 | Schema migration causes data loss, downtime or failed recovery | Base-to-upgrade and fresh PostgreSQL migration rehearsal; idempotent script re-run; logical backup/restore integrity tests | Choose production hosting/provider; rehearse against representative sanitized data; define expand/contract policy, physical backup/PITR, RPO/RTO, encryption, retention and scheduled provider-native restore drills. |
| TM-13 | Development-only insecure configuration is reused in production | API rejects HTTP OIDC metadata outside Development; local compose explicitly opts into Development; local ports are loopback-bound | Add deployment configuration validation and environment-specific release gates. Independently review production TLS, CORS, host headers, network boundaries and secret injection. |

## 5. Required decisions and release gates

### Product, hospital and privacy decisions

- Name the hospital functions permitted to read tenant-wide audit events and approve the exact permission/role mapping.
- Define event retention by category, archival, legal hold, deletion and backup expiry with qualified privacy/legal review.
- Define the purpose, approval, review and alerting requirements for audit exports.
- Confirm patient, clinical, workforce and financial data classifications, data minimization and retention/deletion obligations for the pilot.
- Define incident/breach handling, support access and emergency/break-glass responsibilities.

### Engineering and security controls

- Complete a data-flow and abuse-case review for every new module, background process, cache, export, file store and integration.
- Complete a focused browser security review for XSS/CSP/token storage and an API security review for rate limiting, resource authorization and tenant isolation.
- Review secrets management, signing-key/identity-provider key rotation, TLS, encryption at rest, package credentials, dependency scanning and release provenance.
- Select the production database provider and implement/verify physical backup/PITR, encrypted off-site storage, retention, RPO/RTO and provider-native restore drills.
- Run authenticated local Docker/Keycloak/API/PostgreSQL acceptance using synthetic records; retain commit SHAs, test results and safe screenshots.
- Obtain qualified penetration testing, privacy/legal review and formal production sign-off. Do not claim regulatory compliance based only on this threat model or technical tests.

## 6. Evidence references

- API PR #66 — same-hospital cross-department isolation tests across appointment and queue operations.
- API PR #67 — audit persistence failure rolls back business writes.
- API PR #68/#69 — authenticated endpoint audit metadata coverage.
- API PR #71/#72 — bounded tenant-wide audit query/export, export audit event and CSV formula/control-character hardening.
- API PR #75 — PostgreSQL integration tests prove PatientRegistrar, ClinicalUser, FacilityManager and FacilityViewer cannot query or export audit events, even when the signed token contains forged audit.read, tenant-role and standard role claims; authorization remains derived from persisted membership.
- API PR #73 — PostgreSQL migration rehearsal and integration-test isolation.
- Web PR #43 — skip-link keyboard focus regression; hosted browser coverage is not a full WCAG or real-identity acceptance.
- Database PR #4/#5 — logical backup/restore integrity and weekly synthetic CI regression.
- Local PR #9 — local infrastructure smoke script merged as d9e52044051b3fadb15e8ed3f679f0346ad31952; shell syntax, Keycloak Docker import/reconciliation, Compose validation and observability Docker smoke passed. The script itself still needs to be run on the developer Mac.

## 7. Review cadence

Revisit this threat model whenever authentication, tenant membership, roles/scopes, patient/clinical/billing data flows, audit/export, storage, hosting, background processing or external integrations change. At minimum, review it before the pilot production decision and after any security incident or material architecture change.
