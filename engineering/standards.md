# Engineering Standards

**Status:** Proposed baseline.

## Git workflow
Short-lived branches: `feature/<ticket>-<name>`, `bugfix/<ticket>-<name>`, `docs/<name>`. Avoid direct commits to protected main/release branches. PRs state purpose, scope, tests, risks, and screenshots/API examples where useful. Focused commits. Never commit secrets, patient data, exports, or local configuration.

## .NET
Nullable enabled; readable code; DI; thin endpoints; rules in application/domain layers; boundary validation; async I/O; centralized errors; structured logging; validated options; migrations; unit and integration tests.

## Frontend
TypeScript; feature-oriented structure; typed API service abstractions; avoid duplicating business rules; accessibility/keyboard support; responsive layouts; loading/empty/error/unauthorized states; critical-flow tests.

## Database
Document schema ownership and naming; enforce keys/constraints; design indexes for actual queries; tenant scope; audit/history; migrations; review destructive changes with recovery plan.

## API
Versioned/documented contracts; consistent errors; auth/authorization; pagination/filtering; idempotency/rate limits where needed; never expose internal exception details or sensitive fields.

## Testing
Unit business-rule tests; integration tests for DB/API/auth/tenancy/integrations; contract tests; architecture tests; tenant isolation/security tests. CI build/test/format/lint.

## Documentation
Update docs when behaviour, API, configuration, schema, deployment, or security changes. Requirements have acceptance criteria and observable outcomes.
