# Workflow: reception and clinical workspace UX foundation

**Status:** Interaction specification with authenticated reception lookup merged; patient registration UI implementation in progress; remaining workflows require pilot validation  
**Phase:** 5 — User experience  
**Related workflows:** [Patient registration](patient-registration.md), [Appointments and queue](appointments-and-queue.md), [OPD encounter and clinical documentation](opd-encounter-and-clinical-documentation.md)  
**Related architecture:** [ADR-005 — Frontend architecture](../../architecture/decisions/ADR-005-frontend-architecture.md)

## Purpose and boundaries

Define the first authenticated web workspace around the already implemented patient registration, appointment/queue and OPD encounter APIs. The first vertical slice is staff-assisted outpatient care from patient lookup through reception hand-off and clinician documentation.

This document specifies user tasks, screen responsibilities, information boundaries and interaction states. It does not invent new backend capabilities, grant permissions, define clinical policy or approve a final visual design. Server-side authorization and workflow state remain authoritative.

The existing `CompileCare.Web` repository is the authenticated hospital application. `CompileCare.Site` remains a separate public-facing site. The first shell and reception lookup increments are merged in CompileCare.Web; authenticated patient workflows remain out of the public site.

## Primary personas and scope

| Persona | Initial workspace tasks | Access boundary |
|---|---|---|
| Reception / registration | Search patient, complete registration when authorized, book appointment, register walk-in, check in, view queue | Assigned tenant, hospital and permitted departments; patient and appointment permissions are server-enforced |
| Queue operator | View operational queue, call/recall, start and complete hand-off, resolve permitted queue actions | Assigned hospital/department and queue permissions |
| Clinical user | Open an in-progress queue entry, create/read encounter, edit draft note, sign or append amendment when separately permitted | Explicit clinical permission plus tenant/hospital/department scope |
| Hospital administrator | Navigate facility context and manage only capabilities already exposed by authorized APIs | Explicit provisioned permissions; administrator status must not imply clinical permissions |
| Auditor | Future controlled review of audit records | Not part of the first interactive workspace until an approved audit-query contract exists |

A user may hold multiple roles. The UI must render capabilities from the authenticated server-authorized context; hiding a control is not authorization. Every mutation must still be rejected safely by the API when permission or resource scope is missing.

## Information architecture

The authenticated application shell should provide:

- **Tenant and facility context:** visible active organization/hospital and, where supported, department context. Switching context must be explicit and must not silently carry a stale resource selection across facilities.
- **Primary navigation:** Reception, Queue and Clinical workspace entries only when the user is authorized for their underlying capabilities.
- **Patient context:** a patient summary is shown only inside an authorized workflow and only with fields returned by the relevant API contract.
- **Session controls:** authenticated user identity and sign-out; do not display raw tokens or identity-provider claims.
- **System feedback:** clear loading, empty, validation, conflict, forbidden, not-found and service-unavailable states.

The initial release should not add patient self-service, billing, pharmacy, diagnostics, inpatient, workforce administration or a patient portal.

## Screen inventory and responsibilities

### 1. Reception home

Purpose: orient reception staff to the selected facility and expose permitted next actions.

- Provide direct actions for patient search, new patient registration (when permitted), appointment booking, walk-in registration and queue view.
- Show only operational summaries that are available through existing APIs. Do not invent counts or call an unavailable endpoint.
- Do not expose clinical note content, patient contact details or financial data on the landing screen.
- If the user has no reception permissions, show an access explanation rather than an empty or misleading dashboard.

### 2. Patient search and selection

- Search within the selected hospital registration boundary using the existing patient search contract.
- Present duplicate candidates as review suggestions, never as an automatic merge or association.
- Make hospital registration status explicit. A tenant patient without registration at the selected hospital must follow the authorized registration workflow before appointment booking or queue entry.
- Distinguish no results, multiple candidates, inactive registration, invalid search and service failure.
- Avoid displaying unnecessary demographic or contact fields in result rows.

### 3. Patient registration

- Use the existing patient-master and hospital-registration workflow and field requirements; do not add unapproved contact capture.
- Separate patient master creation from hospital registration in the interaction where the API models them separately.
- Preserve entered values after recoverable validation errors without writing sensitive data to browser logs or telemetry.
- Require a deliberate confirmation before creating a new patient when likely duplicate candidates are present.
- Never silently merge patient records or infer cross-hospital association.

### 4. Appointment booking

- Require an active registration at the selected hospital before showing booking as available.
- Guide the user through department, provider, date and available slot selection using existing availability and booking contracts.
- Display times in the hospital's configured timezone and make the selected facility clear.
- Treat slot conflicts and patient-overlap conflicts as recoverable outcomes: refresh availability and ask staff to choose again.
- Provide explicit confirmation and appointment reference after successful booking.
- Reschedule and cancellation are separate deliberate actions with the API-approved reason inputs; do not implement destructive inline editing.

### 5. Walk-in and check-in

- Walk-in flow starts from a selected active hospital registration and creates a queue entry without creating an appointment.
- Appointment check-in verifies the selected appointment and registration, then displays the returned queue token.
- Make retries safe and communicate when an existing queue entry was returned.
- Never generate queue tokens in the browser.
- Clearly distinguish appointment status from queue-entry status.

### 6. Queue operations

- Provide a department-scoped operational list with queue token, time/status and only the minimum patient context required by the authorized operator.
- Support only server-defined transitions: Waiting → Called; Called → Waiting (recall); Called → InProgress; InProgress → Completed; Waiting/Called → Cancelled or NoShow where permitted.
- Confirm consequential actions such as cancellation and no-show; collect only reason codes/text accepted by the API.
- Refresh or reconcile after conflicts rather than optimistically displaying a transition that the server rejected.
- Public waiting-room display is a separate future surface and must use queue token/minimum status only, never patient name, MRN or clinical data.

### 7. Clinical queue and encounter

- Show only in-progress queue entries visible to the authenticated ClinicalUser in the selected hospital/department.
- Starting an encounter must use the queue-linked encounter API; do not create an encounter directly from an appointment.
- Display a clear distinction between queue status and encounter status.
- Show the approved note sections: chief complaint, history, examination, assessment and plan.
- Draft edits remain available only while the encounter is InProgress.
- Signing requires a deliberate review/confirmation step and communicates that signed note content becomes immutable through normal update operations.
- Amendments must be presented as append-only additions with reason and author/time attribution; never offer an edit-in-place experience for signed text.
- Closing/cancelling follows the API lifecycle and must not imply queue completion.
- Do not introduce diagnosis, vitals, prescriptions, orders, referrals, triage, clinical decision support or emergency workflows in this UX increment.

## Interaction and accessibility requirements

- Keyboard operation for all navigation, search, selection, forms, dialogs and workflow actions.
- Visible focus indicators, semantic headings/landmarks, associated labels, accessible validation summaries and screen-reader announcements for async results.
- Do not communicate status by colour alone; pair status with text and, where helpful, an icon.
- Support responsive layouts for desktop and tablet first; small-screen behavior must remain usable but does not imply a mobile-native application.
- Use consistent date/time formatting with the hospital timezone and unambiguous date entry.
- Prevent accidental double submission; use request state and server idempotency/concurrency contracts where available.
- Preserve non-sensitive form state across recoverable errors; clear sensitive values when leaving an authorized context where appropriate.
- Provide explicit loading, empty, error, forbidden, not-found, conflict and stale-data states for every data-backed screen.
- Avoid auto-refresh behavior that can overwrite in-progress user input.
- Use plain, operational language; avoid clinical interpretation or recommendations.

## API and data ownership rules

- The API is authoritative for identity, tenant/facility scope, permissions, validation, queue transitions, appointment availability and encounter lifecycle.
- The frontend must consume versioned API contracts and must not duplicate business rules as a source of truth.
- Do not infer permissions from role labels, local storage, URL parameters or client-side claims.
- Do not persist access tokens or patient/clinical payloads in local storage, analytics, crash reports or general browser logs.
- Do not put patient names, MRNs, contact details, note text or other protected data in URLs, metric labels or telemetry.
- Use generated or centrally maintained typed API clients where practical; keep module-specific request/response mapping within the owning feature.
- Handle 401, 403, 404, 409 and validation Problem Details distinctly; never convert a forbidden or not-found response into a misleading empty state.

## Initial delivery sequence

1. **Application shell — implemented:** React + TypeScript in `CompileCare.Web`, OIDC Authorization Code + PKCE, accessible responsive navigation, session controls and shared loading/error patterns. Web PR #1 is merged.
2. **Tenant and hospital context — implemented:** API PR #26 exposes audited current-user active tenant memberships using validated issuer/subject. The web client lets the user select a tenant and loads only hospitals visible under the persisted organization scope. No client-supplied role or tenant header grants access.
3. **Reception patient lookup — implemented:** Web PR #2 uses the hospital-scoped patient search API with bounded pagination. Search results are kept in component memory, not browser storage; the API remains authoritative for patient.read permission and hospital scope. Patient registration with duplicate-candidate review and MRN assignment is being implemented in a separate focused UI increment.
4. **Appointment and arrival:** booking, walk-in, check-in and queue-token confirmation.
5. **Queue operations:** scoped queue view and permitted lifecycle actions.
6. **Clinical workspace:** clinical queue, encounter draft, sign and append-only amendment flows.
7. **Hardening:** keyboard/accessibility review, responsive checks, authorization-state tests, API contract tests and Docker-based verification where supported by the repository.

Each increment should be a small vertical slice with tests and documentation updates. Do not implement all screens against mocked contracts and then retrofit API behavior.

## Acceptance criteria for the UX foundation

- Reception, queue and clinical tasks are mapped to existing backend workflows and APIs.
- Every screen has an explicit persona, purpose, required permission/scope, data source and error/empty/forbidden state before implementation.
- The public site and authenticated hospital application remain separate.
- Patient registration and appointment booking preserve the existing hospital-registration prerequisite.
- Queue actions and encounter lifecycle are presented as separate workflows.
- Signed clinical note content is never editable in place; amendments are append-only.
- No unimplemented billing, contact capture, vitals, diagnoses, prescriptions, diagnostic orders or patient-facing workflows are represented as available.
- Accessibility, keyboard operation, responsive behavior and data-minimization requirements are included in implementation tests/review.
- Server-side authorization remains authoritative and is exercised for both allowed and denied UI/API paths.
- Build, automated tests and Docker-based verification pass before a UI increment is merged.

## Open product and design questions

- Which hospital roles map to the existing appointment, queue and clinical permissions for the pilot?
- Which patient demographic fields are necessary in reception search results and the patient banner?
- What facility/department switching behavior is needed for users assigned to multiple facilities?
- Which languages are required for the first pilot, and which content must be localized?
- Which supported browsers and minimum desktop/tablet viewports must be tested?
- What is the approved session timeout, re-authentication and workstation-lock policy?
- Which clinical note templates and signing policies are approved by pilot clinicians?
- What design system and visual brand tokens should be adopted?

These questions do not block an initial shell and API-backed patient lookup, but they must be resolved before exposing role-sensitive or clinical workflows to pilot users.


## Implementation status — 2026-11-24

The authenticated shell, tenant/hospital context and patient lookup are merged in `CompileCare.Web` PRs #1 and #2. Patient registration, duplicate-candidate review and hospital-local MRN assignment are merged in Web PR #3. Its paired documentation update is merged in `Compilecare.Doc` PR #30.

The next implementation slice is being developed on `phase5/appointments-and-arrival` with the paired documentation branch `phase5/appointments-and-arrival-ux`. The web API client now consumes the existing department, availability-session, slot, appointment-booking, appointment-check-in, walk-in and department-queue contracts. The reception UI is being extended with selectable patient search results, guided appointment booking against server-published slots, a separate walk-in flow, returned queue-token confirmation and a department queue view. Department, hospital and tenant scoping remain explicit on every request; queue tokens and booking outcomes are server-owned.

The UI implementation must follow the [CompileCare experience quality bar](../design/experience-quality-bar.md): calm clinical visual hierarchy, responsive desktop/tablet layouts, keyboard-complete selection, visible focus, status text, accessible async feedback and privacy-preserving data handling. This is an implementation standard, not a claim that the current increment has passed visual/accessibility review.

The current increment is not yet merged. Required release gates are API contract tests, UI behavior tests, production TypeScript build, CI Docker build/container smoke test, and manual desktop/tablet/narrow viewport plus keyboard/accessibility review. No unsupported provider data, appointment lists, queue lifecycle actions or clinical workflows will be fabricated in the UI; missing capabilities will remain explicit follow-up work.
