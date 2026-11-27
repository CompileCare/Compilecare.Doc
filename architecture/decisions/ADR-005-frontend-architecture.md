# ADR-005: Frontend Architecture

- **Status:** Proposed
- **Date:** 2026-11-23
- **Decision owner:** HeapCare Product & Engineering (proposed)
- **Review trigger:** Before implementation of the affected capability, or when assumptions/constraints materially change

## Context
CompileCare needs a responsive web experience for different hospital roles and workflows, with maintainable module boundaries and API contracts.

## Decision
- React + TypeScript is the proposed frontend baseline.
- Organize by product feature/module rather than a large shared component bucket.
- API clients and contracts are generated or maintained from versioned OpenAPI where practical.
- Keep business rules authoritative on the server; frontend logic supports interaction and validation.
- Shared UI components must remain domain-neutral; module-specific workflows live with their feature.
- Include accessibility, keyboard operation, responsive layouts, loading/empty/error/unauthorized states, and critical-flow tests.

## Open questions
Routing, state management, design system, localization strategy, frontend hosting, and deployment topology.
