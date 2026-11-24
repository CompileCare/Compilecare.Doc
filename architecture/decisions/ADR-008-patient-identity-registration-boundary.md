# ADR-008: Patient identity and registration boundary

- **Status:** Accepted product direction; implementation and pilot configuration pending
- **Date:** 2026-11-24
- **Decision owners:** CompileCare product and architecture; hospital-specific configuration requires pilot approval
- **Scope:** Tenant-owned patient identity, hospital registration, identity matching and lifecycle boundaries

## Context

CompileCare must support a patient visiting multiple hospitals within one customer tenant without duplicating the patient master, while preserving hospital-level access controls. Patient identity must remain separate from appointments, encounters, billing and other clinical/business activity.

The product direction below is accepted for architecture and implementation. It does not replace hospital-specific operational approval or qualified legal/privacy review. In particular, retention schedules, consent purposes, ABDM exchange and hospital-required demographics must be configured and reviewed for each applicable deployment.

## Decisions

### 1. Identity and identifier scopes

Use three distinct concepts:

| Identifier | Scope | Purpose |
|---|---|---|
| Internal Patient ID | Tenant-wide | Immutable UUID/GUID and primary database identity |
| MRN/UHID | Hospital-local (or explicitly configured registration scope) | Human-readable identifier assigned by each hospital |
| ABHA ID | External, patient-linked | Optional ABDM identity reference, subject to applicable policy and consent |

A patient visiting Hospital A and Hospital B in the same tenant has one patient master and may have a separate hospital registration and MRN at each hospital. Link a registration to an existing master only after identity is sufficiently established through an authorized workflow.

Rules:
- Never use MRN, mobile number, Aadhaar or ABHA as the database primary key.
- Enforce MRN uniqueness within its hospital or explicitly configured registration scope.
- Make MRN generation configurable per hospital; define normalization, format, sequencing, correction and reuse behavior before enabling it.
- Do not automatically link patients across hospitals based only on matching names, phone numbers or other weak demographics.
- Do not perform cross-tenant patient matching or linking by default.
- ABHA and Aadhaar are optional references and must not be general registration prerequisites.

### 2. Demographic data and requiredness

Collect minimum registration data. Patient name is required and must support single names and locally appropriate formats. Date of birth is required only when known; otherwise support estimated age or age with unit and preserve that it is estimated. Mobile, email and address are optional initially. Sex is required only where justified by the hospital's registration/clinical workflow, with explicit support for unknown/not-recorded where approved.

Emergency contact and guardian details are conditional. Blood group, photo and other additional attributes are optional and must not be treated as clinically verified without verification.

Keep registration-required fields separate from clinical-workflow-required fields. Hospitals may configure additional registration requirements without changing the patient master schema. Demographic attributes are mutable facts, not immutable identity keys. Patients without mobile phones, email addresses or identity documents must be registrable under the configured minimum-data policy.

### 3. Duplicate detection and matching

Use configurable, auditable, multi-factor candidate matching. Conceptual signal priority (not numeric weights):

| Signal | Relative priority |
|---|---|
| Exact existing hospital MRN | Very high |
| Exact verified ABHA reference | Very high, subject to approved linkage |
| Exact verified mobile number | High |
| Name + date of birth | Medium to high |
| Name + guardian name | Medium |
| Name + age + sex | Medium |
| Similar name + matching address | Low to medium |
| Name alone | Low |

Normalize names and phone numbers; support spelling variations and transliteration appropriate to Indian names. Account for missing/estimated DOB and shared family phone numbers. A shared phone number or address is never sufficient proof of identity.

Strong matches are prioritized for staff verification; possible matches are presented for manual review; weak matches may allow registration with an advisory warning. Conflicting identifiers block assignment of the conflicting identifier and route the case for resolution. Matching thresholds and behavior remain configurable and auditable. No automatic patient linking or merge is permitted based solely on a score.

### 4. Duplicate resolution and merge authority

Separate duplicate reporting, review and merge approval:
- Receptionist: search and report a potential duplicate.
- Registration staff: review candidates and submit a merge request.
- Registration supervisor: verify identity and approve routine merges.
- Patient identity administrator: resolve complex identity conflicts and exceptional cases.
- System administrator: configure technical permissions, but does not automatically receive clinical identity-merge authority.

High-risk cases require two-person approval under configurable policy.

A merge must verify that both records refer to the same person; identify the surviving master; preserve both original patient IDs as historical references; reconcile registrations and identifiers; preserve clinical, billing and encounter references; record reason, evidence, requester, approver and timestamp; and support a governed correction/reversal process. Never physically delete the merged record. Maintain an auditable merged-record reference and prevent unsafe cycles or repeated merges.

### 5. Mobility, guardians and emergency registration

A patient has one tenant-owned master with multiple hospital associations. Authorized staff may associate an existing master with another hospital after identity verification; a new hospital-local MRN may be assigned according to that hospital's configuration. Department visits reuse the same master and hospital association.

Guardian/dependant/authorized-representative relationships are separate relationship records, not free-text fields on the patient. Support multiple relationships, effective dates, authority/status and contact preferences. A guardian may be another registered patient, but that is not mandatory. Guardian changes are audited.

Support a temporary emergency registration for an unknown patient with a temporary identifier distinct from a permanent MRN. When identity is established, reconcile the temporary record through an authorized, audited workflow; do not casually delete it. Patients without documents may be registered using the minimum fields allowed by approved hospital policy. Conflicting identities enter manual review.

### 6. Retention, correction, consent and deletion

Use governed lifecycle management, not ordinary CRUD deletion:
- Retention schedules are configurable by record category and deployment, based on applicable law, hospital policy and qualified legal/compliance review. Do not hardcode one universal retention period.
- Demographic corrections require authorization and record old/new values, reason, actor and timestamp, with sensitive values protected.
- Identity conflicts and merges use separate governed workflows.
- Consent records, where applicable, capture purpose, scope, status, source, timestamp and withdrawal history.
- Normal patient CRUD never physically deletes a patient.
- De-identification is allowed only through an approved process where legally permitted.
- Legal holds prevent disposal while active.
- Audit evidence has its own access and retention policy.

Distinguish consent to collect/process data, consent to share data, consent for ABDM-related exchange, and consent for treatment or a specific clinical procedure. These are not interchangeable. Consent and retention details require qualified review for each deployment.

### 7. Data model boundaries

Keep these concepts separate:
- **Patient Master:** tenant-owned, immutable internal UUID, minimal core demographics.
- **Patient Identifiers:** typed MRN, ABHA reference and other external identifiers, with issuer/scope, verification state and lifecycle metadata.
- **Patient Relationships:** guardian, dependant and authorized-representative relationships with effective dates and authority.
- **Hospital Registration:** patient-to-hospital association, local MRN, status and registration metadata.
- **Identity Management:** duplicate candidates, identity cases, merge requests, approvals and correction/reversal history.
- **Consent and Audit:** governed consent records and append-only identity/registration audit evidence.

Appointments, encounters, prescriptions, diagnoses, billing and clinical documents remain in their owning modules and reference the internal Patient ID. They must not duplicate the patient identity master.

### 8. Discovery and authorization

Default search is limited to the selected hospital's registrations. Cross-hospital patient discovery is a separate permission and workflow for authorized transfer, referral or registration scenarios. A receptionist at Hospital A must not gain unrestricted access to Hospital B's registrations or clinical information merely because both hospitals share a tenant.

Cross-hospital candidate search returns only the minimum identity information needed to verify a candidate. Access to actual patient information remains subject to hospital/department permissions. Search, create, associate, update, identity review and merge are distinct capabilities. Tenant membership alone does not grant unrestricted patient access.

## Consequences and implementation constraints

- Patient master, identifiers, hospital registrations, relationships and identity governance are separate persistence and API boundaries.
- Database constraints enforce tenant ownership and the configured hospital-local identifier scope.
- Search and association require tenant plus organization-scope authorization; cross-hospital discovery is separately authorized.
- Patient and registration endpoints need idempotency/concurrency handling and PostgreSQL integration tests.
- No MRN format/sequence, demographic mandatory-field set, matching numeric score, retention period or consent legal basis is globally hardcoded.
- Implement the smallest safe patient-master and hospital-registration slice first. Add identifiers, relationships and identity-governance workflows as separately reviewed increments.
- Clinical modules reference the internal Patient ID and remain responsible for their own data lifecycle.

## Deployment validation still required

Before a pilot is enabled, each hospital must approve its MRN format/sequence and uniqueness scope, additional mandatory fields, matching configuration, merge authority and high-risk approval policy, emergency workflow, cross-hospital discovery roles, and retention/consent settings. Qualified privacy/legal review is required for applicable Indian requirements, Aadhaar handling and ABDM exchange. These are deployment decisions, not reasons to weaken the architecture boundaries above.


## Implementation status — 2026-11-24

The first API implementation slice is being delivered in `CompileCare.Api` on `feature/patient-master-registration-foundation` (tracking PR #14). The initial code boundary is intentionally narrower than the full ADR:

- `Patient` is a tenant-owned master with an internal GUID, given name, optional middle/family name and optional date of birth.
- `HospitalPatientRegistration` associates the master with a hospital using tenant-composite foreign keys. The database enforces one registration per patient per hospital.
- Registration and lookup/search routes are nested under the selected hospital. Search queries only that hospital's registration rows and is paginated.
- A dedicated `PatientRegistrar` membership role receives only `patient.read` and `patient.register`. Existing facility roles do not implicitly receive patient permissions, and the patient role does not receive facility administration permissions. The hospital-nested patient API requires tenant-wide or explicit hospital scope; a department-only membership cannot read the entire hospital's patient registrations.
- Patient writes participate in the existing audited request transaction. Audit metadata does not persist request bodies or patient demographics.

The first slice deliberately does **not** implement MRN generation/storage, external identifiers (including ABHA), estimated-age fields, duplicate matching/linking, cross-hospital discovery, guardian relationships, merge workflows, clinical records or retention/consent workflows. Those remain separate increments governed by this ADR. MRN is omitted from the initial registration entity until the hospital-specific identifier scope and generation policy are approved; identifiers will remain a separate model rather than being used as the patient primary key.

The API migration `20261124143001_AddPatientMasterRegistration` has been generated and committed. The guarded migration workflow completed restore, build, migration-snapshot alignment and tests successfully (3 unit tests and 45 PostgreSQL integration tests). The final implementation commit also passed standard CI and Docker Compose verification: each ran 3 unit tests and 47 PostgreSQL integration tests with zero failures. Coverage includes explicit department-scope and hospital-scope isolation. The accepted product direction above remains subject to the deployment validation requirements in this ADR.
