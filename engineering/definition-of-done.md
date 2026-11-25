# Definition of Done

**Status:** Proposed baseline; tailor by change type.

A feature or fix is complete only when applicable items are satisfied:

- [ ] Requirement/workflow and acceptance criteria are linked or documented.
- [ ] Scope, design, and architectural implications reviewed; ADR created/updated where needed.
- [ ] Code reviewed and follows repository standards.
- [ ] Unit and integration tests pass; contract/architecture/security tests added where applicable.
- [ ] Tenant and facility scope enforced; cross-tenant isolation verified for affected paths.
- [ ] Authentication, authorization, validation, and sensitive-data handling reviewed.
- [ ] Database migration and recovery/rollback considerations documented.
- [ ] Logs, metrics, traces, alerts, and redaction updated as needed.
- [ ] API/event contracts and user-facing documentation updated.
- [ ] CI build, tests, formatting, linting, and security checks pass.
- [ ] No secrets, patient data, or unsafe generated artifacts committed.
- [ ] Demo/acceptance evidence available for user-facing or workflow changes.
- [ ] User-facing UI changes follow the experience quality bar; review desktop, tablet and narrow viewports, keyboard-only completion, visible focus, semantic labels, contrast and asynchronous states.
- [ ] Visual evidence is captured from the actual application with synthetic/local data; do not use fabricated screens or expose patient data, tokens or credentials.
- [ ] Release notes and deployment considerations included when required.
