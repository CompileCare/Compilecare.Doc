# ADR-006: Tenant, Organization and Patient Data Boundaries

- **Status:** Proposed — requires product, hospital-user, privacy and legal validation
- **Date:** 2026-11-24
- **Decision owners:** CompileCare Product & Engineering; hospital stakeholders validate workflows
- **Related:** [Product Blueprint v1](../../product/blueprint-v1.md), [ADR-002](ADR-002-multi-tenancy.md), [ADR-0004 tenant registry](ADR-0004-tenant-registry-and-membership-provisioning.md)
- **Review trigger:** Before tenant registry, patient master, facility scope or cross-facility access is implemented.

## Context

CompileCare must support both a single-hospital customer and customers operating multiple hospitals or branches. The current API has a tenant identifier and facility hierarchy, but the customer/group/hospital relationship and patient-sharing rules are not fully defined. These choices affect keys, uniqueness, relationships, authorization, patient search, audit, migration and future deployment.

## Proposed decision

### Tenant and organization
- One tenant represents one customer relationship/contract and is the top-level data-isolation boundary.
- A tenant may operate one hospital or an organization/group of hospitals.
- Branches, campuses and departments are not separate tenants merely because they are separate sites.
- Use the proposed hierarchy: **Tenant → optional Hospital Group → Hospital → Branch/Campus → Department → Service/Unit → Ward/Room/Bed**.
- A tenant is created, suspended, reactivated or closed only through a trusted platform operation.
- Cross-tenant access and patient identity linking are denied by default.

### Identity and authorization
- Authentication establishes the external principal through validated issuer and subject.
- Tenant membership binds the principal to a tenant and a CompileCare-controlled role/status.
- Additional scope grants constrain access to hospitals, branches, departments, services or resources.
- Tenant-wide membership does not imply unrestricted access to every patient record.
- Platform operators use a separate trusted identity/policy; tenant roles never grant platform administration.

### Patient ownership
- A patient master record belongs to exactly one tenant by default.
- Patient registrations/affiliations may reference multiple hospitals/branches within that tenant, subject to explicit access policy.
- Encounters, orders, prescriptions, billing records and audit retain tenant and originating facility context.
- Patient identifiers follow tenant policy; facility aliases may be added where required.
- Duplicate detection may suggest matches, but merge is privileged, reviewable and audited; no silent automatic merge.
- Cross-tenant matching, linking and sharing are out of scope for the initial model.

This is a recommended starting point. Validate patient mobility, record ownership, consent, guardian, emergency and retention workflows before final schema approval.

## Consequences

Tenant identity remains stable when customers add hospitals/branches. Patient access across branches can be authorized without treating every branch as a separate tenant. Facility scope becomes a first-class authorization concern. Cross-tenant exchange requires a deliberate future legal, privacy and technical decision. Patient identifier and duplicate-handling policy must be defined before patient registration implementation.

## Required validation and tests

- One tenant can represent one or multiple hospitals; branches do not create tenants.
- Users can belong to multiple tenants only through separate explicit memberships.
- Membership without suitable organizational grants cannot access out-of-scope facilities/patients.
- Patient data is not visible across tenants, including indirect relationships and search.
- Within-tenant access follows configured facility scope.
- Duplicate detection never automatically merges records.
- Merges and identifier corrections are permission-checked and audited.
- Tenant, patient, encounter and facility relationships cannot cross tenant boundaries.
- Tenant suspension/closure denies access fail-closed.

## Open questions

1. Does the commercial tenant map to one legal entity, a hospital group, or either?
2. Can clinicians work across hospitals/branches, and how are temporary assignments represented?
3. Which demographics are required at pilot registration?
4. Is MRN/UHID tenant-wide, hospital-specific, or both with aliases?
5. Which clinical record types may be shared within a group, and under whose authorization/consent?
6. What are approved correction, merge, retention, export and closure policies?
7. What identity proof, guardian, minor, emergency and deceased-patient workflows are required?

Until validated, implement tenant scoping and extension points without assuming unrestricted cross-hospital patient sharing.
