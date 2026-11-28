# HeapCare — Enterprise Experience Uplift

**Status:** Phase 5 product and engineering quality track; core workspace slices are implemented and integrated. Web PRs #27–#35 and the later workspace improvements are merged. Local PR #39's exact-head run #46086998413 passed the nine-service stack and 11 real-Keycloak browser tests, including all six role accessibility-baseline checks. Real authenticated visual evidence, full manual keyboard/screen-reader evaluation, contrast and zoom/reflow review, and qualified WCAG 2.2 AA assessment remain open product-acceptance items.  
**Scope:** Authenticated HeapCare hospital workspace in `HeapCare.Web`. The public-facing site is outside this initiative.

## 1. Intent

HeapCare has delivered the first operational workflows for reception, patient registration, appointments, queue operations, clinical encounters and OPD billing. The next quality step is to make these capabilities feel like one cohesive, premium enterprise hospital application rather than a collection of feature screens.

"World-class" means more than visual decoration. Staff should understand where they are, what hospital and department context is active, what needs attention, what action is safe to take, and what happened after every operation. The interface must remain calm, fast to scan, accessible, responsive and trustworthy.

This initiative improves the presentation and usability of existing capabilities. It does not authorize new clinical, financial or administrative functionality.

## 2. Experience principles

1. **Clinical confidence:** light-first surfaces, deep ink/navy text, restrained teal/blue accents and sparing semantic status colors. Avoid noisy gradients, glass effects, ornamental charts and excessive shadows.
2. **Task-first hierarchy:** every workspace makes the current module, active organization context, primary action and outstanding state clear.
3. **Operational density:** staff need clear, aligned lists, tables, filters and compact metadata. Avoid oversized dashboard cards and excessive whitespace, while preserving legibility and comfortable touch targets.
4. **Consistency:** shared typography, spacing, controls, headings, buttons, status badges, alerts, dialogs and empty/loading/error states should behave and look alike across modules.
5. **Progressive disclosure:** expose consequential actions only when relevant and authorized. Keep booking, check-in, queue transitions, clinical signing and payment steps distinct.
6. **Trustworthy feedback:** show loading, success, validation, empty, conflict, forbidden, not-found and service-error states distinctly. Never optimistically claim an API operation succeeded.
7. **Accessible interaction:** target WCAG 2.2 AA; provide semantic landmarks/headings, keyboard-complete flows, visible focus, labels, meaningful accessible names, contrast, reduced motion and screen-reader announcements.
8. **Responsive operation:** optimize for real hospital desktop and tablet workflows, then reflow for narrow screens without losing essential actions or forcing horizontal scrolling.
9. **Privacy by design:** show only information required for the task. Never place patient/clinical data, tokens or credentials in URLs, browser storage, general telemetry or unapproved screenshots.

## 3. Visual language and design system

### Foundations

- Typography: define a legible type family and named sizes/weights for page title, section title, body, form labels, helper text and metadata. Use tabular numerals for dates, times, amounts and queue tokens.
- Color: centralize canvas, surface, raised surface, border, primary/secondary text, primary action, focus, success, warning and danger tokens. Status always includes text; color is never the only signal.
- Spacing: use a documented spacing scale and consistent page gutters, form gaps and panel padding.
- Shape and elevation: standardize corner radii and low-elevation surfaces; avoid layering cards inside cards without a clear grouping purpose.
- Icons: use one consistent, accessible icon family with labels for unfamiliar or consequential actions. Do not use emoji as operational icons.
- Motion: use restrained transitions for orientation and feedback, respect `prefers-reduced-motion`, and never animate data in a way that hides a workflow change.

### Shared components

Establish reusable patterns for:
- Application navigation, page heading, breadcrumbs where useful and active facility context.
- Primary, secondary, quiet and destructive buttons, with consistent focus, disabled and in-flight states.
- Text inputs, searchable selects, date/time controls, field hints and accessible validation.
- Panels, cards, tables/lists, pagination and filter rows with consistent alignment and information density.
- Status chips that always include readable text.
- Inline alerts, toasts/announcements, confirmation dialogs, empty states and loading skeletons.
- Error and recovery patterns for 401, 403, 404, 409, validation failures and service errors.

Shared components must not hide permission checks or replace server-side authorization. They must not invent fallback data or silently turn an access failure into an empty state.

## 4. Application shell

The shell should provide:
- Clear primary navigation grouped by actual modules the signed-in membership can access.
- Responsive collapse/navigation behavior without losing access to essential workflows.
- Visible tenant and hospital context, with department context where the current workflow requires it.
- A consistent page title and contextual actions region.
- Predictable session/user controls and clear sign-in/session-expiry states.
- Content width, spacing and responsive breakpoints that are consistent across every module.

Only modules returned by the established permission/context contract may be presented as available. The API remains the authorization boundary.

## 5. Module refinement order

1. **Shell and visual foundations:** tokens, base typography, navigation, page heading, context selector, common panels and controls.
2. **Reception and patient registration:** scan-friendly patient search, patient result selection, registration form grouping, duplicate review and MRN confirmation.
3. **Appointments and arrival:** guided but compact booking steps, clear provider/session/date/slot selection, timezone clarity and conflict recovery.
4. **Queue operations:** high-readability token/status rows, role-safe contextual actions, clear state transitions and accessible confirmation for cancellation/no-show.
5. **Clinical workspace:** patient identity verification, readable note sections, unmistakable draft/signed/closed states and append-only amendment presentation. Clinical data must never appear in operational queue lists.
6. **Billing:** invoice/settlement hierarchy, balance and payment history clarity, receipt recovery/print and precise financial state feedback.
7. **Cross-module polish:** responsive navigation, keyboard/focus consistency, empty/loading/error states, date/time formatting and browser-level visual regression.

Implement each step as a focused Web PR paired with the relevant workflow/design documentation. Do not combine a broad visual rewrite with backend/domain changes.

## 6. Verification and evidence

For each user-visible slice:
- Capture a baseline and an updated screenshot from the actual running authenticated app using synthetic/local data.
- Record repository commit, browser, viewport and environment in the evidence note.
- Verify primary and recovery workflows at desktop, tablet and 375px mobile width.
- Complete the flow using keyboard only; inspect focus order, visible focus, labels, semantic headings, dialogs, announcements and contrast.
- Add/extend component and Playwright tests for changed interactions and responsive behavior.
- Keep mocked UI regression results separate from real Keycloak/API/PostgreSQL acceptance results.
- Run production build, tests and Docker build/container smoke verification. Do not merge based only on a workflow that failed before runner allocation.
- Never use generated mockups as proof of the real product. Never capture real patient data, tokens, passwords or private identity-provider details.

## 7. Out of scope and guardrails

- No new dashboard KPIs, business metrics, clinical decision support or invented operational data.
- No client-side role/tenant authorization or weakening of OIDC/API security.
- No unapproved patient-contact collection, clinical fields, prescribing, diagnostics, insurance or advanced billing capabilities.
- No full rewrite of the current application component before a staged component decomposition plan is reviewed.
- No production-readiness or WCAG conformance claim without the corresponding verification evidence.

## 8. Current implementation baseline — 2026-11-25

The first shared foundation is under review in [HeapCare.Web PR #29](https://github.com/HeapCare/HeapCare.Web/pull/29). Its current branch contains:
- Shared design tokens for canvas/surface, typography, text, borders, primary/focus/semantic status colors, spacing, radii and elevation.
- Consistent shell, navigation, page header, context panel, forms, status, queue, clinical and billing surface styling.
- Permission-driven active navigation with `aria-current`, a direct Patient registration link only when the signed-in context includes the required registration permissions, and a keyboard skip-to-main-content link.
- The page heading follows the selected workspace section, while the context panel identifies the selected hospital (or tenant while hospital selection is pending) and shows the membership role plus context-selection state. The section hash is accepted only when it matches a module exposed by the current permission context; a stale hash cannot label a hidden module as active.
- Mobile navigation overflow handling that retains the module links rather than wrapping them into an unstable multi-row menu.
- A component test assertion for the skip link.

This is a foundation increment, not completion of the uplift. Build, test, Docker, real-browser visual review and desktop/tablet/mobile evidence are still required. The current hosted Web CI run failed before runner allocation (`runner_id=0`, no runner name and zero steps); it did not execute the build or tests. Do not merge until the required checks actually execute and pass.

### Local acceptance findings — 2026-11-25

The baseline build (`HeapCare.Web` `5a90b0a`, `HeapCare.Api` `5a6d882`) was exercised against the real local Docker stack: 48 of 50 executed acceptance cases passed; AUTH-005 and SEC-002 failed and are tracked in Local PR #7 and API PR #55. The mocked UI suite separately reports 35/35 Vitest tests passing and 11/14 Playwright tests passing on macOS arm64; the three native-select/Select-All portability failures are addressed in Web PR #27. The same run verified no horizontal overflow at 1440×900, 768×1024 and 375×812 and a visible 3px focus indicator with labelled inputs. Eight unimplemented UI/future-scope scenarios remain Not Run. See [the complete local acceptance record](../qa/local-stack-acceptance-2026-11-25.md). These local results do not replace hosted CI, Docker verification on the PR head commits, or real-app visual evidence.

## Reception search component refinement — 2026-11-25

The follow-on [HeapCare.Web PR #30](https://github.com/HeapCare/HeapCare.Web/pull/30), stacked on the shell foundation in PR #29, extracts the patient result list into a focused feature component. Each patient result is a native button with an exposed pressed/selected state; Enter and Space work through standard button behavior, and in-flight operations disable selection. This replaces a focusable `article role="button"` with manually implemented keyboard handling.

The component has focused tests for accessible naming, selected state, activation, disabled state and empty results. The PR does not change patient data, server search, permissions or API contracts. Build, test and actual browser verification remain required before merge.

## 10. Success criteria

The workspace looks and behaves like one coherent enterprise product; staff can scan and complete core workflows efficiently; the active tenant/hospital context is always clear; controls and feedback are consistent; essential workflows are keyboard-complete and responsive; and every implementation remains aligned with existing API contracts, data minimization and tenant/organization authorization.


## Appointment slot selection refinement — 2026-11-25

The follow-on [HeapCare.Web PR #31](https://github.com/HeapCare/HeapCare.Web/pull/31), stacked behind shell PR #29 and Reception PR #30, extracts the time-slot controls into an `AppointmentSlotGrid` component. It presents available and unavailable slots as native buttons, exposes selected state with `aria-pressed` and visible “Selected” text, and names each control with its formatted time and availability. Times are formatted using the selected provider availability session's IANA timezone. Slot changes are disabled while a workflow operation is in progress and after the booking has been confirmed.

The component consumes only the existing slot list returned by the selected hospital/department/provider-session availability API. It does not decide availability, publish slots, reserve a slot, alter scheduling rules or change the booking API. The API remains the authority for concurrency, registration, authorization and reservation.

Focused tests cover accessible names, availability/disabled state, selected-state semantics, selection callback, in-flight disabling and empty slots. Test fixtures use UTC explicitly for deterministic formatting. Hosted build, browser and Docker verification passed before merge.


## Queue row accessibility refinement — 2026-11-25

The follow-on [HeapCare.Web PR #32](https://github.com/HeapCare/HeapCare.Web/pull/32), stacked behind shell, Reception and appointment refinements, extracts each displayed queue entry into a reusable `QueueEntryRow` component. The token row is a named article, its server-returned status is an announced status region, and the transition actions are grouped under a descriptive accessible name. The existing action matrix and behavior are preserved: Waiting can be called/cancelled/marked no-show; Called can be recalled/started/cancelled/marked no-show; InProgress can be completed; terminal states have no transition controls. Cancellation and no-show remain disabled until a reason is present, and all actions remain disabled while a transition is in flight.

Focused tests cover current status/action combinations, required-reason behavior, callback identity, in-flight disabling, terminal-state controls and the presentation for users without the queue-management capability. The component displays only API-returned queue records; the API remains authoritative for state transitions and authorization. Build, browser and Docker verification remain required before merge.


## Clinical queue orientation refinement — 2026-11-25

The follow-on [HeapCare.Web PR #33](https://github.com/HeapCare/HeapCare.Web/pull/33), stacked after the shared shell, Reception, appointment, and general queue-row slices, extracts the clinical queue item into a focused `ClinicalQueueRow` component. Each row is named by its operational token, the server-returned InProgress/Completed status is exposed as a status announcement, and the open/resume action receives a token-specific accessible name. The existing loading and busy state disables the action.

This increment is deliberately limited to the clinical queue list. The identity confirmation required before viewing an encounter, department patient lookup, draft editing, signing, closing, cancellation, append-only amendment handling, and all API authorization flows remain in the existing clinical workflow and are not changed. The queue row continues to display operational tokens only; patient identity and clinical content are not added to queue lists.

Tests cover row/status/action names, passing the selected API-returned QueueEntry to the existing open handler, completed-visit resume label and disabled states. Build, browser and Docker verification remain required before merge.


## Verification update — 2026-11-26

The public-repository GitHub Actions workflows are now allocating hosted runners and executing their steps. The previous zero-runner condition is no longer blocking the active Web UI work.

### Completed and merged

- [HeapCare.Web PR #27](https://github.com/HeapCare/HeapCare.Web/pull/27) — cross-platform Playwright interactions for native selects and macOS/Linux Select All behavior. Hosted browser and Docker smoke checks passed before merge.
- [HeapCare.Web PR #28](https://github.com/HeapCare/HeapCare.Web/pull/28) — local Playwright HTML report configuration. Hosted CI passed before merge.
- [HeapCare.Local PR #7](https://github.com/HeapCare/HeapCare.Local/pull/7) — silent-renewal redirect registration plus idempotent reconciliation for an existing Keycloak client. Local CI passed the Docker Keycloak realm-import smoke test and exercises removal/restoration of the silent callback without resetting the realm.
- [HeapCare.Api PR #55](https://github.com/HeapCare/HeapCare.Api/pull/55) — consistent 404 behavior for inaccessible hospital-scoped patient search and department collection listing. .NET CI and Docker/PostgreSQL integration verification passed before merge.
- [HeapCare.Local PR #6](https://github.com/HeapCare/HeapCare.Local/pull/6) — optional observability overlay. Docker runtime smoke runs 45740981547 and 45740976578 passed for Collector, Prometheus, Jaeger v3, Elasticsearch and Kibana using synthetic telemetry before merge.

### UI stack verification

| PR | Slice | Hosted verification |
|---|---|---|
| [#29](https://github.com/HeapCare/HeapCare.Web/pull/29) | Shared visual foundation and shell | Run 45738737945 passed Vitest, build, Chromium E2E and Docker smoke |
| [#30](https://github.com/HeapCare/HeapCare.Web/pull/30) | Reception patient selection | Run 45738814930 passed |
| [#31](https://github.com/HeapCare/HeapCare.Web/pull/31) | Appointment slot selection | Run 45738824899 passed |
| [#32](https://github.com/HeapCare/HeapCare.Web/pull/32) | Queue-row accessibility | Run 45738835427 passed |
| [#33](https://github.com/HeapCare/HeapCare.Web/pull/33) | Clinical queue-row accessibility | Run 45739127780 passed, including the OPD journey E2E and Docker smoke |
| [#34](https://github.com/HeapCare/HeapCare.Web/pull/34) | Facility hierarchy CRUD workspace | Runs 45744960504 and 45744951178 passed 67 Vitest tests, 14 Chromium E2E tests, build and Docker smoke |
| [#35](https://github.com/HeapCare/HeapCare.Web/pull/35) | Tenant membership administration | Runs 45745881098 and 45746024118 passed 75 Vitest tests, 14 Chromium E2E tests, build and Docker smoke |

PRs #29–#34 have been merged in sequence into Web main, followed by membership administration PR #35. Merge commits: #29 `37f30508a23add4a8af0918307a524dd9f9f4bdb`; #30 `f0e548737ac05a1886640be742ed64a32f892e38`; #31 `0c10cafacb43183b47bc2f18ea0deadf8a90a95e`; #32 `68a8c9798a29df34031e2e13d7857550f887546b`; #33 `23cf6dcf503fc4e781e9b343d244ea9665566e29`; #34 `c8a5a5ec9a31e85cab05d60d05e8852616319be3`. The PR #29 CSS/test corrections ensure the skip-to-content link is the first keyboard stop and the narrow-screen navigation remains within the viewport; PR #33's OPD journey selector now matches the queue-token-specific accessible name.

### Remaining acceptance gate

The UI stack has been merged, but the screenshot evidence described in [UI screenshot evidence](ui-screenshot-evidence.md) remains outstanding as a product acceptance item. Capture the actual authenticated application with synthetic tenant/hospital/department/staff/patient data at desktop, tablet and narrow viewports, and include a representative keyboard-focus state. Include the Facilities hierarchy view, Memberships view for a synthetic HospitalAdmin, and Platform tenants view for a synthetic persisted platform operator. Capture read-only and permitted write roles where available. Record the exact Web main commit SHA, viewport and capture date. Do not substitute mocked Playwright screenshots or design mockups for this evidence.

[HeapCare.Local PR #6](https://github.com/HeapCare/HeapCare.Local/pull/6) is merged after the Docker runtime smoke verified synthetic metrics, traces and logs through the observability backends. A real HeapCare.Api request-to-log/metric/trace check on the developer Mac remains post-merge local acceptance, not a merge gate.


The Facility workspace implementation is documented in [Facility hierarchy management](../workflows/facility-management.md). It uses existing tenant-scoped facility APIs, applies membership-scope-aware controls and keeps server authorization authoritative. Web PR #34 is merged as `c8a5a5ec9a31e85cab05d60d05e8852616319be3`. The original Antigravity scenario was Not Run in the earlier tested build; it is now implemented but remains pending authenticated local-stack acceptance with synthetic facility records.

Tenant-scoped membership administration is documented in [Tenant membership administration](../workflows/tenant-membership-administration.md) and merged in Web PR #35 as `b8e4a4ec8864884f5e71dc070a319a5f4df07185`. It is shown only to memberships with `membership.manage`, supports exact OIDC subject-based provisioning, role/scope editing and confirmed membership lifecycle operations. The global platform tenant lifecycle workspace is documented in [Platform tenant lifecycle](../workflows/platform-tenant-lifecycle.md) and merged in Web PR #36 (`54589758db04848d25e2e078f3619c1932cc645a`) after 85 Vitest tests, all 14 Chromium E2E tests, production build and Docker smoke passed. It remains a separate platform-operator-only control plane, not part of the tenant HospitalAdmin workspace. API PR #57 adds the self-status signal; API PR #58 enforces the initial tenant-wide HospitalAdmin prerequisite before activation.

Tenant-scoped membership administration is documented in [Tenant membership administration](../workflows/tenant-membership-administration.md) and merged in Web PR #35 as `b8e4a4ec8864884f5e71dc070a319a5f4df07185`. It is shown only to memberships with `membership.manage`, supports exact OIDC subject-based provisioning, role/scope editing and confirmed membership lifecycle operations. The global platform tenant lifecycle workspace is merged in Web PR #36 (`54589758db04848d25e2e078f3619c1932cc645a`) after hosted Vitest, production build, all 14 Chromium E2E tests and Docker smoke passed. It remains a separate platform-operator-only control plane, not part of the tenant HospitalAdmin workspace.


## Merge completion — 2026-11-26

The complete Web experience, facility and tenant administration stack (#29–#36) is merged into `main`. Final facility-workspace verification on head `b36c7115003384eee8f809511a0b6808b36e76f2` passed 67 Vitest tests, all 14 Chromium E2E tests, production build and Docker smoke (runs 45744960504 and 45744951178). Tenant membership administration PR #35 passed 75 Vitest tests, all 14 Chromium E2E tests, production build and Docker smoke (runs 45745881098 and 45746024118). Platform tenant lifecycle PR #36 passed 85 Vitest tests, all 14 Chromium E2E tests, production build and Docker smoke (runs 45747601458 and 45747606834).

The stabilization PRs for malformed API request handling, concurrent booking, route landing/responsive UI/contrast, OTLP/Npgsql telemetry integration, Kibana data-view provisioning, API rate limiting/HSTS/trusted proxies, and the Web npm lockfile/reproducible install are merged. Remaining acceptance work is to capture authenticated screenshots with synthetic data and exercise facility CRUD, tenant membership, platform tenant lifecycle, Keycloak renewal, production CSP policy, trusted ingress configuration, and real API telemetry against the updated local Docker stack. Hosted CI remains distinct from developer-Mac authenticated acceptance.


## 11. Route-based module architecture — new product requirement (2026-11-26)

The current Web workspace uses a single top-level workspace composition with hash-selected sections. This is an interim implementation and does not yet provide the independent route-level workspaces expected of a full hospital information platform. The reported “everything looks like one page” experience is therefore an information-architecture issue as well as a visual-design issue.

### Target architecture
- One shared authenticated shell owns global navigation, user/session controls, tenant/hospital context and the main content outlet.
- Each module has an independent route-level page/workspace and feature-owned components, data loading and tests.
- Use stable browser paths, browser history, refresh-safe deep links and clear page headings/breadcrumbs.
- Keep platform administration in a separate platform-operator control plane.
- Navigation is generated from effective permissions and selected organization context; route guards are UX controls only. Every API operation remains protected by server-side authorization.
- Preserve existing workflow semantics and API contracts while extracting screens. Do not bundle domain changes into the route migration.
- Avoid a large rewrite. Migrate in focused, testable slices and remove legacy hash behavior only after redirects and regression coverage exist.

### Proposed route map
| Module | Proposed route |
|---|---|
| Overview | `/app/overview` |
| Reception | `/app/reception` |
| Patient registration | `/app/patients/register` |
| Patient record | `/app/patients/:patientId` |
| Appointments | `/app/appointments` |
| Queue | `/app/queue` |
| Clinical/OPD | `/app/clinical` |
| Billing | `/app/billing` |
| Facilities | `/app/facilities` |
| Staff | `/app/staff` |
| Memberships | `/app/memberships` |
| Patient identifier policy | `/app/settings/patient-identifiers` |
| Platform tenant lifecycle | `/platform/tenants` |

Final route names may be refined during implementation. Existing deep links should receive safe redirects where applicable.

### Migration sequence
1. Establish route layout, authenticated shell and permission-aware navigation.
2. Extract overview and reception/patient registration.
3. Extract appointment management and queue operations.
4. Extract clinical/OPD, preserving identity confirmation and signed-record safeguards.
5. Extract billing, facilities, staff, memberships and identifier policy.
6. Keep platform tenant lifecycle separate.
7. Add route-level states, responsive/browser/keyboard tests and authenticated evidence.
8. Remove hash-based section navigation after migration and redirects are verified.

### Acceptance criteria
- A module URL renders its own page, not merely a different heading over the same long page.
- Refresh, back/forward, bookmarks and deep links work predictably.
- The active tenant/hospital and required department context remain visible.
- Unauthorized routes show a safe access-denied/not-found experience without leaking data; APIs independently enforce authorization.
- Each route has explicit loading, empty, validation, conflict and service-error states as relevant.
- Keyboard navigation, visible focus, accessible names/headings, responsive desktop/tablet/mobile and WCAG 2.2 AA target checks are covered.
- Playwright verifies navigation and critical workflows; Vitest verifies focused components.
- Actual authenticated screenshots use synthetic data and record commit, browser, viewport and date. Mockups and mocked screenshots are not acceptance evidence.

### UI scope guardrails
This initiative restructures and improves the experience of existing capabilities. It does not add new KPIs, clinical functions, financial rules or unapproved data collection. Do not claim WCAG conformance or production readiness without evidence.


## Stabilization verification update — 2026-11-26

The first responsive/landing/contrast corrections are merged in Web PR #45:
- Reception now wins the default workspace selection when the effective membership grants patient-search access; Overview remains available as a secondary section.
- The authenticated Overview has a 390×844 Playwright overflow regression and narrow-width layout rules.
- Appointment cancellation uses an explicit high-contrast destructive style with a component assertion.

Static workspace response hardening is merged in Web PR #46 and covered by a Docker header check: `nosniff`, strict-origin referrer policy, `X-Frame-Options: DENY`, permitted cross-domain policy, same-origin CORP and restrictive Permissions-Policy. CSP remains intentionally deferred until deployment-specific identity and API origins are known and tested.

These are hosted-CI verified corrections, not final visual acceptance. The user-local authenticated experience still needs desktop/tablet/mobile screenshots, keyboard/screen-reader review and confirmation against the actual OIDC/API environment. The route-based module migration described above is still future work; the current single-workspace/hash-selected composition remains an interim architecture. See [the roadmap stabilization evidence](../roadmap.md#stabilization-evidence-update--2026-11-26-post-full-platform-regression).


## Security and supply-chain follow-up — 2026-11-26

- API PR #79 is merged: per-subject/per-IP API rate limiting, RFC 9457 429 + Retry-After, trusted X-Forwarded-For/X-Forwarded-Proto handling restricted to configured proxy IPs, one-hop/symmetry checks, HSTS outside Development, and Kestrel server banner suppression. The built-in limiter is per API instance, not a fleet-wide quota. Target deployment must configure the trusted proxy IPs and tune limits; use gateway/shared enforcement if multi-replica global quotas are required.
- Web PR #47 is merged: the npm-generated lockfile is committed; CI and Docker use npm ci; npm audit is blocking. Final PR CI passed Vitest, TypeScript/production build, Chromium E2E and Docker header smoke; the lockfile audit reported zero vulnerabilities.
- Production CSP remains deferred pending exact Keycloak/API origins and browser validation. This response-header and package-audit work must not be described as complete security compliance.
- Hosted nine-service Docker and telemetry verification is merged in Local PR #13. Local PR #14 verifies real Keycloak-issued user tokens and PostgreSQL-persisted two-tenant roles, hospital isolation and membership revocation/reactivation. Local PR #15 verifies browser Authorization Code + PKCE sign-in/callback and tenant switching. Local PR #16 verifies real reception patient registration/search for HospitalAdmin and API-side HTTP 403 denial for FacilityViewer. Remaining acceptance is the developer-Mac stack run, browser/session renewal there, appointments/queue/OPD/billing browser workflows, authenticated hospital-user screenshots, keyboard/accessibility and visual review, and production CSP/ingress validation. See the roadmap stabilization gates.


## Full-stack telemetry verification — 2026-11-26

Local PR #13 is merged. Hosted Docker CI built and started all nine local services and passed the full `verify-stack.sh`: PostgreSQL, Keycloak, API, Web, OTel Collector, Jaeger, Prometheus, Elasticsearch and Kibana. It confirmed HeapCare.Api traces in Jaeger including an Npgsql `postgresql` operation, and ASP.NET Core request metrics through the Collector exporter. The Kibana Logs data view was provisioned.

This removes the prior hosted-integration evidence gap. It does not replace the developer-Mac stack check or authenticated browser workflows, screenshots, keyboard/accessibility review or production security acceptance.


## Real Keycloak and persisted multi-tenant API acceptance — 2026-11-26

Local PR #14 is merged. Hosted full-stack CI used a temporary Keycloak client to obtain real user tokens and verified the API issuer/audience/subject contract, platform bootstrap, two PostgreSQL-backed tenants and distinct persisted roles for the same user. It passed the tenant context/permissions, hospital data-isolation, cross-tenant membership mutation denial, and immediate membership revocation/reactivation assertions.

The test exposed and corrected the local Keycloak subject-claim setup: the checked-in compilecare-web client now receives the built-in basic scope, and the local reconciler verifies its Subject (sub) mapper is enabled in access tokens. This is essential for the API to resolve tenant memberships using the validated issuer and subject. A separately managed production Keycloak realm must be checked independently.

The hosted API identity test on its own is not browser sign-in evidence; that separate gap is now covered at CI level by Local PR #15's real Authorization Code + PKCE login/callback and role-based tenant-switch suite. Developer-Mac acceptance, session renewal on that machine, real role-based hospital-user workflows beyond tenant selection, visual review, accessibility and production configuration remain open.


## Real browser PKCE and tenant-switch acceptance — 2026-11-26

Local PR #15 is merged. Hosted full-stack CI passed the real Authorization Code + PKCE browser flow against the actual Docker Web app, local Keycloak and HeapCare.Api. The synthetic user signed in through the Keycloak login page and Web OIDC callback, switched from HospitalAdmin in tenant A to FacilityViewer in tenant B, saw the correct CI Hospital A/B and role-specific navigation, and generated real API hospital-list requests with bearer authorization and the selected X-Tenant-ID.

This complements the persisted PostgreSQL tenant isolation and revocation checks in Local PR #14. The test uses a temporary CI credential file with restrictive permissions and cleans it up; tokens and authorization codes are not logged.

The hosted test closes the **browser PKCE + tenant switching CI** gate only. The developer-Mac run, browser session renewal and real local workflow validation, hospital-staff visual review, responsive screenshots, keyboard/screen-reader accessibility, and production CSP/ingress acceptance remain open.


## Real reception patient registration/search and role enforcement — 2026-11-26

Local PR #16 is merged. Hosted Local CI run `45807187585`, attempt 2, checked out HeapCare.Web main at `300afed` and passed the full nine-service job. Its real Keycloak browser test completed PKCE sign-in, selected the persisted HospitalAdmin membership, submitted a synthetic patient registration after the duplicate-review acknowledgement, searched for the resulting patient through Reception, then switched to FacilityViewer and confirmed patient-registration navigation was absent.

The API acceptance script independently called the patient registration endpoint using the FacilityViewer token and asserted HTTP 403. This validates server-side authorization in addition to permission-aware UI navigation. The browser observer retained only safe request metadata (method/path, tenant ID and bearer-token presence) and did not capture tokens or patient payloads.

The hosted result closes this specific reception workflow acceptance slice. Appointments, queue operations, OPD/clinical workflows, billing workflows, developer-Mac session renewal, hospital-user visual review, responsive screenshots, keyboard/screen-reader accessibility, and production CSP/ingress validation remain open.
