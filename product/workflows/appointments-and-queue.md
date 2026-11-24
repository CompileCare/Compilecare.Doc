# Workflow: Appointments, walk-ins, check-in and queue

**Status:** Proposed workflow for pilot validation  
**Related decision:** [ADR-014 — Appointment scheduling and queue boundaries](../../architecture/decisions/ADR-014-appointment-scheduling-and-queue-boundaries.md)

## Purpose

Coordinate outpatient visits from availability publication through booking or walk-in arrival, check-in and department queue completion. Appointment scheduling, arrival/queueing and clinical encounters remain separate records and responsibilities.

## Actors

- Hospital administrator / authorized scheduler
- Reception or registration staff
- Patient
- Provider (doctor/consultant)
- Authorized queue operator
- Authorized auditor

## Preconditions

- Tenant is active and the selected hospital and department exist.
- Acting user has a validated OIDC identity, active tenant membership and the specific permission for the requested action.
- Patient has a tenant-owned Patient ID and a valid registration at the selected hospital. If the patient is new to the tenant, an authorized user must first create the patient master and then the hospital registration. If the patient already exists in the tenant but is not registered at the selected hospital, an authorized user must verify identity through the patient-registration workflow and create that hospital association before booking. Demographic similarity alone must never create or establish an association.
- The hospital has an IANA timezone configured.
- A provider has an active staff profile, provider designation and effective assignment to the selected hospital/department for the scheduled time. This is not credential verification or a clinical privilege grant.

## A. Publish provider availability

1. Authorized scheduler selects hospital, department and provider.
2. Scheduler creates a date-specific availability session with local start/end and slot duration.
3. API validates provider profile and effective assignment for the whole session, department/hospital relationship, timezone conversion, positive duration and non-overlap with existing sessions.
4. API stores UTC instants and generates discrete slots from the session.
5. Scheduler reviews the published slots. Changes to a session with active bookings require an explicit audited workflow; silently invalidating existing bookings is prohibited.

Recurring schedules, roster/leave integration and external calendar synchronization are deferred.

## B. Book an appointment

1. Reception searches/selects the patient through the selected hospital's patient-registration workflow.
2. If no patient master exists in the tenant, authorized registration staff create the patient master and the corresponding registration at the selected hospital before returning to booking.
3. If a patient master exists in the tenant but has no registration at the selected hospital, authorized registration staff verify the patient's identity and explicitly create the hospital registration before booking. A demographic match or duplicate-candidate result is only a review aid; it must never automatically associate the patient with a hospital or another patient record.
4. Appointment booking is enabled only after the selected hospital registration is successfully created and active. The booking API independently validates that registration; it never creates appointment-only patient records or bypasses registration.
5. Reception selects department, provider, date and an available slot.
6. API verifies the patient has an active hospital registration for that hospital and that the provider remains assigned to the department at the appointment time.
7. API checks that the slot is available and that the patient has no overlapping active appointment in the tenant.
8. API reserves the slot and creates a Booked appointment in one transaction. Concurrent attempts for the same slot cannot both succeed.
9. Confirmation returns appointment ID, scheduled time in hospital timezone, department and provider. The appointment reason is optional operational text; clinical notes are not captured here.

The pilot uses staff-assisted registration and booking only. Self-service registration and online booking for new patients are separate future workflows. A no-slot response does not automatically create a waitlist entry. Staff may select another available slot or use the walk-in workflow if permitted by the hospital.

## C. Cancel or reschedule

- Only users granted the appointment.manage capability within the appointment's tenant/hospital/department scope may cancel or reschedule a Booked appointment; every action records the actor and an approved reason code. The initial pilot is staff-assisted; patient self-service is deferred. The exact staff-role mapping and any role-specific cancellation restrictions must be approved by the hospital before production.
- A successful cancellation releases the provider slot in the same transaction, making it bookable again immediately unless the availability session itself has been withdrawn or otherwise made unavailable. A failed/rolled-back cancellation must not release the slot.
- Rescheduling creates a new appointment and cancels the original in one transaction. Both records retain explicit links and audit history; the new slot must be successfully reserved before the old booking is finalized as rescheduled.
- A checked-in appointment cannot be silently cancelled or rescheduled; front desk must resolve its queue entry through the appropriate workflow.
- Completed, cancelled or no-show appointments cannot be rebooked by editing the old record.
- Patient self-service cancellation/rescheduling is not part of this increment.

## D. Check in a booked appointment

1. Patient arrives at the hospital.
2. Reception verifies the patient and appointment using the selected hospital registration.
3. API checks appointment status, hospital/department scope and the hospital-configured early/late check-in window. The provisional pilot window is 60 minutes before through 30 minutes after the scheduled start; the values are configurable and must be confirmed by the hospital.
4. API records check-in actor/time and creates one queue entry transactionally.
5. Repeated requests return the same queue entry; duplicate tokens are not created.
6. Patient receives the queue token and waiting instructions.

Check-in does not change appointment state to a separate CheckedIn status. The appointment remains Booked until completed, cancelled or marked no-show; check-in is represented independently.

## E. Register a walk-in

1. Reception selects an existing patient registration at the hospital, or completes patient registration first.
2. Reception selects the destination department and, if known, the provider.
3. API creates a queue entry directly without creating an appointment or inventing an appointment slot.
4. Queue token is allocated for the hospital, department and hospital-local service date.
5. Patient receives the token and waiting instructions.

A walk-in must not bypass patient registration, tenant authorization or hospital scope. Emergency arrivals follow the hospital emergency pathway, not the ordinary FIFO queue.

## F. Operate the queue

Initial queue order is FIFO by check-in timestamp within the hospital/department/service date. Queue operators can:

- View the queue within authorized hospital/department scope.
- Call the next waiting entry.
- Recall an entry by returning Called to Waiting.
- Start consultation by moving Called to InProgress.
- Complete an entry after the operational hand-off is finished.
- Cancel a waiting/called entry or mark it NoShow with an approved reason. For appointment no-shows, an authorized front-desk user explicitly marks the appointment NoShow after the hospital-configured grace period has elapsed and confirms the patient has not arrived; the initial pilot does not automatically infer or mark no-shows.

Allowed transitions:

| Current state | Action | Next state |
|---|---|---|
| Waiting | Call | Called |
| Called | Recall | Waiting |
| Called | Start consultation | InProgress |
| InProgress | Complete | Completed |
| Waiting / Called | Cancel | Cancelled |
| Waiting / Called | Mark no-show | NoShow |

Terminal states cannot be reopened through ordinary queue actions. Any correction requires an explicit privileged and audited correction operation. The initial queue does not support acuity scoring, emergency prioritization, manual reorder or automated triage.

When an appointment-linked queue entry is completed, the appointment may be marked Completed through the same coordinated application operation. Queue completion does not create an OPD encounter automatically; encounter creation belongs to the Clinical/OPD workflow.

## Appointment and queue lifecycles

**Appointment:** Booked → Cancelled | Completed | NoShow. Only appointment.manage users with matching facility scope may cancel/reschedule; role mapping and reason policy require hospital approval. A successful cancellation releases its slot transactionally. No-show is an explicit, audited staff action after the configured grace period in the initial pilot, not an automatic demographic, clock-only or background inference. Rescheduling cancels the old appointment and creates a new linked appointment. Check-in is a separate timestamp and queue-entry reference.

**Queue entry:** Waiting → Called → InProgress → Completed, with Called → Waiting for recall, and Waiting/Called → Cancelled | NoShow.

Queue token is an operational daily identifier, not a patient identifier or MRN.

## Authorization and audit

- availability.manage, appointment.read, appointment.manage, appointment.checkin, queue.read and queue.manage are separate capabilities.
- Hospital and department scope is validated server-side for every request and referenced resource.
- Role mapping is a separate reviewed step; do not assume every HospitalAdmin, FacilityManager or PatientRegistrar automatically receives all permissions.
- Audit availability publication/change, booking, cancellation, rescheduling, check-in, queue transition, no-show and privileged correction.
- Audit events include actor, tenant, hospital/department, target ID, action, outcome, timestamp and correlation ID. Do not duplicate patient name, contact details, reason text or clinical data in audit metadata.
- Public waiting-room displays use queue token and minimum operational status only; no patient name, MRN or clinical information.

## Failure and concurrency paths

- **Slot race:** only one concurrent booking can reserve a provider slot; losing request receives a safe conflict response and must refresh availability.
- **Patient overlap:** overlapping active appointments for the same tenant patient are rejected; staff must resolve the conflict explicitly.
- **Provider assignment ended/deactivated:** new booking is rejected; existing appointments remain visible for authorized rescheduling/cancellation and require a reviewed continuity process.
- **Session change with bookings:** reject destructive edits or require explicit per-appointment resolution; never orphan bookings.
- **Duplicate check-in:** idempotently return the existing queue entry.
- **Invalid appointment state:** reject check-in/cancel/reschedule transitions that are not allowed.
- **Token allocation race:** allocate a unique token transactionally per hospital/department/local date.
- **Timezone ambiguity:** reject ambiguous/nonexistent local times and require explicit offset resolution.
- **Tenant or facility mismatch:** reject cross-tenant patient, registration, provider, session or queue references.
- **Service outage/retry:** make booking, reschedule and check-in idempotent or concurrency-safe; never create duplicate reservations or queue entries.

## Acceptance criteria

- Appointment can only be booked after a valid patient registration exists at the selected hospital. New-to-tenant patients require patient-master creation plus hospital registration first; existing tenant patients without that hospital association require authorized identity verification and explicit registration first. Demographic matching never auto-associates patients, and booking never creates appointment-only patient records or bypasses registration.
- Initial pilot registration and booking are staff-assisted; self-service registration and online booking for new patients are deferred.
- Availability session and appointment are tenant-owned and scoped to hospital/department.
- Provider has an effective assignment to the selected hospital/department at appointment time.
- Provider designation is operational metadata and does not imply credential verification or clinical authority.
- Availability sessions are date-specific and generate deterministic slots; recurring templates are deferred.
- Concurrent requests cannot double-book a provider slot.
- Overlapping active appointments for the same tenant patient are rejected.
- Appointment rescheduling preserves old/new links and is atomic.
- Cancellation/rescheduling is restricted to appointment.manage users with tenant/hospital/department scope; approved role mapping and reason rules are validated with the pilot hospital.
- A successful cancellation releases the provider slot transactionally; failed cancellation does not release it.
- Check-in enforces hospital-configured opening/closing windows, with the provisional 60-minute-early/30-minute-late defaults validated by the hospital.
- Appointment no-show is explicitly marked by authorized staff after a configured grace period in the pilot; automatic no-show detection is deferred.
- An appointment can have at most one queue entry; repeated/concurrent check-in returns that same entry and cannot create another token.
- Cancelled, completed and no-show appointments cannot be checked in.
- Check-in creates exactly one queue entry; retries return the existing entry.
- Walk-in creates a queue entry without creating an appointment.
- Queue token is unique within hospital/department/local date and is distinct from MRN.
- Queue transitions are enforced server-side and audited.
- Queue ordering is FIFO; no acuity score or clinical priority is inferred.
- Queue display responses expose no patient name, MRN, contact or clinical details.
- Timezone conversion uses hospital IANA timezone and UTC persistence.
- Tenant/hospital/department authorization is tested for reads and mutations.
- PostgreSQL integration tests cover booking concurrency, patient overlap, reschedule, check-in idempotency, token allocation and queue transitions.
- Docker-based build and tests pass before implementation PR is considered ready.
- Pilot hospital stakeholders validate visit types, durations, check-in opening/closing windows, no-show grace period and staff identification procedure, cancellation/rescheduling permissions and reason rules, role mapping, and queue operating rules before production use.

## Deferred

Patient self-booking, recurring rosters, leave integration, waitlists, service catalogue/tariffs, payments, reminders, external calendar sync, queue prioritization, triage, emergency workflow, clinical encounters and cross-tenant patient discovery.
