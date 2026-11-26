# CompileCare Full-Platform E2E Validation — 2026-11-26

**Validation verdict: CONDITIONAL PASS**  
**Environment:** Developer macOS 27.2.1, Apple Silicon (arm64), Docker Desktop  
**Execution:** Official `CompileCare.Local` launcher; synthetic records only  
**Purpose:** Local full-stack functional, security, responsive, observability and performance baseline. This is not production certification or hospital-pilot approval.

## Executive result

Antigravity started the platform through `local/Start CompileCare.command` in 109 seconds. The launcher fast-forwarded `CompileCare.Api` to `f423424`, built the API and Web images, reconciled local Keycloak scopes/redirect configuration and completed `verify-stack.sh`.

All nine Compose containers were running and their application/readiness endpoints were checked: PostgreSQL, Keycloak, CompileCare API, CompileCare Web, OpenTelemetry Collector, Jaeger, Prometheus, Elasticsearch and Kibana.

The service-health artifact records nine running containers. Docker healthcheck fields are present only for services that define them; the remaining observability services were validated through their readiness, metrics, trace and log evidence rather than an invented Docker health status.

| Validation group | Result |
|---|---:|
| Live browser E2E/responsive checks | 34 / 35 passed |
| Security, multi-tenant, RBAC and rate-limit checks | 24 / 24 passed |
| Total live full-stack checks | 58 / 59 passed |
| Blocked live checks | 0 |
| Deferred/not implemented areas | 7 |
| Baseline Web unit tests | 107 / 107 passed |
| Serialized billing Playwright specs | 4 / 4 passed |
| Billing Playwright specs with 5 workers | 15 / 16 passed in the baseline run |

The single live failure was `E2E-APPT-STALE`: the Appointment Management provider dropdown did not refresh after a staff profile or assignment mutation until a full page reload. Additional UI/state defects were recorded separately.

## Functional coverage

The browser and API suites exercised the implemented workflows using synthetic records:

- Authentication, OIDC Authorization Code + PKCE and session initialization
- Platform tenant lifecycle and tenant switching
- Hospital, department, ward, room and bed management
- Staff profiles and hospital/department assignments
- MRN policy administration
- Patient registration, duplicate review and MRN assignment
- Reception name and MRN search
- Appointment availability, booking, listing and cancellation
- Walk-in registration and queue state transitions
- OPD clinical note entry, signing and encounter lifecycle
- Billing catalogue, invoice, payment, receipt and daily collection report
- Membership administration, permission gating and RBAC

Features that were not implemented or were deliberately deferred are listed below; they were not counted as passed.

## Security and multi-tenancy

All 24 security checks passed in the baseline run.

Evidence included anonymous, tampered JWT, wrong issuer/audience and expired-token requests rejected with HTTP 401; cross-tenant hospital and patient access denied (403/404 anti-enumeration behavior); FacilityViewer read-only behavior enforced by the API; department-scoped PatientRegistrar containment; immediate denial after membership deactivation; HTTP 400 invalid-request binding; and HTTP 429 rate limiting with `Retry-After: 60`.

These results validate the tested API surface and synthetic scenarios; new endpoints still require explicit positive/negative permission, tenant-scope and audit tests.

## Observability evidence

The report recorded Prometheus targets for `compilecare-api` and `compilecare-otel-collector` at 1/1; 155 HTTP metric series and approximately 1,309 Collector-exposed application metric lines; 50 CompileCare.Api traces and 169 spans in Jaeger including PostgreSQL database spans; synthetic correlation IDs in Elasticsearch/Kibana; and no credential/JWT leakage found in the sampled synthetic log search.

The report's individual endpoint/service-health evidence should be retained alongside these aggregate observations; an open dashboard alone is not proof of a working telemetry pipeline.

## Performance baseline

| Measurement | Result |
|---|---:|
| Initial workspace load | 548 ms |
| Keycloak PKCE login | 780–1,202 ms |
| API calls measured | 134 |
| API latency minimum | 4 ms |
| API latency p50 | 12 ms |
| API latency p95 | 27 ms |
| API latency maximum | 289 ms |

These figures describe one local developer environment and synthetic workload. They are not production capacity, concurrency, resilience or hospital-traffic certification.

## Defect register and remediation

| Finding | Original observation | Current status |
|---|---|---|
| DEF-001 | Appointment provider dropdown stayed stale after staff changes | **Fixed** — Web PR #53; staff mutations now increment a shared directory revision and Appointment Management reloads providers |
| DEF-002 | Selecting the already-active tenant cleared hospital state | **Fixed** — Web PR #53; identical tenant selection is now a no-op |
| DEF-003 | Duplicate department HTTP 409 showed delete-specific guidance | **Fixed** — Web PR #53; conflict copy is operation-neutral |
| DEF-004 | MRN prefix pattern caused a Chromium Unicode Sets regex SyntaxError | **Fixed** — Web PR #53; hyphen is escaped and regression-tested with the `v` flag |
| DEF-005 | Overview metrics stayed stale and tenant/hospital context was lost on hard reload | **Fixed in code** — Web PR #54; Overview refreshes on re-entry and tab-scoped tenant/hospital IDs are restored only after validating them against current memberships and API-filtered hospitals |
| DEF-006 | CSP absent; Audit Viewer UI and OTP service integration unavailable | **Reclassified** — CSP is a deployment-specific security gate; Audit Viewer UI remains deferred under ADR-016; OtpService runtime integration remains deferred/not implemented |
| DEF-007 | Mocked billing Playwright test was timing-flaky with five workers | **Fixed in test** — Web PR #55 waits for the successful queue API response before asserting visit options |

### Merged remediation

- [Web PR #53](https://github.com/CompileCare/CompileCare.Web/pull/53) — merged as `e62d97a2e04a5760f4b47fbbd7a2054a6ee31588`. CI run `45827629738` passed unit tests, production build, Chromium E2E, real-OIDC suite discoverability and Docker smoke.
- [Web PR #54](https://github.com/CompileCare/CompileCare.Web/pull/54) — merged as `0a3bb144060f27ebff76693a40f30e503b7a8278`. CI run `45828086411` passed 113 unit tests across 14 files, 16 browser E2E tests, production build, real-OIDC suite discoverability and Docker smoke.
- [Web PR #55](https://github.com/CompileCare/CompileCare.Web/pull/55) — merged as `256c6df4d702071d3c4a0c13577b86cefc90e938`. CI run `45828251071` passed 108 unit tests across 13 files, all 16 browser E2E tests, production build, real-OIDC suite discoverability and Docker smoke. The PR's Playwright run used the configured parallel workers.

CI run totals above are reported per PR; PR #55 was tested from its own base before PR #54's merge. The PRs touch separate implementation/test areas. A fresh full local Antigravity validation against the combined current `main` remains recommended.

## Deferred and not implemented

1. Patient phone/email contact verification and OTP — deferred pending ADR-012 privacy/security/key-management/retention decisions.
2. IPD admission, transfer, medication administration record and discharge workflows.
3. LIS, RIS and PACS diagnostics integrations.
4. Pharmacy and inventory.
5. Billing GST/tax, refunds and insurance/TPA claims. Current billing scope is the OPD catalogue, invoices, payments, receipts and daily collection.
6. Customer-facing Audit Log Viewer UI — explicitly deferred pending hospital/compliance approval and ADR-016 governance decisions. Do not broaden audit permissions or implement customer-facing audit browsing/export UX yet.
7. CompileCare.OtpService runtime integration — its main branch remains a skeleton and it is not wired into the current application.

## Remaining release and pilot gates

- [ ] Re-run Antigravity's full local E2E and visual evidence collection against the combined current Web `main` after PRs #53–#55; specifically recheck DEF-001 through DEF-005 and the parallel-worker billing test.
- [ ] Review the sessionStorage selection behavior with multiple real Keycloak memberships, revoked memberships and hospital-scope changes. The API remains the authorization boundary.
- [ ] Complete authenticated hospital-user visual/accessibility review, including keyboard-only and screen-reader workflows. The baseline found zero horizontal overflow at 1440×900, 1366×768, 768×1024 and 390×844, but that is not a complete WCAG audit.
- [ ] Execute the Windows 10/11 Docker Desktop launcher acceptance on a real Windows host; Linux CI cannot prove Windows-host behavior.
- [ ] Define exact production Keycloak/API origins, implement and browser-test a restrictive CSP, and configure trusted reverse-proxy addresses.
- [ ] Complete provider-native backups/PITR, encryption/key ownership, retention, RPO/RTO and scheduled restore drills.
- [ ] Complete hospital pilot, security, privacy and applicable legal/regulatory sign-off.

The absence of a CSP header is not permission to add an untested blanket policy. Web PR #46 deliberately deferred CSP until deployment-specific origins are known and allow-listed safely.

## Release decision

**CONDITIONAL PASS for the tested local full-stack workflows and security scenarios. Not approved for hospital pilot or production deployment.**

The platform has strong evidence for its implemented OPD, tenant isolation, RBAC and observability foundations. The merged PRs address the reported code/test defects, but the original local report is a baseline and must be rerun against the combined current main. Deployment security, Windows-host acceptance, backup/PITR, hospital-user accessibility review and pilot approvals remain release gates.

## Evidence artifacts

Antigravity generated the full HTML report and supporting artifacts on the developer workstation:

- HTML report: `e2e-validation-2026-11-26/index.html`
- Evidence directory: `e2e-validation-2026-11-26/`
- Supporting files include results, execution summary, failed/blocked/not-implemented cases, security results, service health, performance and observability JSON, console/network logs and 32 screenshots.

The local evidence bundle is not embedded in this documentation repository. Keep the original bundle unchanged as the historical baseline and generate a new dated directory for the post-remediation rerun.
