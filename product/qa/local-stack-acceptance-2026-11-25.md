# CompileCare — Local full-stack acceptance record

**Execution window:** 2026-11-25 23:21–23:36 IST  
**Assessment:** Ready for further QA; not production or pilot sign-off  
**Environment:** Docker Compose — CompileCare.Web, CompileCare.Api, Keycloak 26.2.5 and PostgreSQL 17  
**Tested commits:** Web `5a90b0a`; API `5a6d882`; Local `c6890fa`  
**Browser:** Chromium 159.0.8154.6; Playwright 1.65.1

## 1. Real-stack result

| Result | Count |
|---|---:|
| Executed | 50 |
| Passed | 48 |
| Failed | 2 |
| Blocked | 0 |
| Executed pass rate | 96.0% |
| Unimplemented / Not Run | 8 |

All four local containers were reported healthy. Web returned HTTP 200; API live/readiness checks returned Healthy, including PostgreSQL; Keycloak OIDC discovery and API OpenAPI were reachable. Core OPD journeys exercised patient registration and MRN assignment, appointment booking/check-in, walk-in and FIFO queue transitions, department-scoped clinical encounters, and OPD billing/settlement/receipts.

The 96% figure is 48/50 executed tests. The eight Not Run cases are not included in the denominator and must not be counted as failures or delivered functionality.

## 2. Defects requiring correction

### AUTH-005 — silent OIDC renewal redirect is not registered

- **Observed:** Keycloak rejected `http://localhost:8080/auth/silent-callback` with `Invalid parameter: redirect_uri`.
- **Cause:** The Web OIDC client configures `silent_redirect_uri` at that path, but the local `compilecare-web` Keycloak client only registered `/auth/callback`.
- **Impact:** Silent renewal fails when the 300-second access token approaches expiry; the user may see API 401/session-not-authorized errors until signing in again.
- **Correction:** Local PR #7 is merged. It adds the missing redirect URI to the checked-in realm and extends the idempotent Keycloak reconciliation script to repair an existing client without deleting its realm volume/users.
- **Hosted verification:** Local CI run 45738522025 passed realm validation, the Docker Keycloak import smoke test and Compose validation. The test deliberately removes the silent callback from an existing client, reruns the reconciliation script and verifies that the callback is restored.
- **Remaining local acceptance:** On the developer Mac, pull the merge, reconcile the existing realm, sign in, allow token renewal to occur and confirm the renewed token is accepted by the API. The hosted smoke test does not replace this full browser/API session-renewal check.

### SEC-002 — foreign/out-of-scope hospital collection returns 200 instead of 404

- **Observed:** Patient search for a hospital ID belonging to another tenant returned HTTP 200 with an empty page. The report also observed the same response-contract issue on department collection listing for an out-of-scope hospital.
- **Data exposure:** The report found zero patient/department rows returned; tenant query filters and department scoping prevented data leakage in the tested case.
- **Expected contract:** Return 404 when the hospital is not visible in the caller's tenant and organization scope, consistent with single-resource anti-enumeration behavior.
- **Correction:** API PR #55 is merged as `07fb9cea2dc6f706fc5c1a6fff67850f585b79d9`; it checks hospital visibility before patient search and returns a typed not-found result for inaccessible department collections. PostgreSQL cases cover tenant-wide foreign-tenant patient search and same-tenant out-of-scope/foreign-tenant department listing.
- **Hosted verification:** CI run 45739052864 passed restore, Release build, EF model-drift validation and all unit/integration tests; Docker Integration Verification run 45739052930 passed.
- **Remaining local acceptance:** Repeat both real-stack cases on the updated local stack and confirm authorized hospital requests still return 200.

## 3. Mocked Web regression results — separate from real-stack tests

| Suite | Result |
|---|---:|
| Vitest unit/component | 35/35 passed |
| Playwright mocked E2E | 11/14 passed on macOS arm64 |
| Playwright mocked E2E failures | 3 |

The three local macOS failures were caused by native `<select>` keyboard behavior and use of `Control+A` rather than macOS `Meta+A`. They were not failures in the live API/Keycloak/PostgreSQL acceptance run.

Web PR #27 is merged. It makes native-select changes deterministic with Playwright's selection API while retaining keyboard focus-order assertions, and uses the platform-appropriate Select All shortcut. Hosted Linux browser E2E passed; no separate macOS hosted-run result is claimed.

## 4. Responsive and accessibility observations

The real-stack run reported no horizontal overflow at desktop 1440×900, tablet 768×1024 and mobile 375×812. The representative keyboard-focus check reported a visible 3px solid focus indicator, navigation and main landmarks, and zero unlabeled inputs among 22 inputs in the inspected view.

These are scoped observations from the executed cases, not a full WCAG 2.2 AA conformance assessment.

## 5. Eight Not Run scenarios

The report marks the following scenarios Not Run because the relevant UI capability is not implemented in the tested build or the capability is deferred by the MVP blueprint:

1. Ongoing tenant lifecycle and membership administration screens — Not Run in the original build; tenant membership UI is now merged in Web PR #35 and the global platform tenant lifecycle UI in Web PR #36. Authenticated local-stack acceptance remains pending for both.
2. Facility hierarchy CRUD screens for hospitals, departments, wards, rooms and beds — Not Run in the original Antigravity-tested build; now implemented and merged in Web PR #34. Authenticated local-stack acceptance remains pending.
3. Staff directory and provider assignment screens.
4. MRN policy configuration and search-by-MRN screens.
5. Publishing appointment availability and appointment list/reschedule/cancel screens.
6. Inpatient ADT, bed occupancy, nursing MAR and discharge workflows.
7. Laboratory/radiology/PACS and pharmacy dispensing.
8. Notifications, patient portal, workforce HR/payroll and ABDM/TPA integrations.

These eight scenarios were Not Run in the original tested build and must not be reclassified as failures. Three are now implemented and merged: facility hierarchy CRUD (Web PR #34), tenant membership administration (Web PR #35) and platform tenant lifecycle administration (Web PR #36). All three still require authenticated local-stack acceptance. The remaining six are staff/provider UI; MRN policy/search UI; appointment availability/list/reschedule/cancel UI; inpatient ADT/bed occupancy/MAR/discharge; LIS/RIS/PACS/pharmacy; and notifications/patient portal/HR/payroll/ABDM/TPA. Deliver deferred capabilities in dependency order with approved product, security and operational requirements.

## 6. Hosted CI status and merge rule

The earlier runner-allocation symptom (`runner_id=0`, empty runner name, zero steps) is no longer blocking the active public-repository workflows. After the relevant repositories were made public, GitHub Actions assigned hosted runners and executed real .NET, browser and Docker steps.

Verified current-head runs include:
- API PR #54 membership audit: PostgreSQL integration and Docker verification passed before merge.
- API PR #55 SEC-002 correction: CI 45739052864 and Docker Integration Verification 45739052930 passed before merge.
- Database PR #2 backup/restore: PostgreSQL Docker verification 45738452765 passed before merge.
- Caching PR #5 tenant isolation: workflow passed before merge.
- Local PR #5 and #7: Local validation 45658651667 and 45738522025 passed before merge.
- Local PR #6 observability overlay: Docker runtime smoke runs 45740981547 and 45740976578 passed before merge; the test verifies synthetic OTLP metrics in Prometheus, traces through Jaeger v3, logs in Elasticsearch and Kibana readiness.
- Web PR #27 and #28: browser/Docker CI 45734073286 and 45670894903 passed before merge.
- Web PR #29–#33: CI/Docker runs 45738737945, 45738814930, 45738824899, 45738835427 and 45739127780 passed on the respective PR heads; all five PRs were merged in sequence.
- Web PR #34 facility hierarchy: final CI/Docker runs 45744960504 and 45744951178 passed on head `b36c7115003384eee8f809511a0b6808b36e76f2`; 67 Vitest tests and all 14 Chromium E2E tests passed, followed by successful production build and Docker smoke. PR #34 merged as `c8a5a5ec9a31e85cab05d60d05e8852616319be3`.
- Web PR #35 tenant membership administration: runs 45745881098 and 45746024118 passed 75 Vitest tests, 14 Chromium E2E tests, production build and Docker smoke; merged as `b8e4a4ec8864884f5e71dc070a319a5f4df07185`.
- Web PR #36 platform tenant lifecycle workspace: runs 45747601458 and 45747606834 passed 85 Vitest tests, 14 Chromium E2E tests, production build and Docker smoke; merged as `54589758db04848d25e2e078f3619c1932cc645a`.
- API PR #57 platform-operator self-status: runs 45746214399 and 45746214402 passed Release build, EF model-drift, 12 unit tests, 97 PostgreSQL integration tests and Docker verification; merged as `916b75a9a8a81e37216d48562b65c23d07682fc1`.
- API PR #58 initial-admin activation guard: runs 45747380466 and 45747380428 passed Release build, EF model-drift, 12 unit tests, 97 PostgreSQL integration tests and Docker verification; merged as `2d74b412f6b26274381ebd0d0ed461eb1b75ead8`.

Do not merge an implementation PR based on a green-looking local report alone or on a hosted job that never received a runner. Record the tested PR head SHA and the exact workflow run. No implementation merge gates remain in the current PR queue. Authenticated screenshots for the integrated UI are still required as product acceptance evidence. Local PR #6 is merged after Docker runtime pipeline smoke; a real CompileCare.Api-to-observability integration check on the developer Mac remains post-merge local acceptance. All 11 CompileCare repositories are now public and [CompileCare.Api issue #56](https://github.com/CompileCare/CompileCare.Api/issues/56) is closed as resolved.
 
## 7. PR grouping and merge order

- **Web experience and administration stack:** PRs #29 → #30 → #31 → #32 → #33 → #34 → #35 → #36 are merged into `main`. Merge commits: #29 `37f30508a23add4a8af0918307a524dd9f9f4bdb`; #30 `f0e548737ac05a1886640be742ed64a32f892e38`; #31 `0c10cafacb43183b47bc2f18ea0deadf8a90a95e`; #32 `68a8c9798a29df34031e2e13d7857550f887546b`; #33 `23cf6dcf503fc4e781e9b343d244ea9665566e29`; #34 `c8a5a5ec9a31e85cab05d60d05e8852616319be3`; #35 `b8e4a4ec8864884f5e71dc070a319a5f4df07185`; #36 `54589758db04848d25e2e078f3619c1932cc645a`. The screenshot evidence remains post-merge product acceptance.
- **Web test work:** PR #27 (cross-platform OPD browser interactions) and PR #28 (local Playwright HTML report) are merged.
- **API security and platform lifecycle:** PR #54 membership audit coverage, PR #55 inaccessible-hospital collection contract fix, PR #57 platform-operator self-status and PR #58 initial-admin activation guard are merged after PostgreSQL/Docker verification.
- **API scheduling and audit:** PR #60 PostgreSQL deadlock-to-409 handling and PR #61 cross-hospital OPD denial/audit coverage are merged after Release build, EF model-drift, 12 unit tests, 97 PostgreSQL integration tests and Docker verification.
- **Local environment:** PR #5 GSSAPI warning, PR #7 Keycloak silent-renewal reconciliation and PR #6 optional observability overlay are merged. PR #6 passed a Docker runtime smoke of the synthetic metrics, traces and logs pipeline; a real API instrumentation smoke remains local post-merge acceptance.
- **Database and caching:** Database PR #2 backup/restore helpers and Caching PR #5 tenant isolation tests are merged after hosted Docker/test verification.
- **Documentation:** PR #50 is merged, paired with Local PR #5. Keep future docs changes paired with implementation status.

No open implementation PRs remain across the 11 CompileCare repositories. The Web experience/facility stack is merged; API, Database, Caching, Local and paired documentation workstreams are also merged. This update records the final acceptance state.
 
## 8. Evidence and source artifacts

This record is based on the Antigravity local full-stack acceptance report, its `acceptance-results.json` and `execution-log.txt`, plus the separate mocked Playwright HTML report. The tested build SHAs are recorded above. Screenshots are referenced by test ID in the original local report; this document does not claim that those screenshots have been copied into the documentation repository.


## 9. Merge completion update — 2026-11-26

The owner has made all 11 CompileCare repositories public. Hosted Actions runner allocation is functioning, and API issue #56 was closed as resolved.

Web PRs #29–#36 have all been merged into main. Facility hierarchy PR #34 passed 67 Vitest tests, tenant membership PR #35 passed 75 Vitest tests, and platform tenant lifecycle PR #36 passed 85 Vitest tests; all three passed all 14 Chromium E2E tests, production build and Docker smoke. PR #36 merged as `54589758db04848d25e2e078f3619c1932cc645a` (runs 45747601458 and 45747606834).

The Facility hierarchy, tenant membership administration and platform tenant lifecycle scenarios were Not Run in the original Antigravity-tested build. They are now implemented and merged; do not count them as locally accepted until the authenticated local Docker stack is exercised with synthetic hospitals, departments, wards, rooms, beds, memberships and platform tenants. Verify read-only/write membership boundaries, platform-operator visibility and the initial-admin activation prerequisite.

The screenshot-evidence checklist is now a post-merge product acceptance item. Capture the integrated Web main build at desktop, tablet and mobile widths, with a keyboard-focus state, Facilities views, Memberships for a synthetic HospitalAdmin and Platform tenants for a synthetic persisted platform operator. Do not use real patient data or real identity subjects.


## 10. Scheduling concurrency and cross-hospital audit — 2026-11-26

- API PR #60 merged as `b31832ef8a416373532bcfe46b9e171dcc9890e7`. PostgreSQL exclusion/unique conflicts and deadlocks (23P01, 23505, 40P01) now return HTTP 409 Conflict rather than an unhandled 500. Hosted runs 45748869864 and 45748869951 passed 12 unit tests, 97 PostgreSQL integration tests and Docker verification.
- API PR #61 merged as `ac8ef92aa8063321bc07f838c4e512945f8f2182`. Five cross-hospital OPD operations (encounter read, queue-linked encounter read, note update, signing and encounter creation) return 404, create rejected audit records with requested tenant and validated actor issuer/subject, and leave the owning hospital's clinical note unchanged. Hosted runs 45749260863 and 45749260926 passed 12 unit tests, 97 PostgreSQL integration tests and Docker verification.
- No open implementation PRs remain across the 11 public CompileCare repositories. The remaining acceptance work is authenticated local Keycloak/API/PostgreSQL verification with synthetic data and screenshot evidence; hosted CI does not replace that local end-to-end acceptance.

## 11. Audit transaction and endpoint coverage — 2026-11-26
- API PR #69 merged as `bb97d951c8d8eb042e7d381501a9f3f29862fb4e`. The authenticated-route audit guard now also asserts that at least one route is discovered, preventing a false-positive empty-source test. Hosted Release CI run 45751984878 and Docker Integration Verification run 45751984843 passed.


- API PR #66 merged as `a88a105624dd717c49aca83a7e6fb69e85dd75d0`. A Department B PatientRegistrar is denied ten availability, appointment and queue operations targeting Department A in the same hospital; each denial returns 404 and is audited with validated actor/tenant and exact resource attribution. Hosted CI runs 45751436654 and 45751402961 passed Release build, EF model-drift, 12 unit tests and 99 PostgreSQL integration tests; Docker run 45751436573 passed.
- API PR #67 merged as `29ccf37cc75273ea98d99e39ccf456a429434210`. A PostgreSQL trigger injects an audit-event persistence failure during patient registration. The regression verifies HTTP 503 and confirms both patient and hospital-registration writes are rolled back. Hosted CI runs 45751563380 and 45751571863 passed; Docker run 45751571879 passed.
- API PR #68 merged as `4f110e56970cbc3dfdca27416fef25fdb9e9f908`. An endpoint-registration integration guard enumerates authenticated routes, including inherited route-group metadata, and verifies each carries tenant, user or platform-operator audit metadata. Hosted CI runs 45751665129 and 45751674630 passed; Docker run 45751674546 passed.

These hosted checks strengthen the security regression suite but do not replace the outstanding developer-Mac acceptance: rerun AUTH-005 silent OIDC renewal and SEC-002 inaccessible-hospital responses on the latest local Docker stack; exercise Facilities, Memberships and Platform tenant administration with synthetic records; verify API-to-observability telemetry; and capture authenticated desktop/tablet/mobile screenshots plus keyboard/accessibility evidence. Do not use real patient data or real identity subjects. The original 48/50 result remains the historical 2026-11-25 baseline, not a claim that the latest build has been locally re-accepted.


## 12. Reception MRN lookup — 2026-11-26

- Web PR #37 merged as `150a61a20e824f3e82f4450bc76e536a390417fd`. Reception supports exact MRN lookup within the selected hospital, in addition to patient-name search. The MRN selector is permission-gated by `patient.identifier.read`; tenant/hospital changes and permission loss clear the MRN search state. Queries/results are not written to browser storage.
- Hosted `build-test` runs 45752346956 and 45752342891 passed on PR head `a9a44eaf0d1f38d6336fc23dc2478bf050b77acd`. Both completed successfully, covering the configured Vitest, Chromium E2E, production build and Docker smoke workflow.
- **Local acceptance still required:** exercise MRN search using synthetic patient records under an authorized identifier-reader membership; verify the control is absent for a membership without `patient.identifier.read`; verify hospital/tenant switching and permission revocation clear search state; verify not-found and API failure behavior. Do not use real patient identifiers. The historical 48/50 run is unchanged and does not represent acceptance of this merged feature.
