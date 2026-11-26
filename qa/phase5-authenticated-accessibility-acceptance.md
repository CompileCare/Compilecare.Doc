# Phase 5 — Authenticated workspace accessibility acceptance

**Status:** Planned / not yet executed against the authenticated local stack  
**Scope:** Existing CompileCare OPD workspaces only. This is an acceptance record and execution checklist, not a WCAG conformance statement.

## Evidence already present

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
