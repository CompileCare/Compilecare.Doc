# ADR-019: Facility Sites, Branches and Clinic Scope

- **Status:** Proposed — requires product and hospital-operations review; not accepted for implementation.
- **Date:** 2026-11-28
- **Decision owners:** HeapCare Product & Engineering with hospital operations, clinical and finance stakeholders.
- **Related:** [ADR-006: Tenant, Organization and Patient Data Boundaries](ADR-006-tenant-organization-patient-boundaries.md), [ADR-018: Inpatient ADT and Bed-Management Boundaries](ADR-018-inpatient-ADT-bed-management-boundaries.md), HeapCare.Doc issue #198.

## Context

The accepted customer boundary is **Tenant = hospital organization/customer account**. A Tenant may operate one or multiple physical hospitals and may also operate branches, campuses and clinics. A Tenant is not equivalent to one physical hospital. No separate `HospitalGroup` aggregate is required.

The current `Hospital` entity has a narrower, important meaning: it is a physical hospital and owns the existing Department → Ward → Room → Bed hierarchy. The current implementation is already tenant-aware and supports multiple Hospital records under one Tenant.

Source inspection on 2026-11-28 found that `HospitalId` is embedded throughout existing operational contracts and persistence:
- `HospitalPatientRegistration` is hospital-scoped and is the prerequisite for patient-facing workflows.
- `StaffAssignment` stores HospitalId and optional DepartmentId.
- Provider availability and appointments store HospitalId and DepartmentId.
- Billing catalogue, invoices, invoice lines and payments carry HospitalId; invoices and many related rows also carry DepartmentId.
- Inpatient admissions and bed placements are hospital-scoped with composite tenant/facility constraints.
- Membership scope types currently support Tenant, Hospital and Department only.
- Departments own Wards, Rooms and Beds, so a clinic cannot safely be represented as an ordinary Hospital row without inheriting hospital-only concepts.

A site model therefore cannot be a display label or a cosmetic rename. It affects patient registration and visibility, staffing, scheduling, clinical access, billing/reporting, audit and the existing API route contracts.

## Problem to solve

Introduce explicit physical operating locations without changing the meaning of Tenant or silently broadening access. A site may represent a hospital, campus, branch or clinic. Some sites may be children of a hospital; an independently operated clinic may be a top-level site within the same Tenant. The final design must express these cases rather than forcing every site into the Hospital entity.

## Options considered

### Option A — Treat every site as a Hospital

**Not recommended.** This would make a clinic appear to be a physical hospital, create hospital-only hierarchy assumptions, and obscure which sites can admit inpatients or own beds. It would also make hospital registrations, billing and analytics indistinguishable from site registration and reporting.

### Option B — Keep Hospital as the organization boundary

**Rejected.** This conflicts with the accepted customer model. A single hospital organization may operate several hospitals and sites under one customer account; using Hospital as the tenant boundary would force account duplication or later data consolidation.

### Option C — Add a first-class FacilitySite aggregate

**Recommended for detailed design, not yet approved.** Introduce a Tenant-owned site registry for physical operating locations, while retaining the current Hospital entity as the hospital-specific operational model during a compatibility period. A one-to-one mapping can represent every existing Hospital as a Hospital-type site. New clinic/branch/campus sites can be represented without pretending they are physical hospitals.

This option is the most flexible but requires a carefully staged transition of operational ownership and authorization.

## Proposed target concepts

### Tenant

- Remains the customer-account and top-level data-isolation boundary.
- Owns users/memberships, site registry records and tenant-wide administrative configuration.
- Tenant-wide membership does **not** automatically grant unrestricted access to every patient's clinical information or every site's operations.

### FacilitySite

A proposed Tenant-owned record representing a physical care-delivery location.

Potential attributes for evaluation:
- Stable site identifier and TenantId.
- Site type, initially considering Hospital, Campus, Branch and Clinic.
- Display/legal name and a Tenant-unique normalized code.
- Optional ParentSiteId for an explicit site hierarchy.
- Active/inactive lifecycle, effective dates and time zone.
- Structured address and operational contact references, with appropriate privacy controls.
- Optional capability flags or related capability records for outpatient services, inpatient beds and other explicitly supported functions.

Parent relationships must remain within one Tenant. A site hierarchy must not imply patient sharing, staff access, billing consolidation or permission inheritance unless those rules are independently defined.

### Hospital

- Retains its current meaning as a physical hospital and continues to own the inpatient bed hierarchy during the compatibility phase.
- Each current Hospital should map to exactly one Hospital-type FacilitySite if this option is approved.
- A Clinic or Branch must not become a Hospital merely to reuse existing APIs.
- The eventual relationship between a Hospital and its sites must distinguish the hospital's clinical/operational authority from the location where a service is delivered.

### Departments and care capabilities

A Department must eventually be associated with the site where the service is delivered. A site may have outpatient departments without having inpatient beds. Bed, ward and room capabilities remain restricted to a site/hospital explicitly approved for inpatient operations.

The design must distinguish:
- A hospital department that operates across multiple campuses.
- A department that exists only at one site.
- A standalone clinic service that is not a department of a physical Hospital.

Do not settle this distinction by adding optional foreign keys without defining uniqueness, parentage and scope rules.

## Patient identity and registration

- Keep one Tenant-owned Patient master; do not create a duplicate Patient per site.
- The current HospitalPatientRegistration remains valid for the existing Hospital workflows during migration.
- New site workflows need an explicit rule for whether registration belongs to a FacilitySite, a Hospital, or both.
- The preferred direction is an explicit registration/access relationship at the site where care is delivered, with separately approved sharing/access grants for cross-site use.
- Registration at one site must not automatically authorize a user at another site to search for or view the patient's identity, clinical record or history.
- MRN uniqueness must be defined at the agreed registration boundary. Existing hospital-local MRNs must remain stable through any migration.

## Staff, appointments, billing and audit

The target model must explicitly represent:
- Staff assignments and temporary coverage at one or more sites, including effective dates and the scope of allowed duties.
- Provider availability and appointments at the site where the service is delivered.
- Queue and encounter site context and resource visibility.
- Whether billing catalogues and invoices are site-specific, hospital-specific or centrally configured with explicit reporting ownership.
- Audit metadata that includes the relevant Tenant, Hospital (where applicable), FacilitySite, resource and validated actor without copying patient names or clinical details into audit payloads.

Reports may aggregate across sites only through explicit tenant-level permissions and defined financial/clinical ownership. Aggregation is not an authorization grant.

## Authorization proposal

The current Tenant, Hospital and Department scopes remain fail-closed while the target is designed.

Evaluate adding an explicit FacilitySite scope. A site scope should grant access only to resources linked to that site and its explicitly declared child scope, not to sibling sites. Parent/child inheritance must be capability-specific and must not silently turn a clinic membership into hospital-wide patient access.

Required rules:
1. Tenant isolation remains mandatory.
2. Site access is explicit and checked at the service/resource boundary.
3. Department-scoped access is constrained to that department and site.
4. Hospital-wide access does not automatically include standalone clinics or sibling hospitals.
5. Cross-site patient visibility requires a separately modelled, auditable grant or relationship.
6. PlatformOperator remains a separate global identity and receives no implicit tenant clinical access.
7. Caller-supplied claims cannot create a site membership or permission.

## Backward-compatible migration approach

If Option C is accepted, implement the change as separate, reviewable increments:

1. **Model and inventory:** confirm site types, parentage, hospital-to-site mapping, registration semantics, scope inheritance and capability rules with a pilot organization.
2. **Additive site registry:** introduce FacilitySite and Tenant-scoped constraints without removing or renaming current HospitalId fields. Backfill one Hospital-type site per existing Hospital and preserve historical identifiers.
3. **Scope and authorization:** add site membership contracts and positive/negative tests. Do not grant new site scopes to existing roles automatically.
4. **Module-by-module site context:** add site references and compatible query/resource checks to departments, staff assignments, patient registrations, appointments/queues, encounters, billing and audit. Keep existing Hospital APIs functional during the expansion period.
5. **Pilot verification:** test one single-hospital organization, one multi-hospital organization, one hospital with campus/branch locations and one independently operated clinic. Verify patient discovery, cross-site denials, staff coverage, schedules, billing ownership and audit trails.
6. **Contract transition:** only after consumers are migrated and data verified, decide whether any legacy HospitalId API or persistence contracts can be deprecated. Do not delete/rename identifiers as part of the initial expansion.

Every schema step requires EF model snapshot updates, fresh-install and upgrade migration rehearsal, PostgreSQL integration tests, tenant/site isolation tests and Local real-Keycloak verification. No destructive migration is implied by this proposal.

## Required product decisions before acceptance

- Is a branch a child physical site of a Hospital or an independent operating site under the Tenant?
- Can a Clinic be a top-level Tenant site, or must every clinic be affiliated with a Hospital?
- Can one Department operate at multiple sites, or must each site have its own Department record?
- Is patient registration hospital-wide, site-specific or a combination? What explicit sharing is permitted between sites?
- What MRN uniqueness boundary is required for hospitals, campuses and clinics?
- Which roles may be tenant-wide, hospital-wide, site-wide or department-only?
- How are staff assignments and temporary cross-site coverage authorized and audited?
- Which services and bed capabilities may a clinic or branch operate?
- Which entity owns billing catalogues, invoices, receipts and financial reports across sites?
- What is the pilot organization's representative hierarchy and which sites must be isolated?

## Guardrails

- Do not introduce a redundant HospitalGroup entity.
- Do not equate Tenant with one physical Hospital.
- Do not represent every Clinic or Branch as a Hospital row solely to reuse existing API contracts.
- Do not make Tenant-wide membership synonymous with unrestricted clinical access.
- Do not infer patient, staff, appointment, billing or audit sharing from parent/child site relationships.
- Do not change runtime schema or existing HospitalId contracts until the site definitions, mappings and rollback plan are approved.

## Current implementation status

This ADR is a proposal only. No FacilitySite entity, migration, scope grant or runtime behavior is introduced by this document. HeapCare.Doc issue #198 remains open until the product and hospital-operational decisions above are resolved and accepted.
