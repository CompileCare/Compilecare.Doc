# ADR-003: Database and Data Ownership

- **Status:** Proposed
- **Date:** 2026-11-23

## Context
The platform has relational healthcare workflows, tenant isolation requirements, audit/history needs, reporting, and possible independently deployable services.

## Decision
- Use a relational database as the primary transactional store.
- SQL Server is a candidate, not yet approved.
- Each bounded module/service owns its schema and migration lifecycle.
- The initial modular core may share one database instance, but modules must not directly manipulate another module's tables; use module contracts.
- Independently deployed services must own their data and avoid runtime shared-table coupling.
- Database changes are versioned migrations with review, backups, and rollback/recovery planning.
- Tenant-aware keys, constraints, indexes, and query filters are required where applicable.

## Alternatives
SQL Server, PostgreSQL, and other relational options require an evaluation based on licensing, hosting, operational skills, client requirements, performance, and ecosystem.
