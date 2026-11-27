# HeapCare Brand Migration — Continuation Report

**Date:** 2026-11-27  
**Status:** Core source migration recorded as merged; administrative cutover and production validation remain open.

## Verified repository state

The connected GitHub account currently exposes these repositories under the GitHub organization `CompileCare`:

- `CompileCare.Api`
- `CompileCare.ApiClient`
- `CompileCare.Caching`
- `CompileCare.Database`
- `CompileCare.Local`
- `CompileCare.Mobile`
- `CompileCare.Observability`
- `CompileCare.OtpService`
- `CompileCare.Site`
- `CompileCare.Web`
- `Compilecare.Doc`

The repositories are still hosted under the old organization and repository names. The GitHub connector available in this session supports repository content, branches, commits and pull requests, but does not expose an organization-rename or repository-rename operation. Therefore, no GitHub administrative rename is claimed as completed here.

The connected organization-scoped pull-request search returned no open PRs at the time of this check.

## Migration state carried forward

The preceding migration work log records 11 source/project migration PRs merged, including .NET solution and namespace updates, NuGet package identity changes, React/Web and site updates, database documentation, and the local launcher update. It also records successful checks for the nine-service Docker stack, multi-tenant flows, PKCE, telemetry and local launchers. These are carried-forward results; they were not all re-executed in this continuation.

## Important documentation drift found

The current `product/roadmap.md` still uses CompileCare as the product name throughout its title, status and implementation references. The migration ADR and dated migration report were not found at the repository root under the expected filenames. Locate their actual paths and reconcile them before calling documentation migration complete.

## Remaining cutover checklist

### GitHub identity and source control
- [ ] Rename the GitHub organization from `CompileCare` to the approved HeapCare organization identity, subject to GitHub availability and administrative permissions.
- [ ] Rename all 11 repositories consistently, including `Compilecare.Doc` casing.
- [ ] Update repository references, badges, clone URLs, package source URLs, CODEOWNERS, workflows, release automation, deployment manifests and external integrations.
- [ ] Revalidate GitHub Actions secrets, environments, branch protection, webhooks, app installations and package permissions after the rename.
- [ ] Confirm redirects and update all local clones/remotes and external consumers.

### Identity and production services
- [ ] Inventory production Keycloak realms, clients, redirect URIs, issuer URLs, audience values, signing-key/JWKS consumers and service credentials.
- [ ] Plan the production issuer/client cutover without changing stable user subject identifiers or tenant membership mappings.
- [ ] Validate login, refresh, logout, PKCE and token validation end-to-end against the production-equivalent configuration.
- [ ] Verify private NuGet feed permissions and package restore from every consuming repository and clean build environment.

### Data and infrastructure
- [ ] Inventory PostgreSQL database names, schemas, connection strings, backup jobs, restore jobs and deployment references.
- [ ] Inventory Docker Compose project names, named volumes, networks, container names and bind mounts.
- [ ] Do not rename or recreate persisted PostgreSQL/Docker volume identifiers until data ownership, migration/rollback and backup/restore behavior are verified. Preserve existing data.
- [ ] Verify observability labels, dashboards, alerts, log queries and telemetry resource attributes use the intended HeapCare identity.

### Product and documentation
- [ ] Update `product/roadmap.md` and all product-facing documentation to use HeapCare consistently.
- [ ] Retain historical CompileCare references only where required for migration history, GitHub redirect/compatibility notes, or immutable package/version provenance.
- [ ] Search every repository for stale product names, namespaces, package IDs, URLs, secrets names and deployment identifiers.
- [ ] Publish a final migration matrix with repository, commit/PR, CI evidence, runtime evidence and remaining exceptions.

## Release gate

Do not declare the migration complete until the GitHub organization/repository cutover, production identity configuration, private package restore, persisted-volume compatibility and full regression evidence are all verified. No production database or user data migration is implied by this report.
