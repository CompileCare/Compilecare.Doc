# ADR-012: Verified patient contact data and duplicate-matching boundaries

- **Status:** Design proposed for implementation; deployment-specific security, privacy and contact-channel decisions remain open
- **Date:** 2026-11-24
- **Decision owners:** CompileCare product, architecture and security; pilot hospitals validate registration, identity-review and contact workflows
- **Scope:** Optional patient-owned phone/email contact methods, proof-of-control verification, protected storage and their permitted use in hospital-local duplicate review

## Context

The merged patient master currently contains names and optional date of birth. The normalized duplicate matcher in ADR-011 uses exact DOB plus its existing name rules. Phone and email are not currently stored as patient contact attributes or used by the matcher.

Contact details may be shared by family members, controlled by a guardian, recycled, reassigned or become inaccessible. A successful OTP proves that someone controlled a channel at a particular time; it does not establish that the person is the patient or that the contact is unique to one patient. Contact data is sensitive personal information and must not be introduced as ordinary strings on the Patient row or used as an unreviewed identity key.

This ADR defines a bounded first contact increment. It does not select an SMS/email vendor, establish a legal basis or retention period, or claim regulatory compliance.

## Decisions

### 1. Keep contact methods separate from the patient master

- Model optional phone and email as separate PatientContactMethod records associated with a tenant-owned Patient.
- Support multiple contacts per patient and allow the same contact value to appear on multiple patient records. Do not create a tenant-wide uniqueness constraint for phone or email.
- Do not add phone/email as required registration fields. A patient without a phone or email remains registrable under the hospital's approved minimum-data policy.
- Store patient-owned contact methods only in the first increment. Guardian, dependant and representative contact details belong to their own relationship/person model. A guardian's phone must never be represented as the patient's own verified contact or used as a patient identity signal.
- Do not duplicate contact values into MRN, identifier, appointment, encounter or clinical records. Other modules reference the patient/contact contract and follow their own permissions.

### 2. Use conservative, versioned canonicalization

Canonicalization is for exact comparison and delivery, not fuzzy matching.

**Phone**
- Parse with a maintained phone-number metadata library and persist a canonical E.164 representation only as protected/encrypted data.
- Require an explicit country/region or a hospital-configured default region. Do not infer a country code from IP address, browser locale or the user's estimated location.
- Reject ambiguous or invalid numbers rather than guessing. Do not normalize extensions, short codes or incomplete numbers into identity-matchable values.
- Record the canonicalization algorithm/metadata version so a future library metadata change can be reprocessed deliberately.

**Email**
- Trim outer whitespace and apply a documented Unicode normalization policy.
- Normalize the domain using IDNA and lowercase the domain. Preserve the local part conservatively; do not strip plus tags, dots or provider-specific aliases, and do not assume different mailbox strings are equivalent.
- Do not treat DNS/MX lookup or successful delivery as proof of patient identity or mailbox ownership.

For both channels, preserve the original entered representation only as encrypted display data. Do not expose canonical values or lookup keys in ordinary API responses.

### 3. Protect stored values and equality lookup

- Encrypt the entered/display value using application-level envelope encryption backed by a deployment-managed KMS/HSM or equivalent secret-management service. Encryption keys must not be stored in source control, application configuration committed to Git, database rows or telemetry.
- For exact lookup, compute a keyed HMAC-SHA-256 digest over channel, canonicalization version and canonical value using a tenant-scoped secret. Persist the digest and key-version metadata; never persist an unkeyed hash of a phone number or email address.
- Persist ciphertext, nonce/authentication tag, encryption-key version, lookup-key version and canonicalization version as required by the selected cryptographic provider. Do not persist a plaintext normalized contact value.
- Index the non-unique tuple needed for scoped lookup (tenant, channel, key version, canonicalization version, digest). The digest is not a unique identity constraint and must not create automatic record association.
- Define key rotation, re-encryption, digest re-computation, backup handling and access review before production use. A key-version mismatch or failed decryption must fail closed for matching; it must not fall back to plaintext or an unkeyed hash.

### 4. Make verification an explicit, revocable lifecycle

Use an explicit state machine such as PendingVerification, Verified, Expired, Revoked and Superseded. Store verification time, method, challenge reference and the minimum actor/context metadata required for audit.

- Verification must prove control of the exact phone/email value using a one-time challenge delivered through an approved channel adapter. Delivery adapters are interfaces; no provider is assumed by this ADR.
- Challenges must be random, one-time, short-lived and rate-limited. Store only a suitable challenge verifier/digest, enforce maximum attempts and resend limits, bind completion to the intended patient/contact/tenant and prevent replay.
- Never log or persist OTP plaintext, verification URLs/tokens, full contact values or challenge secrets. Delivery success is not verification success.
- A contact can become Verified only after a successful challenge. Staff cannot set Verified by toggling a field or by recording an unverified verbal claim. A manual exception must be a separate, reasoned identity-review case and must not masquerade as channel verification.
- Editing a contact creates a new pending value/version or supersedes the previous value; verification never carries over after the value changes. Revocation, reported compromise, reassignment risk or configured expiry removes eligibility for identity corroboration.
- Verification proves control at a point in time, not permanent ownership, patient identity, consent to messaging or authority to share data. Re-verification cadence and expiry are deployment policy, not hardcoded universal constants.

### 5. Treat shared contacts and contact ownership conservatively

- Multiple patient records may legitimately reference the same family phone or email. Do not reject a registration or contact update because a value is already present on another patient.
- Detect when the same active lookup digest is associated with more than one patient in the relevant tenant/hospital search scope. Treat it as shared/ambiguous and suppress its use as a patient identity corroboration signal; do not reveal the other patient records.
- Do not infer that a patient owns a guardian's, spouse's or caregiver's contact merely because that person can complete a challenge. The first increment accepts only patient-owned contact methods for matching.
- A contact match must never by itself create a duplicate candidate, block registration, assign or change an MRN, associate patients, merge records or alter clinical data.

### 6. Keep communication purpose, verification and consent separate

Verification is proof of control of a channel. It is not consent to receive marketing, appointment reminders, clinical results or other messages, and it is not automatically a legal basis for processing or sharing data.

Store and enforce communication purpose/preferences separately, with purpose, channel, status, source and effective timestamps where applicable. A withdrawal or preference change must not silently erase clinical audit history or change identity verification facts. Consent/notice wording, purposes and applicable legal basis require qualified privacy/legal review for each deployment.

### 7. Hospital-scoped API and authorization

The proposed API surface is hospital-nested and requires an active patient registration at the selected hospital plus validated tenant and hospital scope. Exact route names may be finalized with the API contract, but the operations must remain distinct:

- Add/update/revoke a contact.
- Read masked contact summaries.
- Initiate and complete a verification challenge.
- Read unmasked contact data only through an explicitly privileged, purpose-limited operation.

Use separate permissions rather than extending PatientRegistrar implicitly. The initial permission proposal is:

- patient.contact.manage — add, supersede or revoke a patient-owned contact.
- patient.contact.read.masked — read masked contact summaries.
- patient.contact.read.full — read the full contact value for an approved operational purpose; every unmasked read is audited.
- patient.contact.verify.initiate — send a verification challenge.
- Challenge completion is bound to the contact challenge and patient context; it is not an unrestricted staff permission to mark a contact verified.

Mask values by default (for example, reveal only a small suffix of a phone number or a limited email prefix/domain). Never return ciphertext, HMAC digests, normalized values, key versions or verification challenges. Cross-hospital access remains separately authorized; tenant membership alone does not expose a patient's contacts to every hospital.

### 8. Restrict the first matcher integration to corroboration

The existing ADR-011 candidate inclusion rule remains authoritative: exact DOB plus normalized given-name match, or exact DOB plus both normalized middle and family-name matches.

For the first contact-aware matcher increment:
- A verified contact may add a non-numeric corroborating signal only to a candidate already returned by the ADR-011 rule.
- Do not return a candidate solely because a phone or email matches, even when verified.
- The contact must be patient-owned, active, currently verified, canonicalized under a supported version and non-shared/non-ambiguous within the selected hospital scope.
- Unverified, expired, revoked, superseded, guardian-owned or shared contacts contribute no identity signal.
- Return only a generic signal label (for example, “Verified contact corroboration”); do not expose the value, suffix, digest or another patient's existence.
- Do not change candidate order, confidence scoring or the hard DOB/name inclusion rules in this increment. Any future proposal to widen candidate inclusion requires a new reviewed ADR, false-positive/false-negative evaluation and pilot approval.

Matching remains hospital-local, advisory and manually reviewed. No cross-hospital or cross-tenant contact discovery is included.

### 9. Audit, telemetry and retention

- Audit contact add/update/revoke, verification initiation/completion/failure/expiry, unmasked reads and permission changes.
- Audit only opaque record IDs, tenant/hospital scope, channel type, operation, outcome, actor, timestamp, correlation ID and reason/evidence reference where necessary. Do not write contact values, canonical values, hashes, OTPs or full payloads into audit metadata.
- Never place contact values or digests in logs, traces, metric labels, exception messages, dashboards, URLs or general health details. Avoid contact IDs as metric labels.
- Retention is configurable by data category and deployment, based on applicable law, hospital policy and qualified legal/compliance review. Do not hardcode a universal retention period. Revocation stops use but does not automatically erase evidence subject to an approved retention schedule or legal hold.
- Correction, deletion, export, legal hold and de-identification use governed workflows. Ordinary patient CRUD cannot physically delete contact history.

## API, migration and verification requirements

Before enabling the feature, define a reviewed migration/backfill plan (existing patient records have no contact values to backfill), provider adapter contract, secrets/KMS integration, key rotation strategy, masked DTOs and permission matrix.

Required tests include:
- Phone normalization with country/region explicitness, invalid/ambiguous input and versioned metadata.
- Email domain normalization, local-part preservation, plus-tag/dot preservation and Unicode handling.
- Ciphertext-at-rest and tenant-scoped HMAC equality lookup; no plaintext normalized value or unkeyed digest is stored.
- Same contact attached to multiple patient records is allowed and does not create a unique-constraint failure.
- Shared-contact detection suppresses corroboration without leaking linked patient records.
- Pending, verified, expired, revoked and superseded state transitions; changed values lose previous verification.
- Challenge expiry, attempt limit, resend throttling, tenant/patient/contact binding and replay prevention.
- Guardian/representative contacts are not patient-owned matching signals.
- Masking by default; full-value reads require the separate permission and generate audit evidence.
- Tenant/hospital isolation, patient registration checks, role/scope denial and cross-hospital denial.
- No raw contact, digest, OTP or challenge secret in logs, traces, metrics, exceptions or audit metadata.
- Key rotation/re-hash and migration rollback/recovery behavior.
- Existing ADR-011 exact-DOB/name candidate inclusion remains unchanged; contact-only matches return no candidate.

Every API PR that adds or changes this feature must pass standard CI, EF model-drift checks and Docker Compose PostgreSQL integration verification.

## Consequences and open deployment decisions

This design enables contact capture and proof-of-control without treating a channel as a unique person identifier. It intentionally does not select a provider, default country region, verification expiry, retention period, consent wording or role assignment for every hospital.

Before implementation is enabled for a pilot, decide and review: permitted contact channels/types, region defaults, SMS/email provider and delivery operations, OTP TTL/attempt/resend limits, re-verification cadence, contact retention/deletion schedule, privacy notice and purposes, role-to-permission mapping, full-value read purposes, key-management provider and recovery process. Qualified privacy/legal/security review is required for applicable Indian requirements and each customer's deployment. This ADR is an engineering design, not legal advice or a compliance certification.
