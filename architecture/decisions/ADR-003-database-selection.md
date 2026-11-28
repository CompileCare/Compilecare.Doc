# ADR-003: PostgreSQL and EF Core for the Initial Transactional Store

- **Status:** Accepted for the current API foundation
- **Date:** 2026-11-23
- **Decision owner:** HeapCare Product & Engineering
- **Review trigger:** Material change in hosting, customer deployment, licensing, performance, data residency, or operational requirements

## Context
The initial hospital platform needs a relational transactional store with referential integrity, tenant-scoped constraints, migrations, operational tooling, and mature .NET integration. The API repository has implemented PostgreSQL through Npgsql and EF Core 10.

## Decision
- PostgreSQL is the primary database provider for the current HeapCare API foundation.
- EF Core 10 is the primary ORM/migration mechanism.
- Tenant-aware keys, unique indexes, composite relationships, query filters, and persistence checks are required where applicable.
- Production schema changes must be versioned EF Core migrations. Do not use EnsureCreated for production.
- Module/schema ownership must remain explicit. A modular monolith may share a database instance initially; independently deployed services own their data and migrations.
- SQL/Dapper may be used for measured query-specific needs, without bypassing tenant authorization or integrity rules.

## Alternatives considered
- SQL Server: viable, but not selected for the current foundation.
- PostgreSQL: selected for the implemented initial platform.
- Database-per-tenant: remains a future deployment/isolation option, not the current default.

## Consequences
- API and integration tests use PostgreSQL/Npgsql to reflect production behaviour.
- The initial facility hierarchy migration and model snapshot are generated and validated in CI; migration-based clean-install tests and model-drift checks are implemented in the current Phase 3 PR.
- Production upgrade rehearsal, backup/restore validation, and migration runbooks remain required before a production rollout.
- Revisit only with documented workload, customer, licensing, deployment, or operational evidence.
