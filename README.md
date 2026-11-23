# CompileCare Documentation

CompileCare is an India-focused hospital technology platform intended to support hospitals and healthcare providers through an integrated, modular software suite.

> **Status:** Foundation documentation — draft v1. These documents establish direction and proposed decisions. They are not approved specifications or claims of regulatory compliance.

## Start here
- [Product vision and scope](product/vision-and-scope.md)
- [Hospital module catalogue](product/module-catalogue.md)
- [Initial roadmap](product/roadmap.md)
- [Architecture overview](architecture/overview.md)
- [Tenant model](architecture/multi-tenancy.md)
- [Security and privacy principles](security/security-and-privacy.md)
- [Engineering standards](engineering/standards.md)
- [Existing repository assessment](engineering/repository-assessment.md)

## Status
Product vision, module catalogue, roadmap: Draft. Architecture and tenancy: Proposed. Security: baseline guidance requiring legal/security review. Repository assessment: preliminary connector-based review.

Important decisions should be captured as ADRs under `architecture/decisions/`. Documents must distinguish confirmed decisions from proposals and assumptions.

CompileCare is healthcare/hospital-focused. The initial direction is a modular hospital information/management platform with India-specific workflows and integrations considered from the outset. Begin as a modular monolith unless an ADR approves otherwise.

## Contributing
Use feature branches, focused commits, reviewable pull requests, tests, and documentation updates. See [Engineering Standards](engineering/standards.md).
