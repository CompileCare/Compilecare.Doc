# HeapCare Production Identity and Infrastructure Cutover Inventory

**Inventory date:** 2026-11-28  
**Status:** Repository-backed, non-production inventory completed; production provider/account inventory and cutover decisions remain open.  
**Purpose:** Separate HeapCare product branding from technical identifiers that still carry identity, data or persistence compatibility.

## 1. Executive result

The GitHub organization and all 11 repository slugs use the HeapCare name. The inspected application code and documentation also use HeapCare as the product-facing identity.

Several older identifiers remain in the **local development and CI compatibility path**: the Keycloak realm `compilecare`, OIDC client/audience IDs, PostgreSQL connection-string fallback, the local Compose project and persistent volume names, and CI-only migration database labels. These values are not equivalent to public product branding. Some are bound to issuer identity, persisted users or existing local data.

The accessible repository configuration does **not** establish which identity provider, PostgreSQL service, domain, DNS zone, email provider, TLS issuer, secrets manager or hosting region is currently used in production. No production credentials or patient data were inspected. Do not infer production values from local Compose, CI fixtures or sample documentation.

**Decision:** Keep the existing identifiers unchanged until the real production environment has been inventoried and a reversible migration plan has passed staging and restore verification. A text-only rename of a realm, issuer, database or volume is not a safe migration.

## 2. Evidence scope and confidence

This is a read-only inspection of the default branches and tracked configuration available on 2026-11-28. It records what the repositories prove, not what may exist in an external provider account.

- Local Compose, Keycloak import and launch documentation describe a developer-only stack using synthetic data and randomly generated local credentials.
- Local GitHub Actions acceptance uses an isolated disposable stack, synthetic users and records, and temporary CI credentials.
- API and Web configuration read environment/configuration values; sample files name keys but do not provide evidence of the production values.
- The production ingress and disaster-recovery documents explicitly identify provider-specific settings and recovery objectives as decisions that still require deployment-owner evidence.

No secret values, access tokens, private keys, connection-string values or patient data are included in this inventory.

## 3. Verified local OIDC identity contract

The checked-in local identity setup currently specifies:

| Item | Verified local value | Boundary |
|---|---|---|
| Keycloak image | `quay.io/keycloak/keycloak:26.2.5` | Local Compose only |
| Realm ID / issuer path | `compilecare` | Local realm; not evidence of production realm |
| Local issuer / authority | `http://keycloak.localhost:8082/realms/compilecare` | Loopback-only development topology |
| Web OIDC client ID | `compilecare-web` | Local browser client |
| API audience | `compilecare-api` | Local token audience |
| Flow | Authorization Code with PKCE S256 | Local acceptance flow |
| Sign-in callback | `http://localhost:8080/auth/callback` | Local only |
| Silent-renew callback | `http://localhost:8080/auth/silent-callback` | Local only |
| Web origin / post-logout origin | `http://localhost:8080` | Local only |
| Subject claim | Keycloak `sub` supplied by the configured basic scope | Required for identity binding |

The Local README explains that the API binds a persisted tenant membership to the validated issuer and subject. Consequently, a realm/issuer change is an identity migration, not a cosmetic rename: the pair of issuer and subject is part of the trusted external identity reference. Preserve user subject identifiers where the provider supports it, and explicitly map old identity references to new ones where issuer changes. Do not assume that copying usernames or email addresses preserves identity.

The Web client uses Authorization Code flow, automatic silent renewal and session-scoped browser storage. The API's HTTPS metadata requirement defaults to enabled; the local Compose environment explicitly permits HTTP only for Development. Production must use an HTTPS issuer and must not inherit the local HTTP exception.

### Configuration keys present (names only)

- Local: `OIDC_AUTHORITY`, `OIDC_AUDIENCE`, `OIDC_CLIENT_ID`, `OIDC_SCOPE`, `OIDC_REQUIRE_HTTPS_METADATA`.
- API: `Authentication:Authority`, `Authentication:Audience`, `Authentication:RequireHttpsMetadata`.
- Web: `VITE_OIDC_AUTHORITY`, `VITE_OIDC_CLIENT_ID`, `VITE_OIDC_REDIRECT_URI`, `VITE_OIDC_POST_LOGOUT_REDIRECT_URI`, `VITE_OIDC_SCOPE`.

The tracked `.env.example` files contain variable names and placeholders only. They are not a source of production values.

## 4. Verified database and connection compatibility

| Item | Repository-backed finding | Required interpretation |
|---|---|---|
| Primary relational provider | PostgreSQL; API uses EF Core and Npgsql | Product architecture decision; production service/provider is not identified here |
| Local container image | `postgres:17-alpine` | Developer Compose only |
| Local published port | `127.0.0.1:5432` | Loopback binding; not a production network design |
| Preferred non-development key | `ConnectionStrings:HeapCare` | Use this key for new deployment configuration |
| Legacy fallback key | `ConnectionStrings:CompileCare` | Compatibility fallback; do not remove without consumer/configuration audit |
| Non-development TLS policy | PostgreSQL connection must use `SSL Mode=VerifyFull` | Enforced by API startup validation |
| CI migration databases | `compilecare_upgrade` and `compilecare_fresh` | Disposable PostgreSQL 16 migration-rehearsal databases only |

The API connection-security validator skips Development and Testing. Outside those environments it requires either the HeapCare connection-string key or the legacy fallback, validates the connection string without logging it, and rejects any TLS mode other than `VerifyFull`.

The current database CI evidence includes PostgreSQL migration-chain rehearsal and logical dump/restore against disposable databases. Those checks validate migration mechanics and helper scripts. They do not prove production database names, provider-native backups, WAL/PITR, off-site backup encryption, key recovery, tenant-level restore, or approved RPO/RTO.

## 5. Verified local Docker and persistent-volume identifiers

The local Compose files currently define:

| Component | Local identifier | Notes |
|---|---|---|
| Compose project | `compilecare-local` | Project namespace for local resources |
| Keycloak volume | `compilecare-keycloak` | Contains local identity-provider data |
| PostgreSQL volume | `compilecare-postgres` | Contains local database data |
| Prometheus volume | `compilecare-prometheus` | Local metrics data |
| Elasticsearch volume | `compilecare-elasticsearch` | Local log/index data |
| Application/observability containers | `heapcare-*` | Current visible container names use HeapCare |

Compose prefixes named volumes with the project name in the normal project-scoped setup. The Local README explicitly documents the Keycloak volume as `compilecare-local_compilecare-keycloak` when describing a destructive reset. The stop workflow retains data volumes; the explicit `down --volumes` path removes local persisted data.

**Do not rename the Compose project or volume definitions as a branding cleanup.** Changing the project name can cause Compose to create new empty volumes while the existing volumes remain on disk, making existing local users or data appear to have vanished. Deleting the old volumes is destructive. If local names are ever changed, use a separately tested copy/restore or explicit volume migration, verify the target, and retain a rollback path.

## 6. Production configuration not established by repository evidence

The following must be collected from the actual production/staging provider accounts and approved deployment configuration. Values belong in the approved restricted operations/secret-management system, not in public source, PR comments or ordinary documentation.

| Inventory domain | Evidence still required | Status |
|---|---|---|
| Identity provider | Provider/account, realm/tenant, issuer URL, client IDs, API audience, redirect/logout URIs, token lifetimes, accepted signing algorithms | Not verified |
| Identity continuity | User subject IDs, issuer+subject membership references, role/claim mapping, disabled users, service identities and migration/rollback approach | Not verified |
| Signing keys | Active/passive key IDs (kid), JWKS URI, key ownership, refresh/cache behavior across API replicas, rotation and emergency revocation process | Not verified |
| Service credentials | Secret manager, OIDC confidential-client/service credentials if applicable, rotation owners and expiry alerts | Not verified |
| Database | Provider/service, region, database/schema/role names, extensions, replicas, TLS certificate trust chain and connection-secret owner | Not verified |
| Database recovery | Provider-native base backup, WAL/PITR, encryption, off-site storage, retention/legal hold, restore cadence and measured RPO/RTO | Not verified |
| Ingress and domains | Purchased/controlled domain, DNS zones, Web/API hostnames, TLS certificates, ingress/proxy IPs, CORS/CSP origins and WAF/rate limits | Not verified |
| Email and messaging | Provider, verified sender domains, secrets, delivery/abuse controls and purpose-specific retention | Not verified |
| Observability | Production collector/backend, access controls, data retention, alert delivery, health/metrics ingress and log redaction | Not verified |
| Runtime and network | Hosting account/region, service identities, private networks, firewall/security groups, egress allowlist and deployment rollback | Not verified |
| Data policy | Hospital/client-approved retention, deletion, legal hold, residency, audit access and incident contacts | Not verified |

The documentation checkpoint says `heapcare.in` has not been confirmed as purchased or configured. This inventory does not establish domain ownership, production DNS, email, TLS or public origin readiness.

## 7. Safe staged cutover sequence

Do not begin a production identity or data cutover until the following stages have owners, an approved change window and rollback authority.

1. **Discover and freeze the current state.** Inventory actual production and staging provider resources, exported configuration metadata, service-to-service consumers, DNS, secret ownership, database topology and volume/storage identifiers. Capture identifiers and configuration references only; never copy secret material into Git. Confirm which systems are actually production rather than assuming local names carry over.
2. **Approve identity continuity.** Select the target provider/realm/client model and record the exact issuer, audience, callback and logout origins in a restricted environment record. Determine whether the current issuer and `sub` values can be preserved. If the issuer changes, prepare an explicit, reviewed identity-reference mapping and prove that every active tenant membership continues to resolve to the intended user and role. Do not merge accounts by email alone.
3. **Prove staging OIDC behavior.** With synthetic users for all six roles, test discovery/JWKS, Authorization Code + PKCE, callback, token exchange, issuer/audience/signature/lifetime validation, refresh/silent renewal, logout, session revocation, tenant switching and denied cross-tenant access. Test new and still-valid old tokens during any planned overlap. Verify every serving API replica. Keep the old provider available until the approved rollback/overlap condition has passed.
4. **Prove database migration and recovery.** Produce a verified provider-native backup/PITR recovery point; restore to a new isolated target; apply the exact application migration chain; validate schema, migration history, tenant boundaries, critical counts and synthetic workflows. Record actual restore duration and recoverable data-loss window against approved RPO/RTO. Never test first by overwriting the live database.
5. **Preserve local and deployment persistence.** Keep the local Compose/volume names and existing persistent data intact. For a separately approved resource-name change, prove data copy, service cutover, old-resource retention and rollback using disposable or isolated environments before touching any shared environment.
6. **Configure public origins only after ownership.** Once the selected domain and DNS provider are under the organization's control, configure DNS, TLS, ingress, HTTPS issuer, API/Web origins, CORS, CSP, redirect/logout URIs, trusted proxy IPs and health/metrics exposure. Verify the actual browser and deployed endpoints; do not put guessed origins in configuration.
7. **Cut over and observe.** Execute the approved runbook with named operator and decision owner. Verify login, all six role workflows, tenant isolation, appointment/clinical/billing regression, audit attribution, backups, alerts, error rates and restore readiness. Retain sanitized evidence and keep the old path until the rollback window is formally closed.
8. **Retire legacy identifiers last.** Remove compatibility fallbacks, old clients/realms, old database names or old volumes only after consumer searches, production verification, backup/restore, retention and rollback sign-off. Record the irreversible deletion approval separately.

## 8. Explicit no-go actions before the gates pass

- Do not rename the local Keycloak realm ID or change the issuer string solely to remove the word `compilecare`.
- Do not change `compilecare-web` or `compilecare-api` without reviewing every client, audience and token consumer.
- Do not remove the legacy `ConnectionStrings:CompileCare` fallback until all production deployment variables and consumers are inventoried and tested.
- Do not rename or delete `compilecare-local` resources or existing Keycloak/PostgreSQL/Prometheus/Elasticsearch volumes as part of the brand-only migration.
- Do not assume the CI logical dump/restore smoke is production backup/PITR evidence.
- Do not configure a guessed domain, issuer, redirect URI, email sender, TLS origin, region or provider.
- Do not publish real user identities, tenant mappings, secret values, keys, connection strings or production configuration into GitHub documentation.

## 9. Acceptance evidence to close this inventory

- [ ] Production and staging identity-provider inventories are completed by the identity owner.
- [ ] Issuer, audience, client and user-subject continuity design is approved by security and product owners.
- [ ] Staging OIDC migration/overlap and rollback tests pass for all six roles and tenant isolation.
- [ ] Production database/provider, roles, network and connection-secret ownership are documented in the restricted operations system.
- [ ] Provider-native encrypted backup/PITR is configured and monitored.
- [ ] An isolated provider-native restore drill passes and records actual RPO/RTO.
- [ ] DNS/domain ownership, TLS, ingress, CORS/CSP and trusted-proxy settings are verified against the real deployment.
- [ ] Local Compose/volume compatibility has a documented non-destructive migration plan before any rename.
- [ ] Full API/Web/Local regression passes on exact release SHAs.
- [ ] Security/privacy, hospital/client operations and release owners approve cutover and rollback.
- [ ] Legacy identifiers are retired only after the above checks and a separately approved compatibility-removal release.

## 10. Source evidence snapshot

The following repository files were inspected for this inventory. The commit/blob identifiers help reproduce the review; current branch contents may advance.

| Repository file | Blob SHA at inspection |
|---|---|
| `HeapCare.Local/README.md` | `73e2776e2960f48e16a574cac03f27a5257aa8a3` |
| `HeapCare.Local/local/docker-compose.yml` | `7f11038a779e89768224fb269b76ba73959877e0` |
| `HeapCare.Local/local/docker-compose.observability.yml` | `32935faafa08dd2c1b6f87410dbbc4237d527db8` |
| `HeapCare.Local/local/.env.example` | `0b5ef35f715742fcdd465c0c35be63875b6edb26` |
| `HeapCare.Local/local/keycloak/compilecare-realm.json` | `68192d31b21cfb4ff0dc8b790ab1001040155cb5` |
| `HeapCare.Local/local/scripts/configure-keycloak.py` | `2b0fda941a4fa24478ffdefdc74280ca27388012` |
| `HeapCare.Api/src/HeapCare.Api/Security/OidcAuthenticationOptions.cs` | `be82477705da4088cc3ae3f7cea33fc1185e3ad0` |
| `HeapCare.Api/src/HeapCare.Api/Security/DatabaseConnectionSecurity.cs` | `f38a7c60df58ce53d71d47c257970574922beece` |
| `HeapCare.Api/src/HeapCare.Api/Program.cs` | `4626f4ad82bed60c0e1976e30136ad4964917d91` |
| `HeapCare.Web/.env.example` | `f6999c55b5da22b278cf297bd665060d5bf7beac` |
| `HeapCare.Web/src/auth/oidc.ts` | `a77572cc7029c1fe341c3cd52815544f60110843` |
| `HeapCare.Database/docs/backup-restore.md` | `3210d7c4f2fdf99a6a6ba1599cb1fa7e5eff08bb` |
| `HeapCare.Database/docs/migrations-and-release.md` | `b6e84405e7eb0e4d6b8424af676fc70463dcbb14` |

Related operational sources: `operations/production-ingress-and-network-checklist.md`, `operations/backup-restore-disaster-recovery.md`, and `security/oidc-signing-key-rotation-runbook.md`.

---

**Current status:** Repository-backed local/CI compatibility inventory recorded. Production provider inventory, identity cutover, domain/ingress configuration, provider-native recovery and owner approvals remain open. This document is not production approval and does not authorize a live cutover.
