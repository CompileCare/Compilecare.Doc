# ADR-006: Tenant, Organization and Patient Data Boundaries

- **Status:** Accepted for the customer/tenant boundary; facility/site topology and cross-site sharing remain open.
- **Date:** 2026-11-28
- **Decision owners:** HeapCare Product & Engineering; hospital stakeholders validate operational workflows.
- **Related:** [Product Blueprint v1](../../product/blueprint-v1.md), [ADR-002](ADR-002-multi-tenancy.md), [ADR-0004 tenant registry](ADR-0004-tenant-registry-and-membership-provisioning.md).
- **Supersedes:** The earlier proposal that left the commercial tenant-to-organization relationship unresolved.

## Context

HeapCare must support small independent hospitals as well as larger healthcare organizations operating multiple hospitals, branches, campuses and clinics. The tenant boundary must not be tied to one physical hospital. This decision affects facility modelling, identity, authorization, patient registration, clinical record visibility, reporting and future migration.

## Accepted decision

### Tenant and hospital organization

1. A **Tenant represents one customer account / hospital organization** and is the top-level customer and security-isolation boundary.
2. One Tenant may operate one physical hospital, multiple hospitals, branches/campuses and clinics.
3. A Tenant is not equivalent to one physical hospital.
4. The Tenant itself represents the organization/group boundary. Do not introduce a separate `HospitalGroup` entity merely to duplicate the Tenant.
5. A tenant is created, suspended, reactivated or closed only through a trusted platform operation.
6. Cross-tenant access, identity matching and patient-record sharing are denied by default.

### Existing facility model and migration direction

- In the current implementation, `Hospital` represents a physical hospital record; it is not the Tenant and must not be repurposed to mean the customer organization.
- The current operational hierarchy is Hospital → Department → Ward → Room → Bed.
- The accepted product scope includes branches/campuses and clinics, but their precise representation is not yet settled. The facility/site model must be extended deliberately so that a clinic or branch is not falsely represented as a hospital or as a department when its operational meaning differs.
- Before schema changes, define a facility/site abstraction, allowed facility kinds, parent/location relationships, uniqueness, membership scope and migration from the current Hospital hierarchy. Do not add a duplicate group layer.
- Production deployment is deferred; development and schema verification continue locally.

### Identity and authorization

- Authentication establishes the external principal through validated issuer and subject.
- Tenant membership binds the principal to a Tenant and a HeapCare-controlled role/status.
- A user receives access to a hospital/site/department only through explicit, validated organization-scope grants.
- Tenant-wide membership does not imply unrestricted access to every hospital, patient, encounter or clinical record.
- Cross-site staff coverage and multi-site clinical privileges require explicit assignments and approved permissions; they are never inferred from group membership.
- Platform operators use a separate trusted identity/policy. Tenant roles never grant platform administration.

### Patient ownership and records

- A patient master record belongs to exactly one Tenant by default.
- Hospital registrations, identifiers/MRNs, encounters, billing records and audit retain their originating facility context.
- The existing hospital-local MRN behavior remains the compatibility baseline unless an explicit product decision changes it.
- A patient being registered at two hospitals within the same Tenant does not automatically make all records visible to both hospitals.
- Within-tenant patient discovery, record linkage, referral access, consent, emergency access, duplicate handling, merge and retention policies require explicit design and authorization.
- Cross-tenant matching, linking and sharing are outside the current scope.

## Consequences

- Customers can add hospitals, branches and clinics without creating one Tenant per physical site.
- The Tenant remains the stable customer and data-isolation root.
- The current Hospital entity remains a physical-hospital record until a reviewed facility/site migration is implemented.
- Facility/site access becomes a first-class authorization concern. Tenant membership alone is insufficient for clinical access.
- Existing patient registration, MRN and clinical data boundaries must be preserved while the organization/site model evolves.

## Required validation and tests

- One Tenant supports one or multiple physical healthcare locations.
- Users can belong to multiple Tenants only through separate explicit memberships.
- Membership without suitable organization-scope grants cannot access out-of-scope hospitals, sites, departments or patients.
- Patient data is not visible across Tenants, including through indirect relationships and search.
- Within-Tenant cross-site access follows explicit configured grants and approved data-sharing policy.
- Tenant, patient, encounter, billing and facility relationships cannot cross Tenant boundaries.
- Tenant suspension/closure denies access fail-closed.
- Any facility/site migration has fresh-install, upgrade, concurrency, cross-scope and rollback/rehearsal coverage.

## Remaining decisions

1. Is a branch/campus a separately managed facility, a child location of a hospital, or only a reporting/location label?
2. Is a clinic a first-class facility/site, a child site of a hospital, or an outpatient service within a hospital?
3. Which roles may be Tenant-wide, hospital-specific, site-specific or department-specific?
4. How should staff assignments and temporary coverage work across hospitals/sites?
5. Which patient demographics may be discovered across sites, and which clinical record types may be shared, linked by referral or remain isolated?
6. Is MRN tenant-wide, hospital-specific or a combination of a tenant-wide identifier and facility aliases?
7. Which reporting, licensing, billing, audit, retention and data-residency boundaries apply at each level?

Until these decisions are validated, preserve current fail-closed tenant and hospital scoping. Implement extensibility without assuming unrestricted cross-hospital patient sharing.
