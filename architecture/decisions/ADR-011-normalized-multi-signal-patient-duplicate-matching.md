# ADR-011: Normalized multi-signal patient duplicate matching

- **Status:** Implemented and merged in CompileCare.Api PR #17; contact enrichment is governed separately by ADR-012
- **Date:** 2026-11-24
- **Decision owners:** CompileCare product and architecture; pilot hospitals validate registration and identity-review procedures
- **Scope:** Improve advisory duplicate candidates inside the selected hospital using currently supported patient demographics

## Context

ADR-010 introduced a conservative, read-only duplicate-candidate check using exact given name plus exact date of birth. The current patient master stores given, middle and family names and optional date of birth. It does not yet store verified phone/email contacts, sex, address, estimated age or external identity evidence.

The next increment should improve matching for name formatting and use the available optional name components without claiming an identity match. It must stay hospital-local, remain explainable, and never create, link or merge a patient.

## Decisions

### 1. Versioned deterministic name normalization

Use the shared application-level PatientNameNormalizer with normalization version NFKC-INVARIANT-UPPER-WHITESPACE-v1:

1. Apply Unicode Normalization Form KC (compatibility normalization).
2. Collapse consecutive Unicode whitespace into one ordinary space and trim leading/trailing whitespace.
3. Convert to invariant uppercase.

Punctuation and diacritics are preserved. Thus repeated spaces, casing and compatibility forms normalize consistently, while Anne-Marie remains different from Anne Marie, and Jose remains different from José. Do not strip meaningful marks, remove punctuation, transliterate scripts, expand nicknames, use phonetic algorithms or use fuzzy edit-distance matching in this increment.

Persist normalized given, middle and family names separately from the original display values. Maintain them centrally on patient insert/update. Backfill existing rows in the EF migration and add tenant/DOB/name indexes. The original name fields remain authoritative for display; normalized fields are search keys only.

### 2. Multi-signal candidate rules

Exact date of birth remains mandatory for every candidate. Do not execute a name-only search when DOB is missing.

Return a candidate when either:
- normalized given name matches; or
- both normalized middle name and normalized family name match.

A family-name-only or middle-name-only match is insufficient. A matching given name remains a candidate even when optional name parts differ or are absent.

Return explicit match signals for DOB, normalized given name, normalized middle name and normalized family name. When the candidate qualifies only because middle and family names both match, make it clear that the given name differed. Do not assign a numeric confidence score or label the candidate as a confirmed identity.

### 3. Scope and authorization remain unchanged

- Search only registrations for the selected hospitalId and validated tenant.
- Require patient.duplicate.review and tenant-wide or explicit hospital scope. Department-only scope cannot call the hospital-wide endpoint.
- Cross-hospital and cross-tenant search are not included.
- Return no more than 20 deterministic candidates and expose only the existing minimal identity response.
- Use the metadata-only audit pipeline. Never copy request names, date of birth or candidates into general logs, traces, metrics or audit metadata.

### 4. Do not invent unavailable demographic signals

Phone, email, address, sex, estimated age, verified identifiers and guardian details are not part of the current patient master. Phone/email contact design is defined separately in ADR-012; it is not yet implemented. The ADR-011 candidate inclusion rule must not be widened by contact data. A shared, guardian-owned or unverified phone/email must never be treated as patient identity proof.

### 5. No automatic identity action

Candidate detection remains advisory. It must not block registration, allocate an MRN, associate an existing patient, merge records, change clinical references or alter demographic data. Identity verification, association and merge remain separate privileged workflows with reason/evidence, approvals, audit and correction/reversal controls.

## API and verification requirements

- Keep the hospital-nested endpoint and existing dedicated permission.
- Exact DOB is required and remains a hard filter.
- Test compatibility normalization, casing, whitespace, preservation of punctuation/diacritics, optional middle/family matches, middle+family fallback, family-only rejection, exact-DOB enforcement, hospital isolation, role/scope authorization and the 20-candidate cap.
- Verify backfill from the existing patient rows and verify the EF model snapshot matches the migration.
- Docker Compose PostgreSQL integration verification is required for every API PR changing this feature.
- No API response includes normalized search keys; retain original display names.

## Consequences and limitations

This expands recall for formatting differences and records whose given name differs while both middle and family names match. It still does not detect misspellings, transliteration differences, phonetic equivalents, nickname variants, missing DOB or records with different dates of birth. A no-candidate result is not proof that no duplicate exists. Manual hospital verification remains mandatory.

Future contact capture and verification follow ADR-012. The first contact-aware increment may only annotate an already-qualified ADR-011 candidate with a verified, patient-owned, non-shared contact corroboration signal; it must not create candidates by contact alone. Cross-hospital discovery, duplicate case queues, patient association, merge approval and reversal require their own reviewed increments and tests.


## Implementation status — 2026-11-24

The implementation was squash-merged into CompileCare.Api main through PR #17 as commit c603dc602565b69f25b24b4738e45f24c4090362. It adds persisted normalized name keys, a migration/backfill, tenant/DOB/name indexes and the normalized given-name OR middle-plus-family candidate rule.

The final PR head passed the EF Core model-drift check, standard CI and Docker Compose PostgreSQL verification: 12 unit tests and 58 integration tests passed with zero failures. Post-merge main CI also passed on the merge commit: build succeeded with zero compiler warnings/errors, the EF Core pending-model-change check passed, 12 unit tests passed and 58 PostgreSQL integration tests passed.

The API remains advisory and hospital-local. It requires exact DOB and the documented normalized-name rule, returns at most 20 minimal candidates, and does not link, block or merge records. See ADR-012 before introducing phone/email contact data or contact-based corroboration.
