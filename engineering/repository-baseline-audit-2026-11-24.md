# CompileCare Repository Baseline Audit

**Date:** 2026-11-24  
**Status:** Initial connector-based audit; follow-up source/build review is required.  
**Scope:** All 10 CompileCare repositories, default-branch trees, branch inventory, PR states, selected source/configuration files, and architecture documentation.

This audit is based on GitHub connector inspection. A local checkout, full builds, tests, dependency/security scans, and complete line-by-line review were not performed as part of this baseline.

## Executive summary

- 10 repositories exist in the CompileCare organization.
- 46 branches were observed in the initial snapshot, including 10 `main` branches; 36 non-main branches remained. The audit branch created for this report is excluded from that snapshot.
- 25 pull requests were observed across the repositories: 21 merged, 2 closed without merge, and 2 open.
- The two open PRs are CompileCare.Mobile #1 and CompileCare.OtpService #1.
- The API is the most complete application repository: .NET 10, PostgreSQL/EF Core 10, tenant context, OIDC/JWT membership-based authorization, facility hierarchy CRUD, pagination, migrations, and metadata-only append-only audit.
- ApiClient and Observability have reusable foundations. Site and Docs have meaningful initial content.
- Caching, Mobile, Web, Database and OTP have work on non-main branches or are still skeletons on `main`. Their branch contents must not be assumed to be production-ready or delivered.
- Branch cleanup is significant, but deletion is not available through the currently connected GitHub actions. Branches are classified below for later cleanup through GitHub UI/CLI or a capability that supports deletion.

## Repository inventory

| Repository | Main state | Main branch file count (observed) | Audit status |
|---|---|---:|---|
| CompileCare.Api | Implemented foundation and facility slice | 110 | Review further; active core |
| CompileCare.ApiClient | Reusable typed/named HTTP client | 30 | Review further; baseline merged |
| CompileCare.Observability | Shared telemetry/logging libraries, samples and CI | 60 | Review further; target framework/package hardening |
| CompileCare.Caching | README-only skeleton | 2 | Implementation exists on non-main branches |
| CompileCare.Mobile | README-only skeleton | 2 | Open PR contains an early MAUI starter |
| CompileCare.Web | README-only skeleton | 2 | No actual frontend implementation |
| CompileCare.OtpService | README missing; gitignore only | 1 | Open PR contains implementation; security review required |
| CompileCare.Site | Marketing site | 70 | Initial version merged |
| CompileCare.Database | README-only skeleton | 2 | Scope and ownership unclear |
| Compilecare.Doc | Architecture/product/engineering/security docs | 30 | Source of truth; some proposals remain unapproved |

## PR inventory

### Merged (21)
- CompileCare.Api: #1, #3, #5, #6, #7, #8, #9, #10, #11
- CompileCare.ApiClient: #1, #2
- CompileCare.Observability: #1, #2, #3
- CompileCare.Caching: #1 (merged into `develop`, not `main`)
- CompileCare.Site: #1
- Compilecare.Doc: #2, #3, #4, #5, #6

### Closed without merge (2)
- CompileCare.Api #2 — stacked observability PR; superseded by clean PR #3.
- Compilecare.Doc #1 — superseded by PR #2.

### Open (2)
- CompileCare.Mobile #1 — starter MAUI app; targets `develop`. No check-runs/statuses were found for the inspected head.
- CompileCare.OtpService #1 — security review pending; latest CI run 45221746072 failed (6 tests passed, 1 failed).

## Branch inventory and disposition

### CompileCare.Api (12 non-main branches)
**Merged PR branches — cleanup candidates after confirming merged PRs and current main content:**
- `feature/facility-hierarchy-crud` (PR #7)
- `feature/facility-hierarchy-domain` (PR #6)
- `feature/observability-integration-clean` (PR #3)
- `feature/phase2-api-conventions` (PR #8)
- `feature/phase3-audit-events` (PR #11)
- `feature/phase3-database-security` (PR #9)
- `feature/phase3-identity-rbac` (PR #10)
- `feature/platform-foundation` (PR #1)
- `feature/postgresql-efcore-foundation` (PR #5)

**Stale/superseded cleanup candidates:**
- `feature/facility-hierarchy-crud-clean` — compare shows 0 ahead / 5 behind main.
- `feature/observability-integration` — PR #2 was closed without merge and explicitly superseded by PR #3.

**Review before deletion:**
- `feature/enterprise-security-foundation` — 2 commits ahead of main; includes security/RBAC-related commits. Do not delete until content is compared with merged PR #10 and the current main implementation.

### CompileCare.ApiClient (4 non-main branches)
- `feature/api-client-foundation` and `feature/phase2-developer-platform`: merged PR branches; cleanup candidates.
- `develop`: 0 ahead / 2 behind main; cleanup candidate.
- `feature/CompileCare_Api_client_creation`: old sample/demo branch, 1 commit ahead / 26 behind the foundation branch; inspect for any useful content, then likely archive/delete.

### CompileCare.Observability (2 non-main branches)
- `feature/publish-stable-package`: merged PR #3; cleanup candidate.
- `develop`: diverged from main (20 commits behind and 2 ahead in the observed comparison); inspect before deletion because it may contain changes not represented in main.

### CompileCare.Caching (3 non-main branches)
- `feature/CompileCare1`: merged into develop; 0 ahead / 2 behind `feature/Add_redis_support`; cleanup candidate after confirming the successor branch retains the work.
- `develop`: 5 commits ahead of main.
- `feature/Add_redis_support`: 6 commits ahead of main and 1 commit ahead of develop; retain for implementation review and a possible PR to main.
- Main remains README-only. The caching implementation is not delivered on main.

### CompileCare.Mobile (2 non-main branches)
- `develop`: identical to main; candidate for consolidation/cleanup, but retarget or close PR #1 first.
- `feature/CompileCare1`: 24 commits ahead of main; keep while PR #1 is reviewed.

### CompileCare.Web (2 non-main branches)
- `develop` and `feature/CompileCare1`: both identical to main (README-only); cleanup candidates. No open PR and no implementation observed.

### CompileCare.OtpService (2 non-main branches)
- `develop`: identical to main; cleanup candidate.
- `feature/create-a-service-for-otp-validation`: keep while PR #1 is reviewed.

### CompileCare.Site (2 non-main branches)
- `feature/website`: PR #1 merged; cleanup candidate.
- `develop`: 0 ahead / 1 behind main; cleanup candidate.

### CompileCare.Database
- Main only; no branch cleanup required.

### Compilecare.Doc (7 non-main branches)
**Merged PR branches — cleanup candidates:**
- `docs/compilecare-foundation-v1` (PR #2)
- `docs/compilecare-roadmap-alignment` (PR #3)
- `docs/phase3-migration-status` (PR #4)
- `docs/phase3-oidc-rbac-status` (PR #5)
- `docs/tenant-registry-provisioning-design` (PR #6)

**Superseded/stale cleanup candidates:**
- `docs/foundation-v1` — PR #1 closed without merge; superseded by PR #2.
- `docs/enterprise-security-foundation` — 0 ahead / 4 behind main.

## Findings and risks

### P1 — OTP service must not merge yet
The PR description records a previously committed SMTP credential that must be revoked/rotated; removing it from current configuration/history rewrite is not a substitute for credential rotation. Remaining work includes atomic multi-instance rate limiting/cooldown, attempt limits, OTP hashing and safe comparison, recipient validation, abuse controls, and missing test coverage. Latest CI failed (6/7 tests). Keep PR open and blocked.

### P1 — Mobile starter is not integrated with the backend
PR #1 targets `develop` and contains a .NET 8 MAUI starter. The observed branch has a placeholder API base URL, a blank LoginViewModel, starter counter UI, and OTP client calls to `/auth/send-otp` and `/auth/verify-otp`. The API currently exposes OIDC/JWT authentication and tenant-membership/RBAC; these OTP endpoints were not observed in CompileCare.Api. The mobile branch has no GitHub check-runs/statuses on the inspected head. Do not merge until its purpose, authentication contract, supported SDK baseline, UI, tests, and CI are reviewed.

### P1 — Caching implementation is not on main and needs review
The non-main Redis/HybridCache implementation targets .NET 9, while CompileCare.Api and ApiClient target .NET 10. The implementation contains an empty placeholder test. A source review found `GetAsync` implemented via `GetOrCreateAsync` with a default-value factory, which is not a clear cache-only retrieval contract and needs correction. Configuration is duplicated across CacheSettings, ConnectionStrings, and Caching sections. Tenant-aware cache-key strategy and invalidation/isolation need explicit requirements and tests before use with tenant data. Main is README-only.

### P1 — Inconsistent framework baseline
Observed target frameworks:
- CompileCare.Api: .NET 10
- CompileCare.ApiClient: .NET 10
- CompileCare.Observability: .NET 9
- CompileCare.Caching branch: .NET 9
- CompileCare.Site: .NET 8
- CompileCare.Mobile branch: .NET 8

Choose and document a supported baseline and migration policy; do not assume branches/packages are compatible without build validation.

### P2 — Web frontend has not started
CompileCare.Web main and its observed develop/feature branches contain only README and gitignore. UI architecture, design system, authentication/tenant selection, frontend CI, and API integration remain future work.

### P2 — Database repository ownership is unclear
CompileCare.Api currently owns EF Core entities, configurations, migrations and model snapshot. CompileCare.Database is a skeleton with a broad “all database queries and deployment” description. Decide whether it is for deployment assets/scripts only; application query logic and schema migrations should remain owned by the service/module.

### P2 — Observability hardening
The main Observability repo targets .NET 9 and includes beta OpenTelemetry packages and placeholder tests. The existing Compose/configuration should be reviewed for portable paths, pinned images, security defaults, release permissions/versioning, redaction, and meaningful tests before production use.

### P2 — Documentation and architecture decisions remain proposals
The documentation labels MVP and several architecture choices as proposals. Confirm with stakeholders. The distributed target vs modular-monolith implementation strategy, tenant/group/branch model, patient-sharing rules, identity-provider decision, deployment model, and ownership of database assets need explicit decisions. The product MVP should remain hospital/healthcare scope only.

## Framework and boundary decisions to settle

1. Supported .NET/SDK baseline and package target strategy.
2. Modular monolith boundaries and when/if service extraction is justified.
3. OIDC/JWT identity-provider approach; OTP is not a replacement for identity/session architecture.
4. Tenant/customer/group/hospital/branch hierarchy and patient-sharing policy.
5. Web (React/TypeScript proposed), Mobile (MAUI branch), and Site (marketing) ownership and separation.
6. Database repository scope versus API/module migration ownership.
7. Shared-library packaging/versioning and consuming-app integration.

## Recommended next sequence

1. Finish branch disposition and cleanup through a branch-deletion-capable mechanism; retain branches with unique, unreviewed work.
2. Review and decide the Caching branch: fix cache semantics, tenant-aware keys, configuration, tests, SDK baseline; create a focused PR to main.
3. Review Mobile PR #1; retarget to main only after review; either close and rebuild a clean starter or merge a genuinely complete, tested baseline.
4. Resolve OTP security blockers and rotate the exposed credential before any merge/deployment.
5. Review Observability package compatibility, release pipeline, configuration and tests; reconcile `develop` with main.
6. Establish the Web frontend architecture and initial UI contract against the API.
7. Update the master roadmap/tracker with accepted scope, owners, dependencies, and exit criteria.
8. Continue the healthcare MVP in order: trusted tenant provisioning, staff/roles, patient registration/MRN/search/dedupe, appointment, check-in/queue, OPD encounter, then the agreed next modules.

## Audit limitations and cleanup status

- No local checkout or local build/test execution was performed.
- Branch deletion was not available through the connected GitHub actions at audit time.
- Branch recommendations are dispositions, not completed deletions.
- This is a baseline; source-level review, CI verification, dependency/security scanning, and environment/operational checks are still required.
