# HeapCare Post-Migration Full-Platform QA Report — 2026-11-27

**Date:** 2026-11-27  
**Verdict:** **CONDITIONAL PASS** — Full local nine-container HeapCare stack, multi-tenant API/Web OPD MVP workflows, RBAC/tenant isolation, observability pipelines, and private NuGet package consumption are **verified working on `main`**. Production/hospital-pilot certification remains **gated** on external infrastructure, security, and deferred modules listed in Section 14.  
**Scope:** End-to-end execution across all 11 `HeapCare/*` repositories using synthetic tenants, identities, patients, and transactions only (zero production data, zero secrets exposed).

---

## 1. Executive Summary & Overall Verdict

Following the GitHub organization migration from `CompileCare` to `HeapCare`, full post-migration startup, build, test, security, observability, and live browser QA were executed from a clean local `main` baseline via the official `HeapCare.Local` macOS launcher (`local/Start HeapCare.command`).

Initial startup exposed a live build-blocking migration defect (**`HC-DEF-001`**: `HeapCare.Api/NuGet.Config` still pointed to the 404 legacy `CompileCare` GitHub NuGet feed) alongside stale clone/workflow slugs (**`HC-DEF-002`**), broken `Observability` sample project references (**`HC-DEF-003`**), a walk-in queue token UI state defect (**`HC-DEF-007`**), a Kibana data-view pattern mismatch with `HeapCare.Api`'s live `heapcare-logs-*` Elasticsearch indices (**`HC-DEF-008`**), and minor stale `CompileCare` copy/links across `Web`, `Site`, `ApiClient`, and `Caching`.

All **8 confirmed defects** (`HC-DEF-001` through `HC-DEF-008`, with `HC-DEF-004/005/006` triaged as `Web`/UI branding & docs) were resolved through **8 focused Pull Requests** across 7 repositories, validated by green GitHub Actions CI before squash-merge into `main`, and re-verified on the freshly rebuilt nine-service Docker stack:

- **9 / 9 Docker services** running (`postgres`, `keycloak`, `api`, `web`, `otel-collector`, `jaeger`, `prometheus`, `elasticsearch`, `kibana`), with 5 healthcheck-defined services all `(healthy)`.
- **27 / 27 live browser E2E checks passing** against real Keycloak PKCE + PostgreSQL + API (zero JS/CSP errors, 0 px horizontal overflow at `1440×900` and `390×844`, verified persistence across `api` + `web` container restart).
- **24 / 24 security / RBAC / tenant-isolation / rate-limiting checks passing** (`401`, `403`, `404` anti-enumeration, `400`, `429 Retry-After=60`).
- **100% local & CI test pass rates across implemented codebases**, including `12/12` Api UnitTests, `187/187` Api PostgreSQL IntegrationTests, `131/131` Web Vitest specs, `20/20` Web Playwright specs, `30/30` ApiClient tests, `13/13` Caching tests, `2/2` Observability tests, and `HeapCare.Database` Docker backup/restore verification.

---

## 2. Post-Migration Platform Completion Percentage

To avoid arbitrary scoring, platform readiness is measured transparently across **6 weighted dimensions** using only observed artifacts:

| Dimension | Weight | Score Basis | Observed Completion | Weighted Contribution |
|---|---:|---|---:|---:|
| 1. GitHub org, repo slugs & clean `main` checkouts | 15% | 11 / 11 repos under `HeapCare/*`, clean `main`, `HeapCare` NuGet feed restored | 100% | 15.0% |
| 2. Build, unit, integration & E2E test suites | 20% | All implemented repo suites passing on `main` (`Api`, `Web`, `ApiClient`, `Caching`, `Observability`, `Database`, `Site`, `Local`) | 100% | 20.0% |
| 3. Live 9-service runtime, E2E workflows & restart persistence | 20% | 27/27 live E2E + 2-tenant isolation + persistence after container restart | 100% | 20.0% |
| 4. Security, RBAC, CSP & `OtpService` gate | 15% | 24/24 live security assertions + strict CSP + closed `OtpService#1`; pending external pentest | 85% | 12.8% |
| 5. Observability pipeline (OTel, Jaeger, Prometheus, ES, Kibana) | 10% | Full trace/metric/log ingestion verified (`heapcare-logs-*` + Npgsql span + 0 collector errors) | 100% | 10.0% |
| 6. Production cutover & unimplemented product modules | 20% | `Mobile` & `OtpService` skeletons (0% runtime), unpurchased `heapcare.in` DNS/TLS, deferred IPD/pharmacy/lab/tax billing, pending production PITR/key-rotation | 10% | 2.0% |
| **Weighted Total** | **100%** | **Overall HeapCare Platform Migration & Release Readiness** | — | **79.8% (~80%)** |

- **Implemented OPD MVP + Brand Migration Verification:** **100%** (`51/51` combined live E2E & security assertions passing post-fixes).
- **Overall Production Platform Completion:** **~80%** (remaining ~20% reflects deferred clinical/revenue modules, empty `Mobile`/`OtpService` services, and external production cutover gates).

---

## 3. Repository Inventory & Post-Migration State

All 11 local clones point to `https://github.com/HeapCare/HeapCare.*.git`, are on clean `main` (`dirty=0`), and match post-merge upstream commits:

| Repository | Remote URL | Branch | Verified Commit SHA | Clean | Implementation State |
|---|---|---|---|---|---|
| `HeapCare.Api` | `https://github.com/HeapCare/HeapCare.Api.git` | `main` | `70e6079` | Yes | .NET 10 Web API, EF Core 10, 15 migrations |
| `HeapCare.ApiClient` | `https://github.com/HeapCare/HeapCare.ApiClient.git` | `main` | `c9328a3` | Yes | .NET 10 reusable typed HTTP client library |
| `HeapCare.Caching` | `https://github.com/HeapCare/HeapCare.Caching.git` | `main` | `4988bae` | Yes | .NET 10 HybridCache + Redis library |
| `HeapCare.Database` | `https://github.com/HeapCare/HeapCare.Database.git` | `main` | `8ec6d20` | Yes | PostgreSQL ops runbooks & backup/restore verification |
| `HeapCare.Local` | `https://github.com/HeapCare/HeapCare.Local.git` | `main` | `a2daae6` | Yes | Official 9-container local orchestrator & Keycloak setup |
| `HeapCare.Mobile` | `https://github.com/HeapCare/HeapCare.Mobile.git` | `main` | `40fb31f` | Yes | Scaffold only (`README.md`, `.gitignore`) |
| `HeapCare.Observability` | `https://github.com/HeapCare/HeapCare.Observability.git` | `main` | `1a06382` | Yes | .NET 9 OTel + Serilog ES packages (`1.0.1`) |
| `HeapCare.OtpService` | `https://github.com/HeapCare/HeapCare.OtpService.git` | `main` | `f3a8ded` | Yes | Scaffold only (`README.md`); PR #1 remains CLOSED |
| `HeapCare.Site` | `https://github.com/HeapCare/HeapCare.Site.git` | `main` | `ed3a46c` | Yes | .NET 8 Blazor Server marketing website |
| `HeapCare.Web` | `https://github.com/HeapCare/HeapCare.Web.git` | `main` | `29570fd` | Yes | React 19 / Vite TypeScript workspace (Nginx + CSP) |
| `HeapCare.Doc` | `https://github.com/HeapCare/HeapCare.Doc.git` | `main` | `71cf070` | Yes | Architecture, product, security, and QA docs (base for this PR) |

---

## 4. `HeapCare.Local` Startup Verification & Nine-Service Runtime Status

- **Official Command Executed:** `printf '\n' | bash "local/Start HeapCare.command"` (invoking `local/scripts/start-macos.sh`)
- **Exit Code:** `0` (`HeapCare is ready.`), automatically rebuilding `api` and `web`, reconciling Keycloak client `compilecare-web` scopes (`basic`, `profile`, `email`, `compilecare-api-audience`), and passing `local/scripts/verify-stack.sh`.

| Container Name | Compose Service | Runtime Status | Port / Primary Endpoint | Verified Response |
|---|---|---|---|---|
| `heapcare-postgres` | `postgres` | `Up (healthy)` | `127.0.0.1:5432` | `15` applied EF migrations (`20261125124500_AddInvoiceSettlementStatuses` latest) |
| `heapcare-keycloak` | `keycloak` | `Up (healthy)` | `http://keycloak.localhost:8082` | `HTTP 200` OIDC discovery (`compilecare` realm) |
| `heapcare-api` | `api` | `Up (healthy)` | `http://localhost:8081` | `HTTP 200` on `/`, `/health/ready`, `/openapi/v1.json`, `/metrics` prior to RL burst test (`429` verified during RL burst window) |
| `heapcare-web` | `web` | `Up (healthy)` | `http://localhost:8080` | `HTTP 200` + strict `Content-Security-Policy` (`script-src 'self'; style-src 'self'`), `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff` |
| `heapcare-otel-collector` | `otel-collector` | `Up` | `http://localhost:8889/metrics` | `HTTP 200` (`1,768` HTTP metric lines; `0` error log lines) |
| `heapcare-jaeger` | `jaeger` | `Up` | `http://localhost:16686` | `HTTP 200` (`/api/v3/services` lists `HeapCare.Api`) |
| `heapcare-prometheus` | `prometheus` | `Up` | `http://localhost:9090` | `HTTP 200` (`/-/ready`; both `heapcare-api` and `heapcare-otel-collector` targets `up`) |
| `heapcare-elasticsearch` | `elasticsearch` | `Up (healthy)` | `http://localhost:9200` | `HTTP 200` (`_cluster/health` green/yellow local single-node, `heapcare-logs-2026.11` active) |
| `heapcare-kibana` | `kibana` | `Up` | `http://localhost:5601` | `HTTP 200` (`/api/status`; data view `heapcare-logs-*` provisioned) |

---

## 5. Startup Blockers Found & Resolved

1. **Docker API Build Blocked on Legacy `CompileCare` NuGet Feed (`HC-DEF-001`):**  
   Initial execution of `local/Start HeapCare.command` failed in `dotnet restore HeapCare.Api.slnx` (`NU1101`) because `HeapCare.Api/NuGet.Config` still queried `https://nuget.pkg.github.com/CompileCare/index.json` (HTTP 404), whereas `HeapCare.Observability` and `HeapCare.Observability.AspNetCore` `1.0.1` are published under `https://nuget.pkg.github.com/HeapCare/index.json`. CI had masked the failure via local `.ci-nuget`. Resolved cleanly via **`HeapCare.Api#102`**.
2. **Legacy `CompileCare` Clones & Workflows in `HeapCare.Local` (`HC-DEF-002`):**  
   `local/scripts/start-macos.sh`, `local/scripts/start-windows.ps1`, and `.github/workflows/ci.yml` still referenced `CompileCare/*` git slugs. Resolved cleanly via **`HeapCare.Local#26`**.
3. **Kibana Data View Index Mismatch After ES Migration (`HC-DEF-008`):**  
   `HeapCare.Observability` `LoggerConfigurator.cs` already emitted ES logs to `heapcare-logs-{yyyy.MM}`, causing live `HeapCare.Api` logs to index into `heapcare-logs-2026.11` while Kibana was still provisioned with `compilecare-logs-*`. Resolved across provisioner, stack verifier, CI smoke checks, and docs via **`HeapCare.Local#27`**.

---

## 6. Cross-Repository Build, Test & GitHub Packages Verification Matrix

| Repository | Executed Commands & Scope | Observed Outcome |
|---|---|---|
| `HeapCare.Api` | `dotnet restore HeapCare.Api.slnx --force --no-cache` against `https://nuget.pkg.github.com/HeapCare/index.json`; `dotnet test tests/HeapCare.UnitTests -c Release`; `dotnet test tests/HeapCare.IntegrationTests` against ephemeral `postgres:16` | **PASS:** NuGet restore succeeded cleanly from `HeapCare` feed; **12/12** unit tests passed (39 ms); **187/187** PostgreSQL integration tests passed (1 m 5 s, 0 skipped) |
| `HeapCare.ApiClient` | `dotnet test HeapCare.ApiClient.slnx --configuration Release` | **PASS:** **30/30** tests passed (3 s) |
| `HeapCare.Caching` | `dotnet test HeapCare.Caching.sln --configuration Release` | **PASS:** **13/13** tests passed (449 ms) |
| `HeapCare.Database` | `bash -n scripts/*.sh tests/backup-restore/*.sh`; `docker compose -f docker-compose.backup-test.yml up --abort-on-container-exit --exit-code-from verify` | **PASS:** Shell syntax valid; `PostgreSQL logical backup/restore Docker verification passed.` (exit `0`, checksum & sidecar `0600` checks verified) |
| `HeapCare.Local` | `printf '\n' \| bash "local/Start HeapCare.command"`; `python3 local/scripts/provision-kibana-data-view.py`; `verify-stack.sh` | **PASS:** All 9 containers started, Keycloak configured, `heapcare-logs-*` Kibana data view idempotent, Jaeger DB span & Collector metrics verified |
| `HeapCare.Observability` | `dotnet test HeapCare.Observability.sln -c Release`; `dotnet build samples/Sample.Worker/Sample.Worker.csproj -c Release` | **PASS:** **2/2** unit tests passed (42 ms); `Sample.Worker` built with **0 warnings, 0 errors** after PR `#6` |
| `HeapCare.Site` | `dotnet build src/HeapCare.Site/HeapCare.Site.csproj -c Release`; `dotnet publish -c Release`; live `http://127.0.0.1:5099` HTTP smoke (`/`, `/privacy`, `/contact`, `/images/hospital-dashboard.svg`) | **PASS:** **0 warnings, 0 errors**; Release publish succeeded; all 4 HTTP probes returned `200`; `HeapCare` brand rendered |
| `HeapCare.Web` | `npm run build` (`tsc -b && vite build`); `npm test` (Vitest); `npm run test:e2e` (Playwright 5 workers); `npm audit --omit=dev` | **PASS:** Built in `1.88 s`; **131/131** Vitest specs across 19 files; **20/20** Playwright specs (`25.9 s`); **0 production audit vulnerabilities** |
| `HeapCare.Mobile` | Repository & `README.md` inspection | **SCAFFOLD:** Confirmed intentional minimal scaffold (`README.md`, `.gitignore` only; no build targets yet) |
| `HeapCare.OtpService` | Repository & `gh pr list --state all` gate check | **SCAFFOLD / GATED:** Confirmed `README.md` skeleton only on `main`; PR `#1` remains `CLOSED` pending security review |
| `HeapCare.Doc` | Documentation accuracy review + this QA report PR | **IN REVIEW / UPDATED:** Cross-links & continuation status updated cleanly |

---

## 7. Live End-to-End Functional Workflows Verified (27 / 27 PASS)

Executed against the rebuilt live local Compose stack with real Keycloak OIDC Authorization Code + PKCE sign-in and two freshly created synthetic tenants (**Alpha** `11ffbfef-a0f9-4600-b3aa-7088464acb57` and **Beta** `a7c5e57d-aa63-4082-a4a7-ed695ea5dbdd`):

| Test ID | Workflow Area | Verified Live Outcome |
|---|---|---|
| `E2E-AUTH-01` / `02` | Unauthenticated shell & route guard | Sign-in gate enforced; direct `/billing` blocked before login |
| `E2E-AUTH-INVALID` / `03` | Keycloak PKCE login | Invalid password rejected (`Invalid username or password.`); valid login renders `Hospital workspace` |
| `E2E-TEN-01` / `E2E-TEN-ISO-UI` | Multi-tenant lifecycle & UI isolation | Created & activated Alpha + Beta tenants; switching to Beta completely hides Alpha Hospital (`alphaVisibleInBeta=false`) |
| `E2E-HOSP-01` | Hospital onboarding | Created & selected synthetic hospital `HeapCare Alpha General` |
| `E2E-DEPT-01` / `02` | Department & 409 conflict UX | Created `General Medicine (GEN)`; duplicate returned `HTTP 409` with operation-neutral guidance |
| `E2E-FAC-01` | Facility hierarchy | Ward → Room → Bed hierarchy created and visible |
| `E2E-MRN-01` | MRN policy | Configured immutable `HCP` prefix MRN policy cleanly |
| `E2E-STAFF-01` / `E2E-MEM-01` | Staff & memberships | Created `Dr. Priya Menon` assignment; provisioned `ClinicalUser` & `FacilityViewer` memberships |
| `E2E-PAT-01` / `02` | Patient registration & MRN search | Duplicate check + registration assigned `HCP000001`; Reception MRN search selected `HCP000001` |
| `E2E-APPT-01` / `02` / `CHECKIN` | Appointments & check-in | Provider appeared immediately without reload; slot booked & checked in to queue token `#01` |
| `E2E-QUEUE-01` | Walk-in & queue transitions | Walk-in issued `#02`; `Waiting` → `Called` → `InProgress` transitions succeeded |
| `E2E-DASH-01` | Overview metrics | Revisiting Overview auto-refreshed metrics (`Appointments=1`, `Queue=1`/`2` after walk-in) |
| `E2E-RESP-1440x900` / `390x844` | Responsive layout | `0px` horizontal overflow on desktop and mobile (`390×844`) |
| `E2E-BILL-01` | OPD billing | Service catalogue item, invoice, UPI payment, receipt modal, and daily collection report passed |
| `E2E-AUDIT-01` | Tenant audit query | `GET /api/v1/audit/events` returned `HTTP 200`, `20` persisted audit items |
| `E2E-CLIN-01` | Clinical OPD workflow | `ClinicalUser` opened encounter, confirmed identity, documented, signed (read-only + amendment panel) |
| `E2E-RBAC-01` | Role UI restriction | `FacilityViewer` showed `Facilities` only (`receptionCount=0`) and blocked `/reception` |
| `E2E-PERSIST-01` | Restart persistence | After `docker compose restart api web`, `GET .../by-mrn/HCP000001` returned `HTTP 200`, `givenName=Siddharth` |

Console & Network Logs: **0 JavaScript runtime/CSP errors** (`5` expected HTTP `404`/`409` console resource logs matching intentional unconfigured MRN policy / duplicate / pre-invoice probes).

---

## 8. Security, Tenant Isolation & `OtpService` Gate Verification (24 / 24 PASS)

- **Authentication (`SEC-AUTH-01..04`):** Anonymous (`401`), tampered JWT signature (`401`), wrong issuer/audience (`401`), expired/forged claims (`401`).
- **Tenant Isolation (`SEC-TEN-01..07`):** Cross-tenant hospital/staff/membership reads & mutations (`403`), cross-tenant patient/department anti-enumeration reads (`404`).
- **RBAC & Scopes (`SEC-RBAC-01..08`, `SEC-SCOPE-01..02`, `SEC-MEM-01`):** `FacilityViewer` read-only enforcement (`200` facilities/depts, `403` patient/staff writes); department-scoped `PatientRegistrar` confined to own department (`200` D1, `404` D2); `HospitalAdmin` blocked from `PlatformOperator` (`403`); `PatientRegistrar` blocked from audit export (`403`); deactivated membership rejected immediately despite still-valid JWT (`403`).
- **Binding, Rate Limiting & Headers (`SEC-ERR-01`, `SEC-RL-01`):** Malformed query returned `400 ProblemDetails`; anonymous burst triggered `HTTP 429` with `Retry-After: 60`; Web serves `default-src 'self'`, `script-src 'self'`, `style-src 'self'`, `object-src 'none'`, `frame-ancestors 'none'`.
- **`HeapCare.OtpService` Gate:** Confirmed `HeapCare/HeapCare.OtpService#1` (`feat(otp): add OTP service (security review pending)`) stays **`CLOSED`** and `main` contains only `README.md`.

---

## 9. Observability & Telemetry Verification

- **Prometheus (`:9090`):** `upJobs = ["heapcare-api", "heapcare-otel-collector"]`; `174` HTTP metric series observed.
- **OTel Collector (`:8889/metrics`):** `HTTP 200`; `1,768` exported HTTP metric lines; `0` error/drop lines in container logs.
- **Jaeger (`:16686`):** Service `HeapCare.Api` active with `50` traces, `127` spans, and PostgreSQL (`postgresql`) DB child spans confirmed.
- **Elasticsearch (`:9200`):** Live `HeapCare.Api` logs actively written to `heapcare-logs-2026.11`. Correlation-ID lookup for allowed authenticated traffic returned `10` hits; sampled credential/JWT secret-leak query (`eyJhbGci OR password1 OR Shatru11`) returned `0` hits.
- **Kibana (`:5601`):** Data view `heapcare-logs-*` (`id=82343359-aba8-484c-a2ea-706df49f958c`) provisioned cleanly alongside legacy `compilecare-logs-*`.

---

## 10. Branding & Migration Reference Classification

| Category | Examples Inspected | Disposition |
|---|---|---|
| **Fixed Genuine Migration Defects** | `Api/NuGet.Config` feed URL; `Local` script slugs + Kibana `heapcare-logs-*`; `Observability/Sample.Worker` project refs + `RepositoryUrl`; `Web` UI strings (`HEAPCARE PLATFORM`, `HeapCare.Local`, `HeapCare.Api`) & walk-in token state; `Site` footer GitHub links & SVG title; `ApiClient`/`Caching` `RepositoryUrl` | **Fixed & merged via PRs** (Section 11 & 15) |
| **Intentional Legacy Compatibility Contracts** | Compose project `compilecare-local`; PostgreSQL DB/user `compilecare`; Keycloak realm `compilecare` & client `compilecare-web` / audience `compilecare-api`; Docker named volumes `compilecare-*`; `ConnectionStrings:CompileCare` fallback | **Preserved intentionally** to avoid breaking active DBs, OIDC subject mappings, or local volumes |
| **Intentional Pre-Cutover Domain Reference** | `HeapCare.Site/Pages/Contact.razor` (`mailto:support@compilecare.com`) and Web mock tests referencing `https://compilecare.in/problems/...` | **Retained** until `heapcare.in` DNS, TLS, and mailbox infrastructure are provisioned |

---

## 11. Defect Register & Fixes Merged During Validation

| ID | Severity | Repository & Location | Root Cause | Resolution PR | Retest Status |
|---|---|---|---|---|---|
| `HC-DEF-001` | **Blocker** | `HeapCare.Api/NuGet.Config`, `.github/workflows/migration-rehearsal.yml` | NuGet pointed to legacy 404 `CompileCare` GitHub package feed; migration rehearsal script used old slnx/DbContext names | `HeapCare.Api#102` | **Verified Fixed** (local Docker build + `--no-cache` restore + CI passed) |
| `HC-DEF-002` | **High** | `HeapCare.Local/local/scripts/start-{macos,windows}`, `ci.yml` | Launcher & CI scripts cloned/referenced legacy `CompileCare/*` repo slugs | `HeapCare.Local#26` | **Verified Fixed** |
| `HC-DEF-003` | **Medium** | `HeapCare.Observability/samples/Sample.Worker/Sample.Worker.csproj`, `.csproj` URLs | Sample Worker `<ProjectReference>` pointed to non-existent `CompileCare.Observability*`; `RepositoryUrl` outdated | `HeapCare.Observability#6` | **Verified Fixed** (`0 Warnings, 0 Errors`) |
| `HC-DEF-004` | **Low** | `HeapCare.ApiClient` & `HeapCare.Caching` `.csproj` | Package `RepositoryUrl` still pointed to `CompileCare/*` | `ApiClient#5`, `Caching#7` | **Verified Fixed** |
| `HC-DEF-005` | **Low** | `HeapCare.Site` `MainLayout.razor`, `hospital-dashboard.svg` | Footer GitHub links & hero SVG header still referenced `CompileCare` | `HeapCare.Site#13` | **Verified Fixed** |
| `HC-DEF-006` | **Low** | `HeapCare.Web` `App.tsx`, `PlatformTenantManagement.tsx`, `TenantMembershipManagement.tsx` | Residual UI copy (`COMPILECARE PLATFORM`, `CompileCare.Api`, `CompileCare.Local`) | `HeapCare.Web#70` | **Verified Fixed** (`03-tenants-created.png` confirmed `HEAPCARE PLATFORM`) |
| `HC-DEF-007` | **Medium** | `HeapCare.Web/src/app/App.tsx` (`handleWalkIn`) & `QueueWorkspace.tsx` | Creating walk-in after booking an appointment did not clear `bookedAppointment`, hiding `WALK-IN QUEUE TOKEN` | `HeapCare.Web#70` | **Verified Fixed** (Vitest regression + `09-queue-workspace.png`) |
| `HC-DEF-008` | **Medium** | `HeapCare.Local` `provision-kibana-data-view.py`, `verify-stack.sh`, `ci.yml` | Kibana provisioned `compilecare-logs-*` while `HeapCare.Observability` writes to `heapcare-logs-*` | `HeapCare.Local#27` | **Verified Fixed** (`corrAllowHits: 10` in `heapcare-logs-*`) |

---

## 12. Deferred / Blocked / Not Implemented Features

These areas remain **intentionally unimplemented or deferred** and are **not counted as passed**:

- **`HeapCare.Mobile`:** Scaffold README only; no Android/iOS/MAUI runtime exists yet.
- **`HeapCare.OtpService` & Verified Patient Contacts (ADR-012):** Skeleton README only (`PR #1` intentionally `CLOSED`); patient phone/email capture and OTP delivery remain disabled.
- **Customer-Facing Audit Log Viewer UI (ADR-016):** Backend `/api/v1/audit/events` query/CSV export work cleanly, but customer-facing Web screen remains withheld pending governance/compliance sign-off.
- **Billing Extensions:** GST/tax, discounts/waivers, refunds/credit notes, TPA/insurance, payment gateways, and ERP export are outside baseline OPD billing.
- **Inpatient, Bed Allocation, Pharmacy, Diagnostics & Inventory:** Beyond `Ward → Room → Bed` facility configuration, inpatient ADT, LIS/RIS, and pharmacy dispensing are not yet implemented.

---

## 13. Visual Evidence Catalog

Fresh post-fix headless Chromium screenshots (`15` PNG files) and machine-readable logs (`e2e-results.json`, `security-results.json`, `observability-results.json`, `console-errors.log`, `network-errors.log`) were captured outside git in `artifacts/heapcare-post-migration-qa-2026-11-27/`:

- `01-unauthenticated.png` — Unauthenticated sign-in gate (`HeapCare Workspace`)
- `02-authenticated-workspace.png` — Authenticated initial workspace shell
- `03-tenants-created.png` — Visually verified `HEAPCARE PLATFORM` badge & `52 tenants across the HeapCare platform`
- `04-facilities-hierarchy.png` — Hospital → Department → Ward → Room → Bed tree
- `05-staff-management.png` — Provider profile and department assignment (`Dr. Priya Menon`)
- `06-patient-registered.png` — Patient registration with sequential MRN `HCP000001`
- `07-reception-search.png` — Reception MRN lookup & patient context binding
- `08-appointments-workspace.png` — Appointments slot availability, booking & check-in banner
- `09-queue-workspace.png` — Visually verified `WALK-IN QUEUE TOKEN 02` + live department queue (`#01 InProgress`, `#02 Waiting`)
- `10-overview-dashboard.png` — Visually verified auto-refreshed metrics (`APPOINTMENTS 1`, `QUEUE ENTRIES 2`, `COLLECTED TODAY ₹0`)
- `11-responsive-1440x900.png` & `11-responsive-390x844.png` — `0px` horizontal overflow desktop/mobile
- `12-billing-settlement.png` — Paid invoice, UPI receipt, and daily collection summary
- `13-clinical-signed.png` — Signed read-only OPD encounter with amendment controls
- `14-facilityviewer-rbac.png` — Least-privilege `FacilityViewer` navigation boundary

---

## 14. Operational & Production Readiness Gaps

Before approving any live hospital pilot or production launch:
1. Acquire/configure the target domain (e.g., `heapcare.in`), TLS termination, and explicit production CORS/CSP origins (`connect-src` / `frame-src`).
2. Establish managed production Keycloak realms with signing-key rotation and non-resetting identity subject mapping.
3. Configure managed PostgreSQL automated physical backups/PITR, encrypted off-site retention, and provider-native restore drills.
4. Complete independent penetration testing, DPDPA / Indian healthcare privacy/legal sign-off, and pilot-hospital RBAC matrix approval.

---

## 15. Pull Requests Created & Merged During This Validation

| Repository | PR | Title | CI Result Before Merge |
|---|---|---|---|
| `HeapCare/HeapCare.Api` | [#102](https://github.com/HeapCare/HeapCare.Api/pull/102) | Fix NuGet feed URL and migration rehearsal paths for HeapCare | 4/4 checks passed (`CI` PR + push, `Docker Integration Verification`, `PostgreSQL Migration Rehearsal`) |
| `HeapCare/HeapCare.Local` | [#26](https://github.com/HeapCare/HeapCare.Local/pull/26) | Update clone and CI repository slugs to HeapCare | 3/3 active checks passed (+ 1 skipped push check) |
| `HeapCare/HeapCare.Observability` | [#6](https://github.com/HeapCare/HeapCare.Observability/pull/6) | Fix Sample.Worker project references and RepositoryUrl metadata | 3/3 active checks passed (`build`, `test`, `code-quality`; publish skipped) |
| `HeapCare/HeapCare.Site` | [#13](https://github.com/HeapCare/HeapCare.Site/pull/13) | Update HeapCare footer GitHub URLs and dashboard illustration title | 1/1 passed (`HeapCare Site CI/build`) |
| `HeapCare/HeapCare.ApiClient` | [#5](https://github.com/HeapCare/HeapCare.ApiClient/pull/5) | Update HeapCare.ApiClient RepositoryUrl to HeapCare org | 1/1 active passed (`build-and-test`; publish skipped) |
| `HeapCare/HeapCare.Caching` | [#7](https://github.com/HeapCare/HeapCare.Caching/pull/7) | Update HeapCare.Caching RepositoryUrl to HeapCare org | 1/1 active passed (`build-test-pack`; publish skipped) |
| `HeapCare/HeapCare.Web` | [#70](https://github.com/HeapCare/HeapCare.Web/pull/70) | Update remaining UI branding to HeapCare and clear prior booking state on walk-in | 2/2 passed (`Web CI/build-test` push + PR) |
| `HeapCare/HeapCare.Local` | [#27](https://github.com/HeapCare/HeapCare.Local/pull/27) | Provision heapcare-logs-* Kibana data view matching API ES index format | 3/3 active passed (incl. `Full nine-service Docker stack and API telemetry verification`) |

---

## 16. Exact Verification Commands & Reproducibility Log

All secrets (`local/.env`, `local/.secrets/nuget-source-credentials`, JWTs) were kept out of command logs and reports. Key execution commands:

```bash
# 1. Clean-main full nine-service startup & infrastructure smoke verification
cd HeapCare.Local && printf '\n' | bash "local/Start HeapCare.command"

# 2. Private GitHub Packages feed verification
dotnet restore HeapCare.Api/HeapCare.Api.slnx --force --no-cache

# 3. Repository suites (unit, isolated Postgres integration, backup/restore Docker verify, Web Vitest & Playwright)
dotnet test HeapCare.Api/tests/HeapCare.UnitTests/HeapCare.UnitTests.csproj -c Release
ConnectionStrings__HeapCareTests="Host=127.0.0.1;Port=15432;Database=heapcare_test;Username=postgres;Password=postgres" \
  dotnet test HeapCare.Api/tests/HeapCare.IntegrationTests/HeapCare.IntegrationTests.csproj --no-build
docker compose -f HeapCare.Database/docker-compose.backup-test.yml up --abort-on-container-exit --exit-code-from verify
cd HeapCare.Web && npm run build && npm test && npm run test:e2e

# 4. Live E2E + restart persistence, 24-check security matrix & telemetry verification
node artifacts/.../scratch/e2e_heapcare_post_migration.mjs
python3 artifacts/.../scratch/phase5_8_9_heapcare.py --rate-limit
```
