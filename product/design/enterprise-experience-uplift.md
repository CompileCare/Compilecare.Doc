# CompileCare — Enterprise Experience Uplift

**Status:** Phase 5 product and engineering priority; implementation is incremental and must be verified per slice.  
**Scope:** Authenticated CompileCare hospital workspace in `CompileCare.Web`. The public-facing site is outside this initiative.

## 1. Intent

CompileCare has delivered the first operational workflows for reception, patient registration, appointments, queue operations, clinical encounters and OPD billing. The next quality step is to make these capabilities feel like one cohesive, premium enterprise hospital application rather than a collection of feature screens.

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

The first shared foundation is under review in [CompileCare.Web PR #29](https://github.com/CompileCare/CompileCare.Web/pull/29). Its current branch contains:
- Shared design tokens for canvas/surface, typography, text, borders, primary/focus/semantic status colors, spacing, radii and elevation.
- Consistent shell, navigation, page header, context panel, forms, status, queue, clinical and billing surface styling.
- Permission-driven active navigation with `aria-current`, a direct Patient registration link only when the signed-in context includes the required registration permissions, and a keyboard skip-to-main-content link.
- The page heading follows the selected workspace section, while the context panel identifies the selected hospital (or tenant while hospital selection is pending) and shows the membership role plus context-selection state. The section hash is accepted only when it matches a module exposed by the current permission context; a stale hash cannot label a hidden module as active.
- Mobile navigation overflow handling that retains the module links rather than wrapping them into an unstable multi-row menu.
- A component test assertion for the skip link.

This is a foundation increment, not completion of the uplift. Build, test, Docker, real-browser visual review and desktop/tablet/mobile evidence are still required. The current hosted Web CI run failed before runner allocation (`runner_id=0`, no runner name and zero steps); it did not execute the build or tests. Do not merge until the required checks actually execute and pass.

## Reception search component refinement — 2026-11-25

The follow-on [CompileCare.Web PR #30](https://github.com/CompileCare/CompileCare.Web/pull/30), stacked on the shell foundation in PR #29, extracts the patient result list into a focused feature component. Each patient result is a native button with an exposed pressed/selected state; Enter and Space work through standard button behavior, and in-flight operations disable selection. This replaces a focusable `article role="button"` with manually implemented keyboard handling.

The component has focused tests for accessible naming, selected state, activation, disabled state and empty results. The PR does not change patient data, server search, permissions or API contracts. Build, test and actual browser verification remain required before merge.

## 10. Success criteria

The workspace looks and behaves like one coherent enterprise product; staff can scan and complete core workflows efficiently; the active tenant/hospital context is always clear; controls and feedback are consistent; essential workflows are keyboard-complete and responsive; and every implementation remains aligned with existing API contracts, data minimization and tenant/organization authorization.
