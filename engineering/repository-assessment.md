# Existing Repository Assessment — Initial Review

**Review date:** 2026-11-23  
**Review type:** Connector-based review of repository metadata, README files, selected project files, and initial commits. Full directory-tree enumeration and local build/test execution were unavailable; findings are preliminary.

## Inventory
| Repository | Observed purpose/state | Initial assessment |
|---|---|---|
| `CompileCare.Observability` | .NET 9 solution with core, ASP.NET Core and Worker components, samples, health checks, OpenTelemetry, Serilog, Docker stack, CI/NuGet publishing | Most mature shared foundation; hardening/review before adoption |
| `CompileCare.Caching` | .NET 10 HybridCache + optional Redis package on main; config validation, tenant/global key builder, sample, CI; PR #2 merged with 11 tests and pack | Foundation complete; consumer integration, invalidation strategy and production Redis configuration remain |
| `CompileCare.ApiClient` | .NET 10 typed/named clients, auth/tenant/correlation providers, resilience, Problem Details; correlation validation fix merged in PR #3 | Foundation complete; package/consumer validation and API contract generation remain |
| `CompileCare.Database` | Operational ownership, migration/release, backup/restore and environment runbooks merged in PR #1 | Documentation foundation; hosting-specific automation and restore drills remain |
| `CompileCare.OtpService` | PR #1 was closed without merging; reviewed its proposed API/service code | Security-sensitive prototype; not safe to integrate or deploy. Requires a complete threat model and refactor before reuse |
| `CompileCare.Site` | README says CompileCare website; initial commit shows README and gitignore | Skeleton |
| `Compilecare.Doc` | Documentation repository; foundation docs are being added | Documentation source of truth; repository capitalization remains a naming inconsistency |
| `CompileCare.Web` | Renamed from `Complilecare.Web`; README says intended for web app; initial commit shows README and gitignore | Skeleton |
| `CompileCare.Mobile` | Renamed from `Complilecare.Mobile`; README says intended for mobile app; initial commit shows README and gitignore | Skeleton |

## Observability evidence
Inspected project file targets `net9.0`. Packages include OpenTelemetry 1.12.x, health checks, OTLP/Prometheus exporters, ASP.NET Core/HTTP/runtime/process/SQL instrumentation, and Serilog 4.x with console and Elasticsearch sinks.

Solution contains core, ASP.NET Core, and Worker libraries; sample API/Worker; tests; Docker Compose; Collector and Prometheus configs; GitHub Actions CI.

### Preliminary risks
- Compose Collector mount uses a developer-specific Windows absolute path; make it portable.
- Elasticsearch security is disabled in inspected Compose; local-only use should be explicit.
- Some Docker images use `latest`; pin versions/digests.
- Beta OpenTelemetry packages present; evaluate stable versions.
- CI publishes on push; review branch conditions, versioning, and release permissions.
- Tests include a placeholder and a health-check test; increase meaningful coverage.
- Validate README endpoints/topology against actual Compose/config.
- Ensure telemetry/logs redact sensitive information and never include patient data.

## Cross-repository findings
- Web and Mobile repository spelling errors have been corrected through renaming. The `Compilecare.Doc` capitalization inconsistency remains.
- Site/Web/Mobile boundaries and ownership need definition.
- Shared libraries should be packaged/versioned when contracts and consumers are stable; avoid premature proliferation.
- Database repo's “all database queries and deployment” scope may couple unrelated concerns. Prefer module schema/migration ownership and explicit deployment tooling boundaries.
- Inventory suggests the hospital platform itself is not yet implemented; documentation must label planned capabilities clearly.


## OTP service security assessment (2026-11-24)

Reviewed the closed, unmerged [CompileCare.OtpService PR #1](https://github.com/CompileCare/CompileCare.OtpService/pull/1) and its source. It is not approved for deployment or integration.

### Blocking findings
- OTP values are stored in the distributed cache in plaintext.
- Verification has no attempt counter/lockout; a six-digit OTP can be brute-forced while valid.
- Request cooldown is a non-atomic read-then-write and keyed only by the submitted identifier; concurrent requests and distributed abuse are not controlled.
- No IP/device/recipient rate-limit policy or abuse monitoring is demonstrated.
- Endpoints accept primitive parameters, lack a documented identity/authorization contract, and the host configures permissive CORS.
- Redis is configured without demonstrated fail-closed behavior for missing/invalid production configuration.
- SMS provider initialization uses a static/global client configuration and the shown request path does not propagate cancellation.
- Recipient normalization/validation and privacy-safe responses need a formal contract; user existence and channel responses must not enable enumeration.
- Tests cover basic happy/failure cases but do not prove expiry, attempts, concurrency, delivery failures, rate limits, replay prevention or abuse controls.
- A provider credential was previously committed in Git history. **The credential must be revoked/rotated by its owner; deleting it from current files does not invalidate history.** No secret value is reproduced here.

### Required remediation before reuse
1. Confirm purpose and ownership: account verification, login MFA, step-up auth or another use case. Prefer the identity provider's supported MFA where it satisfies the requirement.
2. Establish a versioned request/response contract, recipient/channel policy and generic anti-enumeration response behavior.
3. Hash OTPs at rest using a server-side secret/HMAC (with key rotation), compare in constant time, and store expiry plus attempt count/state.
4. Use atomic Redis operations/Lua or another proven mechanism for issue cooldown, attempt counting, one-time consumption and per-recipient/IP limits across instances.
5. Enforce short expiry, bounded attempts, resend cooldown, abuse/rate limits, one-time verification and cancellation/timeouts.
6. Secure delivery adapters and configuration; avoid global provider client mutation; redact recipient identifiers and never log OTPs.
7. Add comprehensive tests including concurrency and cache outages, plus CI security/dependency checks.
8. Rotate the exposed historical credential and decide whether repository history rewriting is required by the organization; coordinate it to avoid disrupting collaborators.

This service remains deferred until the product's authentication/contact-verification need and security owner are established.

## Next implementation steps
- Finish Phase 2 consumer validation and package publication/release workflow.
- Validate proposed patient identity/registration ADR-008 with pilot stakeholders before implementing patient schema and APIs.
- Continue API cross-tenant, audit, asynchronous and production-hardening tests.
- Defer OTP until its purpose/security owner is agreed and all blockers—including credential rotation—are addressed.
- Start Web after patient/registration API contracts are stable; Mobile follows validated use cases and auth contracts.
