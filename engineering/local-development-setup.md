# HeapCare local development setup

**Audience:** HeapCare engineers running the hospital workspace on macOS or Windows 10/11  
**Source of truth:** `HeapCare.Local/README.md` and the checked-in launchers  
**Status:** The nine-service local stack and authenticated CI harness are implemented. A successful container startup is not a substitute for the developer-machine, manual accessibility or production acceptance gates.

## 1. Prerequisites

- Docker Desktop installed and running.
- Git and GitHub CLI installed; authenticate with `gh auth login`.
- Access to the `HeapCare` organization repositories.
- GitHub Packages `read:packages` access for the HeapCare package owner.
- macOS: use the supplied shell launcher. Windows 10/11: use the supplied Batch launcher.

Never use production credentials or real patient data in local development.

## 2. Clone and launch

Clone the local orchestration repository into a HeapCare-named folder:

```bash
gh repo clone HeapCare/HeapCare.Local HeapCare.Local
cd HeapCare.Local
```

The launchers create or reuse adjacent `HeapCare.Api` and `HeapCare.Web` checkouts. They do not overwrite dirty work or discard local changes. The macOS launcher requires existing application checkouts to be clean and on `main`, then fetches and fast-forward updates them only.

- macOS: double-click `local/Start HeapCare.command`.
- Windows 10/11: double-click `local/Start HeapCare.bat`.

The first launch creates ignored `local/.env` and `local/.secrets/` files as required. It generates independent random local Keycloak/PostgreSQL credentials and a one-time platform bootstrap secret. Package credentials are obtained through the authenticated GitHub CLI token on macOS; Windows prompts for a GitHub username and a `read:packages` PAT. Secrets remain in ignored local files.

Do not paste tokens/passwords into issues, PRs or chat, and never commit local secret files.

## 3. Local endpoints

| Service | URL |
|---|---|
| Web workspace | `http://localhost:8080` |
| API | `http://localhost:8081` |
| API OpenAPI (Development) | `http://localhost:8081/openapi/v1.json` |
| Keycloak | `http://keycloak.localhost:8082` |
| Keycloak admin console | `http://keycloak.localhost:8082/admin` |
| PostgreSQL | `localhost:5432` (loopback only) |
| Prometheus | `http://localhost:9090` |
| Jaeger | `http://localhost:16686` |
| Kibana | `http://localhost:5601` |

The local stack includes PostgreSQL, Keycloak, HeapCare API, HeapCare Web, OpenTelemetry Collector, Jaeger, Prometheus, Elasticsearch and Kibana.

## 4. Local identity and compatibility contract

The checked-in local identity configuration deliberately retains these legacy technical identifiers:

| Setting | Local value |
|---|---|
| Keycloak realm / issuer path | `compilecare` |
| Authority | `http://keycloak.localhost:8082/realms/compilecare` |
| Web client | `compilecare-web` |
| API audience | `compilecare-api` |
| Default audience scope | `compilecare-api-audience` |
| Sign-in callback | `http://localhost:8080/auth/callback` |
| Silent-renew callback | `http://localhost:8080/auth/silent-callback` |
| Web origin / post-logout URI | `http://localhost:8080` |

The Web client uses Authorization Code with PKCE S256. The API validates issuer, audience, signature, lifetime and subject, then binds persisted membership to the validated issuer and `sub`. The `.localhost` issuer resolves to loopback and is registered as a Docker network alias.

These are compatibility values, not the HeapCare public brand. Do not rename the realm/client/audience or change the issuer as cosmetic cleanup; doing so changes the identity contract. Production identity-provider values must be inventoried and migrated separately.

The API permits HTTP identity metadata only in the local Development configuration. Do not carry the local HTTP exception into staging or production.

## 5. Credentials, persistence and safe reset behavior

Fresh setup generates independent random 256-bit Keycloak administrator and PostgreSQL passwords in ignored `local/.env`. Existing values are preserved so the configuration does not silently become inconsistent with data already stored in Docker volumes.

For an existing Keycloak volume, change the local administrator password in Keycloak before updating `local/.env`. For PostgreSQL, change the local database role password before updating the environment file and restarting the API. Do not reuse these credentials outside the developer environment.

The local Compose project and persistent volume names still use legacy `compilecare` identifiers. They are retained to protect existing Keycloak users and PostgreSQL/observability data. Routine stop/start operations preserve the volumes.

To stop the application without deleting data, use `local/Stop HeapCare.command` or `local/Stop HeapCare.bat`.

Only when intentionally deleting all local database, identity and observability data, run:

```bash
docker compose --env-file local/.env \
  -f local/docker-compose.yml \
  -f local/docker-compose.observability.yml down --volumes
```

This operation is destructive. Do not use it as a troubleshooting shortcut.

## 6. First platform operator and hospital setup

1. Create a normal local user in the `compilecare` Keycloak realm. Do not use the Keycloak admin account as a hospital workspace user.
2. Sign in to the HeapCare Web workspace.
3. When the first-run panel requests the platform bootstrap secret, enter the one-time value generated by the launcher and stored in `local/.env`.
4. Create the initial tenant and provision the signed-in identity as the initial tenant-wide HospitalAdmin through the supported workflow.
5. Activate the tenant, create the first hospital and departments, and assign organization scopes through the API/workspace.
6. Reload tenant context and select the appropriate hospital/department.

The bootstrap route is Development-only, requires a valid API access token and a separate high-entropy secret, and closes permanently after a platform operator exists in that database. Do not use direct SQL to bypass provisioning or audit operations.

## 7. Verify the stack

Run the infrastructure and observability smoke checks:

```bash
bash local/scripts/verify-stack.sh
```

The script checks the nine services, PostgreSQL readiness, Web/API/OpenAPI, Keycloak OIDC discovery, Prometheus, Jaeger, Elasticsearch, Kibana, Collector metrics and the API metrics endpoint. It does not read patient data or print credentials.

Passing this script is an infrastructure smoke test, not proof of tenant authorization, patient workflows, accessibility or production readiness. Use synthetic records only.

The Local GitHub Actions full-stack job additionally runs `local/scripts/verify-authenticated-tenancy.py` with temporary Keycloak identities and persisted tenant memberships. It is gated by `HEAPCARE_CI_ACCEPTANCE=1` and is designed for an isolated, ephemeral CI stack only. Do not run it against a persistent developer or production environment.

## 8. Common troubleshooting

### Private NuGet restore returns HTTP 401 or 403

Confirm the GitHub identity has `read:packages`, access to the HeapCare package owner and any required organization SSO authorization. The launcher stores package credentials under ignored `local/.secrets/`; Docker BuildKit receives them as a secret, not as a build argument or image layer. Replace only the credential file if it is stale, then rerun the launcher.

### Keycloak sign-in succeeds but the API returns 401

Check the access token's issuer, audience and subject against the configured authority/API options. Verify the browser sends the access token rather than the ID token. Do not disable token validation or invent a tenant header to bypass authorization. The API still requires an active persisted membership and valid organization scope.

### Existing Keycloak users are missing after a local configuration edit

The realm import runs only when the Keycloak volume is initialized. The launcher runs the idempotent reconciler to update the existing local client while preserving users and volumes. Use:

```bash
python3 local/scripts/configure-keycloak.py local/.env
```

Do not delete the volume unless a full local identity reset is intended.

### Web API/OIDC connectivity or CSP errors occur after changing local URLs

Update the local API/OIDC origins in ignored `local/.env` and align `CSP_CONNECT_SRC` and `CSP_FRAME_SRC` to bare origins (scheme, host and optional port only). The local Compose file explicitly opts into HTTP origins for loopback Development only. Production must use the exact HTTPS origins approved for its deployment.

### Inspect service health and logs

```bash
docker compose --env-file local/.env \
  -f local/docker-compose.yml \
  -f local/docker-compose.observability.yml ps

docker compose --env-file local/.env \
  -f local/docker-compose.yml \
  -f local/docker-compose.observability.yml logs --tail=150 api web keycloak postgres
```

Do not share raw tokens, passwords, patient identifiers, clinical text or unredacted logs.

## 9. Acceptance boundaries

A local start, smoke-test pass or hosted CI run does not prove the platform is production-ready. Separate acceptance is still required for:

- Developer-machine launch on the supported macOS and Windows versions.
- Authenticated role workflows using synthetic users and data.
- Manual keyboard/focus/dialog review, screen-reader review, contrast and zoom/reflow.
- Provider-native encrypted backup/PITR and measured restore objectives.
- Production OIDC issuer/client/audience, signing-key custody, database TLS, ingress, DNS and secret rotation.
- Security/privacy/legal review and hospital pilot sign-off.

See the [Phase 5 authenticated accessibility acceptance checklist](../qa/phase5-authenticated-accessibility-acceptance.md), the [production identity and infrastructure cutover inventory](../operations/heapcare-production-cutover-inventory-2026-11-28.md), and the [backup/restore/DR baseline](../operations/backup-restore-disaster-recovery.md).
