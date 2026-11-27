# Phase 5 — Authenticated workspace accessibility acceptance

**Status:** Automated authenticated full-stack smoke passed on 2026-11-27; manual six-role accessibility and local desktop acceptance remain not yet executed  
**Scope:** Existing HeapCare OPD workspaces only. This is an acceptance record and execution checklist, not a WCAG conformance statement.


### Real-Keycloak persisted-role verification — 2026-11-27

HeapCare.Local PR #29 merged on main as `93ae61f4742c9e5caf41ef4088bcb8a832a5dbb4`. Pull-request workflow run [#45999845006](https://github.com/HeapCare/HeapCare.Local/actions/runs/45999845006) passed both jobs: local tooling validation and the full nine-service Docker stack/API telemetry verification. The harness provisions synthetic identities for all five persisted tenant roles (HospitalAdmin, FacilityManager, FacilityViewer, PatientRegistrar and ClinicalUser), plus a separate PlatformOperator identity. It verifies server-derived role permissions and tenant hospital-read behavior, and checks positive/negative patient-search access for the relevant roles. The test data and identities are ephemeral and confined to the isolated CI stack.

This is automated authorization/identity acceptance evidence, not manual accessibility evidence. It does not execute the six-role keyboard, screen-reader, contrast, zoom/reflow or authenticated desktop review below. It also does not certify every endpoint/capability combination; focused API PostgreSQL tests remain the source for individual resource and action authorization.

## Evidence already present

### Current merged-main full-stack evidence — 2026-11-27

HeapCare.Local run [#45993951443](https://github.com/HeapCare/HeapCare.Local/actions/runs/45993951443) completed successfully on Local main commit `a2daae6a0f2cbb3eeea07d583ddf0459297b6eae`. Both jobs passed:

- **Local tooling validation:** macOS shell and Windows PowerShell syntax, fail-closed secret placeholders, Keycloak realm import (including Docker import smoke), OpenTelemetry Collector configuration, Compose validation and Docker observability pipeline smoke.
- **Full nine-service Docker stack and API telemetry verification:** built and ran the complete stack; verified real Keycloak-issued identities, persisted multi-tenant workflows, real Authorization Code + PKCE browser acceptance, and API telemetry. The run cleaned up only its isolated CI stack and CI-only secrets/volumes.

This is current merged-main automated full-stack evidence, not the manual authenticated accessibility review. It does not execute every role/workflow in the matrix below, test a developer's Mac environment, provide screen-reader evidence, or establish WCAG conformance. The role-by-role manual matrix remains outstanding.

The Web repository currently has an automated e2e/accessibility.spec.ts suite covering the signed-out/public workspace:

- Skip link is the first keyboard stop and moves focus to the main landmark.
- One main landmark and one level-one heading are present.
- Public controls and form fields expose accessible names/labels.
- Images declare alternative text.
- Reduced-motion preference suppresses nonessential transitions.
- The public workspace has no horizontal overflow at 768px, 375px and 320px.

The Web CI workflow runs Vitest, TypeScript/production build, Chromium E2E, a production Nginx Docker smoke, and the browser suite against the container with the enforced Content Security Policy. The roadmap also records merged keyboard-focused workflow coverage across Reception, clinical encounter entry, billing, appointments, queue and patient registration (Web PRs #20–#26; hosted runs #214, #218, #246, #247, #234, #254 and #257).

**Boundary:** These automated checks establish a useful baseline. They do not prove full authenticated-workspace accessibility, assistive-technology compatibility, WCAG 2.2 AA conformance, or local Keycloak/API/PostgreSQL acceptance.

## Execution prerequisites

- Run against the updated merged Web, API and Local commits using the real local Keycloak/API/PostgreSQL Compose stack.
- Use synthetic patients, synthetic staff and non-production identities only.
- Never capture or attach real patient names, MRNs, clinical notes, contact details, credentials, tokens or payment references.
- Record browser/OS, viewport, role, selected tenant/hospital/department, build commit and test date.
- If a scenario cannot be executed, mark it **Not Run** with the missing prerequisite; do not mark it passed based on a mocked browser or hosted Linux CI.

## Authenticated role and workflow matrix

| Role | Workflow to execute | Keyboard / semantics acceptance |
|---|---|---|
| PlatformOperator | Tenant creation, initial administrator provisioning, activation, suspension/reactivation and terminal closure | All controls reachable in logical order; confirmation dialogs announce purpose and outcome; status changes are announced |
| HospitalAdmin | Facilities, MRN policy, staff/provider assignments, memberships, availability, appointments, queue, billing catalogue and collections | Labels and required state are announced; destructive actions require clear confirmation; errors and conflicts are announced |
| FacilityManager | Facility hierarchy read/create/update | Read/write controls are discoverable; delete actions are absent; scope and validation feedback are announced |
| FacilityViewer | Facility hierarchy read-only | Read-only state is clear; mutation controls are absent; tables/lists remain navigable |
| PatientRegistrar | Reception search/registration, duplicate review, MRN, appointment booking/check-in, walk-in/queue and cashier billing | Complete primary workflows without a pointer; patient identity and duplicate review prompts are understandable; token-only queue behavior remains intact |
| ClinicalUser | Clinical queue, identity confirmation, draft note, sign, close and amendment | Identity gate is explicit; form labels/errors are associated; signing and amendment confirmations are clear; signed note immutability is communicated |

## Manual acceptance checklist

### Keyboard and focus
- [ ] Complete each role workflow using keyboard only; no keyboard traps.
- [ ] Focus order follows the visual and task order.
- [ ] Focus indicator is visible against every relevant surface and is not obscured by sticky navigation, dialogs or overlays.
- [ ] Skip link works after authentication and on each major workspace route.
- [ ] Dialogs move focus inside on open, keep focus contained, close with Escape where safe, and restore focus to the invoking control.
- [ ] Tables, selectable rows, slot grids and queue actions expose meaningful keyboard interaction and state.

### Names, labels and announcements
- [ ] Every input has a programmatic label; required, invalid, disabled and read-only states are conveyed.
- [ ] Icon-only controls have an accessible name and a sufficiently descriptive purpose.
- [ ] Validation, API errors, loading completion, booking conflicts, queue transitions and payment outcomes are announced without relying on color alone.
- [ ] Dynamic results and status changes use appropriate live-region semantics without excessively repeating content.
- [ ] Heading levels and landmarks form a coherent page structure.
- [ ] Data tables have meaningful headers and associations; list and grid semantics match interaction behavior.

### Visual and responsive
- [ ] Verify desktop, tablet and 375px mobile layouts; no clipped controls or horizontal page overflow.
- [ ] Check text, control, focus and status contrast against WCAG 2.2 AA target ratios.
- [ ] Confirm text zoom/reflow at 200% and 400% where supported without loss of workflow or content.
- [ ] Verify reduced-motion preference across navigation, status transitions and dialogs.
- [ ] Confirm sticky shell/navigation does not cover focused controls or active content.

### Assistive technology
- [ ] Screen-reader pass on at least one supported desktop screen reader/browser combination.
- [ ] Verify announced role/name/value for context selectors, appointment slots, queue actions, encounter identity gate, billing controls and modal dialogs.
- [ ] Verify token-only queue presentation and clinical privacy boundaries are preserved.

## Evidence record

| Evidence item | Status | Notes |
|---|---|---|
| Public workspace automated accessibility tests | Existing CI baseline | Implemented in e2e/accessibility.spec.ts; not an authenticated workflow audit |
| Feature keyboard browser coverage | Existing CI baseline | Merged Web PRs #20–#26; hosted Chromium coverage |
| Authenticated role matrix execution | Not Run | Requires local Keycloak/API/PostgreSQL stack and synthetic role memberships |
| Desktop/tablet/mobile authenticated screenshots | Not Run | Capture only after local authentication with synthetic data |
| Screen-reader review | Not Run | Requires a real desktop assistive-technology environment |
| Contrast and zoom/reflow review | Not Run | Manual visual verification required |
| WCAG 2.2 AA conformance | Not claimed | Requires complete, qualified evaluation; this checklist alone is not certification |

## Exit criteria

1. All six role workflows have an executed result or a clearly recorded blocker.
2. Keyboard-only completion is demonstrated for the primary tasks of each role.
3. Focus, dialog, accessible-name/label, status-announcement, contrast, zoom/reflow and reduced-motion findings are recorded with reproducible steps.
4. Any defects are fixed in focused Web PRs with regression tests and paired documentation.
5. The full Web test/build/Chromium E2E/Docker/CSP workflow passes for each code change.
6. Local authenticated Keycloak/API/PostgreSQL acceptance is rerun after fixes.
7. No WCAG 2.2 AA or production/pilot approval claim is made without the required complete review and separate security/operational sign-off.


### Route-level workspace navigation — 2026-11-27

Web PR #64 merged as `1a73684e7ac3c4660cf343855499a594792401c3`. Module URLs now display a single active workspace under the shared authenticated shell while retaining mounted component state. The hosted Web pipeline passed Vitest, TypeScript/production build, Chromium E2E, real-OIDC suite discovery and Docker/CSP smoke after the browser journeys were updated to navigate between reception, patient registration, appointments, queue and billing routes.

**Acceptance boundary:** this is automated hosted browser coverage. It does not complete the manual authenticated local Keycloak/API/PostgreSQL role matrix, keyboard-only review for all routes, assistive-technology testing, contrast/zoom evaluation or WCAG 2.2 AA conformance review. Those remain Not Run until executed on the target local environment with synthetic identities and data.


## Automated route-component and real-identity acceptance — 2026-11-27

### Web route workspaces merged

- Web PR #65: patient registration workspace extracted; duplicate candidate review and explicit acknowledgement preserved.
- Web PR #66: reception workspace extracted; MRN lookup remains gated by patient-identifier read permission.
- Web PR #67: appointment booking and queue/walk-in workspaces extracted; appointment slot selection, check-in, queue actions and cancellation/no-show reason behavior preserved.
- All three changes passed the complete hosted Web validation pipeline: Vitest, TypeScript/production build, Chromium E2E, real-OIDC suite discovery and Docker/CSP smoke.

### Full-stack local CI after Web PR #67

[HeapCare.Local run #45946047300](https://github.com/HeapCare/HeapCare.Local/actions/runs/45946047300) passed the nine-service Docker/observability stack and real Keycloak/OIDC acceptance.

- API, Npgsql traces and Collector-fed API metrics were verified through Jaeger/Prometheus; Kibana data view was provisioned over Elasticsearch logs.
- Real Keycloak-issued tokens passed issuer/audience checks.
- Two PostgreSQL-persisted tenant memberships with different roles were verified; hospital data isolation, cross-tenant membership mutation denial and membership revocation/reactivation passed.
- Real OIDC Authorization Code + PKCE browser sign-in passed (1 E2E test).
- Exact API/Web/Observability commit SHAs are recorded in the roadmap entry for this run.

**This does not complete the manual checklist below.** Authenticated role-by-role clinical workflows, manual visual acceptance, keyboard-only review, assistive technology, contrast, zoom/reflow, reduced motion and WCAG 2.2 AA assessment remain Not Run until performed on the target local environment with synthetic data. No accessibility certification or production/pilot approval is claimed.
