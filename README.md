# CompileCare Documentation

CompileCare is an India-focused hospital information and management platform for hospitals, clinics, and healthcare provider organizations. This repository is the product, architecture, workflow, and engineering documentation source of truth.

> **Status:** Foundation documentation — draft v1. Product scope and several architecture decisions remain proposals until reviewed and approved with hospital stakeholders and technical owners.

## Start here
- [Product vision and scope](product/vision-and-scope.md)
- [MVP scope and non-goals](product/mvp-scope.md)
- [Glossary](product/glossary.md)
- [Hospital module catalogue](product/module-catalogue.md)
- [Delivery roadmap and current implementation status](product/roadmap.md)
- [Requirements lifecycle](product/requirements-lifecycle.md)
- [Patient registration workflow](product/workflows/patient-registration.md)
- [Non-functional requirements](product/non-functional-requirements.md)
- [Architecture overview](architecture/overview.md)
- [API conventions and error contracts](architecture/api-conventions.md)
- [Tenant model](architecture/multi-tenancy.md)
- [Architecture Decision Records](architecture/decisions/)
- [Security and privacy principles](security/security-and-privacy.md)
- [Integration testing strategy](engineering/integration-testing-strategy.md)
- [Engineering standards](engineering/standards.md)
- [Definition of Done](engineering/definition-of-done.md)
- [Existing repository assessment](engineering/repository-assessment.md)

## Delivery status
The agreed delivery sequence is:
1. **Phase 1 — Backend foundation:** API foundation and shared observability. Completed and merged.
2. **Phase 2 — Developer platform:** Completed and merged. CompileCare.ApiClient now provides named/typed clients, injectable token/tenant/correlation providers, safe-method resilience, and RFC 9457 error parsing. The API exposes consistent facility Problem Details errors and validated correlation IDs. Integration strategy and conventions are documented.
3. **Phase 3 — Database and security:** Active. PostgreSQL/EF Core migrations and the first OIDC/JWT tenant-membership and facility-RBAC slice are merged in API PRs [#9](https://github.com/CompileCare/CompileCare.Api/pull/9) and [#10](https://github.com/CompileCare/CompileCare.Api/pull/10). Tenant/membership provisioning, append-only audit events, broader asynchronous/cross-tenant tests, and production operational controls remain.
4. **Phase 4 — Core healthcare domain:** Hospital hierarchy CRUD is implemented; tenant/organization onboarding, staff, and patient registration remain.
5. **Phase 5 — UI in parallel:** API conventions are now stable enough to begin the design system and admin shell alongside Phase 3. Build facility/staff/bed screens as their contracts become available; do not wait for all backend modules to finish.

The roadmap records what is implemented versus planned. A merged repository or passing build does not imply that a complete hospital workflow or production readiness has been delivered.

## Documentation governance
Important decisions must be recorded as ADRs. Documents distinguish implemented facts, approved decisions, proposals, assumptions, and open questions. Update affected requirements, workflows, API contracts, and tests when scope or behaviour changes.

CompileCare is exclusively a hospital and healthcare systems product. Unrelated products and repositories are outside this documentation and delivery scope.

## Contributing
Use short-lived branches, focused commits, reviewable pull requests, automated tests, and documentation updates. Follow [Engineering Standards](engineering/standards.md) and [Definition of Done](engineering/definition-of-done.md).
