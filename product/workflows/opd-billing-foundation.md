# OPD billing foundation

**Status:** Backend billing foundation and settlement lifecycle are merged; the permission-driven Web billing workspace is implemented and under CI/Docker verification. Pilot policy validation remains required before production use.

## Implementation status — 2026-11-24

The backend foundation is merged, including explicit `Issued`, `PartiallyPaid` and `Paid` settlement states (CompileCare.Api PR #39) and a permission-scoped lookup for an existing invoice by OPD visit (CompileCare.Api PR #40). The Web billing workspace is implemented in CompileCare.Web PR #10 and is being verified independently. The workspace covers billable visit selection, catalogue-based invoice issue, partial/full payment capture, receipt retrieval and daily collection reporting. CI and Docker verification passed for the API changes; the Web branch has passed its unit tests, production build and Docker smoke test. The deferred policy decisions below remain out of scope and must be validated with the pilot hospital before production activation.

## Purpose

Deliver a deliberately small, hospital-scoped OPD billing capability that can be used with the existing patient registration and queue workflows. The first release covers a service catalogue, visit-linked charge capture, immutable invoice snapshots, payment recording, receipts and a basic daily collection report.

This is an implementation baseline, not a claim of statutory tax-invoice compliance or a replacement for accounting software. The pilot hospital must validate invoice numbering, applicable tax treatment, record retention and operational controls before production activation.

## Included in the first implementation

1. **Service catalogue:** hospital-specific service code, display name, category, active flag and price in INR. Prices are effective for new invoices only; issued invoice lines retain a snapshot of the description, code, unit price and quantity.
2. **Visit-linked charge capture:** create an invoice for an existing in-progress or completed OPD queue visit, for the same hospital and patient. A visit may have at most one initial invoice in this baseline. The queue entry is the billing anchor; clinical note content is never copied into billing records.
3. **Invoice:** immutable issued invoice with a unique internal reference, issue timestamp, patient/visit identifiers, line snapshots, subtotal, amount paid and balance. No tax, discount or waiver is calculated in this baseline.
4. **Payments:** append-only payment records for cash, UPI, card or bank transfer. Supports multiple partial payments and exact settlement; rejects zero/negative amounts and overpayment. Payments cannot be edited or deleted through the API.
5. **Receipt:** a read-only receipt view derived from a recorded payment, with receipt reference, invoice reference, payment amount, method and timestamp.
6. **Daily collection report:** hospital-local business date, totals by payment method and total collected. The report is based on successfully recorded payments, not invoice issue totals.

## Money and invoice rules

- All monetary values are INR, represented as decimal with two fractional digits; no floating-point arithmetic.
- Service prices and invoice snapshots must be non-negative; payment amounts must be positive.
- Invoice totals are calculated server-side from line snapshots.
- Invoice issue and its lines are persisted atomically.
- Once issued, invoice header and lines are immutable. Corrections, credit notes, cancellation and refunds are deliberately not implemented until policy is confirmed.
- Payment recording is append-only. Outstanding balance is derived from invoice total minus recorded payments.
- A payment must not exceed the current outstanding balance. Concurrent payment attempts must be protected by a transaction/concurrency strategy so that the invoice cannot be overpaid.
- Internal invoice and receipt references are operational identifiers only until the pilot confirms statutory numbering and tax requirements.
- No patient clinical note, diagnosis, prescription, contact data or other unnecessary sensitive information is stored in billing rows.

## Invoice settlement lifecycle

The API exposes three invoice states:

- `Issued`: an invoice has a positive outstanding balance and no payments have been recorded.
- `PartiallyPaid`: one or more payments have been recorded, but a positive balance remains.
- `Paid`: the outstanding balance is exactly zero. A zero-total invoice is marked paid when issued.

The payment record, invoice status transition and invoice payment-sequence update are committed atomically. A payment larger than the outstanding balance or a further payment against a fully paid invoice is rejected with HTTP 409. The invoice status is a summary of the append-only payment ledger; the ledger remains the source for payment and receipt history.

## Reception billing workspace

The Web workspace is available only when the selected tenant membership has one or more billing permissions. Each capability is rendered from the effective permission identifiers returned by the API; the API independently enforces authorization on every request.

1. A cashier selects a department and hospital-local business date, then loads queue entries in `InProgress` or `Completed` status. The selector shows the visit token and status, not patient clinical information.
2. On selecting a visit, the client calls the visit-to-invoice lookup. If an invoice already exists, the client displays that invoice and its current settlement state. This is the normal resume path and avoids asking the cashier to issue another invoice after reloading or navigating away.
3. If no invoice exists, an authorized user selects active catalogue items and quantities. The API calculates totals and creates the invoice with immutable line snapshots. Only one initial invoice per visit is supported by the existing unique constraint.
4. An authorized cashier records a payment using Cash, UPI, Card or BankTransfer. The client then reloads the invoice and retrieves the receipt. A remaining balance leaves the invoice `PartiallyPaid`; exact settlement moves it to `Paid`.
5. Users with the reporting permission can request a daily collection summary by business date and time zone. Users with catalogue-management permission can add hospital-specific service items.

The initial Web workspace intentionally avoids displaying patient names in the billing visit selector. Patient identity and clinical records remain subject to their separate patient/clinical permissions. It is not a cashier shift-close or reconciliation system.

## Access and audit

- All data is tenant- and hospital-scoped, with relational constraints tying invoice, queue entry and patient registration to the same tenant and hospital.
- Catalogue management is separate from invoice/payment operations.
- Read and mutation operations require explicit billing permissions; UI visibility is not authorization.
- Invoice creation, invoice reads, payment recording and receipt reads must be auditable. Audit metadata must not contain clinical text or payment credentials.
- No card PAN, CVV, UPI PIN, bank credentials or gateway secrets are stored. A payment reference may be stored only as a short non-sensitive reconciliation reference.
- Billing access must not grant clinical encounter access.

## Explicitly deferred pending pilot confirmation

| Deferred capability | Decision needed |
|---|---|
| GST/tax | Applicable tax categories, rates, inclusive/exclusive pricing, tax registration details and statutory invoice format; validate with the hospital's finance/tax adviser |
| Invoice numbering | Legal numbering format, fiscal-year reset, branch/series rules and cancellation numbering |
| Discounts and waivers | Allowed types, limits, approval matrix, reason codes and audit requirements |
| Refunds and reversals | Refund methods, partial refunds, payment-provider reversal, approval and immutable credit-note model |
| Cancellation / credit notes | Whether invoices can be cancelled, when, and required accounting treatment |
| Insurance / TPA | Payer workflows, pre-authorizations, claim data and settlement handling |
| Payment gateway | Provider selection, hosted checkout, webhook signature validation, idempotency and reconciliation |
| Accounting integration | Target system, chart of accounts, export format and reconciliation ownership |
| Advanced reporting | Cashier close, shift settlement, outstanding ageing, department/provider revenue, tax reports and audit exports |
| Deposits / advances | Advance collection, allocation to future visits and refund rules |
| Multiple currencies | Whether non-INR billing is ever required |
| Production retention | Finance-record retention, archival, access review and applicable Indian legal requirements |

## Pilot acceptance checklist

- Finance/admin approves service codes, names, categories and INR prices.
- Reception confirms when a charge is raised and who may record payments.
- Finance confirms accepted payment methods and what reference data may be retained.
- Finance validates invoice and receipt content, numbering and tax treatment.
- Hospital confirms whether one invoice per visit is sufficient for the pilot.
- Hospital validates correction, cancellation, refund and retention policies before enabling production billing.
- Role assignments and audit access are reviewed; billing permissions do not imply clinical permissions.

## Initial delivery sequence

1. Catalogue CRUD/read with hospital scope and explicit permissions.
2. Visit-linked invoice creation with immutable line snapshots.
3. Payment recording, balance calculation and receipt retrieval.
4. Daily collection report by hospital-local date.
5. Integration tests for tenant/hospital isolation, permission denial, immutability, partial payments, overpayment and concurrent payment attempts.
6. Docker-backed PostgreSQL migration and integration verification before merge.

## Non-goals

No general ledger, accounting journal, tax engine, discounts, waivers, refund, credit note, insurance, TPA, payment gateway, patient portal, pharmacy billing, lab/radiology billing, inpatient billing or cash-register hardware integration in this baseline.
