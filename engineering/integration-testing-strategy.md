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
- Test positive and negative authorization cases once Phase 3 identity is integrated.
- Assert content type and schema of Problem Details, not only status codes.
- Use synthetic tenant/resource data; no real patient data.

The existing test fixture resets a database ending in _test and currently uses EnsureDeleted/EnsureCreated. After migrations are added in Phase 3, change the fixture to apply migrations and test clean install plus upgrade from the previous supported schema.

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
- Tests must be deterministic, independently repeatable, and safe under CI parallelism (or serialize tests that share a database).
- External dependencies should be replaced with controlled test doubles or disposable containers.
- Assert tenant isolation explicitly; do not rely only on EF query filters.

## CI gates
A PR must restore, build, run unit tests, run API/client integration tests, validate OpenAPI/contracts, and complete applicable code/security scans. Report the exact tests executed and the database/provider used.
