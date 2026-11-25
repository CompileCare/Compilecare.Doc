# API Conventions and Error Contracts

**Status:** Working standard for the Phase 2 developer platform. Apply to new endpoints and migrate existing endpoints consistently.

## Resource and route conventions
- Use a versioned root: /api/v1.
- Use plural resource names and stable resource identifiers.
- Use HTTP methods consistently: GET read, POST create/action, PUT full update, PATCH only where partial update semantics are explicitly defined, DELETE remove/retire according to domain policy.
- Do not return persistence entities directly. Use API request/response contracts.
- Propagate request cancellation tokens into application and persistence calls.
- Use pagination on collection endpoints. Set bounded defaults and maxima; define stable deterministic ordering and include page, pageSize, totalCount, and totalPages.
- Successful create returns 201 Created and a Location header to the resource. Successful delete with no representation returns 204 No Content.
- Use 400 for invalid request shape/validation, 401 for missing/invalid authentication, 403 for authenticated but unauthorized, 404 for absent or out-of-scope resource, and 409 for business conflicts or concurrency conflicts where applicable.

## Error representation
Errors use RFC 9457 Problem Details and the media type application/problem+json. The common members are:
- type: stable problem type URI, or about:blank where no custom type is defined.
- title: short, stable summary.
- status: HTTP status code.
- detail: safe, actionable detail; never disclose stack traces, SQL, secrets, tokens, or unrelated patient data.
- instance: request path or occurrence identifier where appropriate.
- traceId: non-sensitive correlation/diagnostic identifier.

Validation may include an errors object mapping fields to safe messages. Do not create a second unrelated error envelope such as { message: ... } for some endpoints while others return Problem Details.

## Correlation
- Accept exactly one inbound X-Correlation-ID containing 1–64 ASCII letters, digits, periods, underscores, or hyphens; otherwise generate one.
- Return the effective value in the response X-Correlation-ID header.
- Include the ID in logs and Problem Details as traceId.
- Do not use correlation IDs as authorization material or place patient identifiers in them.

## Tenant and authorization
- X-Tenant-ID is a tenant selector/context request, not proof of access.
- Resolve authenticated identity first, then validate selected tenant membership from trusted claims/server-side membership data.
- Scope every data query and relationship by authorized tenant and organizational permissions.
- Return the same not-found semantics for absent and out-of-scope resources to reduce cross-tenant enumeration.

## Current-user tenant context and effective permissions

The authenticated UI discovers the user's active tenant memberships through `GET /api/v1/me/tenant-contexts`. Each returned tenant context includes:

- Tenant identity and display metadata.
- The persisted membership role.
- The authorized organization scopes (tenant, hospital and/or department).
- The effective permission identifiers derived by the API from its server-side role-to-permission mapping.

The permission list is a presentation contract for feature navigation and control visibility. The frontend may use it to hide unavailable modules, actions and navigation entries, but it must never treat a client-side permission check as authorization. Every protected API operation independently enforces its server-side permission and organization scope. Never infer permissions from token role claims or hard-code role-to-feature mappings in the browser; use the returned permission identifiers for the selected tenant context.

Tenant context discovery is identity-scoped and returns only active memberships for the validated issuer/subject. The response is paginated and must be refreshed after the user switches tenant context or an administrator changes membership role/scope. The endpoint does not issue, refresh or extend OIDC tokens; token renewal remains the identity provider/client responsibility.

## Concurrency and idempotency
- Use optimistic concurrency for mutable tenant-owned records.
- Map known concurrency/uniqueness/business conflicts consistently to 409.
- Introduce idempotency keys for retry-sensitive commands/operations when the workflow requires safe replay.
- Do not transparently retry POST/PATCH or other unsafe operations in the API client unless the operation has an explicit idempotency contract.

## OpenAPI
- Every endpoint has an operation name, tag, response status documentation, request/response schema, pagination parameters where applicable, and Problem Details responses.
- OpenAPI is generated from the API contract and validated in CI.
- Breaking changes require a versioning/deprecation decision.

## Current implementation status
The facility API now maps application-level 400/404/409 outcomes to RFC 9457 Problem Details. Integration tests assert content type/status/title/detail for duplicate and out-of-tenant not-found cases. Correlation IDs are constrained to the format documented above and are returned in the response. The ApiClient package parses Problem Details and exposes the correlation ID without including server detail in the exception message.

The standard is the target for new endpoints. Route-level OpenAPI response annotations, generated-schema validation, and consumer-package verification remain follow-up hardening tasks; do not assume every existing endpoint already documents every possible error response.
