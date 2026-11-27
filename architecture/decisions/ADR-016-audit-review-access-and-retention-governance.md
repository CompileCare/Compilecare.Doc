# ADR-016: Audit review access, export and retention governance

- **Status:** Partially implemented — bounded tenant-wide query/export exists as a provisional baseline; production governance and any scope expansion remain unapproved
- **Date:** 2026-11-27
- **Decision owners:** Product, security and engineering; pilot hospital operations, privacy/compliance and qualified legal stakeholders must validate before implementation is enabled for customer tenants

## Context

HeapCare persists metadata-only audit events for tenant-scoped, user-scoped and platform-operator API requests. The records are append-only and support accountability and incident investigation. They intentionally exclude request/response bodies, clinical note text, patient payloads, credentials and query strings.

The current audit record stores a validated tenant ID, the separately untrusted requested tenant ID, validated actor issuer/subject, action, resource type, target resource ID, HTTP method, route template, outcome, status, correlation ID and UTC occurrence time. It does **not** store validated hospital or department scope metadata. The audit middleware can identify route parameters, but route values are caller-controlled input and must not be treated as proof that an actor is authorized for a hospital or department.

A general audit listing or export endpoint exposes sensitive security and operational metadata. A deliberately bounded tenant-wide query/export baseline has since been implemented (API PRs #71 and #72). That implementation is provisional and must not be interpreted as blanket approval for production audit access, hospital/department-scoped audit browsing or unrestricted export. Any expansion still requires an explicit access model, organization-scope enforcement, retention policy and export controls.

## Proposed decisions

1. **Keep audit events metadata-only and append-only.** Do not store request/response bodies, clinical text, patient names, patient contact details, tokens, credentials, query strings or free-form payloads in audit records. Preserve database-level update/truncate protection and the request transaction guarantee.
2. **Keep the current endpoint narrowly bounded.** The implemented API exposes tenant-wide listing and CSV export only to a validated tenant membership with the server-derived `audit.read` permission and `OrganizationScopeContext.IsTenantWide`. Hospital/department-scoped memberships are denied; patient-facing audit browsing and unrestricted cross-tenant search remain out of scope. Do not add a hospital/department view until trusted scope attribution and a pilot-approved reader model exist.
3. **Preserve a dedicated permission and treat the present role mapping as provisional.** The implementation has a dedicated `audit.read` permission, currently included in the server-generated `HospitalAdmin` permission set. The query service additionally requires tenant-wide organization scope; a hospital-scoped HospitalAdmin is denied, and PatientRegistrar/ClinicalUser do not receive the permission. This is a provisional pilot baseline, not evidence that the pilot hospital has approved all tenant-wide HospitalAdmins as audit readers. Before production activation, obtain a named reader roster and explicit approval; a distinct ComplianceAuditor role may be preferable but must be validated rather than assumed.
4. **Enforce validated tenant and organization scope before pagination.** Every query must derive tenant ID from the validated tenant context, never from an arbitrary request parameter. Hospital/department-limited readers must see only events within their authorized organization scope. A tenant-scoped administrator must not gain platform-wide audit access.
5. **Add trusted facility attribution before scoped audit reads are implemented.** The current `AuditEvent` schema has no `HospitalId` or `DepartmentId`. Add optional validated scope identifiers only through an explicit request-scoped audit resource context populated by domain services after the target resource and its tenant/hospital/department ownership have been resolved and authorized. Do not infer trusted scope solely from route values, query parameters, headers or the requested tenant ID. Platform events remain separately classified and do not inherit customer tenant scope.
6. **Apply bounded, deterministic queries.** The current tenant-wide listing implementation requires a bounded UTC time range (maximum 90 days), caps page size at 100, applies the validated tenant and time predicates in SQL before sorting/pagination, and uses deterministic offset pagination ordered by `OccurredAtUtc` plus `Id`. It currently supports exact action, resource type and resource ID filters. The export path is separately bounded to 31 days and 5,000 rows. These are implementation limits, not approved retention or customer policy. Before materially increasing audit volume or adding broader query capabilities, evaluate keyset pagination and query plans against representative PostgreSQL data; any hospital/department query must apply trusted organization-scope predicates in SQL before sorting/pagination. Do not add unrestricted full-text search over actor identities or resource identifiers without a specific approved need.
7. **Audit the audit.** Implemented: audit listing and CSV export requests pass through the request audit middleware and create metadata-only audit events identifying the reader, validated tenant, action, outcome and correlation ID. The event generated by a read/export request appears in subsequent queries rather than the page that triggered it (API PRs #71 and #72). Preserve this behavior for future detail views or export jobs.
8. **Keep the current export bounded; defer bulk/asynchronous export.** Implemented CSV export is synchronous and capped at a 31-day range and 5,000 rows, with tenant filtering, metadata-only columns, CSV formula/control-character neutralization and an audit event for the export request. Do not raise these limits or add JSON/bulk/asynchronous exports without an approved purpose, encrypted storage, short-lived download authorization, expiry/deletion, independent review and safeguards against bulk disclosure. The present bounded response is not a general file-delivery platform.
9. **Do not invent a retention period.** No fixed number of days or years is approved by this ADR. The pilot hospital and qualified privacy/legal stakeholders must define retention by event category, archival, legal hold, deletion, backup expiry and any applicable statutory/contractual requirements. Until an approved policy exists, do not add automatic audit deletion or claim that indefinite retention is compliant.
10. **Restrict operational access.** Support and engineering personnel must not receive routine direct database browsing access to customer audit records. Emergency/break-glass access requires a documented reason, time-bounded approval, least privilege, independent logging and post-access review. Platform operators do not automatically receive cross-tenant audit visibility merely because they can manage tenant lifecycle.
11. **Preserve correlation without copying sensitive payloads.** Audit records may be correlated with protected application logs using the trace/correlation ID. Do not copy audit event content into metric labels or general telemetry.

## Current implementation baseline

The following safeguards are already implemented and merged:

- Metadata-only request auditing with trusted actor/tenant attribution and route-resource identification.
- Append-only EF Core and PostgreSQL protections.
- Success audit and business writes committed in one transaction; audit persistence failure rolls back the business operation and returns HTTP 503.
- PostgreSQL failure-injection regression proving patient and hospital-registration writes roll back when audit persistence fails (API PR #67).
- Endpoint-registration integration guard proving every authenticated route carries tenant, user or platform-operator audit metadata (API PR #68).
- Cross-tenant, cross-hospital and cross-department denial/audit coverage across patient, OPD encounter, billing, appointment and queue workflows (API PRs #61, #62, #64, #65 and #66).
- Bounded tenant-wide audit listing (maximum 90-day UTC range, page size 100) and CSV export (maximum 31-day range, 5,000 rows), filtered by the validated tenant ID and denied for hospital/department-scoped memberships (API PR #71).
- Dedicated server-generated `audit.read` permission currently mapped to HospitalAdmin, with a further tenant-wide organization-scope requirement. This role mapping remains provisional pending named-reader approval; it is not a production compliance decision.
- CSV formula/control-character neutralization and an audit event for the export request itself, visible in subsequent queries (API PR #72).

The API implementation details are maintained in [Audit events and traceability](https://github.com/HeapCare/HeapCare.Api/blob/main/docs/security/audit-events.md).

## Implementation gates

Do not broaden the current provisional query/export implementation or enable production use until all of the following are agreed and testable:

- The pilot hospital has named the intended audit readers and approved the current `HospitalAdmin` plus tenant-wide-scope mapping or a narrower dedicated role.
- The tenant, hospital and department scope model for audit events is defined, including how trusted scope is populated for reads, writes, failed requests and platform actions.
- The retention, archival, legal-hold and deletion policy has been approved by qualified stakeholders.
- Export purpose, permitted filters, maximum time range, format, file protection, expiry and review requirements are approved.
- Threat modelling covers enumeration, bulk export, actor-identity disclosure, cross-tenant access, scope changes and break-glass support access.
- Integration tests cover authorized tenant-wide reads, denied hospital/department-scoped reads, cross-tenant denial, audit-of-audit behavior, pagination boundaries, export authorization and any future retention operations.
- Docker/PostgreSQL verification proves the query plan uses the tenant/time predicates and required indexes without loading the entire audit table into memory; any future organization-scoped queries must prove their trusted scope predicates too.

## Open questions for pilot validation

1. Which named hospital functions may review audit events: HospitalAdmin, compliance officer, internal auditor, security officer or a separate role?
2. Should a reader be limited to one hospital or department, or is tenant-wide audit review an explicit responsibility?
3. Which event categories require different retention periods (authentication/security, patient access, clinical mutation, billing/financial, platform administration)?
4. What retention, archive, legal-hold and deletion obligations apply to the pilot hospital and its customer contracts?
5. Which filters are operationally necessary, and may a reader search by actor subject or resource ID?
6. Who may initiate exports, what is the maximum export size/time range, and who approves bulk exports?
7. What is the approved emergency access process and who performs independent post-access review?
8. Which audit events must be forwarded to the hospital's SIEM or security operations tooling, and under what data-sharing agreement?

## Consequences

- HeapCare does not expose sensitive audit metadata before a defensible reader/scope model exists.
- Tenant/hospital/department audit access will be enforced as a data boundary, not merely a UI navigation rule.
- Export and retention remain governed operational capabilities, not incidental API features.
- The existing append-only, metadata-only audit pipeline remains useful for accountability while customer-facing audit review is designed and validated.


## Current status and non-approval

The current code has a bounded tenant-wide API, but policy approval is not inferred from the existence of the code or from a successful test. Before customer production use, the pilot hospital and qualified security/privacy/legal stakeholders must confirm the named reader roster, retention and legal-hold rules, export purpose/review, incident escalation and access-review cadence. Until then, keep the capability limited to the existing tenant-wide membership boundary and do not expand it to hospital/department scopes or bulk export. The API implementation notes are in [Audit events and traceability](https://github.com/HeapCare/HeapCare.Api/blob/main/docs/security/audit-events.md).
