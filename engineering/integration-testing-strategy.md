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
- Wrong issuer, wrong audience, expired token, missing subject and invalid subject shape are rejected.
- Missing or malformed tenant selection is rejected.
- A tenant ID without a matching active membership and an inactive membership are denied.
- `HospitalAdmin`, `FacilityManager`, and `FacilityViewer` permissions are checked through real API requests.
- Forged token role/permission claims cannot grant access beyond the stored membership role.
- Current API CI evidence after PR #10: 3 unit tests and 29 PostgreSQL-backed integration tests passed.

The test signing key and issuer are synthetic and must never be configured in deployed environments.

## CI gates
A PR must restore, build, run unit tests, run API/client integration tests, validate OpenAPI/contracts, and complete applicable code/security scans. Report the exact tests executed and the database/provider used.
