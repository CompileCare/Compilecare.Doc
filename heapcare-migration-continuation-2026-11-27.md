# HeapCare Brand Migration — Continuation Report

**Date:** 2026-11-27  
**Status:** GitHub organization rename verified; repository slug cleanup, source references and production identity cutover remain separately tracked.

## Verified GitHub organization state

The organization login is now `HeapCare` (organization ID `228320081`). The previous `CompileCare` login no longer owns the listed repositories. The organization rename preserved repository IDs and history.

## Repository inventory

All 11 repositories are present under `HeapCare`:

- `HeapCare.Api`
- `HeapCare.ApiClient`
- `HeapCare.Caching`
- `HeapCare.Database`
- `HeapCare.Local`
- `HeapCare.Mobile` — repository slug still needs rename from `CompileCare.Mobile`
- `HeapCare.Observability`
- `HeapCare.OtpService` — repository slug still needs rename from `CompileCare.OtpService`
- `HeapCare.Site` — repository slug still needs rename from `CompileCare.Site`
- `HeapCare.Web`
- `HeapCare.Doc`

The organization-scoped open-PR search and per-repository open-PR checks returned no open PRs at this checkpoint.

## Documentation reconciliation

The central `product/roadmap.md` and documentation README have been updated to use HeapCare product and repository naming. Historical migration notes may retain CompileCare where they describe previous identifiers, commit-era names, or compatibility contracts.

## Remaining cutover checklist

### GitHub repository identity
- [ ] Rename `CompileCare.Mobile` to `HeapCare.Mobile`.
- [ ] Rename `CompileCare.OtpService` to `HeapCare.OtpService`.
- [ ] Rename `CompileCare.Site` to `HeapCare.Site`.
- [ ] Verify README badges, clone URLs, workflows, links, release automation, deployment references, CODEOWNERS, webhooks, app installations, package access and local clones after the renames.

### Source, package and deployment references
- [ ] Search every repository for stale product-facing names, namespace/project/file paths, package IDs, URLs, CI variables, deployment labels and observability resource names.
- [ ] Distinguish true stale identifiers from intentional legacy compatibility identifiers before changing them.
- [ ] Rebuild and test affected repositories; validate private HeapCare NuGet package access from clean consumers.

### Identity and production services
- [ ] Inventory production Keycloak realms, clients, redirect URIs, issuer URLs, audience values, signing-key/JWKS consumers and service credentials.
- [ ] Plan any production issuer/client cutover without changing stable user subject identifiers or tenant membership mappings.
- [ ] Validate login, refresh, logout, PKCE and token validation end-to-end against production-equivalent configuration.
- [ ] Configure domain, DNS, email and certificates only after domain ownership and provider control are established. The `heapcare.in` domain has not been confirmed as purchased/configured.

### Data and infrastructure
- [ ] Inventory PostgreSQL database names, schemas, connection strings, backup jobs, restore jobs and deployment references.
- [ ] Inventory Docker Compose project names, named volumes, networks, container names and bind mounts.
- [ ] Do not rename or recreate persisted PostgreSQL/Docker volume identifiers until data ownership, migration/rollback and backup/restore behavior are verified.
- [ ] Verify observability labels, dashboards, alerts, log queries and telemetry resource attributes.

## Release gate

Do not declare the migration complete until repository slug cleanup, production identity/configuration review, private package restore, persisted-volume compatibility and full post-rename regression evidence are verified. No production database or user data migration is implied by this report.
