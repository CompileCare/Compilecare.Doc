# Requirements Lifecycle

**Status:** Proposed working process.

## Flow
Business Need → Workflow → Functional Requirements → Acceptance Criteria → Technical Design/ADR → Implementation → Testing → Review → Release → Feedback.

## 1. Business need
Record the problem, affected roles, client context, measurable outcome, and scope/non-scope.

## 2. Workflow
Document actors, preconditions, main flow, alternatives, exceptions, data created/updated, permissions, and audit requirements.

## 3. Functional requirements
Use stable IDs (for example, CC-PAT-001). Each requirement should be testable and state expected behavior, permissions, and tenant/hospital scope.

## 4. Acceptance criteria
Prefer observable Given/When/Then scenarios. Include normal, validation, authorization, tenant-isolation, failure, and edge cases as appropriate.

## 5. Technical design
Capture module/service ownership, API/event contracts, data model, migrations, security/privacy, observability, failure/recovery, and relevant ADRs.

## 6. Implementation and tests
Link code and tests to requirement IDs. Include unit, integration, contract, architecture, and security/tenant tests as applicable.

## 7. Review and release
PR review, CI, documentation, migration/recovery plan, release notes, deployment readiness, and acceptance/demo.

## Change management
Changes to scope or workflow update the requirement, acceptance criteria, and design docs. Decisions that affect architecture are recorded in ADRs.
