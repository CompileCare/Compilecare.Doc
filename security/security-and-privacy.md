# Security and Privacy Baseline

**Status:** Baseline guidance; not a legal opinion or compliance certification.

CompileCare may process sensitive patient and operational information. Controls must be designed, implemented, tested, and reviewed before production.

## Baseline
- Reviewed identity solution; secure credential handling if credentials are managed by CompileCare.
- Least privilege, role-based permissions, scope restrictions.
- Tenant and hospital/department isolation.
- TLS in transit and encryption at rest where supported/configured.
- Secrets outside source control.
- Audit authentication, sensitive record access, changes, exports, permissions, privileged actions.
- Input validation, output encoding, CSRF protections where applicable, rate limiting, abuse protection.
- Dependency/container scanning and least-privilege CI.
- Backups, restore tests, incident response, disaster recovery.
- Data minimization, purpose limitation, retention, controlled exports.
- No patient-identifiable information in logs, telemetry, test fixtures, or screenshots unless specifically approved and protected.

## India-specific review
Qualified reviewers must assess applicable Indian data-protection and healthcare requirements, including DPDP Act/rules as applicable, ABDM ecosystem policies/agreements, and other obligations. Documentation alone does not establish compliance.

## Clinical safety
Software supports clinical workflows. AI or automated outputs must not be represented as clinical decisions without an approved clinical safety process; human review/responsibility must remain explicit.

## Production security gate
Threat model and data-flow review; auth/authorization tests; cross-tenant tests; audit review; secret/dependency scans; backup/restore verification; incident/breach procedures; suitable penetration testing; privacy/legal review; production approval.
