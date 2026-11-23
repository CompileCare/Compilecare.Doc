# Existing Repository Assessment — Initial Review

**Review date:** 2026-11-23  
**Review type:** Connector-based review of repository metadata, README files, selected project files, and initial commits. Full directory-tree enumeration and local build/test execution were unavailable; findings are preliminary.

## Inventory
| Repository | Observed purpose/state | Initial assessment |
|---|---|---|
| `CompileCare.Observability` | .NET 9 solution with core, ASP.NET Core and Worker components, samples, health checks, OpenTelemetry, Serilog, Docker stack, CI/NuGet publishing | Most mature shared foundation; hardening/review before adoption |
| `CompileCare.Caching` | README describes shared caching library; inspected initial commit shows README and gitignore | Skeleton/early stage |
| `CompileCare.ApiClient` | README describes API client for microservices communication; inspected initial commit shows README and gitignore | Skeleton/early stage |
| `CompileCare.Database` | README says database queries and deployment; inspected initial commit shows README and gitignore | Scope needs clarification |
| `CompileCare.OtpService` | Root README was not found; inspected initial commit showed gitignore | Need source/tree and service requirements |
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

## Next audit steps
Complete tree/source audit of Caching, ApiClient, Database, OTP, Site, Web, Mobile; inspect dependencies, workflows, tests and configuration. Then agree naming/boundaries, CI templates, and platform repository creation.
