# ADR-010: Conservative hospital-local duplicate candidate detection

- **Status:** Implemented and merged for the initial advisory increment
- **Date:** 2026-11-24
- **Decision owners:** CompileCare product and architecture; each pilot hospital validates its manual identity-verification procedure
- **Scope:** Read-only duplicate candidate detection before patient registration

## Context

ADR-008 establishes a tenant-owned patient master and a separate hospital registration. It requires staff to search before creating a record, allows advisory duplicate candidates, and prohibits automatic linking or merging. The first patient API slice currently stores only given name, optional middle/family name and optional date of birth. Phone, email, estimated age, verified government/ABHA references, guardian relationships and identity evidence are not yet implemented as patient identity data.

The first matching increment must work with the data that exists today, avoid inventing identity evidence, keep discovery inside the selected hospital, and make uncertainty explicit. It is not an identity-proofing service and does not replace hospital staff verification.

## Decisions

### 1. Provide a read-only pre-registration check

Add:

- POST /api/v1/hospitals/{hospitalId}/patients/duplicate-candidates
- Request body: given name, optional middle name, optional family name and optional date of birth.
- Response: whether matching was evaluated, an explanatory message, up to 20 candidates, and a flag indicating that more candidates exist.

Use POST rather than query-string parameters so names and date of birth are not placed in URLs, browser history or ordinary URL access logs. The endpoint does not create a patient, registration or identifier and does not update or associate any record.

### 2. Use a deliberately narrow v1 match rule

A record is returned as a possible candidate only when it has:

1. The exact same date of birth; and
2. The exact same given name, compared case-insensitively after trimming leading/trailing whitespace.

Middle and family names are returned for staff comparison. When the supplied middle/family name also matches case-insensitively after trimming, the response identifies that as an additional match signal; these fields do not exclude a candidate when they differ or are absent.

This v1 algorithm does not perform transliteration, phonetic matching, fuzzy spelling, nickname expansion, punctuation removal, demographic weighting, probabilistic scoring or a universal confidence score. The narrow rule prioritizes explainability and limits the data used until additional fields and hospital-approved matching configuration exist.

### 3. Treat unknown date of birth explicitly

If date of birth is unavailable, do not run a name-only search. Return evaluated=false, no candidates and a message instructing staff to follow the hospital's manual identity-check procedure. A future estimated-age workflow must be designed separately; an estimated age must never be silently treated as an exact date of birth.

### 4. Enforce hospital and tenant boundaries

- Search only registrations belonging to the hospitalId in the route and the validated tenant context.
- Require the dedicated patient.duplicate.review permission and tenant-wide or explicit hospital scope. Department-only membership cannot use this hospital-wide endpoint.
- Grant the permission to the initial PatientRegistrar role; keep it separately named so it can later be assigned to a dedicated registration supervisor/identity-review role.
- Do not search other hospitals or tenants in this increment. Cross-hospital discovery needs a separate permission, explicit business purpose and separately reviewed data-minimizing response.

### 5. Minimize response and protect audit data

Return only the candidate's internal Patient ID, name components, date of birth, hospital registration timestamp and explicit match signals. Do not return clinical data, addresses, contact information, documents, MRNs or identity evidence from this endpoint.

The request uses the existing metadata-only audit path. Do not log or persist the request body, names, date of birth or candidate list in general logs/traces/metrics. Apply the endpoint's normal authorization, tenant scoping and hospital-wide organization-scope checks before querying.

### 6. Bound the response and keep human review mandatory

Return at most 20 candidates in deterministic order and set hasMoreCandidates when the bounded result indicates further matches. Candidate results are suggestions for staff verification, not confirmed identities. A match result must never automatically block a registration, associate an existing patient, allocate an MRN, merge records or change clinical/business references.

Any later association or merge is a distinct command with its own authority, reason/evidence, approval, audit, conflict handling and reversal/correction rules.

### 7. Keep the first increment schema-neutral

The initial check uses existing patient and hospital-registration fields; it introduces no new patient identity fields or database migration. A future richer matcher may add normalized search fields, optional phone/contact verification, estimated age, verified identifiers, search indexes and configurable policy only after the data model, normalization, privacy controls and pilot acceptance criteria are reviewed.

## API and verification requirements

- The endpoint is versioned and hospital-nested.
- Missing/invalid given name, overlong name parts and future date of birth are rejected.
- Unknown date of birth returns evaluated=false and does not search by name alone.
- Matching uses exact given name (case-insensitive after trimming) plus exact date of birth.
- Results never cross the selected hospital or tenant.
- Department-only and other-hospital scopes are denied/not found; roles without the dedicated permission are forbidden.
- Results are bounded and disclose only minimum identity data.
- Tests prove that candidate detection does not associate or merge patient records.
- Docker Compose PostgreSQL integration verification is required for every API PR that changes this feature.

## Implementation status — 2026-11-24

The endpoint was merged in CompileCare.Api PR #16 as commit `41b459975b0e64198ad0fb251f1d43ad3e0bfe0f`. The final API PR head passed Docker Compose verification with 3 unit tests and 55 PostgreSQL integration tests (zero failures), standard CI, and the EF Core model-drift check. No migration was required.

## Consequences and deferred work

This provides a usable, explainable first check but has intentionally low recall: it will not find records with misspelled, transliterated, abbreviated or differently formatted given names, missing DOB, or data-entry variation. A no-candidates result is not proof that no duplicate exists. The front desk must continue its approved manual checks.

Future increments require separate decisions for richer demographic normalization, age-estimate matching, contact/identifier verification, cross-hospital discovery, duplicate case queues, identity review authority, patient association, merge approval, reversal and retention. No fuzzy score or automatic merge is introduced by this ADR.

## Follow-on matcher design

ADR-011 defines the next incremental matcher: versioned Unicode compatibility/case/whitespace normalization and a second candidate path requiring both middle and family names to match while exact DOB remains mandatory. This does not add unverified contact or other demographic signals.
