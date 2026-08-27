<!--
Sync Impact Report
- Version change: unversioned scaffold -> 1.0.0
- Modified principles: none; all nine principles are newly defined from the project brief
- Added sections: Security and API Constraints; Development Workflow and Quality Gates
- Removed sections: none
- Follow-up TODOs: TODO(RATIFICATION_DATE), because the original adoption date is unknown
-->

# Todo/Habit Tracker REST API Constitution

## Core Principles

### I. Test-First Development (NON-NEGOTIABLE)

All business logic and API endpoints MUST begin with failing Pytest tests, followed by the
smallest implementation that makes them pass, then refactoring. Tests are the executable
specification and MUST define boundary conditions and observable behavior. This Red-Green-Refactor
cycle prevents unverified behavior from becoming part of the API contract.

### II. Layered Architecture

The application MUST separate Router, Service, Repository, and Model layers. Each layer MUST have
one primary responsibility and MUST communicate through explicit contracts; a Router handles HTTP,
a Service handles business rules, a Repository handles persistence, and a Model represents data.
Cross-layer shortcuts are technical debt and require explicit remediation because clear boundaries
keep business logic independently testable.

### III. Meaningful Test Design (NON-NEGOTIABLE)

Tests MUST use equivalence partitioning and boundary-value analysis. A test suite MUST cover each
behaviorally distinct equivalence class and its relevant boundaries without repeating the same class
only to vary input types. Coverage is a health indicator and MUST NOT replace traceability from
acceptance criteria to meaningful test cases.

### IV. Service and Repository Coverage

Tests for the Service and Repository layers MUST achieve at least 80% coverage, measured with
`pytest-cov`. Coverage thresholds are quality gates, while the test-design requirements in this
constitution determine whether the covered behavior is meaningful.

### V. Consistent REST API Design

Public endpoints MUST follow RESTful conventions and MUST return a consistent JSON structure,
standard HTTP status codes, and a consistent error-response format. Changes to externally visible
contracts MUST update their tests and generated OpenAPI description so clients can rely on stable,
predictable behavior.

### VI. Secure-by-Default Input and Identity

All external input MUST be validated with Pydantic. Passwords MUST be stored only as secure hashes,
never as plaintext, and protected API operations MUST require JWT authentication. Security behavior
MUST be covered by tests for both accepted and rejected authorization and validation cases.

### VII. Reuse with Proportionate Abstraction

Shared utility functions MUST live in `app/utils/`. Before adding logic, contributors MUST check
whether an existing abstraction can be reused. New abstractions MUST match the project's scale and
remove real duplication or clarify a stable responsibility; speculative generality is prohibited.

### VIII. Disciplined Branching and Version Control

Branches MUST follow `<type>/<scope>-<short-description>`. Git worktrees are prohibited. Commit
messages MUST follow Conventional Commits. Changes MUST be merged through a Pull Request and MUST
NOT be pushed directly to `main`, preserving reviewable history and an auditable quality gate.

### IX. API Documentation

The public API MUST expose automatically generated OpenAPI documentation with request, response,
error, and authentication examples. Documentation MUST remain synchronized with the implemented
contract and its tests so the API is usable without reverse-engineering the source.

## Security and API Constraints

The requirements in this section apply to every feature exposed through the API. Pydantic schemas
MUST reject invalid input before business logic executes. Authentication and authorization failures
MUST use the standard error format and MUST NOT disclose sensitive implementation details. API
responses MUST avoid exposing password hashes, signing secrets, tokens beyond their intended use,
or internal persistence details.

## Development Workflow and Quality Gates

Every feature MUST progress through a written specification, failing tests, implementation, and
refactoring. The plan and task list MUST identify the affected architectural layer, acceptance
tests, security implications, coverage measurement, and OpenAPI impact. A Pull Request is ready for
merge only when its relevant tests pass, Service and Repository coverage meets 80%, API contracts
are documented, and a reviewer has checked compliance with this constitution.

## Governance

This constitution takes precedence over individual feature decisions. `plan.md` and `tasks.md`
MUST be revised when they conflict with these principles; schedule pressure MUST NOT justify
skipping test-first development, layer boundaries, security controls, or Pull Request review.

Amendments require a documented rationale, an updated Sync Impact Report, and review in a Pull
Request. The amendment MUST state affected principles, migration implications, and any resulting
updates required for specifications, plans, tasks, tests, or documentation.

Versioning follows Semantic Versioning: MAJOR for backward-incompatible removals or redefinitions
of governance; MINOR for new principles, sections, or materially expanded requirements; PATCH for
clarifications and non-semantic wording changes. The Last Amended date MUST be updated whenever
the constitution changes.

Every feature Pull Request MUST include a compliance review covering test-first evidence, layer
ownership, meaningful test partitions, coverage, API consistency, security, reuse, version control,
and documentation. Reviewers MUST record justified exceptions and their remediation plan; no silent
waivers are permitted.

**Version**: 1.0.0 | **Ratified**: TODO(RATIFICATION_DATE): original adoption date is unknown | **Last Amended**: 2026-08-27

