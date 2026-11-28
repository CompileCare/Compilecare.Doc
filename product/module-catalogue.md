# Hospital Module Catalogue

**Status:** Draft inventory; workflows and boundaries require validation.

## Platform foundation
Tenant/customer management; hospital/branch/department/unit hierarchy; identity/authentication; users/roles/permissions; audit; configuration; feature flags; notifications; documents.

## Patient services
Registration, demographics, contact details, UHID/MRN, search, duplicate detection/merge governance, appointments, rescheduling/cancellation, queue/token management, waiting-room display, patient portal/mobile (later).

## Appointments and front desk
Date-specific provider availability sessions; appointment booking, cancellation and rescheduling; patient arrival and check-in; walk-in registration; department FIFO queue and token lifecycle. Scheduling reserves a provider time; queue records operational arrival/work; neither is a clinical encounter. Patient self-booking, recurring rosters, waitlists, service catalogue/tariffs and clinical prioritization are later capabilities.

## Clinical care
OPD encounters linked to front-desk queue entries; immutable signed clinical notes with append-only amendments; approved vital observations; diagnosis, allergy/history, care plans, referrals, prescription/medication orders and templates as separately governed later capabilities. The first encounter increment must not infer diagnoses or recommend treatment.

## Inpatient
Admission/discharge/transfer; ward/room/bed; occupancy; nursing assessments, care plans, observations, shift handover; medication administration record; discharge summary and follow-up.

## Diagnostics
**Laboratory (LIS):** test catalogue/panels, orders, collection, labels/barcodes, tracking, result entry/validation, critical-result workflow, reports.
**Radiology (RIS/PACS):** imaging orders/scheduling, modality workflow, reports, DICOM/PACS integration and image access.

## Pharmacy
Medicine catalogue, prescription validation/dispensing, stock/batches/expiry, purchase, returns, inventory movements/reconciliation.

## Revenue cycle
Service/tariff catalogue, OPD/IPD/procedure billing, packages, discounts, deposits, payments, receipts, refunds, reconciliation, insurance/TPA/preauthorization/claims (later).

## Supply chain/administration
Inventory/stores, requisitions/POs/vendors, assets/maintenance, HR/roster/attendance/payroll integration (later), housekeeping, dietary, ambulance, facilities (later/configurable).

## Analytics
Operational dashboards; OPD/IPD census; occupancy; wait time; revenue/collection; pharmacy/lab reporting; role-based exports and audit.

## Integrations
ABDM/ABHA subject to approved requirements; FHIR/HL7; DICOM; payment providers/UPI; SMS/email/WhatsApp; insurance/TPA; accounting/external systems.

## Proposed delivery order
Foundation → Patient + Staff/Provider → Appointments + Walk-ins + Check-in/Queue → OPD/Clinical → Billing (if pilot requires) → Pharmacy → Lab → IPD → integrations and advanced operations.


## Deferred hospital modules and capability boundaries (2026-11-26)

The following modules are retained in the HeapCare product plan. They are not part of the initial OPD MVP merely because their high-level capability appears in the catalogue.

| Module | Bounded responsibilities | Planned capability increments | Status |
|---|---|---|---|
| Inpatient (IPD) | Admission lifecycle, bed occupancy, ward and nursing operations, discharge | ADT, reservation/assignment, transfer, nursing observations, care plans, handover, medication administration, discharge | Deferred |
| Pharmacy | Formulary, prescription validation, dispensing and pharmacy stock | Catalogue, dispense/return, batch/expiry, stock movements, supplier and purchase workflows | Deferred |
| Laboratory (LIS) | Diagnostic orders, specimens, laboratory worklists and validated results | Test catalogue, barcode/sample tracking, processing, result approval, critical-result escalation | Deferred |
| Radiology (RIS/PACS) | Imaging orders, scheduling, modality workflow, reports and image references | Worklists, DICOM integration, PACS storage/access, report approval and retention | Deferred |
| Inventory and procurement | Stores, vendors, purchasing, stock and assets | Requisitions, purchase orders, receiving, transfers, adjustments, expiry, assets and maintenance | Deferred |
| Advanced revenue cycle | Financial rules and reconciliation across OPD/IPD/services | Tax/statutory invoices, deposits, discounts, refunds, credit notes, insurance/TPA, claims, gateways and accounting integration | Deferred |
| Workforce and HR | Staff identity, credentialing, privileges and workforce operations | Verified membership linking, licence renewal, rosters, attendance, leave, employee records and payroll integrations | Foundation exists; extensions deferred |
| Patient engagement | Patient/guardian self-service and communication | Portal/mobile, appointment self-booking, approved records/instructions, reminders and consent preferences | Deferred |
| Interoperability | External healthcare and enterprise contracts | ABDM/ABHA, FHIR/HL7, DICOM, identity, payment, messaging and hospital integrations | Validate per pilot |
| Analytics and intelligent assistance | Permission-controlled operational/financial insight and governed assistance | Reporting/export, capacity/wait-time analysis and human-reviewed AI-assisted workflows | Deferred |

### Cross-module constraints
- Modules own their domain rules, data writes, authorization checks and audit events. Do not allow direct writes into another module's tables.
- Use explicit application contracts and versioned integration events where justified. Do not introduce a broker or microservice solely because a module is listed here.
- Patient, clinical and financial data must remain tenant- and organization-scoped with least-privilege access.
- Diagnostics, prescriptions, medication administration and clinical privileges require qualified hospital governance.
- Pharmacy and inventory require batch, expiry, stock movement and reconciliation invariants.
- Financial extensions require approved customer policies and qualified statutory/accounting review.
- External integrations require validated use cases, partner agreements, privacy/security assessment and conformance testing.
- AI-assisted capabilities must preserve human accountability and must not autonomously diagnose, triage or prescribe.

See [Product Blueprint — delivery roadmap](blueprint-v1.md#14-delivery-roadmap--implemented-foundation-mvp-completion-and-future-scope) for sequencing and entry dependencies.
