# CompileCare Web — experience quality bar

**Status:** Implementation standard for Phase 5 authenticated workflows  
**Applies to:** Reception, appointments, arrival, queue operations and clinical workspace

This document establishes the visual and interaction quality bar for the authenticated hospital workspace. It is a product standard, not a mandate to add decorative UI or expose capabilities that the API does not support.

## Experience principles

1. **Calm, clinical confidence:** Use a restrained, light-first interface with deep ink/navy text, a clear teal/blue primary action, quiet neutral surfaces and sparing semantic status colors. Avoid loud gradients, excessive shadows, glass effects and dashboard decoration.
2. **Task-first hierarchy:** Every screen answers where the user is, which hospital/department is active, what action is primary, and what needs attention. Keep the most common reception tasks reachable in one or two deliberate steps.
3. **Operational density without clutter:** Use strong alignment, consistent spacing, readable tables and compact metadata. Prioritize scanability over oversized cards or excessive whitespace; preserve comfortable touch targets.
4. **Progressive disclosure:** Reveal advanced or consequential actions only in the context where they are needed. Keep booking, cancellation, rescheduling, check-in and queue transitions distinct.
5. **Trustworthy feedback:** Every asynchronous action has loading, success, empty, validation, conflict, forbidden, not-found and service-error states as applicable. Never imply success before the server confirms it.
6. **Accessible by default:** Target WCAG 2.2 AA contrast, semantic landmarks and headings, keyboard-complete workflows, visible focus, associated labels, accessible dialogs, reduced-motion support and screen-reader announcements.
7. **Responsive by design:** Optimize for reception desktop and tablet workflows first, then reflow gracefully to narrow screens. Do not hide essential actions or force horizontal scrolling for core forms.
8. **Privacy in the interface:** Show only the minimum patient data needed for the task. Do not place patient names, MRNs, contact details or clinical information in URLs, telemetry, browser storage or public queue views.

## Visual system

- **Typography:** Use a highly legible sans-serif with a deliberate scale for page title, section title, field label, body and supporting metadata. Use tabular numerals for times, dates and queue tokens.
- **Color:** Define semantic tokens for canvas, surface, elevated surface, border, primary text, secondary text, primary action, focus, success, warning and danger. Status must always include text; color is never the sole signal.
- **Layout:** Use a consistent spacing scale, aligned content grid, stable page gutters and predictable form widths. Keep active tenant/hospital context visible.
- **Components:** Build reusable buttons, inputs, selects, date/time controls, searchable patient pickers, status chips, tables, empty states, inline alerts, confirmation dialogs and loading skeletons. Components must have consistent disabled, hover, focus and error states.
- **Motion:** Use brief, purposeful transitions only for orientation and feedback. Respect reduced-motion preferences; never animate queue data in a way that obscures changes.
- **Icons:** Use one consistent icon family, paired with labels for unfamiliar or consequential actions. Avoid emoji as operational UI icons.

## Appointment and arrival workflow requirements

- Present the flow as a guided sequence: patient selection → department/provider → date and availability → slot → review → booking confirmation.
- Make the selected hospital and appointment timezone explicit. Render slot times in the hospital timezone and retain the server UTC instant for requests.
- Distinguish unavailable slots from loading and failed availability retrieval. Refresh availability after a conflict; do not optimistically mark a slot booked.
- Walk-in is a separate path, not an appointment with missing fields. Require an active hospital registration and department; the server allocates the queue token.
- Check-in starts from a confirmed appointment and displays the returned queue token. Retries must safely show the existing queue entry when the API returns it.
- Do not invent provider/department data or infer permissions. Populate choices from authorized API responses and handle empty/forbidden states explicitly.
- Keep patient search and appointment selection scoped to the currently selected tenant and hospital. Clear dependent selections when context changes.

## Definition of done for every UI slice

- Design reviewed at desktop, tablet and narrow viewport widths.
- Keyboard-only completion verified for the primary and error-recovery paths.
- Contrast, focus, labels, accessible names and async announcements reviewed.
- Loading, empty, stale, conflict, forbidden, validation and server-error states are implemented.
- No patient or clinical payload is written to local/session storage, URL, analytics or logs.
- API contract tests cover route, tenant header, method, payload and no-store behavior.
- Component tests cover success, failure, retry, double-submit prevention and context changes.
- Production build and Docker image/container smoke test pass in CI before merge.
- Screens use existing API contracts only; missing backend capability is recorded rather than mocked into production UI.

## Explicit non-goals

This quality bar does not authorize self-service booking, payment, insurance, triage, clinical recommendations, public patient details, or client-side authorization. The API remains the source of truth for identity, permissions, availability, registration, booking, check-in and queue state.


## Enterprise experience uplift

The [Enterprise experience uplift](enterprise-experience-uplift.md) adds an explicit product-wide refinement track for the authenticated hospital workspace. Apply this quality bar to the shared shell and every module, not only to new screens. The intent is a cohesive, premium enterprise experience: precise visual hierarchy, clear operational density, consistent reusable components, dependable feedback and polished responsive behavior. Aesthetic choices must remain subordinate to clinical clarity, accessibility, privacy and server-authoritative workflows.

Treat screenshots as verification evidence from the real running application. Use synthetic/local data only, record the tested commit and viewport, and never capture credentials, tokens, real patient information or private identity-provider details.
