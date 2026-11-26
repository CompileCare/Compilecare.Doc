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
- **Correction:** Local PR #7 adds the missing redirect URI to the checked-in realm, extends the idempotent Keycloak reconciliation script to update an existing client without deleting its volume/users, and documents the safe reconciliation command.
- **Verification gate:** Apply the reconciliation script to the existing local realm, sign in, allow token renewal to occur, and confirm the renewed token is accepted by the API. Hosted Local CI and Docker verification must also execute.

### SEC-002 — foreign/out-of-scope hospital collection returns 200 instead of 404

- **Observed:** Patient search for a hospital ID belonging to another tenant returned HTTP 200 with an empty page. The report also observed the same response-contract issue on department collection listing for an out-of-scope hospital.
- **Data exposure:** The report found zero patient/department rows returned; tenant query filters and department scoping prevented data leakage in the tested case.
- **Expected contract:** Return 404 when the hospital is not visible in the caller's tenant and organization scope, consistent with single-resource anti-enumeration behavior.
- **Correction:** API PR #55 checks hospital visibility before patient search and changes department listing to return a typed not-found result for inaccessible hospital IDs. PostgreSQL integration cases cover a tenant-wide foreign-tenant patient search and both same-tenant out-of-scope and foreign-tenant department listing.
- **Verification gate:** Run the API build/integration suite against PostgreSQL and the full Docker acceptance scenarios; confirm 404 for both paths and that authorized hospital listing still returns 200.

## 3. Mocked Web regression results — separate from real-stack tests

| Suite | Result |
|---|---:|
| Vitest unit/component | 35/35 passed |
| Playwright mocked E2E | 11/14 passed on macOS arm64 |
| Playwright mocked E2E failures | 3 |

The three local macOS failures were caused by native `<select>` keyboard behavior and use of `Control+A` rather than macOS `Meta+A`. They were not failures in the live API/Keycloak/PostgreSQL acceptance run.

Web PR #27 makes select-value changes deterministic with Playwright's selection API while retaining keyboard focus-order assertions, and uses the platform-appropriate Select All shortcut. It must be rerun on macOS and Linux before merge.

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

On 2026-11-26, the latest retried GitHub Actions jobs across API, Web, Caching, Local and Database repositories completed in approximately two seconds with `runner_id=0`, an empty runner name and zero steps. The workflow job logs were unavailable because no runner log blob was produced. These results do not establish a source-code build/test failure: the jobs did not execute.

Do not merge an implementation PR based on a green-looking local report alone or on a hosted job that never received a runner. Restore GitHub Actions runner allocation, then rerun each PR's required build, tests, Docker checks and relevant browser/acceptance tests. Record the tested PR head SHA, not just the main-branch baseline SHA.

## 7. PR grouping and merge order

- **Web design stack:** PR #29 → #30 → #31 → #32 → #33. These are deliberately stacked; each PR's base is the preceding feature branch. Merge in dependency order only after each required check passes.
- **Web test work:** PR #27 contains authenticated mobile OPD coverage and macOS/Linux keyboard-test portability fixes; PR #28 contains local Playwright HTML report and screenshot configuration. They target Web main independently.
- **API security:** PR #54 membership audit coverage and PR #55 inaccessible-hospital collection contract fix are independent API changes. Verify each on its own head SHA.
- **Local environment:** PR #5 Npgsql GSSAPI warning, PR #6 optional observability stack and PR #7 Keycloak silent-renewal configuration are separate local-environment changes.
- **Database and caching:** Database PR #2 backup/restore helpers and Caching PR #5 tenant cache isolation are independent repository deliverables.
- **Documentation:** Keep documentation PRs paired with their implementation and merge the docs update only when the implementation status is accurately described.

The number of open PRs is not itself evidence of broken code. The Web #29–#33 chain is intentionally stacked, while the other PRs represent independent workstreams. The immediate blocker is hosted runner allocation; once restored, merge small, dependency-ordered slices and close superseded PRs rather than merging unverified changes.

## 8. Evidence and source artifacts

This record is based on the Antigravity local full-stack acceptance report, its `acceptance-results.json` and `execution-log.txt`, plus the separate mocked Playwright HTML report. The tested build SHAs are recorded above. Screenshots are referenced by test ID in the original local report; this document does not claim that those screenshots have been copied into the documentation repository.
