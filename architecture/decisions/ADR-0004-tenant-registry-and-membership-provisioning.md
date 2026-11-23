# ADR-0004: Tenant registry and trusted membership provisioning

**Status:** Proposed  
**Date:** 2026-11-23  
**Scope:** CompileCare.Api — hospital-management platform

## Context

The API currently selects a tenant using `X-Tenant-ID`, then verifies the validated OIDC issuer/subject against an active `tenant_memberships` record. The tenant header is only a selector. Membership provisioning is an operational responsibility and there is no tenant registry or lifecycle model.

A membership record alone must not make an arbitrary tenant identifier a valid tenant. Tenant suspension/offboarding must be enforceable centrally, and provisioning must not be exposed as a tenant-user self-service operation without a separate, reviewed platform-admin authorization model.

## Decision (proposed)

1. Add a persisted `tenants` registry. A tenant has a generated UUID, normalized unique slug, bounded display name, lifecycle status (`Active`, `Suspended`, `Closed`), and UTC creation/update timestamps. The slug is an operational identifier, not an authentication credential.
2. Tenant lifecycle state is authoritative. Tenant-scoped requests must require the selected tenant to exist and be `Active` before membership is resolved. Suspended or closed tenants are denied without revealing tenant existence to unauthorised callers.
3. Memberships remain keyed by the exact validated OIDC `issuer` + `subject` + tenant UUID. Add a tenant FK and preserve unique membership identity within a tenant. Membership roles are allow-listed by CompileCare; unknown roles fail closed.
4. Initial tenant and membership provisioning is performed by restricted, trusted operational tooling—not a public endpoint and not a tenant user's token. Provisioning must be idempotent, validate inputs, create tenant and initial membership transactionally, and emit an audit event.
5. Any future platform-admin API must use a separate platform-operator authorization policy backed by a specifically trusted identity source/claim configuration. Tenant roles such as `HospitalAdmin` must never grant platform administration. This ADR does not define or enable that API.
6. Tenant suspension/closure must deny access immediately. Membership deactivation/role changes must be auditable and invalidate authorization state; do not cache membership decisions until a cache invalidation strategy is designed and tested.
7. Audit records must not contain patient payloads, credentials, bearer tokens, or raw request bodies. Record identifiers and outcome metadata only.

## Proposed registry constraints

- `Id`: UUID primary key.
- `Slug`: normalized, bounded, unique; lowercase ASCII letters, digits and hyphens; no leading/trailing hyphen.
- `DisplayName`: required, bounded.
- `Status`: required enum/string with an allow-list.
- `CreatedAtUtc`, `UpdatedAtUtc`: UTC.
- Memberships reference the tenant registry with restrictive delete behavior; tenants are deactivated/closed rather than deleted after provisioning.

## Provisioning contract

The trusted provisioning operation accepts a tenant slug/display name and initial operator identity (issuer, subject) plus an explicitly allow-listed initial role. It must:

- validate and normalize all values; never use email as the identity key;
- reject unknown tenant statuses and roles;
- be idempotent for a stable tenant slug and identity tuple, while rejecting conflicting requests;
- use a transaction for tenant + membership creation;
- create no tenant-owned facility or patient records implicitly;
- return only safe identifiers and status;
- emit a success/failure audit event with correlation reference and actor;
- be unavailable to ordinary tenant-scoped API callers.

## Required tests before implementation is considered complete

- Active tenant with active membership succeeds.
- Missing, suspended, and closed tenant are denied.
- Active tenant without membership, inactive membership, and unknown role are denied.
- Cross-tenant membership/foreign-key relationships cannot be created.
- Slug normalization and unique-conflict behaviour are deterministic.
- Provisioning is transactional and idempotent; a failure leaves no partial tenant/membership.
- Tenant lifecycle and membership role/status changes produce audit events.
- Provisioning cannot be invoked through ordinary tenant role claims.
- Migration install/upgrade and pending-model-change checks pass.

## Consequences

This introduces a central tenant lifecycle and prevents membership rows from authorizing access to unregistered or inactive tenants. Operational onboarding remains intentionally restricted until a separate platform-operator identity and administration surface is approved. It also creates a prerequisite for the next audit-log milestone.

## Open questions

- Who is the designated platform operator and what trusted identity/credential source is used?
- What is the tenant closure, retention, export and deletion policy?
- Can a customer tenant represent a hospital group, a single hospital, or both? How are branches scoped?
- Which approved roles can be assigned during initial provisioning, and is initial membership necessarily a `HospitalAdmin`?
