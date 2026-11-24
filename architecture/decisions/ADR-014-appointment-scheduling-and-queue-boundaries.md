# ADR-014: Appointment scheduling and queue boundaries

- **Status:** Proposed for implementation
- **Date:** 2026-11-24
- **Decision owners:** Product and engineering; hospital operations, reception and clinical stakeholders must validate the workflow before production use

## Context

CompileCare has tenant-owned patient masters, hospital registrations, a hospital facility hierarchy and tenant-owned staff/provider profiles with effective-dated hospital/department assignments. The first outpatient journey needs to schedule visits, support walk-ins, check patients in and coordinate a department queue.

These are related but distinct workflows. An appointment reserves a future time with a provider. Check-in records arrival. A queue entry represents current operational work. An OPD encounter records clinical care. Combining them into one status-heavy record would make cancellations, walk-ins, rescheduling and clinical hand-off difficult to reason about.

## Decision

1. **Keep scheduling, front-desk queueing and clinical encounters as separate logical capabilities** within the existing CompileCare.Api modular monolith. Scheduling owns availability sessions and appointments. Front desk/queue owns check-in and queue entries. Clinical/OPD owns encounters and clinical documentation.
2. **Make hospital registration a hard booking prerequisite.** Use the tenant-owned Patient ID as the patient reference, and require a valid registration for the same tenant and selected hospital. If no patient master exists in the tenant, authorized staff must create the patient master and then the hospital registration. If the patient master exists in the tenant but is not registered at the selected hospital, authorized staff must verify identity through the patient-registration workflow and explicitly create the hospital association. Demographic similarity or duplicate-candidate matching must never automatically associate patients. Booking must not create appointment-only patient records or bypass registration. Initial pilot registration and booking are staff-assisted; self-service registration and online booking for new patients are separate future workflows. Patient master and hospital registration remain owned by Patient Management.
3. **Scope each appointment to one tenant, hospital and department.** The provider is required for the first scheduled-appointment increment and must be an active staff profile with a provider designation and an effective assignment to that hospital/department at the appointment time. This is an operational scheduling check only; designation does not establish verified credentials or clinical authority.
4. **Start with explicit, date-specific provider availability sessions.** An authorized scheduler publishes a session with start/end, slot duration and provider. The system derives discrete bookable slots; a provider slot can have only one active appointment. Recurring schedule templates, leave integration, multi-provider/resource booking and external calendar sync are later capabilities.
5. **Store instants in UTC and interpret schedules in the hospital's configured IANA time zone.** The API accepts offset-aware timestamps; display and queue-day boundaries use the hospital timezone. Reject ambiguous/nonexistent local times and require a configured timezone before enabling scheduling. Do not assume the server timezone.
6. **Make booking concurrency-safe.** Reserve a slot transactionally with database-enforced uniqueness/locking so simultaneous requests cannot double-book a provider slot. Prevent overlapping active appointments for the same patient within the tenant by default; return a conflict that staff must resolve rather than silently moving either appointment.
7. **Keep appointment lifecycle separate from arrival.** Appointment states are Booked, Cancelled, Completed and NoShow. Check-in is a timestamp/actor plus a link to one queue entry, not a new appointment state. Enforce at most one queue entry per appointment; repeated or concurrent check-in returns the existing entry and never creates another token. Rescheduling atomically reserves a new booking and cancels the old booking, preserving explicit RescheduledFrom/RescheduledTo links and audit history.
8. **Support walk-ins without fabricating appointments.** A walk-in creates a queue entry directly for a patient with a valid hospital registration. It has no appointment reference. A checked-in appointment creates exactly one queue entry. Both paths converge on the same queue workflow.
9. **Use a simple FIFO queue for the first increment.** Queue entries are scoped to tenant, hospital, department and hospital-local service date. Queue tokens are daily, department-scoped operational identifiers, separate from MRN. Initial ordering is check-in time; manual priority changes, clinical triage ranking, emergency routing and multi-queue orchestration require separate clinical/operational approval.
10. **Use explicit queue transitions and explicit no-show handling.** Waiting → Called → InProgress → Completed; a called entry may return to Waiting for recall. Authorized staff may cancel a waiting/called entry or mark it NoShow with a reason. For booked appointments, the initial pilot identifies no-shows through an explicit, audited staff action after a hospital-configured grace period and confirmation that the patient has not arrived; do not automatically infer no-show from elapsed time alone. Terminal states cannot be silently reopened; correction requires an audited action.
11. **Make check-in idempotent and windowed.** Repeated check-in for the same appointment returns the existing queue entry; a database constraint and transactional operation enforce one queue entry per appointment. A cancelled, completed or no-show appointment cannot be checked in again. Hospital-configured opening/closing windows govern appointment check-in; the provisional pilot default is 60 minutes early and 30 minutes late, subject to validation.
12. **Enforce least privilege and audit.** Availability management, appointment read/manage, check-in and queue operations are distinct permissions. Only appointment.manage users with matching tenant/hospital/department scope may cancel or reschedule; exact role mapping, cancellation restrictions and reason policy require pilot-hospital approval. A successful cancellation releases its slot transactionally; failed cancellation does not. Every request validates tenant, hospital and department scope on the server. Audit booking, rescheduling, cancellation, check-in, queue transitions, no-show and schedule changes without copying patient names, reasons or clinical data into general telemetry.
13. **Do not turn scheduling into a clinical record.** Appointment reason is optional, short, operational text only; diagnoses, symptoms, triage findings, notes, orders and clinical priority belong to clinical workflows. Emergency patients must follow the hospital's emergency pathway and must not be delayed by ordinary appointment or FIFO queue rules.

## Initial implementation scope

- Publish and list date-specific provider availability sessions.
- Generate deterministic slots from each session.
- Book, retrieve, list, cancel and reschedule appointments.
- Validate patient registration, provider assignment, hospital/department and slot availability.
- Check in a booked appointment and create a queue entry.
- Register a walk-in and create a queue entry without an appointment.
- View department queue and perform the approved queue transitions.
- Enforce bounded pagination, tenant isolation, organization scope, audit and PostgreSQL concurrency constraints.

## Explicitly out of scope

- Patient-facing self-booking or portal/mobile experience.
- Recurring provider schedules, leave/attendance/roster integration and cross-facility calendars.
- Appointment types/service catalogue, tariffs, billing and payment collection.
- Waitlists, group/family bookings, multi-provider or multi-resource appointments.
- External calendar/SMS/WhatsApp integrations and automated reminders.
- Clinical triage, acuity scoring, emergency-department flow and priority queueing.
- OPD encounter, diagnosis, prescriptions, orders and clinical documentation.
- Cross-tenant patient search or patient identity linking.

## Recommended pilot defaults — validate with hospital users

- Appointment requires an existing patient registration at the selected hospital and a provider-specific slot; authorized staff complete patient-master creation and/or explicit hospital registration before booking, and demographic matches never auto-associate patients.
- Cancellation/rescheduling requires appointment.manage plus matching facility scope; a successful cancellation releases the slot transactionally.
- Check-in uses hospital-configured opening/closing windows (provisional 60 minutes early / 30 minutes late).
- No-show is explicitly marked by authorized staff after a configured grace period in the pilot; automatic no-show detection is deferred.
- At most one queue entry exists per appointment; repeat check-in returns the same entry.
- Availability is published as explicit date-specific sessions; no recurring schedule engine in v1.
- Slot capacity is one for a provider.
- Same-patient overlapping active appointments in the tenant are rejected.
- Department queue is FIFO by check-in time; queue token resets by hospital-local date and department.
- Appointment check-in window defaults to 60 minutes before through 30 minutes after the scheduled start, configurable by hospital.
- Only HospitalAdmin can manage availability initially; HospitalAdmin and explicitly assigned reception/registration roles receive appointment and check-in permissions after role mapping is approved.
- Queue actions are limited to authorized front-desk roles; clinical priority and manual reordering are deferred.
- Do not require phone/email contact data to book or check in.

These are implementation defaults for the first slice, not final hospital policy.

## Security, privacy and audit

- All references must be tenant-consistent: patient, hospital registration, hospital, department, provider, availability session, appointment and queue entry.
- Provider assignment is revalidated for the scheduled time and the selected facility; an ended/deactivated profile or assignment cannot receive new bookings.
- Never expose appointment or queue data across tenant boundaries. Same-tenant membership alone does not grant all-hospital visibility.
- Queue display APIs should expose token and minimum operational information only; do not expose patient names or clinical details to public waiting-room displays.
- Audit actor, tenant, hospital/department, target identifier, action, outcome, timestamp, reason code where applicable and correlation ID. Avoid full request bodies and sensitive patient values.
- Do not log appointment reason, patient identifiers, names, diagnoses, access tokens or raw request payloads in traces, metrics, exceptions or health details.
- Retain cancelled/rescheduled appointments and queue history according to an approved retention policy; do not physically delete operational history through ordinary CRUD.

## Consequences

Scheduling can evolve independently from clinical encounters and queue operations while keeping a coherent patient journey. Explicit sessions and FIFO queueing reduce initial complexity and make concurrency rules testable. The design intentionally postpones recurring rosters, service catalogues, patient self-service and clinical prioritization until hospital requirements are validated.

## Open questions for pilot validation

- Which visit types and appointment durations are required, and who owns their configuration?
- Should some departments book against a service/resource rather than an individual provider?
- Who may publish or change provider availability, and how are schedule changes communicated?
- Which staff roles may cancel or reschedule, and are there role-specific cutoffs, approval requirements or reason codes?
- Should a successfully cancelled slot become immediately bookable again in all appointment states, or are any policy-based holds required?
- What are the hospital's check-in opening/closing windows, no-show grace period and staff confirmation procedure?
- Does the pilot require waitlists, walk-in capacity limits or manual queue reordering?
- Which reception/registration roles receive each appointment and queue permission?
- What hospital timezone and queue-token format should be configured?

## Related documents

- [Appointment, walk-in, check-in and queue workflow](../../product/workflows/appointments-and-queue.md)
- [Patient registration workflow](../../product/workflows/patient-registration.md)
- [Staff and provider management workflow](../../product/workflows/staff-and-provider-management.md)
- [ADR-013: Staff and provider profile boundaries](ADR-013-staff-and-provider-profile-boundaries.md)
- [ADR-006: Tenant, organization and patient data boundaries](ADR-006-tenant-organization-patient-boundaries.md)
- [Product blueprint](../../product/blueprint-v1.md)
- [Delivery roadmap](../../product/roadmap.md)
