# CompileCare Post-Remediation E2E Validation — 2026-11-26

**Verdict: VALIDATION PASS FOR IMPLEMENTED OPD MVP SCOPE**  
**Environment:** Developer macOS, official CompileCare.Local nine-container stack  
**Execution timestamp:** 2026-11-26 13:54:06 UTC (execution summary)  
**Data:** Synthetic tenants, users, patients and clinical workflows only  
**Scope boundary:** This is evidence for the tested implemented OPD MVP workflows. It is not hospital-pilot approval, production certification, a full WCAG audit, or independent penetration testing.

## Executive result

| Measure | Baseline | Post-remediation |
|---|---:|---:|
| Live browser E2E | 34/35 | **41/41** |
| Security/RBAC/tenant isolation | 24/24 | **24/24** |
| Combined live checks | 58/59 | **65/65** |
| Combined live check failures | 1 | **0** |
| Blocked live checks | 0 | **0** |
| Web unit tests (CI evidence) | 107/107 baseline | **113/113** |
| Parallel Playwright specs (CI evidence) | 15/16 baseline at 5 workers | **16/16** |
| Docker stack | 9 services | **9/9 running** |

The live browser suite improved from 34/35 to 41/41. Security checks remained 24/24. The 65/65 figure is the combined live E2E and security check count; the 113 unit tests and 16 Playwright specs are separate CI suites and are not included in that denominator.

The post-remediation execution summary records 41 total, 41 passed, zero failed and zero blocked browser checks. The separate failed-tests and blocked-tests artifacts are empty. Seven areas remain explicitly not implemented/deferred; they are not counted as passed.

## Defect verification matrix

| ID | PR | Status | Live verification evidence |
|---|---|---|---|
| DEF-001 | Web #53 | **Fixed** | E2E-APPT-DEF001 created/reassigned Kavita Joshi and published availability; provider appeared in Appointments without page.reload(). |
| DEF-002 | Web #53 | **Fixed** | E2E-TEN-DEF002 and E2E-CLIN-01 selected the active tenant again; selected hospital and hospital options remained intact. |
| DEF-003 | Web #53 | **Fixed** | E2E-DEPT-02 duplicate General Medicine (GEN) returned HTTP 409 and displayed operation-neutral conflict guidance, not delete-specific text. |
| DEF-004 | Web #53 | **Fixed** | E2E-MRN-DEF004 validated the MRN prefix HTML pattern with checkValidity(); no JavaScript/regex SyntaxError was logged. |
| DEF-005 — Overview | Web #54 | **Fixed** | E2E-DASH-01 revisited Overview without manual Refresh; appointment and queue metrics updated from 0 to 1 after the corresponding workflows. |
| DEF-005 — context restore | Web #54 | **Fixed** | E2E-CTX-RELOAD, E2E-CTX-INVALID and E2E-CTX-REVOKED verified tab-scoped tenant/hospital selection restoration, rejection of forged IDs and rejection of a deactivated membership's stored context. |
| DEF-007 | Web #55 | **Fixed** | CI: 113/113 Vitest tests across 14 files and 16/16 Playwright specs with five configured workers. |

DEF-006 is not represented as a completed defect. It is a grouped release backlog item with distinct governance and deployment dependencies:

- **CSP:** no Content-Security-Policy header was observed during this Antigravity run (before the subsequent Web hardening PR). Web PR #56 was subsequently merged and adds an enforced Nginx CSP rendered from validated runtime API/OIDC origins; CI run `45838673174` passed the header-generation and invalid-wildcard startup checks. CompileCare.Local PR #20 supplies local API/Keycloak origins and adds full-stack CSP assertions; its hosted run started before Web PR #56 merged, so it must be rerun against Web main before merge. Production CSP acceptance remains open until exact HTTPS production origins and real OIDC/API flows are tested.
- **Audit Log Viewer UI:** the API `/api/v1/audit/events` and CSV export are reported implemented, but the customer-facing Web screen is not. Keep UI, export workflow and permissions gated by ADR-016 and hospital/compliance approval.
- **OtpService:** runtime integration is not implemented; the OtpService main branch remains a skeleton. Patient contact verification is separately deferred under ADR-012.

## Browser and functional coverage

The 41 passing browser checks cover authentication and Keycloak PKCE, two-tenant switching and UI isolation, tenant/hospital context restoration, FacilityViewer gating, hospital/department/ward/room/bed hierarchy, MRN policy, patient registration and MRN lookup, staff management, appointment availability and booking, walk-in and queue progression, OPD clinical note signing, billing/receipt workflows, and responsive viewport checks.

The report records zero horizontal overflow at 1440×900, 1366×768, 768×1024 and 390×844. This is a responsive-layout observation, not a complete accessibility conformance assessment.

## Security and tenant isolation

All 24/24 security checks passed. Evidence includes:

- Anonymous, tampered-signature, wrong-issuer/audience and expired JWT requests rejected with HTTP 401.
- Cross-tenant hospital, patient, department, staff and membership access denied with HTTP 403/404 as expected by the anti-enumeration tests.
- FacilityViewer can read assigned facility data but is denied patient registration/search and staff creation.
- Department-scoped PatientRegistrar can access its own department's patient records and is denied another department's records.
- HospitalAdmin is denied PlatformOperator endpoints; PatientRegistrar is denied tenant-wide audit events.
- A deactivated membership is denied immediately despite a still-valid JWT.
- Invalid query binding returns HTTP 400.
- Anonymous burst limiting returns HTTP 429 with `Retry-After: 60`.

These results apply to the tested endpoints and synthetic identities. They do not replace an independent penetration test or review of future endpoints.

## Stack, observability and logging

All nine local containers were running: PostgreSQL, Keycloak, CompileCare API, CompileCare Web, OpenTelemetry Collector, Jaeger, Prometheus, Elasticsearch and Kibana.

- Prometheus: both `compilecare-api` and `compilecare-otel-collector` scrape jobs up; 140 HTTP metric series.
- OpenTelemetry Collector: metrics endpoint HTTP 200; 1,037 exposed HTTP metric lines; zero Collector error-log lines.
- Jaeger: 50 CompileCare.Api traces and 101 spans; PostgreSQL database spans and HTTP 4xx spans present.
- Elasticsearch: `compilecare-logs-2026.11` index; correlation-ID searches found expected allow/deny/bad-request events; sampled secret-leak search returned zero hits.
- Kibana: `compilecare-logs-*` data view available.

A zero-hit secret search is evidence for the sampled test data and query, not a universal proof that no secret can ever be logged.

## Performance observations

The uploaded performance artifact contains 167 API request-duration observations (plus five non-request metrics). Recomputed from those request observations:

| Metric | Result |
|---|---:|
| Initial load | 556 ms |
| API request samples | 167 |
| Minimum | 6 ms |
| p50 | 13 ms |
| p95 | 51 ms |
| Maximum | 111 ms |
| Keycloak login sample | 1,533 ms |

These are local synthetic-workload measurements, not a capacity, concurrency, resilience or production SLO certification.

## Console and network observations

The execution summary records six browser console error entries and 12 network error entries. They must not be described as zero console/network errors.

The six console entries correspond to HTTP responses also present in the network log: three GET requests for an unconfigured patient-identifier policy returned 404, one duplicate-department POST returned the expected 409 conflict, and the billing visit-invoice and OPD encounter lookups returned 404. The remaining six network entries are aborted requests (status 0 / `net::ERR_ABORTED`) during navigation. No JavaScript/regex SyntaxErrors were recorded in the MRN validation check.

The 404 responses should be reviewed against intended API semantics and the UI's handling of an absent policy/invoice/encounter; the duplicate 409 is an intentional negative test. Navigation-aborted requests are not automatically application defects, but should remain distinguishable from unexpected transport failures in future evidence.

## Explicitly deferred / not implemented

1. Patient contact storage and phone/email OTP verification — deferred pending ADR-012 privacy, security, key-management and retention decisions.
2. IPD admission, transfer, medication administration record and discharge.
3. LIS/RIS/PACS diagnostics integrations.
4. Pharmacy and inventory.
5. GST/tax, refunds and insurance/TPA claims; current billing scope is the OPD service catalogue, invoices, payments, receipts and daily collection.
6. Customer-facing Audit Log Viewer UI; API and CSV export are reported implemented, UI remains gated by ADR-016 governance approval.
7. CompileCare.OtpService runtime integration; main branch remains a skeleton.

## Remaining release gates

- [x] Implement an enforced, configurable Web CSP with startup validation and CI header/invalid-origin checks (Web PR #56; CI run `45838673174`).
- [x] Enforce production HTTPS-only CORS allowlist validation with unit/integration coverage (API PR #81; CI `45841339836`, Docker `45841339388`, migration rehearsal `45841339328`).
- [x] Require a configured PostgreSQL connection string and Npgsql `SSL Mode=VerifyFull` outside Development/Testing (API PR #82; CI `45843760937`, Docker `45843760927`, migration rehearsal `45843760942`).
- [ ] Configure the selected provider's trusted CA and verify an actual production-network PostgreSQL TLS connection with hostname validation.
- [ ] Set the exact production Web/API origins and verify CORS preflight plus authenticated browser requests through the selected ingress.
- [x] Verify CompileCare.Local emits the expected CSP through the full nine-service stack and real Keycloak Authorization Code + PKCE browser acceptance (Local PR #20 merged as `792196a2d19afeb128e8bf40205977d6b05e607a`; latest workflow run `45839528532` passed both jobs).
- [x] Set API HSTS to a one-year max-age for trusted HTTPS requests, restrict forwarded headers to configured proxy IPs/one hop, and test that untrusted forwarded schemes cannot enable HSTS (API PR #83; CI `45846891145`, Docker `45846891086`, migration rehearsal `45846891178`).
- [ ] Configure ingress TLS certificates, HTTP-to-HTTPS redirects, edge HSTS and exact trusted proxy IPs; verify production-style OIDC login, callback, silent renewal and API requests.
- [ ] Production KMS/secret ownership, rotation, encryption and operational access controls.
- [ ] Elasticsearch/Kibana authentication and TLS for the target deployment.
- [ ] Provider-native encrypted PITR backups, retention, defined RPO/RTO and automated restore drills.
- [ ] Windows 10/11 Docker Desktop launcher acceptance on a real Windows host.
- [x] Add automated public-shell accessibility regression checks for main landmark/heading, accessible control names, field labels, image alt attributes, keyboard focus/skip link, responsive overflow and reduced motion (Web PR #58; CI run `45844753971`, 19/19 Playwright specs passed).
- [x] Extend accessibility label/name checks into the authenticated HospitalAdmin workspace (Web PR #59; CI run `45845404253`, 113/113 unit tests and 20/20 Playwright specs passed).
- [x] Verify authenticated keyboard users can focus the skip link and move focus to the main landmark (Web PR #60; CI run `45846148079`, 113/113 unit tests and 20/20 Playwright specs passed).
- [ ] Complete authenticated hospital-user visual review, keyboard-only and screen-reader workflows; independent accessibility assessment.
- [ ] Independent penetration testing and qualified privacy/legal/compliance review for applicable Indian requirements.
- [ ] Hospital pilot acceptance and clinical governance approval.
- [ ] Resolve ADR-012 patient contact verification decisions before OTP implementation.
- [ ] Resolve ADR-016 named-reader, scope, retention, legal-hold and export approvals before customer-facing Audit Review UI or expanded audit permissions.

## Release decision

**VALIDATION PASS FOR IMPLEMENTED OPD MVP SCOPE. NOT HOSPITAL-PILOT OR PRODUCTION READY.**

The post-remediation evidence verifies all seven targeted functional/test defects (DEF-001 through DEF-005 and DEF-007; DEF-005 has two separately tested aspects) and records zero failures in the 65 live checks. DEF-006 and the release gates above remain open. Continue engineering only within approved scope; do not bypass ADR-012 or ADR-016 governance gates.

## Evidence artifacts received

This report is based on the post-remediation files supplied in the conversation: `results.json`, `execution-summary.json`, `failed-tests.json`, `blocked-tests.json`, `not-implemented.json`, `security-results.json`, `observability-results.json`, `performance-results.json`, `console-errors.log`, `network-errors.log` and `index.html`. The original evidence bundle should be retained unchanged for traceability.

### Follow-up local CSP integration acceptance

After Web PR #56 merged, CompileCare.Local PR #20 was merged as `792196a2d19afeb128e8bf40205977d6b05e607a`. Latest Local workflow run `45839528532` passed both jobs. The full-stack job built and ran the nine-service Compose environment, passed persisted multi-tenant API acceptance and completed the real Keycloak Authorization Code + PKCE browser suite. The Local smoke script verified the Web response CSP contained the explicit local API and Keycloak origins. The validation job also passed Keycloak realm reconciliation, Compose configuration assertions, and the Docker observability pipeline smoke test. Web CI run `45840521245` additionally verifies wildcard, insecure HTTP-without-opt-in and missing CSP origin configurations are rejected at container startup.

This is hosted local-stack integration evidence, not proof of the production CSP origins, production TLS/HSTS, ingress configuration or third-party origin inventory. The production CSP gate remains open until the exact deployed HTTPS origins are configured and the real sign-in, callback, silent-renew and API flows are validated in that deployment.
