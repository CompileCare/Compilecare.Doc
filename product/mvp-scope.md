# MVP Scope and Non-Goals

**Status:** Draft proposal. Scope must be validated with hospital stakeholders before it becomes an approved delivery commitment.

## MVP objective
Deliver a secure, tenant-aware foundation and one end-to-end outpatient journey that demonstrates the platform's core value and operational readiness.

## In scope — proposed
1. Tenant provisioning and tenant configuration.
2. Hospital hierarchy: tenant, optional group, hospital, branch/campus, department, and unit.
3. User identity, roles, permissions, and organizational scope.
4. Audit logging for security-sensitive and key business actions.
5. Patient registration with demographic data and configurable identifier/MRN rules.
6. Patient search and duplicate-patient detection support.
7. Date-specific provider availability and appointment booking, cancellation and rescheduling.
8. Patient check-in, walk-in registration and department FIFO queue/token lifecycle. Appointment-linked check-in and walk-ins converge on the same queue workflow; a walk-in does not create a fictitious appointment.
9. Basic OPD encounter workflow with clinician notes and a minimal set of structured clinical fields, subject to workflow validation.
10. Basic service catalogue and billing/receipt workflow only if required to complete the pilot journey; otherwise defer to the next milestone.
11. Admin screens for tenant/hospital setup and essential user management.
12. API contracts, automated tests, CI, local development environment, logging, metrics, tracing, and deployment documentation.

## Explicit non-goals for the first MVP
- Full ERP/HR/payroll.
- Advanced financial accounting and multi-company consolidation.
- Full pharmacy inventory, procurement, and supply chain.
- Full laboratory information system and instrument integrations.
- IPD bed management, nursing, medication administration, and complex discharge.
- PACS/DICOM imaging workflows.
- Insurance/TPA claims automation.
- Advanced analytics/data warehouse.
- Patient mobile app and broad patient portal capabilities.
- AI-assisted diagnosis, treatment recommendations, or autonomous clinical decisions.
- ABDM/ABHA or other regulated integrations until requirements, approvals, and implementation responsibilities are validated.
- Multi-region active-active architecture or service extraction without an evidenced need.

## MVP success criteria — proposed, measurable targets to be agreed
- Tenant provisioning and sign-in can be completed through documented flows.
- Automated tests demonstrate tenant isolation for relevant data-access and API scenarios.
- A patient with a valid hospital registration can be booked into a provider slot or registered as a walk-in, checked in, queued and handed off to a separately recorded OPD encounter.
- Key actions are permission-checked and audit-recorded.
- Critical journey tests run in CI.
- A documented deployment and recovery process exists for the pilot environment.
- Stakeholder acceptance is recorded against the workflow and requirements.

## Scope control
New capabilities should be added only after identifying the user/problem, workflow, acceptance criteria, dependencies, and operational/security implications. Non-MVP requests should be tracked for later roadmap decisions rather than silently added to MVP.
