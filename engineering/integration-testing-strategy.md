# Integration Testing Strategy

**Status:** Phase 2 working strategy. The current API test project contains PostgreSQL-backed facility integration tests and a WebApplicationFactory; extend the existing approach rather than creating a parallel test stack.

## Test layers

### 1. Unit tests
Test deterministic application/domain logic without infrastructure:
- Validation, normalization, pagination and permission rules.
- HTTP client delegating handlers and provider behaviours.
- Problem Details deserialization and exception mapping.
- Retry-safety predicate and options validation.

### 2. API integration tests
Use WebApplicationFactory<Program> and a dedicated PostgreSQL test database:
- Exercise routing, model binding, middleware, exception handling and response contracts.
- Verify database constraints, migrations, tenant filters and concurrency behaviour.
- Test positive and negative OIDC/JWT authentication, tenant membership and role-based authorization cases.
- Assert content type and schema of Problem Details, not only status codes.
- Use synthetic tenant/resource data; no real patient data.

The API integration fixture resets only a dedicated database ending in `_test`, then uses `Database.Migrate()` to apply the versioned EF Core migrations. The current migration integration tests verify that a clean database receives the facility hierarchy schema, the expected migration is recorded, no migrations remain pending, all hierarchy sets can be queried, and reapplying migrations is idempotent. The fixture does not use `EnsureCreated`.

CI also runs `dotnet ef migrations has-pending-model-changes` to detect a model change that was not captured in a new migration and snapshot. When a prior schema version is released, add explicit upgrade tests from that version and keep representative schema/data fixtures for any data-preserving migrations.

### 3. API-client tests
Use a controlled in-memory HttpMessageHandler/test server:
- Base address, named and typed client resolution.
- Access-token, tenant and correlation header injection and overwrite rules.
- Missing/invalid provider values and cancellation.
- RFC 9457 Problem Details parsing, including validation errors and correlation ID.
- Transient GET retry, unsafe POST non-retry, timeout and circuit-breaker behaviour.
- Ensure exceptions and logs never include access tokens or patient-sensitive payloads.

### 4. Contract tests
- Validate generated OpenAPI JSON in CI.
- Verify status codes, response schemas, pagination envelope, and Problem Details shape.
- Exercise a typed client against an API contract fixture or a running API integration host.
- Treat incompatible API changes as explicit versioning decisions.

## Isolation and safety
- Each test run uses a dedicated isolated database and known synthetic data.
- Never point a destructive test fixture at development, staging or production databases.
- Do not use real names, contact details, patient records, tokens, or credentials in fixtures.
- Tests must be deterministic and independently repeatable. Integration tests sharing the same PostgreSQL test database are serialized through an xUnit collection fixture; do not parallelize destructive database resets.
- External dependencies should be replaced with controlled test doubles or disposable containers.
- Assert tenant isolation explicitly; do not rely only on EF query filters.

### 5. Authentication and authorization integration tests

The facility API integration suite uses the real ASP.NET Core JWT Bearer handler and the production issuer-stamping callback. It issues short-lived synthetic HS256 JWTs with a test-only key; only OIDC discovery metadata and the signing key are replaced for deterministic tests. The production validation pipeline is not replaced with a fake authentication handler.

Coverage includes:
- Valid token without membership is authenticated but denied tenant access.
- A valid hospital-scoped membership in tenant A cannot read a patient registered in tenant B; the API returns not-found and audit metadata is attributed to tenant A (API PR #46; PostgreSQL CI #779 and Docker Integration Verification #161).
- Wrong issuer, wrong audience, expired token, missing subject and invalid subject shape are rejected.
- Missing or malformed tenant selection is rejected.
- A tenant ID without a matching active membership and an inactive membership are denied.
- `HospitalAdmin`, `FacilityManager`, and `FacilityViewer` permissions are checked through real API requests.
- Forged token role/permission claims cannot grant access beyond the stored membership role.
- Current API CI evidence after PR #10: 3 unit tests and 29 PostgreSQL-backed integration tests passed.

The test signing key and issuer are synthetic and must never be configured in deployed environments.

## CI gates
A PR must restore, build, run unit tests, run API/client integration tests, validate OpenAPI/contracts, and complete applicable code/security scans. Report the exact tests executed and the database/provider used.


## 6. Browser end-to-end tests

The authenticated web application uses Playwright with Chromium. The suite lives in `HeapCare.Web/e2e` and is run in CI through `npm run test:e2e` after installing the pinned browser runtime.

Current browser coverage includes:
- Billing workspace navigation is hidden when the selected membership has no billing permission.
- Invoice issue, payment capture, reload, payment-history recovery, prior receipt retrieval and print.
- Full walk-in OPD user journey: patient registration and duplicate review, hospital-local MRN assignment, patient search, walk-in queue token, call/start transitions, department-scoped identity verification, encounter note/sign/close, queue completion and invoice/payment.
- Appointment booking: hospital-registered patient search, provider availability discovery, slot selection, booking confirmation, check-in and server-issued queue token.
- OIDC silent session renewal: a synthetic short-lived session exchanges its refresh token at a controlled identity-provider endpoint; the workspace remains signed in and subsequent API requests use the renewed bearer token.
- OIDC renewal failure: a controlled `invalid_grant` refresh-token response causes the UI to display the safe re-authentication warning while the existing access token remains unexpired.
- Appointment slot contention: the first booking request returns 409; the UI clears the stale selection, reloads the session's slots, disables the claimed slot and allows booking the next available time.

The browser tests install a synthetic OIDC session and intercept the API with deterministic in-memory fixtures. They validate user-visible workflows and permission-driven feature gating; they do not replace API integration tests, real JWT/tenant membership tests or PostgreSQL persistence/concurrency checks. Never use real patient data, credentials, identity-provider tokens or production APIs in browser fixtures.

Vitest discovery is limited to `src/**/*.{test,spec}.{ts,tsx}`; Playwright discovery is limited to `e2e`. Keep the runners separate so each executes only its intended tests. The Web CI gate runs unit/component tests, TypeScript/Vite production build, Chromium browser tests and Docker build/startup smoke tests. OIDC tests must use synthetic sessions and intercepted discovery/token endpoints; never connect browser tests to a real identity provider or include real credentials.
