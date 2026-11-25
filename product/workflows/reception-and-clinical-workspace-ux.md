# Workflow: reception and clinical workspace UX foundation

**Status:** Reception, appointment/arrival, queue, OPD clinical and initial billing workspaces are implemented and merged. Browser-level billing recovery, full walk-in-to-billing journey, appointment booking/check-in and successful OIDC silent-renewal coverage are merged. Availability-conflict recovery, renewal-failure/re-authentication behavior and remaining operational workflows are next. Staff/provider administration remains gated on hospital-user validation.  
**Phase:** 5 — User experience  
**Related workflows:** [Patient registration](patient-registration.md), [Appointments and queue](appointments-and-queue.md), [OPD encounter and clinical documentation](opd-encounter-and-clinical-documentation.md)  
**Related architecture:** [ADR-005 — Frontend architecture](../../architecture/decisions/ADR-005-frontend-architecture.md)

## Purpose and boundaries

Define the authenticated hospital workspace around patient registration, appointment/queue, OPD encounter and initial billing APIs. The delivered outpatient slice runs from patient lookup and reception hand-off through clinician documentation, invoice settlement and receipt recovery.

This document specifies user tasks, screen responsibilities, information boundaries and interaction states. It does not invent new backend capabilities, grant permissions, define clinical policy or approve a final visual design. Server-side authorization and workflow state remain authoritative.

The existing `CompileCare.Web` repository is the authenticated hospital application. `CompileCare.Site` remains a separate public-facing site. The first shell and reception lookup increments are merged in CompileCare.Web; authenticated patient workflows remain out of the public site.

## Primary personas and scope

| Persona | Initial workspace tasks | Access boundary |
|---|---|---|
| Reception / registration | Search patient, complete registration when authorized, book appointment, register walk-in, check in, view queue | Assigned tenant, hospital and permitted departments; patient and appointment permissions are server-enforced |
| Queue operator | View operational queue, call/recall, start and complete hand-off, resolve permitted queue actions | Assigned hospital/department and queue permissions |
| Clinical user | Open an in-progress queue entry, create/read encounter, edit draft note, sign or append amendment when separately permitted | Explicit clinical permission plus tenant/hospital/department scope |
| Cashier / billing operator | Select a completed visit, issue or resume an invoice, record payment, view payment history, retrieve/reprint receipt and view daily collection | Explicit billing permissions plus tenant/hospital/department scope; financial data is not shown in reception or clinical summaries |
| Hospital administrator | Navigate facility context and manage only capabilities already exposed by authorized APIs | Explicit provisioned permissions; administrator status must not imply clinical permissions |
| Auditor | Future controlled review of audit records | Not part of the first interactive workspace until an approved audit-query contract exists |

A user may hold multiple roles. The UI must render capabilities from the authenticated server-authorized context; hiding a control is not authorization. Every mutation must still be rejected safely by the API when permission or resource scope is missing.

## Information architecture

The authenticated application shell should provide:

- **Tenant and facility context:** visible active organization/hospital and, where supported, department context. Switching context must be explicit and must not silently carry a stale resource selection across facilities.
- **Primary navigation:** Reception, Queue, Clinical workspace and Billing entries only when the selected tenant membership has the effective permission identifiers for those capabilities.
- **Patient context:** a patient summary is shown only inside an authorized workflow and only with fields returned by the relevant API contract.
- **Session controls:** authenticated user identity and sign-out; do not display raw tokens or identity-provider claims.
- **System feedback:** clear loading, empty, validation, conflict, forbidden, not-found and service-unavailable states.

The initial release includes the minimal OPD billing workflow described below. Patient self-service, pharmacy, diagnostics, inpatient, workforce administration and a patient portal remain deferred.

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

- Show in-progress queue entries and completed visits visible to the authenticated ClinicalUser in the selected hospital/department; completed visits are available only to resume an existing encounter, never to create a new one.
- Starting an encounter must use the queue-linked encounter API; do not create an encounter directly from an appointment. Retrieve an existing encounter first; create only for an in-progress queue entry and recover a concurrent create conflict by reading the existing encounter.
- Display a clear distinction between queue status and encounter status.
- Show the approved note sections: chief complaint, history, examination, assessment and plan.
- Draft edits remain available only while the encounter is InProgress.
- Signing requires a deliberate review/confirmation step and communicates that signed note content becomes immutable through normal update operations.
- Amendments must be presented as append-only additions with reason and author/time attribution; never offer an edit-in-place experience for signed text.
- Closing/cancelling follows the API lifecycle and must not imply queue completion.
- Do not introduce diagnosis, vitals, prescriptions, orders, referrals, triage, clinical decision support or emergency workflows in this UX increment.

### 8. Billing, payment history and receipt recovery

- Show the billing workspace only when the selected membership has the relevant billing permissions. The API enforces each read and mutation independently.
- Load billable visits from the department queue and only allow invoice creation for the supported in-progress/completed visit states validated by the API.
- Retrieve an existing invoice by visit before offering invoice creation. Reopening a visit must resume the original invoice rather than presenting a second-issue action.
- Render immutable invoice line snapshots, subtotal, paid amount, balance due and the API settlement state (`Issued`, `PartiallyPaid`, `Paid`).
- Record only supported payment methods and references accepted by the API; never infer payment success from client state.
- Re-fetch invoice state and payment history after a successful payment. Allow prior receipt retrieval and print the selected API receipt record without printing unrelated workspace content.
- Keep service catalogue administration and daily collection reporting behind their distinct permissions. Do not display clinical notes or unnecessary patient demographics in the billable-visit selector.
- GST/tax, discounts/waivers, refunds/credit notes, insurance/TPA, gateway/accounting integration, deposits/advances and cashier shift reconciliation are not represented as available until separately designed and approved.

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
3. **Reception patient lookup and registration — implemented:** Web PR #2 uses the hospital-scoped patient search API with bounded pagination. Patient registration, duplicate-candidate review and hospital-local MRN assignment are merged in Web PR #3 with paired documentation in Doc PR #30. Search results remain in component memory, not browser storage; the API remains authoritative for patient permissions and hospital scope.
4. **Appointment and arrival — implemented:** booking, walk-in, check-in and queue-token confirmation are merged in Web PR #4 with paired documentation in Doc PR #31.
5. **Queue operations — implemented:** scoped queue view and permitted lifecycle actions, merged in CompileCare.Web PR #5 with paired documentation in Compilecare.Doc PR #32.
6. **Clinical workspace — implemented:** scoped clinical queue discovery, queue-linked encounter create/resume, draft note editing, signing, append-only amendments, cancellation and closure against the existing OPD API (CompileCare.Web PR #9). The clinical UI does not expose patient details in queue rows or represent unsupported vitals, diagnoses, prescriptions or diagnostic orders.
7. **Billing workspace — implemented:** department-scoped visit selection, invoice create/resume, partial/full payment capture, payment history, receipt retrieval/reprint, service catalogue and daily collection (CompileCare.Web PRs #10–#11). API permission and department boundaries remain authoritative.
8. **Browser acceptance — implemented for the walk-in path:** Playwright Chromium tests cover billing recovery after refresh and permission-denied navigation (Web PR #12), plus patient registration/MRN → walk-in token → queue call/start → clinical identity check and sign/close → queue completion → invoice and payment (Web PR #13). API workflow correctness remains covered by PostgreSQL-backed integration tests.
9. **Appointment and check-in browser acceptance — implemented:** Web PR #14 verifies hospital patient selection, API-published availability and slot selection, booking confirmation and check-in with a server-issued queue token.
10. **OIDC session renewal browser acceptance — implemented:** Web PR #15 verifies the short-lived synthetic OIDC session obtains a refreshed access token through the refresh-token grant and uses it on subsequent API requests. No real identity provider, credentials or patient data are used.
11. **Next browser acceptance:** cover availability conflict/slot contention recovery and renewal failure/re-authentication behavior. **Hardening:** keyboard/accessibility review, responsive checks, authorization-state tests, API contract tests and Docker-based verification where supported by the repository.

Each increment should be a small vertical slice with tests and documentation updates. Do not implement all screens against mocked contracts and then retrofit API behavior.

## Acceptance criteria for the UX foundation

- Reception, queue and clinical tasks are mapped to existing backend workflows and APIs.
- Every screen has an explicit persona, purpose, required permission/scope, data source and error/empty/forbidden state before implementation.
- The public site and authenticated hospital application remain separate.
- Patient registration and appointment booking preserve the existing hospital-registration prerequisite.
- Queue actions and encounter lifecycle are presented as separate workflows.
- Signed clinical note content is never editable in place; amendments are append-only.
- The delivered invoice/payment/receipt capabilities are permission-gated and use API-generated amounts, settlement state and receipt records. Deferred tax, discounts/waivers, refunds/credit notes, insurance/TPA, external payment gateways, accounting, deposits/advances and cashier shift reconciliation are not represented as available.
- No unimplemented contact capture, vitals, diagnoses, prescriptions, diagnostic orders or patient-facing workflows are represented as available.
- Accessibility, keyboard operation, responsive behavior and data-minimization requirements are included in implementation tests/review.
- Server-side authorization remains authoritative and is exercised for both allowed and denied UI/API paths.
- Build, automated tests and Docker-based verification pass before a UI increment is merged.

## Open product and design questions

- Validate appointment availability conflict and rescheduling/check-in edge cases with the pilot workflow; add them to the Playwright suite without replacing the PostgreSQL API tests.
- Confirm the approved silent-renew failure, access-token expiry and workstation re-authentication experience with hospital IT/security.
- Validate the staff/provider administration workflow with hospital stakeholders before building the workforce UI. Profile-to-account linking, credentialing and clinical privileging remain separate decisions.

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


## Queue operations UI increment — 2026-11-24

The authenticated reception workspace now has a department/date queue view. The next vertical slice adds operator controls to each queue row, using the existing API route POST /api/v1/hospitals/{hospitalId}/queue/{queueEntryId}/{action} and the queue.manage policy. The UI must not attempt a transition unless the server response succeeds; after success, reconcile the affected row from the returned DTO.

| Current state | Available action(s) | Result |
|---|---|---|
| Waiting | Call, Cancel, No-show | Called, Cancelled, NoShow |
| Called | Recall, Start consultation, Cancel, No-show | Waiting, InProgress, Cancelled, NoShow |
| InProgress | Complete | Completed |
| Terminal | No ordinary action | Remains closed |

Cancellation and no-show require an explicit 1–250 character reason accepted by the API. No-show is an explicit staff decision, not an automatic timeout. Call/recall/start/complete do not collect a reason. The UI exposes only the returned queue token and operational status; it does not display patient name, MRN or clinical information. Server-side authorization and hospital/department scope remain authoritative for every action. Conflicts and forbidden responses are surfaced without pretending the action succeeded.

The operator controls are responsive and keyboard operable, with status text in addition to color, visible focus, disabled in-flight controls and an accessible error announcement. Manual viewport/accessibility review remains a release gate.

### Visual evidence and screenshot policy

Production UI screenshots must be captured from the actual running authenticated web application, not generated mockups. Use a staging OIDC authority and API with synthetic patient/queue data only. Capture desktop, tablet and narrow-screen views of reception context, patient lookup/registration, appointment booking, walk-in/check-in, queue operations and the clinical encounter workspace. Never capture real patient data, access tokens, browser storage, or private identity-provider details. Keep the screenshot source environment and commit/build SHA in the accompanying evidence note. Until a staging identity/API configuration and browser capture environment are available, screenshot evidence is marked pending rather than substituted with fabricated images.


## OPD clinical workspace UI increment — 2026-11-24

The clinical UI consumes the existing OPD encounter API; it does not create a second clinical data model. The paired API increment adds a queue-entry encounter lookup for safe resume after refresh and grants ClinicalUser `patient.identity.read` for linked-patient identity verification without general patient-search access, plus scoped `facility.read`, `queue.read` and the existing encounter permissions. Queue mutations remain unavailable to ClinicalUser.

The first UI slice includes:
- A department/date-filtered clinical queue that shows only operational queue tokens and in-progress state.
- Open/resume behavior: look up the existing encounter first, create only when none exists, and recover a concurrent create conflict by retrieving the newly created encounter.
- Explicit draft save for the five approved note sections with API-aligned character limits.
- Require explicit confirmation that the displayed patient identity has been checked before showing or editing note content. The review-and-sign action saves the current draft before requesting the server-side signature; chief complaint and assessment are required by the API.
- Read-only signed/closed note presentation, append-only amendments with required reason/content, unsigned cancellation with required reason, and signed encounter closure.
- Server-returned state reconciliation only; permission, validation and concurrency errors remain visible.
- Queue rows remain token-only; the encounter editor retrieves and displays only the patient name and date of birth returned by the hospital-scoped patient read contract. No MRN, contact data or note text is placed in queue rows, URLs or general telemetry. The encounter workspace may show the minimal patient identity returned by the authorized hospital-scoped patient read API.

The UI deliberately does not implement coded vitals, diagnoses, medication reconciliation, prescriptions, laboratory/radiology orders, referrals, triage or clinical decision support. Those remain gated on pilot clinician validation and separate API/product decisions. Encounter signing is not queue completion; queue and clinical lifecycle remain independent.

Release gates include Web API contract tests, Vitest suite, production TypeScript build, CI Docker image build and HTTP smoke test, plus manual keyboard and desktop/tablet/narrow viewport review. Authenticated staging screenshots remain pending until the staging OIDC/API and secure browser capture environment are available.