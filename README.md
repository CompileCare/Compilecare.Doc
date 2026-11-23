# CompileCare Documentation

CompileCare is an India-focused hospital technology platform intended to support hospitals and healthcare providers through an integrated, modular software suite.

> **Status:** Foundation documentation — draft v1. These documents establish direction and proposed decisions. They are not approved specifications or claims of regulatory compliance.

## Start here
- [Product vision and scope](product/vision-and-scope.md)
- [Hospital module catalogue](product/module-catalogue.md)
- [Initial roadmap](product/roadmap.md)
- [Requirements lifecycle](product/requirements-lifecycle.md)
- [Patient registration workflow](product/workflows/patient-registration.md)
- [Non-functional requirements](product/non-functional-requirements.md)
- [Architecture overview](architecture/overview.md)
- [Tenant model](architecture/multi-tenancy.md)
- [Architecture Decision Records](architecture/decisions/)
- [Security and privacy principles](security/security-and-privacy.md)
- [Engineering standards](engineering/standards.md)
- [Definition of Done](engineering/definition-of-done.md)
- [Existing repository assessment](engineering/repository-assessment.md)

## Status
Product vision, module catalogue, roadmap, workflows, and non-functional requirements: Draft. Architecture and tenancy: Proposed. Security: baseline guidance requiring legal/security review. Repository assessment: preliminary connector-based review.

Important decisions should be captured as ADRs under `architecture/decisions/`. Documents must distinguish confirmed decisions from proposals and assumptions.

CompileCare is healthcare/hospital-focused. The proposed target is a distributed system with clear domain boundaries, independently deployable services where justified, and a modular core to avoid premature microservice complexity. See ADR-001.

## Contributing
Use branches following [Engineering Standards](engineering/standards.md), focused commits, reviewable pull requests, tests, and documentation updates.
