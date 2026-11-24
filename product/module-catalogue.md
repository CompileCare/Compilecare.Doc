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
