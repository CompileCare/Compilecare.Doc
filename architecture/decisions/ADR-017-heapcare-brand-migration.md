# ADR-017: HeapCare brand migration

- **Status:** Proposed / in progress
- **Date:** 2026-11-27
- **Decision owner:** Product owner
- **Scope:** Product brand, source identifiers, repositories, packages, deployment and operational references

## Context

The product brand is changing from CompileCare to HeapCare. The domain `heapcare.in` has not been purchased or configured. The migration must not assume ownership of any HeapCare domain, email domain, DNS zone, certificate, OAuth callback origin or hosting custom domain.

The product remains an India-focused hospital and healthcare information/management platform. This is a coordinated identity migration, not a product rewrite. Existing functionality, tenant isolation, audit behavior, security controls, database data and Git history must be preserved.

## Decision

Use **HeapCare** as the proposed product-facing brand. Apply changes in controlled, reviewable stages. Do not perform blind global replacement of `CompileCare` / `compilecare`: many occurrences are technical identifiers, persisted values, migration history, package coordinates, OIDC identifiers, local developer state or external integration contracts.

## Migration rules

1. Preserve existing Git history and repository contents. Do not delete or recreate repositories to change names.
2. Do not rename the GitHub organization or repositories until an authorized GitHub organization/repository rename operation is available and all dependent references are inventoried.
3. Product-facing copy, public site metadata, documentation headings and UI brand presentation may adopt HeapCare first.
4. Treat .NET namespaces, assembly names, project names, NuGet package IDs, Docker image names, service names, environment-variable keys, configuration sections, OIDC realm/client IDs, database names/schemas, telemetry service names, dashboards, alert names, workflow names and deployment resource names as technical contracts. Rename them only in coordinated changes with compatibility, rollout and rollback plans.
5. Do not rewrite existing EF Core migration history or rename persisted database identifiers in place as a cosmetic change. Any database identifier transition requires a separately reviewed migration and verified backup/restore plan.
6. Publish new HeapCare package IDs only as an intentional versioned migration. Keep old CompileCare package IDs available during a documented compatibility window; update all consumers and verify restore/build/test before deprecation.
7. Preserve existing API routes and authentication contracts unless an explicitly versioned migration is approved. Update OIDC issuers, audiences, redirect URIs, CORS and CSP only after the new identity provider and exact HTTPS origins are configured.
8. Do not expose secrets or copy credential values into documentation. Existing GitHub Actions secrets and deployment variables must be inventoried by name and updated by an authorized operator; secret values must never be retrieved or committed.
9. Update source code, tests, generated clients, Docker/Compose, CI/CD, observability, docs, local launchers and cross-repository links together. Search for both `CompileCare` and `compilecare`, including case variants and URLs, before declaring completion.
10. Keep migration pull requests as drafts until all impacted repositories are updated and the relevant CI, Docker, integration, E2E and local-stack verification passes.

## Delivery sequence

### Stage 0 — Inventory and safe brand introduction
- Inventory all organization repositories, open PRs, package consumers, deployment targets, domains, OIDC configuration, secrets references and public assets.
- Introduce HeapCare in product-facing documentation and README descriptions.
- Clearly mark technical identifiers that have not yet changed.
- Record the migration plan and dependency matrix in this ADR.

### Stage 1 — Source and package identity
- Rename .NET namespaces, project/solution/assembly identifiers and package metadata in dependency order.
- Update package IDs and consumers with a compatibility release strategy.
- Update frontend package names, imports, API client identifiers and generated code.
- Update tests and fixtures. Preserve persisted values and historical migrations unless a separately approved data migration is required.

### Stage 2 — Runtime and local infrastructure
- Update Docker image tags, Compose project/service/container/network/volume names where safe, environment-variable names, configuration sections, Keycloak realm/client identifiers and local scripts.
- Provide a non-destructive local-state migration. Never delete local volumes or identity-provider state as part of a brand-only change.
- Update observability resource attributes, dashboards, scrape labels and alert routing without losing historical telemetry.

### Stage 3 — GitHub and delivery infrastructure
- Rename the organization and repositories only through GitHub's supported administrative UI/API after confirming organization-wide effects and redirect behavior.
- Update Git remotes, branch protections, CODEOWNERS, workflow permissions, package feeds, action references, badges, documentation links and automation references.
- Update CI/CD secret names and values through authorized settings workflows; never expose secret values.

### Stage 4 — External identity and production endpoints
- Only after domain purchase and DNS control, configure HTTPS, certificates, canonical URLs, email identity, OIDC issuer/audience/redirects, CORS/CSP and hosting custom domains.
- Keep old origins and identifiers during a measured transition where needed. Remove them only after consumer telemetry and owner confirmation.

### Stage 5 — Verification and closure
- Run full .NET restore/build/test, EF model-drift and migration rehearsal, package pack/restore tests, frontend lint/test/build, Docker builds, security checks, API integration tests, browser E2E and full local-stack verification.
- Search all repositories and documentation for residual brand references and classify each remaining occurrence as historical, compatibility or defect.
- Update deployment runbooks, roadmap, architecture diagrams, screenshots, support material and release notes.
- Close the migration only when every repository and external dependency has an owner, evidence and sign-off.

## Current execution state

- HeapCare product branding has been introduced in README files on a dedicated `migration/heapcare-branding` branch in each of the 11 repositories.
- Draft PRs are open in all 11 repositories. The changes are documentation-only and do not imply that source namespaces, package IDs, runtime configuration or repository names have been migrated.
- The available GitHub connector supports repository reads and file/branch/PR writes, but does not expose organization rename or repository rename operations. Those administrative renames remain blocked until performed through an authorized GitHub administrative workflow.
- The connector status lookup returned no status contexts for the initial README commits; CI must be verified in GitHub Actions before any PR is marked ready or merged.
- The domain `heapcare.in` has not been purchased. No new domain, email address, DNS, certificate or production origin is assumed.

## Risks and rollback

- **Package breakage:** use parallel package IDs and a compatibility window; rollback consumers to the existing package if restore/build or runtime validation fails.
- **Authentication lockout:** preserve old OIDC issuer/audience/redirects until the new identity configuration is tested end-to-end; maintain a documented rollback configuration.
- **Data loss:** do not rename/drop database objects or volumes as part of a brand change. Require verified backups and rehearsed restore before any data migration.
- **Broken deployments:** update deployment settings and code in a coordinated release; keep previous image tags and configuration available for rollback.
- **Broken repository links:** preserve Git history and verify redirects; update all cross-repository references after administrative renames.

## Consequences

HeapCare becomes the proposed product-facing identity immediately in the documentation-first stage. Technical identifiers remain CompileCare until each dependent contract can be migrated safely. This staged approach is intentionally slower than a global search-and-replace, but protects production behavior, patient data, tenant isolation and developer workflows.
