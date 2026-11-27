# HeapCare migration report

**Date:** 2026-11-27  
**Decision:** HeapCare is the accepted product-facing brand.  
**Scope:** All 11 existing CompileCare repositories and their source, build, package, local-development and documentation identifiers.  
**Domain status:** No HeapCare domain has been purchased or configured.

## Executive status

The coordinated source/project migration is merged across all 11 repositories. The API, Web and public Site received additional technical-identifier PRs after the initial README/branding PRs. Post-merge API restore/build/test and EF checks passed; post-merge Web audit/unit/build/browser E2E/Docker checks passed. The existing GitHub organization and repository slugs remain CompileCare because the connected GitHub actions in this session do not expose the organization/repository rename administration operation.

No production database, Keycloak realm, persistent Docker volume, external email identity or production origin was renamed. Those are compatibility contracts and require separate, rehearsed transitions.

## Repository completion

| Repository (current GitHub slug) | HeapCare migration delivered |
|---|---|
| `CompileCare.Api` | HeapCare solution, project/folder names, namespaces, DbContext/model snapshot/migration namespaces, package references, API service identity, Docker/EF/CI paths; new database connection key with legacy fallback |
| `CompileCare.ApiClient` | HeapCare project/namespace/package identity; package publishing workflow |
| `CompileCare.Caching` | HeapCare project/namespace/package identity; package publishing workflow |
| `CompileCare.Database` | HeapCare database operations documentation and script descriptions; persisted database/role/backup identifiers preserved |
| `CompileCare.Local` | HeapCare launchers, local checkout directories, environment path variables, ephemeral container names, Prometheus labels, API telemetry service identity and CI checkout paths |
| `CompileCare.Mobile` | HeapCare scaffold documentation; no mobile project/app identifiers exist yet |
| `CompileCare.Observability` | HeapCare project/namespace/package identities and NuGet publishing |
| `CompileCare.OtpService` | HeapCare scaffold documentation; no service project/runtime identifiers exist yet |
| `CompileCare.Site` | HeapCare .NET solution/project/assembly/namespaces, Razor pages, styles, metadata, SVG brand mark and Docker build |
| `Compilecare.Doc` | HeapCare ADR, migration plan, roadmap and report |
| `CompileCare.Web` | HeapCare npm package name, API client module path/imports, product UI text and session-storage selection keys |

## Merged pull requests

- Initial coordinated branding/documentation PRs: API #101, API Client #4, Caching #6, Database #7, Local #25, Mobile #4, Observability #5, OTP #2, Site #11, Documentation #151, Web #68.
- Additional technical identifier PRs: API #101 (solution/runtime migration), Site #12 (project/assembly migration), Web #69 (frontend module/package/session-key migration).
- All listed PRs were merged. The repository slugs in the list above are still their current GitHub names.

## Verification evidence

### API
The final API PR head passed:
- .NET restore, build and unit/integration tests.
- EF Core pending-model-change detection.
- Docker integration verification using an ephemeral local HeapCare NuGet feed.
- PostgreSQL migration rehearsal against both the legacy API revision and the HeapCare API revision, including upgrade and fresh-database paths.

### Web
The final Web PR head passed:
- npm dependency audit, unit tests and production build.
- Browser end-to-end tests.
- Docker build and smoke/CSP validation.
- Legacy OIDC client ID was intentionally retained in the isolated browser test setup.

### Public Site
The final Site PR head passed:
- .NET build.
- Docker build and container health.
- Public route smoke tests.

### Local environment
- Shell/PowerShell syntax, secret-placeholder checks, Keycloak realm import, OpenTelemetry Collector configuration, Compose validation and observability pipeline smoke checks passed.
- A previous full-stack run passed the nine-service stack, persisted multi-tenant workflow, real Keycloak Authorization Code + PKCE browser acceptance and telemetry checks.
- The latest migration-triggered Local run exposed a Keycloak import startup defect: Keycloak requires an import filename matching the preserved realm ID (`compilecare-realm.json`), while the branding change had renamed the file to `heapcare-realm.json`. The realm ID, client IDs and issuer remain unchanged intentionally. The file and all Compose, configuration-script and CI references have now been restored to the compatible filename. The post-fix Local full-stack workflow is running; its final result is not yet available.

### Packages
HeapCare Observability, API Client and Caching package publishing workflows have all completed successfully on the current main branches. The API Client and Caching publish jobs both passed their Pack and Publish to GitHub Packages steps; the Observability publish job also passed. API and Docker CI use ephemeral local feeds to validate the actual package contract without depending on cross-repository package permissions.

**Production requirement:** grant the intended consumer repositories read access to the new private HeapCare packages, or configure a valid package-read credential in their deployment environments. Successful publishing and local-feed CI do not prove production consumer authorization.

## Compatibility retained intentionally

1. **GitHub coordinates:** The organization and repository slugs remain CompileCare. Repository rename and organization rename must be performed through authorized GitHub organization/repository administration, then remotes, badges, workflow repository coordinates, package URLs and cross-repository links must be updated.
2. **Domain/email:** No HeapCare domain, DNS, TLS certificate, support mailbox or hosting custom domain is assumed. Existing support email and public origins remain unchanged.
3. **Identity:** Existing Keycloak realm, issuer, client IDs and API audience remain unchanged. The frontend session-storage tenant/hospital selections migrate from legacy keys to HeapCare keys on first read without losing the selected context. A production realm/client migration needs a user migration and dual-client/issuer rollout.
4. **Database:** PostgreSQL database, role and schema names and EF migration IDs/history remain unchanged. The API prefers `ConnectionStrings:HeapCare` and falls back to legacy `ConnectionStrings:CompileCare`. Do not rewrite migration history or rename persisted data as a cosmetic operation.
5. **Local data:** Persistent Docker volume and Compose project names remain unchanged to keep existing local Keycloak/PostgreSQL data available. Ephemeral container names and telemetry labels use HeapCare.
6. **Legacy local scripts:** Launchers prefer `HEAPCARE_*` paths and local HeapCare checkout folders, while accepting old `COMPILECARE_*` variables and detecting existing CompileCare-named checkouts.

## Remaining closure checklist

- [ ] Rename GitHub organization and all 11 repositories using authorized GitHub administration; verify redirects and update remotes/workflows/links.
- [ ] Purchase/configure the chosen HeapCare domain only when the owner is ready; then configure DNS, HTTPS, email, canonical URLs, hosting origins, OIDC redirect URIs, CORS and CSP.
- [ ] Provision and validate HeapCare Keycloak realm/client/audience identities; migrate users/roles with a tested recovery path; keep legacy identity until the transition is verified.
- [ ] Grant private HeapCare package read access to production consumer repositories and validate deployment restore/publish using the intended credentials.
- [ ] Decide whether PostgreSQL database/role/schema names and persistent Docker volume names need to change. If yes, prepare backup, copy/restore, cutover and rollback procedures; never delete old volumes during a brand-only release.
- [ ] Post-merge API CI/EF checks, Web audit/unit/build/browser E2E/Docker checks and Local full-stack verification passed. Continue monitoring normal release CI after future changes.
- [ ] Run a final residual-reference audit and classify each remaining CompileCare reference as GitHub slug/owner, legacy fallback, identity/database/persisted-data compatibility or defect.
- [ ] Complete the post-fix Local full-stack run after restoring the Keycloak import filename to match the preserved `compilecare` realm.

## Closure rule

Do not declare the brand migration fully closed until the GitHub administrative rename and any chosen domain/identity changes are completed, private package access is verified for production consumers, and all post-merge checks pass. The source/project migration is merged; the external identity and administrative migration is not yet complete.
