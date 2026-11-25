# Local development setup and troubleshooting

**Audience:** CompileCare developers running the full hospital workspace on macOS (Windows launcher is also provided).  
**Last updated:** 2026-11-25  
**Status:** The Docker stack has been started locally and Keycloak/PostgreSQL health checks passed. End-to-end API authorization and first-tenant onboarding are still being verified. Do not interpret a running Web container as a fully working local environment.

This runbook records the actual local setup, observed failures, fixes and remaining verification. Keep it updated whenever the launcher or local authentication model changes.

## 1. Repository layout

Clone the repositories as siblings:

```text
~/CompileCare/
  CompileCare.Api/
  CompileCare.Web/
  CompileCare.Local/
  Compilecare.Doc/
```

The local Compose file is `CompileCare.Local/local/docker-compose.yml`. Compose build contexts must resolve to the sibling API and Web repositories. Because the Compose file is inside `local/`, relative path interpretation can be surprising; the most reliable manual invocation is to export absolute paths:

```bash
cd ~/CompileCare/CompileCare.Local
export COMPILECARE_API_PATH="$(cd ../CompileCare.Api && pwd)"
export COMPILECARE_WEB_PATH="$(cd ../CompileCare.Web && pwd)"
```

The macOS launcher is `local/Start CompileCare.command`; the Windows launcher is `local/Start CompileCare.bat`.

## 2. First-time setup

1. Install and start Docker Desktop.
2. Install GitHub CLI and authenticate with an account that can read the private CompileCare NuGet packages:
   `brew install gh`, then `gh auth login`.
3. From `CompileCare.Local`, start the launcher:
   `./local/Start\ CompileCare.command`
4. Copy `local/.env.example` to `local/.env` if it does not exist. Keep `local/.env` untracked.
5. Set a private local Keycloak administrator password in `KEYCLOAK_ADMIN_PASSWORD`. Do not commit or share it.
6. At the launcher's hidden GitHub PAT prompt, supply a token with `read:packages` and access to the CompileCare organization packages. Authorize the token for organization SSO if required. Never paste the token into chat or source control.
7. The launcher stores the NuGet credential in `local/.secrets/nuget-source-credentials` with restrictive file permissions. This file must remain untracked and must not be printed in logs.

Relevant local defaults in `local/.env`:

```dotenv
OIDC_AUTHORITY=http://keycloak.localhost:8082/realms/compilecare
OIDC_AUDIENCE=compilecare-api
OIDC_CLIENT_ID=compilecare-web
OIDC_SCOPE=openid profile
OIDC_REQUIRE_HTTPS_METADATA=false
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=<choose-a-local-password>
POSTGRES_PASSWORD=<choose-a-local-password>
COMPILECARE_WEB_PATH=../CompileCare.Web
```

HTTP discovery is allowed only for Development and only with the explicit opt-in. Never carry `RequireHttpsMetadata=false` into Staging or Production.

## 3. Local services and endpoints

| Service | Local address |
|---|---|
| Web workspace | http://localhost:8080 |
| API | http://localhost:8081 |
| API OpenAPI (Development) | http://localhost:8081/openapi/v1.json |
| Keycloak realm | http://keycloak.localhost:8082/realms/compilecare |
| Keycloak admin console | http://keycloak.localhost:8082/admin |
| PostgreSQL | localhost:5432 |

The local stack uses Compose-managed Keycloak 26.2.5, PostgreSQL 17, CompileCare.Api and CompileCare.Web. Persistent data is stored in named Docker volumes. Do not use `docker compose down -v` as a routine troubleshooting step: it deletes local database and Keycloak data.

Check status and logs:

```bash
docker compose --env-file local/.env -f local/docker-compose.yml ps
docker compose --env-file local/.env -f local/docker-compose.yml logs --tail=150 api keycloak web postgres
```

Check Keycloak discovery:

```bash
curl -f http://localhost:8082/realms/compilecare/.well-known/openid-configuration
```

## 4. Keycloak realm and OIDC configuration

The realm import is maintained at `local/keycloak/compilecare-realm.json`. The intended client is:

- Realm: `compilecare`
- Client: `compilecare-web`
- Public client using Authorization Code + PKCE S256
- Redirect URI: `http://localhost:8080/auth/callback`
- Web origin and post-logout redirect: `http://localhost:8080`
- API audience: `compilecare-api`, supplied by the default `compilecare-api-audience` client scope
- Issuer/authority: `http://keycloak.localhost:8082/realms/compilecare`

The API audience mapper must remain enabled for access tokens. Do not remove the `compilecare-api-audience` default client scope or weaken API issuer/audience/signature validation to make local sign-in appear successful.

### Observed local scope issue (2026-11-25)

The imported realm's client-scope screens showed only `compilecare-api-audience` and `offline_access`; the expected built-in `profile`, `email`, `roles`, `web-origins` and `acr` scopes were not visible. The client requested `openid profile` and Keycloak returned:

```text
error=invalid_scope
error_description=Invalid scopes: openid profile
```

For diagnostic progress only, the local `.env` scope was changed to:

```dotenv
OIDC_SCOPE=openid
```

After rebuilding the Web image, the sign-in flow advanced to the workspace. This is a **temporary local workaround**, not the final OIDC configuration. It may omit profile claims and must not be treated as a completed realm setup. Restore the intended scopes only after the realm's built-in scopes and client assignments are correctly provisioned and tested. Keep the API audience mapper intact.

### API returned 401 after the workspace loaded

The workspace displayed `Your sign-in session is not authorized for the API.` This is distinct from the earlier Keycloak `invalid_scope` response. The UI shell can load while API calls fail. Diagnose using the API logs and a safe inspection of access-token claims (issuer, audience, expiry and scope only; never share the raw token):

- Confirm the access token's `iss` equals the configured authority.
- Confirm the access token's `aud` contains `compilecare-api`.
- Confirm the browser sends the **access token**, not the ID token, as the Bearer token.
- Confirm the API uses the same issuer and audience and is healthy.
- Do not disable token validation or substitute an arbitrary tenant header.

A successful Keycloak login is not proof that the API authorized the identity, and API authorization is not proof that the user has an active tenant membership.

## 5. Issues encountered and fixes

### A. Placeholder identity-provider hostname

**Symptom**

Browser attempted to load `https://your-development-identity-provider.example.com/.well-known/openid-configuration` and failed with `ERR_NAME_NOT_RESOLVED`.

**Cause**

The local `.env` still contained the example authority.

**Fix**

Set:

```dotenv
OIDC_AUTHORITY=http://keycloak.localhost:8082/realms/compilecare
```

The authority is injected into the Vite build. Rebuild the Web image after changing it; restarting an old image does not change the embedded JavaScript configuration.

### B. Private NuGet restore returned HTTP 401 in the API Docker build

**Symptom**

Docker build failed restoring `CompileCare.Observability` and `CompileCare.Observability.AspNetCore` from GitHub Packages.

**Cause and fix**

API PR #31 updated the API Dockerfile to consume the Compose BuildKit secret as NuGet's `NuGetPackageSourceCredentials_github` environment variable. The secret is not passed as a build argument or persisted in an image layer. The launcher prompts for a PAT only when the saved secret is absent or empty.

If package restore still fails, verify PAT owner, `read:packages`, organization package access and SSO authorization. To replace a stale local token, remove only the credential file and rerun the launcher:

```bash
rm -f local/.secrets/nuget-source-credentials
./local/Start\ CompileCare.command
```

Do not remove Docker volumes or repository clones for a NuGet authentication problem.

### C. Web Docker build context pointed inside CompileCare.Local

**Symptom**

Docker reported a missing path ending in `CompileCare.Local/CompileCare.Web`, even though `CompileCare.Web` existed as a sibling repository.

**Cause**

The Compose build context was resolved relative to the Compose file's directory and/or a relative override was interpreted from that base.

**Fix**

Use absolute paths for the build invocation:

```bash
export COMPILECARE_API_PATH="$(cd ../CompileCare.Api && pwd)"
export COMPILECARE_WEB_PATH="$(cd ../CompileCare.Web && pwd)"
```

The launcher and Compose path handling should continue to be hardened so users do not need to export these manually.

### D. API startup rejected HTTP OIDC authority

**Symptom**

API logged:

```text
Authentication:Authority must be an absolute HTTPS URL.
HTTP is allowed only in Development with explicit opt-in.
```

**Cause and fix**

The API intentionally allows HTTP metadata only when the environment is Development and `Authentication:RequireHttpsMetadata=false`. Local Compose supplies Development and maps `OIDC_REQUIRE_HTTPS_METADATA=false` to that option. Verify the resolved Compose configuration and container environment if the validation error returns. Do not relax the validation in non-development environments.

### E. Keycloak legacy standalone container name conflict

**Symptom**

An earlier manually started Keycloak container could reserve the `compilecare-keycloak` name even when stopped.

**Fix**

Local PR #3 updated the launcher to inspect all containers and remove the old standalone container only when it is not managed by the `compilecare-local` Compose project. The Compose-managed Keycloak instance uses the local realm import and persistent volume.

### F. API health check failed after startup

**Symptom**

The API container was unhealthy and the Web service could not satisfy its dependency.

**Observed cause**

The API host failed its startup options validation for the HTTP authority (see D). PostgreSQL migration logs showed the database was reachable and up to date. Keycloak and PostgreSQL health checks passed.

**Fix**

Correct the effective API OIDC authority/Development opt-in, rebuild/recreate the API and inspect API logs and health status. A message about optional `libgssapi_krb5.so.2` should be investigated separately if integrated authentication is required; it was not the fatal startup exception shown in this run.

### G. Workspace displayed but API authorization failed

**Symptom**

The reception shell rendered, but tenant-context discovery displayed `Your sign-in session is not authorized for the API.`

**Status**

Not yet resolved in the recorded local session. The remaining work is to verify the access token's issuer/audience and the API's effective configuration, then provision a trusted platform operator, tenant, initial tenant administrator, hospital, departments and user organization scopes. Do not create database records manually or bypass authorization as a workaround.

## 6. Safe rebuild and restart commands

After changing OIDC settings, rebuild the Web image:

```bash
docker compose --env-file local/.env -f local/docker-compose.yml build --no-cache web
docker compose --env-file local/.env -f local/docker-compose.yml up -d web
```

After changing API configuration or API source:

```bash
docker compose --env-file local/.env -f local/docker-compose.yml build --no-cache api
docker compose --env-file local/.env -f local/docker-compose.yml up -d api
```

For both services:

```bash
docker compose --env-file local/.env -f local/docker-compose.yml build --no-cache api web
docker compose --env-file local/.env -f local/docker-compose.yml up -d
docker compose --env-file local/.env -f local/docker-compose.yml ps
```

Use `docker compose down` only when a full stop is required. Do not add `-v` unless intentionally resetting all local database and identity-provider data.

## 7. Remaining local acceptance checklist

- [x] Docker Compose stack definition and launchers merged.
- [x] Keycloak realm import and PKCE/audience configuration added.
- [x] Legacy standalone Keycloak container migration handled by launcher.
- [x] Private GitHub NuGet restore uses a BuildKit secret.
- [x] Local PostgreSQL and Keycloak health checks passed on the developer Mac.
- [x] Web shell loaded after setting the local authority and temporarily narrowing scope to `openid`.
- [ ] Restore and verify standard OIDC client scopes (`profile`, `email`, roles and related defaults) in a fresh realm import.
- [ ] Verify the Web obtains an access token with the expected issuer and `compilecare-api` audience.
- [ ] Verify authenticated API requests succeed; diagnose the observed 401 before claiming local setup complete.
- [ ] Bootstrap a trusted platform operator and provision the first tenant and initial administrator through supported APIs.
- [ ] Create a hospital, departments and membership organization scopes through supported workflows.
- [ ] Complete browser-level reception patient search, registration, appointment and queue smoke tests.
- [ ] Validate the macOS launcher end-to-end without manual environment exports; validate Windows launcher separately.

## 8. Security and data handling reminders

- Never commit `local/.env`, `local/.secrets/` or PATs/passwords.
- Never paste access tokens, refresh tokens, passwords or private credentials into issues, PRs or chat.
- Do not use Keycloak's temporary admin account as a hospital workspace user.
- Do not bypass OIDC, API audience validation, tenant membership checks or hospital/department scope enforcement.
- Use supported platform and tenant provisioning APIs; preserve auditability.
- Keep local-only HTTP metadata opt-in confined to Development.
- Preserve named Docker volumes unless deliberately resetting the developer environment.
