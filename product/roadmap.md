# Product Roadmap — Proposed

**Status:** Draft; capability milestones, not calendar commitments.

## M0 — Foundation
Tenant/hospital hierarchy; identity/users/roles/permissions; audit; configuration; API versioning/OpenAPI/errors; CI/tests; Docker environment; observability; basic admin UI.
**Exit:** tenant provisioning and sign-in work; cross-tenant isolation tests pass.

## M1 — Patient and appointment journey
Registration/MRN; search/duplicate detection; appointments; queue/token; communication hooks.
**Exit:** patient can register, book, check in, and progress through queue states.

## M2 — OPD and clinical foundation
Encounter lifecycle; vitals, notes, diagnosis, allergies/history; templates; orders/prescriptions; doctor workflow.
**Exit:** complete OPD encounter documented/reviewed with permissions and audit.

## M3 — Billing and payment
Services/tariffs; OPD billing; payment/receipt/discount/refund; daily reconciliation.
**Exit:** tested billing scenarios reconcile.

## M4 — Pharmacy
Medicine catalogue; dispensing; stock/batches/expiry; returns and inventory transactions.

## M5 — Laboratory
Catalogue/orders; sample workflow; result entry/validation; reports.

## M6 — IPD
Admission/transfer/discharge; beds/wards; nursing; observations/care plans; medication administration; discharge summary.

## M7 — Integrations and expansion
ABDM/ABHA subject to approval; FHIR/HL7; DICOM/PACS; insurance/TPA; procurement/inventory; analytics; patient portal/mobile; hospital operations.

## Release gates
Approved requirements, validated workflow, design, implementation, automated tests, security review, docs, migration/rollback plan, demo/acceptance.
