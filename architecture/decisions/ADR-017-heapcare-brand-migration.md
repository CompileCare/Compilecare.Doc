# ADR-017: HeapCare brand migration

- **Status:** Accepted / in progress
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

### Completed and merged
- HeapCare product branding is present in the public website and authenticated Web workspace. The public Site CI and Web CI checks passed for the branding changes.
- The HeapCare.ApiClient solution, project/test project names, namespaces, public types, package ID and source paths have been migrated and merged (PR #4). Its CI build/test passed.
- The HeapCare.Caching solution, library/sample/test projects, namespaces, public extension method, package ID, Dockerfile paths, Compose references, CI paths and README examples have been migrated and merged (PR #6). Restore/build/test/pack passed.
- The HeapCare.Observability solution, core/ASP.NET Core/worker/test project names, namespaces, extension methods, package IDs, sample API references and project paths have been migrated and merged (PR #5). Build, tests, code-format validation and the GitHub Packages publish job passed. The new HeapCare observability packages were published at version 1.0.1.
- Local macOS/Windows start/stop launcher filenames and user-facing instructions were renamed to HeapCare. Local tooling validation, Keycloak import smoke tests, Docker Compose validation and observability pipeline smoke tests passed on the latest completed validation run.

### In progress
- The API migration branch now consumes HeapCare.Observability and HeapCare.Observability.AspNetCore 1.0.1, and its NuGet source mapping recognizes both old and new package families. API build, migration rehearsal and integration verification are being rerun.
- The Local full-stack workflow is being updated to test the API migration branch against the new HeapCare observability packages. The prior full-stack run failed because its pack step still referenced the old observability project paths; this CI integration defect is being corrected.
- The Site and Web PRs remain open for review/merge despite their branding CI checks passing.
- The API, Local and documentation PRs remain open. The API's broad source/project folder rename is not yet complete; a proposed automated branch-push job was rejected by the repository's write restrictions and was removed rather than masking the failure. A repeatable migration script is retained for controlled execution, but it has not been applied to the API source tree.

### Remaining compatibility contracts and blockers
- GitHub organization/repository names remain CompileCare; the available connector does not expose administrative rename operations. No repository has been deleted or recreated.
- The domain heapcare.in has not been purchased or configured. No DNS, email, HTTPS certificate, production URL, OAuth callback, CORS or CSP origin is assumed.
- The local Keycloak realm (compilecare), OIDC client/audience IDs, existing issuer, database names/roles, EF migration history, connection-string key and persistent Docker volume names remain intentionally unchanged. Renaming these without migrating users, persisted data and local state would risk authentication lockout or data loss.
- HeapCare.ApiClient and HeapCare.Caching package identities have been migrated in source, but their publication/consumer rollout still needs verification. The Observability package publication is confirmed.
- CI status is evaluated against each latest PR head; earlier green runs do not certify later commits.

## Risks and rollback

- **Package breakage:** publish new package IDs, verify restore/build of every consumer, and keep old IDs available through a documented compatibility window.
- **Authentication lockout:** preserve old OIDC issuer/audience/redirects until a tested identity migration provisions the HeapCare realm/clients and migrates users and roles.
- **Data loss:** do not rename/drop database objects or Docker volumes as a cosmetic change. Require verified backups and rehearsed restore before any data migration.
- **Broken deployments:** update deployment settings and code in a coordinated release; retain previous image tags and configuration for rollback.
- **Broken repository links:** preserve Git history and verify redirects; update cross-repository references only after the administrative organization/repository rename.

## Consequences

HeapCare is now the product-facing brand, and three shared .NET libraries have been migrated and merged with their CI checks passing. The main API and local integration workflows are still in progress. Technical compatibility identifiers remain until their dependent contracts can be migrated safely. This staged approach avoids breaking package consumers, authentication, patient data, tenant isolation and existing local developer environments.
## Risks and rollback

- **Package breakage:** use parallel package IDs and a compatibility window; rollback consumers to the existing package if restore/build or runtime validation fails.
- **Authentication lockout:** preserve old OIDC issuer/audience/redirects until the new identity configuration is tested end-to-end; maintain a documented rollback configuration.
- **Data loss:** do not rename/drop database objects or volumes as part of a brand change. Require verified backups and rehearsed restore before any data migration.
- **Broken deployments:** update deployment settings and code in a coordinated release; keep previous image tags and configuration available for rollback.
- **Broken repository links:** preserve Git history and verify redirects; update all cross-repository references after administrative renames.

## Consequences

HeapCare becomes the proposed product-facing identity immediately in the documentation-first stage. Technical identifiers remain CompileCare until each dependent contract can be migrated safely. This staged approach is intentionally slower than a global search-and-replace, but protects production behavior, patient data, tenant isolation and developer workflows.


## Progress update — 2026-11-27

### Completed and merged
- Documentation and roadmap now use HeapCare as the product brand.
- HeapCare Observability, API Client and Caching project identities are migrated; NuGet packages were published to the existing CompileCare GitHub Packages owner namespace.
- Public Site project/solution paths, .NET namespaces, assembly identity, UI branding, logo asset and Docker build paths are migrated and merged.
- Local developer environment scripts, container identities, Prometheus scrape labels, telemetry service name, Compose build paths and CI checkouts now prefer HeapCare local names. Existing local checkouts are detected and reused.
- Database operations documentation and scripts use HeapCare while preserving the actual database/role and historical backup/restore identifiers.
- Mobile and OTP repositories are currently scaffolds; their product-facing documentation is branded HeapCare, with no app/service code identifiers yet to migrate.
- Web workspace technical migration PR #69 is merged: package name, API client module path/imports and session-storage selection keys are migrated. Existing session values are moved from legacy keys on first read.
- API technical migration PR #101 is merged: solution/project/folder names, namespaces, EF context/snapshot namespaces, package references, Docker paths, API service identity and connection-string key support are migrated. The API supports new ConnectionStrings:HeapCare and the legacy ConnectionStrings:CompileCare fallback.

### Verification evidence
- HeapCare API .NET build/unit tests and package-feed restore have passed on a recent branch head.
- PostgreSQL migration rehearsal has passed: the legacy API revision and HeapCare API revision were built from source with isolated package feeds, then upgrade and fresh-database migrations were exercised.
- Local run #45978681883 passed the nine-service stack, real Keycloak-issued identities, persisted multi-tenant workflows, OIDC/PKCE browser acceptance and telemetry. A follow-up run #45979293490 is validating the merged API main branch and HeapCare.Local checkout directory.
- API, Web and Site migration PRs are merged. Latest post-merge API, Web and Site CI checks are green.

### Remaining external/admin work
- The GitHub organization and repository slugs are still CompileCare. The connected GitHub actions available in this session do not expose the organization/repository rename administration operation. Renames must be done through authorized GitHub organization settings; after that, update remotes, workflow repository coordinates, package owner URLs, badges and links.
- The domain has not been purchased or configured. Existing support email and public origins remain unchanged.
- Production Keycloak realm, issuer, client IDs and API audience remain compatibility identifiers. The Web local-storage selection key migration is complete, but the identity realm/client migration requires an explicit user/account migration plan and dual-issuer/dual-client rollout.
- PostgreSQL database/role/schema names, EF migration IDs/history and persistent Docker volume names remain unchanged to protect existing data. A separate, rehearsed data migration is required before changing them.
- New HeapCare NuGet packages are published under the existing GitHub Packages owner. Production consumers need package read access granted to their repositories or a valid package-read credential; CI uses isolated local package feeds and does not prove production package authorization.
- The final residual-reference audit and all post-merge main-branch, Docker, integration, E2E and deployment verification remain mandatory before declaring the brand migration complete.


### Merge completion update
All 11 original repository migration PRs are merged, and the additional technical identifier PRs for the API, Site and Web are also merged. The API and Web solution/source/package identities and the Site project/assembly identity now use HeapCare on main. The Local main branch now prefers HeapCare local checkout paths and HEAPCARE_* environment variables while detecting old checkout folders. The only remaining CompileCare identifiers are classified compatibility or administrative references described above, not overlooked source-project names.


## Execution refresh — 2026-11-27 15:30 IST

The source/project migration PRs are merged across the 11 existing repositories; an open-PR audit currently returns no open PRs in any of them. Main-branch tree inspection confirms no CompileCare-named file paths in API, API Client, Caching, Database, Mobile, Observability, OTP, Site, documentation or Web. Local retains one intentionally legacy-named Keycloak realm import file, `local/keycloak/compilecare-realm.json`, because its internal realm ID remains `compilecare` and Keycloak requires the import filename to match the realm ID.

Latest CI runs are green for API, Web, public Site, Database documentation, Observability, API Client and Caching. The API Client and Caching GitHub Packages publish jobs both passed; Observability publishing also passed. Production package read permissions still need validation for deployment consumers.

The most recent earlier Local full-stack run failed because the realm import file was renamed to a filename that did not match its preserved realm ID. The filename and all Compose, script and CI references have been corrected. Post-fix Local run #45978681883 passed both jobs, including the complete nine-service Docker stack, persisted multi-tenant workflows, real Keycloak-issued identities, PKCE browser acceptance and telemetry. Follow-up run #45979293490 is validating the merged API main branch and HeapCare.Local checkout directory.

The connected GitHub actions available in this session do not provide organization or repository rename administration. The existing organization and repository slugs therefore remain unchanged. No HeapCare domain/DNS/email/certificate/production origin is configured. Production OIDC identity and persisted database/volume identifiers remain compatibility contracts pending a separately rehearsed migration.

## Execution correction — 2026-11-27

The following current state supersedes earlier progress paragraphs in this ADR that described the GitHub administrative rename as pending:

- The GitHub organization is **HeapCare** and all 11 repository names use the HeapCare brand. Repository IDs and history were preserved.
- CI repository coordinates and NuGet feed owner references were corrected in the API, API Client and Caching workflows. The post-fix API, API Client, Caching and Observability main-branch runs (#45984268087, #45984273706, #45984278571 and #45984332659) all passed.
- API Client and Caching publication jobs to GitHub Packages passed in runs #45984273706 and #45984278571. Actual production-consumer package-read authorization remains to be verified from consuming identities.
- Web #45983835225, Local tooling #45983816999 and Database backup/restore #45970305638 also passed. The Site run #45972538371 passed.
- No open PRs were returned by the HeapCare organization-wide open-PR search at this checkpoint.

The OTP service is intentionally not treated as a completed implementation: its main branch is a scaffold, and the earlier implementation PR #1 was closed without merge because of unresolved security blockers. The PR also recorded a previously committed SMTP credential that requires revocation/rotation. The OTP feature branch must not be revived until the credential owner confirms rotation and a new security review, implementation and regression suite pass.

The domain `heapcare.in` remains unpurchased/unconfigured. Existing Keycloak realm/client/audience IDs, PostgreSQL names/roles, EF migration history and persistent volume names remain compatibility contracts. Their migration requires a separately rehearsed, reversible identity/data transition.


## Execution refresh — 2026-11-27 (latest verified repository work)

### Completed since the previous checkpoint
- Web PR #71 merged after both Web CI runs passed. Membership list and mutation failures now have regression coverage for accessible alert announcements and preservation of the OIDC subject on rejected creation.
- Web PR #72 merged; the browser-test guide now uses HeapCare product-facing branding while preserving compatibility-sensitive OIDC identifiers.
- Web PR #73 merged after its CI checks passed; membership lifecycle confirmation tests assert the named accessible group and confirm/cancel actions.
- API PR #111 merged after API CI and Docker integration verification passed. PostgreSQL-backed coverage now verifies deterministic audit ordering across page boundaries when events share the same timestamp.
- API PR #112 merged after API CI and Docker integration verification passed. Invalid, empty-GUID and repeated `X-Tenant-ID` values are verified to return HTTP 400 Problem Details.
- Site PR #14 merged after Site CI passed. The legacy `support@compilecare.com` address is no longer published by the HeapCare public contact page.
- Site PR #15 merged after Site CI passed. The page now accurately says the official HeapCare support contact is not yet published and does not direct visitors to an unsupported repository issue channel.
- Docs PR #168 records the source-verified tenant-to-hospital hierarchy and leaves group/branch semantics explicitly undecided.
- Docs PR #169 refreshes the Phase 3–5 pending-task register and execution controls.
- Docs PR #170 updates ADR-012 decision-owner branding without changing its proposed status or unresolved privacy/security gates.
- Docs PR #171 updates decision-owner labels in ADR-001 through ADR-006 and ADR-008 through ADR-011, preserving historical implementation references and technical identifiers.

### In progress at this checkpoint
- API PR #113 adds PostgreSQL-backed CSV export regression coverage for all spreadsheet formula prefixes (`=`, `+`, `-`, `@`) and leading whitespace before a formula. Its API CI and Docker integration checks must pass before merge.
- Phase 3 release readiness remains open: pilot-approved audit reader/role mapping, purpose and export governance, retention/deletion/legal-hold policy, representative-data migration and rollback rehearsal, provider-backed encrypted backup/PITR and isolated restore evidence, production key/secret custody and rotation, and qualified privacy/legal review are not established by repository CI.
- Phase 4 remains partial. Existing tenant-owned multiple-Hospital support does not define a HospitalGroup/Branch business model. Do not add that hierarchy until the product owner and pilot hospitals approve the model and migration requirements.
- Phase 5 still requires authenticated six-role acceptance, keyboard/focus and responsive/zoom evaluation, and an actual screen-reader pass. Automated CI is not a substitute for manual accessibility evaluation and does not establish WCAG 2.2 AA conformance.

### External identity and compatibility status
- HeapCare is the GitHub organization and product-facing brand. Repository IDs and Git history were preserved during renames.
- The official HeapCare support email/domain is not configured. `heapcare.in` remains unpurchased/unconfigured; no DNS, certificate, production origin, OAuth callback, CORS or CSP origin is assumed.
- The old support email was removed from the public contact page rather than replaced with an invented address.
- Keycloak realm/client/audience IDs, PostgreSQL database/role names, EF migration history, legacy connection-string fallback and persistent Docker volume names remain compatibility contracts. Any transition requires an independently reviewed, reversible migration and verified backup/restore.
- OTP remains on security hold pending confirmation that the historical SMTP credential has been revoked/rotated and a fresh security review is complete.

This refresh supersedes earlier status sentences in this ADR where they conflict with the verified merges and checks listed above. Historical migration evidence and old PR links are retained intentionally.
