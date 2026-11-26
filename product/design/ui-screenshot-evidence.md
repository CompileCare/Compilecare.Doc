# CompileCare UI screenshot evidence

## Purpose

Keep product documentation aligned with the real authenticated hospital workspace. Screenshots are implementation evidence, not design mockups.

## Capture prerequisites

- A deployed or locally running CompileCare.Web build with its exact commit SHA recorded.
- A non-production OIDC authority/client configuration and a staging CompileCare.Api environment.
- Synthetic tenant, hospital, department, staff and patient records. Do not use production patient information.
- A browser capture environment that can complete the OIDC sign-in flow.

## Required views

1. Reception workspace with selected synthetic tenant and hospital.
2. Patient lookup and registration/duplicate-review states.
3. Appointment booking, availability and review/confirmation.
4. Walk-in registration and server-issued queue token confirmation.
5. Department queue in Waiting, Called and InProgress states, including the permitted actions for each state.
6. Clinical queue with in-progress and completed entries, minimal synthetic patient identity banner and explicit identity-confirmation step.
7. Draft clinical note with the approved five sections, signed read-only note, and append-only amendment history.
8. Facility hierarchy administration showing the active hospital, department/ward/room/bed navigation, and permission-appropriate controls. Capture both a read-only view and a permitted write view where the staging roles allow it.
9. Tenant membership administration for a synthetic HospitalAdmin identity, showing the membership list, role/scope editor and explicit deactivate/reactivate confirmation. Do not expose real identity subjects or use production user data in screenshots.

Capture each view at a desktop viewport (1440×900), tablet viewport (1024×768) and narrow viewport (390×844), where the workflow remains meaningful. Include keyboard focus in at least one representative capture and keep labels/statuses legible.

## Privacy and integrity

- Never include real patient identifiers, names, dates of birth, contact details, clinical notes, access tokens or identity-provider secrets.
- Do not alter a screenshot to imply a workflow or state that the application did not render.
- Do not create synthetic screenshots using image generation or static HTML and describe them as application captures.
- Store approved captures under product/design/screenshots/ with descriptive names and a short evidence note containing the application commit SHA, viewport, capture date and staging environment identifier (not credentials).
- If authenticated staging or browser capture is unavailable, state that screenshot evidence is pending. Do not substitute mockups.

## Review checklist

- [ ] Correct app build SHA and viewport recorded.
- [ ] Synthetic-only data verified.
- [ ] No secrets or personal/clinical data visible.
- [ ] Screen reflects a real state reached through the application.
- [ ] Responsive layout and text remain legible.
- [ ] Keyboard focus and status feedback are visible where applicable.
