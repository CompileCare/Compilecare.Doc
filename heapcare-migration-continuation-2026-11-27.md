# HeapCare Brand Migration — Continuation Report

**Date:** 2026-11-27  
**Status:** GitHub organization and all 11 repository slug renames verified; source-reference cleanup and production identity cutover remain separately tracked.

## Verified GitHub organization state

The organization login is now `HeapCare` (organization ID `228320081`). The previous `CompileCare` login no longer owns the listed repositories. The organization rename preserved repository IDs and history.

## Repository inventory

All 11 repositories are present under `HeapCare`:

- `HeapCare.Api`
- `HeapCare.ApiClient`
- `HeapCare.Caching`
- `HeapCare.Database`
- `HeapCare.Local`
- `HeapCare.Mobile`
- `HeapCare.Observability`
- `HeapCare.OtpService`
- `HeapCare.Site`
- `HeapCare.Web`
- `HeapCare.Doc`

The organization-scoped open-PR search and per-repository open-PR checks returned no open PRs at this checkpoint.

## Documentation reconciliation

The central `product/roadmap.md` and documentation README have been updated to use HeapCare product and repository naming. Historical migration notes may retain CompileCare where they describe previous identifiers, commit-era names, or compatibility contracts.

## Remaining cutover checklist

### GitHub repository identity
- [x] Rename `CompileCare.Mobile` to `HeapCare.Mobile`.
- [x] Rename `CompileCare.OtpService` to `HeapCare.OtpService`.
- [x] Rename `CompileCare.Site` to `HeapCare.Site`.
- [ ] Verify README badges, clone URLs, workflows, links, release automation, deployment references, CODEOWNERS, webhooks, app installations, package access and local clones after the renames.

### Source, package and deployment references
- [x] Search every repository for stale product-facing names, namespace/project/file paths, package IDs, URLs, CI variables, deployment labels and observability resource names.
- [x] Distinguish true stale identifiers from intentional legacy compatibility identifiers before changing them.
- [x] Rebuild and test affected repositories; validate private HeapCare NuGet package access from clean consumers (verified via `HeapCare.Api#102`, `Observability#6`, `Local#26`/`#27`, `Web#70`, `Site#13`, `ApiClient#5`, `Caching#7`; see [qa/post-migration-full-platform-qa-2026-11-27.md](qa/post-migration-full-platform-qa-2026-11-27.md)).

### Identity and production services
- [ ] Inventory production Keycloak realms, clients, redirect URIs, issuer URLs, audience values, signing-key/JWKS consumers and service credentials.
- [ ] Plan any production issuer/client cutover without changing stable user subject identifiers or tenant membership mappings.
- [ ] Validate login, refresh, logout, PKCE and token validation end-to-end against production-equivalent configuration.
- [ ] Configure domain, DNS, email and certificates only after domain ownership and provider control are established. The `heapcare.in` domain has not been confirmed as purchased/configured.

### Data and infrastructure
- [ ] Inventory PostgreSQL database names, schemas, connection strings, backup jobs, restore jobs and deployment references.
- [ ] Inventory Docker Compose project names, named volumes, networks, container names and bind mounts.
- [ ] Do not rename or recreate persisted PostgreSQL/Docker volume identifiers until data ownership, migration/rollback and backup/restore behavior are verified.
- [x] Verify observability labels, dashboards, alerts, log queries and telemetry resource attributes (local stack verified with `HeapCare.Api` Jaeger spans + `heapcare-logs-*` Kibana data view).

## Release gate

Do not declare the migration complete until repository slug cleanup, production identity/configuration review, private package restore, persisted-volume compatibility and full post-rename regression evidence are verified. No production database or user data migration is implied by this report.


## Engineering continuation — 2026-11-27 (latest checkpoint)

### Merged since the previous checkpoint
- `HeapCare.Local#29` — expanded the isolated PostgreSQL/Keycloak acceptance harness to exercise all five persisted tenant roles (HospitalAdmin, FacilityManager, FacilityViewer, PatientRegistrar, ClinicalUser) plus a separate PlatformOperator identity. The test verifies server-derived permission claims and role-appropriate hospital/patient access boundaries. Synthetic test identities/data are ephemeral; this does not modify production identity or tenant data. Both Local CI jobs passed in run [#45999845006](https://github.com/HeapCare/HeapCare.Local/actions/runs/45999845006), including the nine-service Docker stack and telemetry verification.
- `HeapCare.Doc#157` — corrected the audit-governance ADR's HeapCare branding and current API reference.
- `HeapCare.Doc#158` — recorded the new persisted-role acceptance evidence and retained the distinction between automated authorization checks and manual six-role accessibility acceptance.
- `HeapCare.Doc#159` — aligned audit pagination/export documentation with the current implementation: UTC query window capped at 90 days, page size capped at 100, deterministic offset ordering by OccurredAtUtc and Id, and export capped at 31 days / 5,000 rows. Keyset pagination remains a future scale evaluation, not a current feature.
- `HeapCare.Api#103` — merged the user-visible audit CSV filename correction to `heapcare-audit-events.csv`, with an integration assertion. PR checks passed: Release build/tests/model-drift, Docker integration and PostgreSQL migration rehearsal. Merge commit: `3903df4c9e7260e51c7358bbf2b251e6da62f946`.

### Current CI and pull-request state
- The HeapCare organization-wide open-PR search returned no open pull requests at this checkpoint.
- The post-merge API main-branch CI run [#46003258837](https://github.com/HeapCare/HeapCare.Api/actions/runs/46003258837) is still running. The PR-level checks passed, but the main-branch run must complete before its status is recorded as green.

### Remaining acceptance and release gates
- The automated persisted-role matrix is additional evidence, not full endpoint-by-endpoint authorization certification. Continue positive and negative tenant/hospital/department boundary coverage for each sensitive capability.
- Phase 5 still requires manual authenticated acceptance for all six roles, including keyboard-only operation, focus visibility/order, dialogs, responsive reflow/zoom, contrast and screen-reader behavior. Do not claim WCAG conformance without qualified evaluation.
- Audit governance remains constrained by unapproved external decisions: authorized audit-reader roster and scope, retention/archive/legal-hold periods, export purpose/approval and emergency-access review. Do not broaden audit access or export behavior without hospital/security/privacy/legal approval.
- Production readiness remains on hold pending production signing-key and secret custody/rotation, backup/PITR and isolated restore evidence, incident-response ownership, privileged-access review, monitoring/alert validation and privacy/legal review.
- OTP remains on security hold. Do not revive the closed unmerged implementation branch until the historically committed SMTP credential is confirmed revoked/rotated and a new implementation passes security review.
- Existing Keycloak realm/client/audience, PostgreSQL database/role, connection-string fallback and persistent Docker volume identifiers remain compatibility contracts. No domain, DNS, mail, certificate or production origin is assumed configured.

This checkpoint records engineering evidence only; it is not a production approval or a declaration that the brand/identity migration is fully closed.


## Engineering continuation — 2026-11-27 (follow-up checkpoint)

### Platform-operator authorization regression
- `HeapCare.Api#104` — merged commit `3f58326b3168035bde036a082766080409ad9c91`. Added a PostgreSQL-backed integration regression proving that a tenant HospitalAdmin cannot obtain platform tenant-administration access by supplying forged `role`, `platform_operator` or permission claims in the bearer token. Authorization continues to depend on the validated identity and active persisted PlatformOperator record.
- All three reported PR checks passed for head `983ab483d33b6a8bf32c0945d20ce1eed8465199`: two build-and-test checks and Docker integration verification. See [PR #104](https://github.com/HeapCare/HeapCare.Api/pull/104).
- Post-merge API main-branch CI run [#46005866195](https://github.com/HeapCare/HeapCare.Api/actions/runs/46005866195) was observed initializing its PostgreSQL test containers at this checkpoint; its final outcome is intentionally not inferred here.

### Repository metadata cleanup
- A read-only inventory found legacy/placeholder descriptions still present on several renamed repositories (including `CompileCare.Api` and `CompileCare Web Site`). This is repository metadata, separate from source code and runtime compatibility identifiers. No metadata mutation has been made in this checkpoint; the available connected GitHub write actions do not expose repository-description updates.

### Current release boundaries
- The authorization regression is automated evidence for this platform-operator escalation path only; it is not a complete endpoint-by-endpoint authorization certification.
- OTP remains on security hold, and production identity, data-volume, DNS/domain and release-governance gates above remain open.
