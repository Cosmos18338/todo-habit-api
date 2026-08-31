# Implementation Plan: Todo/Habit Tracker API

**Branch**: `001-todo-habit-tracker` | **Date**: 2026-08-28 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

Build a Todo/Habit Tracker REST API with user isolation, covering registration and login, Todo and Habit CRUD, daily/weekly check-ins, streaks, and progress queries. Use a layered FastAPI architecture with SQLAlchemy 2.x and PostgreSQL, managing the schema through Alembic; all mutations use a Unit of Work to write business data and the immutable audit log in the same transaction.

Authentication uses PyJWT to issue and verify a 15-minute Access Token and a 7-day Refresh Token. The Access Token is carried only in client memory, while the Refresh Token is stored in an `HttpOnly; Secure` Cookie and uses `jti`, token family, and rotation state to support automatic renewal and replay revocation; no Token is written to `localStorage`. Passwords use `pwdlib`'s Argon2id recommended configuration.

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: Python 3.11+

**Primary Dependencies**: FastAPI, Pydantic v2, SQLAlchemy 2.x, psycopg 3, Alembic, PyJWT, pwdlib[argon2], pytest, pytest-cov, httpx

**Storage**: PostgreSQL; `audit_logs` and business data share the same database and transaction, with an isolated PostgreSQL schema preferred for testing

**Testing**: pytest + pytest-cov; HTTPX ASGI TestClient/AsyncClient; unit tests validate schemas/services, and integration tests validate PostgreSQL transactions, constraints, audit records, and the API contract

**Target Platform**: Docker Compose local Linux container (API + PostgreSQL), deployable to a standard Linux container runtime

**Project Type**: web-service / REST API

**Performance Goals**: Under normal service conditions, at least 95% of registration, login, and Todo/Habit CRUD requests complete within 2 seconds; validated according to SC-001 and SC-002

**Constraints**: Business writes, streak updates, and audit inserts for a mutation must use the same SQLAlchemy Session/transaction; success must not be returned before commit; request logs and audit logs are separate; collaboration, sharing, notifications, and push notifications are excluded

**Scale/Scope**: The first phase uses a single API service and a single PostgreSQL instance; it covers individual users, Todos, Habits, Habit check-ins, progress queries, and OpenAPI documentation; no existing data migration is needed

## CI/CD

Use GitHub Actions `.github/workflows/ci.yml` as the continuous-integration workflow. The workflow must trigger when a Pull Request targeting `main` receives `opened`, `synchronize`, or `reopened`, ensuring checks rerun for every new commit on the source branch; it must also run on direct pushes to `main` as the final post-merge validation.

The CI job runs in this order:

1. Install the pinned Python version and dependencies.
2. Run `ruff check` for linting.
3. Run Pytest tests.
4. Run `pytest --cov=app/services --cov=app/repositories` and apply an 80% minimum coverage threshold separately to Service and Repository; the workflow must fail if either layer is below its threshold.

Create this workflow immediately after the project skeleton is established so that every subsequent implementation task receives CI feedback for Pull Requests and pushes to `main`.

## Constitution Check

_GATE: Must pass before Phase 0 research. Re-check after Phase 1 design._

| Gate                        | Status | Evidence / plan                                                                                                                                  |
| --------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Test-first development      | PASS   | `tasks.md` creates acceptance, contract, and service/repository failing tests before implementation                                              |
| Layered architecture        | PASS   | `app/routers`, `app/services`, `app/repositories`, `app/models`, `app/schemas`, `app/utils`; Router does not access Session directly             |
| Meaningful test design      | PASS   | Covers password 7/8 boundaries, Daily/Weekly targets, date ranges, empty results, authorization isolation, and interrupted and duplicate streaks |
| Service/repository coverage | PASS   | CI runs pytest-cov, with at least 80% for Service and Repository individually                                                                    |
| REST consistency            | PASS   | A unified success/error envelope, HTTP status semantics, and OpenAPI contract, with contract tests fixing the format                             |
| Secure-by-default identity  | PASS   | Pydantic v2, pwdlib Argon2id, PyJWT issuer/audience/type validation, HttpOnly Secure refresh cookie                                              |
| Reuse and abstraction       | PASS   | Shared authentication, clock, request context, error, transaction, and logging helpers are placed in `app/utils/`                                |
| Branching/version control   | PASS   | Uses the existing `001-todo-habit-tracker` feature branch; this plan does not commit or push                                                     |
| API documentation           | PASS   | FastAPI OpenAPI schema with request/response/error/auth examples, with quickstart and contract validation kept in sync                           |

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

<!--
  ACTION REQUIRED: Replace the placeholder tree below with the concrete layout
  for this feature. Delete unused options and expand the chosen structure with
  real paths (e.g., apps/admin, packages/something). The delivered plan must
  not include Option labels.
-->

```text
app/
├── main.py
├── config.py
├── db/
│   ├── session.py
│   └── uow.py
├── models/
├── schemas/
├── repositories/
├── services/
├── routers/
└── utils/
alembic/
├── env.py
└── versions/
tests/
├── contract/
├── integration/
└── unit/
docker-compose.yml
Dockerfile
pyproject.toml
.github/
└── workflows/
  └── ci.yml
```

**Structure Decision**: Single backend web-service project rooted at `app/`, with explicit Router -> Service -> Repository -> Model boundaries. `schemas/` owns Pydantic request/response contracts; `db/uow.py` owns transaction scope; `utils/` owns authentication, request context, errors, clock and logging helpers. Alembic and tests remain top-level. This matches the requested layered architecture and keeps public API contracts separate from persistence models.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Phase 1 design re-check: PASS. The design preserves test-first delivery, explicit Router -> Service -> Repository -> Model boundaries, Pydantic validation, JWT security, the 80% Service/Repository coverage gate, consistent REST contracts, and generated OpenAPI documentation. No constitution violation requires an exception.

No complexity exceptions.

| Violation | Why Needed | Simpler Alternative Rejected Because |
| --------- | ---------- | ------------------------------------ |
| None      | N/A        | N/A                                  |

