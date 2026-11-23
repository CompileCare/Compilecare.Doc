# Hospital Module Catalogue

**Status:** Draft inventory; workflows and boundaries require validation.

## Platform foundation
Tenant/customer management; hospital/branch/department/unit hierarchy; identity/authentication; users/roles/permissions; audit; configuration; feature flags; notifications; documents.

## Patient services
Registration, demographics, contact details, UHID/MRN, search, duplicate detection/merge governance, appointments, rescheduling/cancellation, queue/token management, waiting-room display, patient portal/mobile (later).

## Clinical care
OPD encounters, vitals, notes, diagnosis, allergies/history, care plans, referrals, prescription/medication orders, templates.

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
Foundation → Patient + Appointments → OPD/Clinical → Billing → Pharmacy → Lab → IPD → integrations and advanced operations.
