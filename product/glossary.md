# HeapCare Glossary

**Status:** Draft; terminology should be validated with hospital stakeholders and updated as workflows become concrete.

| Term | Working definition |
|---|---|
| Tenant | A customer organization and top-level isolation boundary within HeapCare. |
| Hospital Group | An optional organization grouping multiple hospitals under a tenant. |
| Hospital | A healthcare organization/site operating clinical and administrative services. |
| Branch/Campus | A physical location or campus belonging to a hospital. |
| Department | An organizational or clinical department, such as OPD, Laboratory, or Pharmacy. |
| Unit | A sub-area within a department, such as a ward, nursing unit, or specialty unit. |
| Patient | A person receiving or seeking healthcare services. |
| Patient Identifier / MRN | A hospital- or tenant-assigned identifier used to identify a patient record; scope and uniqueness rules must be defined. |
| Appointment | A scheduled booking for a patient to receive a service or consult with a provider. |
| Encounter / Visit | A documented interaction with healthcare services. The exact lifecycle and distinction between encounter and visit must be standardized. |
| OPD | Outpatient Department; services where the patient is not admitted as an inpatient. |
| IPD | Inpatient Department; care involving admission. |
| Queue / Token | A mechanism to organize and call patients for a service or provider. |
| Clinical Record | Structured and/or narrative healthcare information recorded during care, including notes, diagnoses, observations, and orders. |
| Tenant Context | Trusted server-side context identifying the tenant and relevant organizational scope for a request or operation. |
| RBAC | Role-Based Access Control; permissions assigned through roles. |
| Scope-Based Authorization | Authorization constrained by organizational, tenant, department, resource, or workflow scope. |
| Audit Event | A record of security- or business-relevant actions, including actor, action, target, timestamp, and applicable context. |
| Bounded Module | A cohesive domain area with defined responsibilities and explicit interfaces. |
| Modular Monolith | One deployable application organized into enforced internal modules and boundaries. |
| Distributed Service | An independently deployable component communicating through defined contracts. |
| ADR | Architecture Decision Record documenting context, decision, alternatives, and consequences. |
| NFR | Non-Functional Requirement; a quality or operational requirement such as performance, availability, security, or recoverability. |
