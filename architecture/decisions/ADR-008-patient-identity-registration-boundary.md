# ADR-008: Patient identity and registration boundary

- **Status:** Proposed — requires hospital/customer validation
- **Date:** 2026-11-24
- **Decision owners:** Product, hospital stakeholders, architecture and security

## Context

CompileCare must support patients visiting multiple hospitals/branches within a customer tenant while protecting identity and clinical data. Patient identity must not be coupled to appointments, encounters, billing or a single physical facility. The MVP registration workflow must prevent avoidable duplicates but cannot silently merge uncertain records.

The Product Blueprint proposes a tenant-owned patient master with facility affiliations, but patient identifier policy, patient mobility, emergency registration and legal/consent requirements remain open.

## Proposed decisions

1. **Tenant boundary:** Patient master records are owned by one tenant. Cross-tenant matching, linking or sharing is disabled by default and requires a separately reviewed legal, consent, identity and technical design.
2. **Internal identity:** Use an immutable internal GUID for the patient master. It is not a human-facing identifier.
3. **Facility affiliation:** Represent hospital registration/affiliation separately from the patient master. This allows a tenant's patient to be registered at multiple authorized facilities without duplicating the master.
4. **Human identifiers:** Do not finalize MRN/UHID generation or uniqueness scope in the initial technical slice. The pilot must choose tenant-wide identifier, facility-local identifier, or a documented combination. Identifier normalization, display, correction and reuse rules must be defined.
5. **Duplicate handling:** Search before create. Exact identifier matches may be deterministic; demographic matching is advisory and must present candidates for authorized review. No silent merge. Merge/correction requires a privileged, audited, governed workflow and a recovery plan.
6. **Minimum data:** Store only the demographic fields necessary for the validated registration workflow. DOB is date-only and may be unknown; phone/email are optional and are not unique identity keys. Do not include clinical data in the patient master.
7. **Access:** Search/create/associate/update are separately authorized. Patient visibility is tenant- and organization-scoped; tenant membership alone is not unrestricted patient access. Candidate lists and search responses return minimal data.
8. **Lifecycle:** Ordinary API CRUD does not physically delete patient records. Correction, merge, retention and deletion are governed workflows.

## Consequences

- Patient master and facility affiliation are separate concepts and should have separate contracts.
- Database uniqueness constraints must encode the approved identifier scope, not an assumption.
- Search requires tenant and organization scope checks at the service layer, with tenant filters and database constraints as defense-in-depth.
- Registration endpoints need retry/idempotency strategy and concurrency tests.
- Before production, validate with hospital users and qualified privacy/legal reviewers, including patient mobility, guardianship, emergency workflows, retention and applicable Indian obligations.

## Open questions

- What identifier should the first pilot use (tenant UHID, hospital-local MRN, or both)?
- Which demographic fields are required at registration and which can be deferred?
- What is the approved candidate matching algorithm and minimum match threshold?
- Who may resolve duplicates and merge records, and how can merge be reversed?
- How are guardians, minors, emergency/unknown patients, aliases and name changes handled?
- What patient access is allowed across hospitals/branches within the same tenant?
- What retention, correction, deletion and consent rules apply to the pilot?
