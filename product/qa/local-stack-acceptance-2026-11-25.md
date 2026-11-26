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
- **Correction:** API PR #55 is merged as `07fb9cea2dc6f706fc5c1a6fff67850f585b79d9`? No — see merge SHA below; it checks hospital visibility before patient search and returns a typed not-found result for inaccessible department collections. PostgreSQL cases cover tenant-wide foreign-tenant patient search and same-tenant out-of-scope/foreign-tenant department listing.
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

1. Ongoing tenant lifecycle and membership administration screens.
2. Facility hierarchy CRUD screens for hospitals, departments, wards, rooms and beds.
3. Staff directory and provider assignment screens.
4. MRN policy configuration and search-by-MRN screens.
5. Publishing appointment availability and appointment list/reschedule/cancel screens.
6. Inpatient ADT, bed occupancy, nursing MAR and discharge workflows.
7. Laboratory/radiology/PACS and pharmacy dispensing.
8. Notifications, patient portal, workforce HR/payroll and ABDM/TPA integrations.

These items remain roadmap scope decisions. Do not mark them as failed tests, and do not create implementation PRs solely to make the Not Run count disappear. Deliver them in dependency order with approved product, security and operational requirements.

## 6. Hosted CI status and merge rule

The earlier runner-allocation symptom (`runner_id=0`, empty runner name, zero steps) is no longer blocking the active public-repository workflows. After the relevant repositories were made public, GitHub Actions assigned hosted runners and executed real .NET, browser and Docker steps.

Verified current-head runs include:
- API PR #54 membership audit: PostgreSQL integration and Docker verification passed before merge.
- API PR #55 SEC-002 correction: CI 45739052864 and Docker Integration Verification 45739052930 passed before merge.
- Database PR #2 backup/restore: PostgreSQL Docker verification 45738452765 passed before merge.
- Caching PR #5 tenant isolation: workflow passed before merge.
- Local PR #5 and #7: Local validation 45658651667 and 45738522025 passed before merge.
- Web PR #27 and #28: browser/Docker CI 45734073286 and 45670894903 passed before merge.
- Web PR #29–#33: CI/Docker runs 45738737945, 45738814930, 45738824899, 45738835427 and 45739127780 passed on the respective PR heads.

Do not merge an implementation PR based on a green-looking local report alone or on a hosted job that never received a runner. Record the tested PR head SHA and the exact workflow run. The current remaining merge gates are authenticated screenshot evidence for Web PR #29 and full developer-Mac observability-overlay startup/telemetry smoke for Local PR #6. The runner/billing distinction is tracked in [CompileCare.Api issue #56](https://github.com/CompileCare/CompileCare.Api/issues/56); `CompileCare.Site`, `CompileCare.Mobile` and `CompileCare.OtpService` still appear private in the organization inventory and remain subject to the Free organization's included Actions allowance.
 
## 7. PR grouping and merge order

- **Web design stack:** PR #29 → #30 → #31 → #32 → #33. These remain intentionally stacked. All five PR heads have passed hosted CI and Docker smoke checks, but merge remains blocked on the real authenticated screenshot evidence required for PR #29. Merge sequentially after that evidence is recorded.
- **Web test work:** PR #27 (cross-platform OPD browser interactions) and PR #28 (local Playwright HTML report) are merged.
- **API security:** PR #54 membership audit coverage and PR #55 inaccessible-hospital collection contract fix are merged after PostgreSQL/Docker verification.
- **Local environment:** PR #5 GSSAPI warning and PR #7 Keycloak silent-renewal reconciliation are merged. PR #6 optional observability stack remains open pending full Docker runtime/telemetry smoke.
- **Database and caching:** Database PR #2 backup/restore helpers and Caching PR #5 tenant isolation tests are merged after hosted Docker/test verification.
- **Documentation:** PR #50 is merged, paired with Local PR #5. Keep future docs changes paired with implementation status.

The number of open PRs is now 6: five deliberately stacked Web experience slices (#29–#33) and one independent Local observability overlay (#6). The API, Database, Caching, Web test and Local Keycloak/GSSAPI workstreams listed above have been merged; they are not duplicates or abandoned PRs.
 
## 8. Evidence and source artifacts

This record is based on the Antigravity local full-stack acceptance report, its `acceptance-results.json` and `execution-log.txt`, plus the separate mocked Playwright HTML report. The tested build SHAs are recorded above. Screenshots are referenced by test ID in the original local report; this document does not claim that those screenshots have been copied into the documentation repository.
