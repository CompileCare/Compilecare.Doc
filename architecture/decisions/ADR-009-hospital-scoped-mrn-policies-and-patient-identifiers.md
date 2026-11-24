# ADR-009: Hospital-scoped MRN policies and patient identifiers

- **Status:** Accepted for the initial MRN implementation increment
- **Date:** 2026-11-24
- **Decision owners:** CompileCare product and architecture; each hospital approves its own prefix and numbering length
- **Scope:** Hospital-local Medical Record Number (MRN) policy, allocation, lookup and authorization

## Context

ADR-008 establishes an immutable tenant-owned Patient ID and a separate hospital registration. Hospitals need a human-readable identifier for local operations, but MRNs must not become patient primary keys or leak across hospital boundaries. Allocation must remain correct when multiple registration users issue identifiers concurrently.

## Decisions

### 1. Keep the MRN separate from patient identity

- The internal Patient GUID remains the permanent database identity.
- An MRN is an identifier attached to one existing patient registration at one hospital.
- A patient registered at two hospitals may have two different MRNs while retaining the same tenant-owned Patient GUID.
- MRN lookup is always scoped to the hospital in the route. This increment does not add cross-hospital or cross-tenant discovery.

### 2. Configure one MRN policy per hospital

The initial policy contains:
- Prefix: 1–20 uppercase letters, digits or hyphens; the first character must be a letter or digit.
- NumberLength: 1–12 digits, used as minimum zero-padding width.
- NextNumber: an internal positive 64-bit sequence counter, initially 1.

Example: prefix AKH, number length 6 produces AKH000001, AKH000002, and so on. The complete identifier is limited to 40 characters.

A hospital policy is created once in this increment. The API does not replace or edit an existing policy. Prefix or width changes require a separately reviewed, versioned policy/reconfiguration workflow; an existing sequence must never be reset or reused.

### 3. Allocate identifiers transactionally and safely under concurrency

- Allocation is serialized using a PostgreSQL row lock on the hospital's policy row.
- Incrementing the sequence and inserting the MRN occur in the same transaction as the request's metadata-only audit event.
- A retry or concurrent repeat assignment for the same patient registration returns the existing MRN; it does not allocate another number.
- Database uniqueness constraints enforce one MRN per patient registration and one normalized MRN value per hospital.
- An identifier cannot be attached to a patient/hospital combination unless that hospital registration exists; this is enforced by a tenant-composite foreign key.
- Identifiers are append-only in the initial increment. Ordinary API operations cannot edit or delete an issued MRN.

A failed transaction does not commit the allocation. The system does not intentionally reuse an identifier that has already been committed.

### 4. Separate configuration and operational permissions

| Capability | Permission | Initial role |
|---|---|---|
| Read MRN policy | patient.identifier.policy.read | HospitalAdmin |
| Create MRN policy | patient.identifier.policy.manage | HospitalAdmin |
| Read patient identifiers and search by MRN | patient.identifier.read | PatientRegistrar |
| Assign an MRN to an existing hospital registration | patient.identifier.assign | PatientRegistrar |

Every operation also requires tenant-wide or explicit hospital scope. Department-only membership cannot use hospital-wide identifier routes. The PatientRegistrar role does not gain facility administration or policy-management permissions.

### 5. Initial API boundary

| Method | Route | Purpose |
|---|---|---|
| POST | /api/v1/hospitals/{hospitalId}/patient-identifier-policy | Create the hospital MRN policy |
| GET | /api/v1/hospitals/{hospitalId}/patient-identifier-policy | Read the policy |
| POST | /api/v1/hospitals/{hospitalId}/patients/{patientId}/identifiers/mrn | Idempotently assign the next MRN |
| GET | /api/v1/hospitals/{hospitalId}/patients/{patientId}/identifiers | Read identifiers for that registration |
| GET | /api/v1/hospitals/{hospitalId}/patients/by-mrn/{mrn} | Resolve an MRN within that hospital |

The first implementation supports the MRN type only. It does not issue, verify, exchange or search ABHA IDs, Aadhaar numbers or other external identifiers. Those require their own issuer/scope, verification, privacy and consent decisions before implementation.

### 6. Audit and data minimization

All routes are tenant-scoped and use the existing request audit middleware. Audit records retain operation metadata, actor, tenant, route, outcome and correlation ID; they do not store patient demographics, request bodies or the MRN value. An MRN is returned only through an authorized hospital-scoped API response.

## Consequences

- MRN generation is a distinct service and persistence boundary, not a property on Patient.
- Hospital-specific uniqueness and patient-registration association are enforced in PostgreSQL.
- Application logic must use the same transaction for sequence allocation, identifier insertion and request audit.
- Policy reconfiguration, identifier correction/revocation, external identifiers, duplicate matching, identity merge and cross-hospital search remain separate reviewed increments.
- The registration workflow must create or locate the patient master and hospital registration before assigning the hospital MRN.

## Verification requirements

- PostgreSQL migration and model snapshot must remain aligned.
- Docker Compose integration verification is required for every pull request that changes this feature.
- Tests must cover policy validation, sequence formatting, concurrent/idempotent assignment, duplicate prevention, hospital/tenant isolation, role separation and MRN lookup.
