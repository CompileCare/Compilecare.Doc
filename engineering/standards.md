# Engineering Standards

**Status:** Proposed baseline.

## Git workflow
Short-lived branches:
- `feature/<ticket>-<name>` (e.g., `feature/CC-123-patient-registration`)
- `bugfix/<ticket>-<name>` (e.g., `bugfix/CC-124-tenant-scope`)
- `docs/<name>` (e.g., `docs/heapcare-foundation-v1`)
- `refactor/<name>`
- `hotfix/<name>`
- `infra/<name>`

Ticket IDs may be omitted until an issue-tracking system is established. Avoid direct commits to protected main/release branches. PRs state purpose, scope, tests, risks, and screenshots/API examples where useful. Focused commits. Never commit secrets, patient data, exports, or local configuration.

## Architecture
Follow approved ADRs. The target is distributed, but delivery may use a modular core and selective independently deployable services. Preserve module/service ownership, explicit contracts, tenant context propagation, resilience, and observability. Do not introduce a new service without documenting the reason and its data ownership.

## .NET
Nullable enabled; readable code; DI; thin endpoints; rules in application/domain layers; boundary validation; async I/O; centralized errors; structured logging; validated options; migrations; unit and integration tests.

## Frontend
TypeScript; feature-oriented structure; typed API service abstractions; avoid duplicating business rules; accessibility/keyboard support; responsive layouts; loading/empty/error/unauthorized states; critical-flow tests.

## Database
Document schema ownership and naming; enforce keys/constraints; design indexes for actual queries; tenant scope; audit/history; migrations; review destructive changes with recovery plan. Independently deployed services own their data.

## API and messaging
Versioned/documented contracts; consistent errors; auth/authorization; pagination/filtering; idempotency/rate limits where needed; never expose internal exception details or sensitive fields. Events are versioned and consumers are idempotent. Define timeout, retry, and failure behavior.

## Testing
Unit business-rule tests; integration tests for DB/API/auth/tenancy/integrations; contract tests; architecture tests; tenant isolation/security tests. CI build/test/format/lint.

## Documentation
Update docs when behaviour, API, configuration, schema, deployment, or security changes. Requirements have acceptance criteria and observable outcomes. Use [Requirements Lifecycle](../product/requirements-lifecycle.md) and [Definition of Done](definition-of-done.md).
