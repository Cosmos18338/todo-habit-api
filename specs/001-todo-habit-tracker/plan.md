# Implementation Plan: Todo/Habit Tracker API

**Branch**: `001-todo-habit-tracker` | **Date**: 2026-08-28 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

建立具備使用者隔離的 Todo/Habit Tracker REST API，涵蓋註冊登入、Todo 與 Habit CRUD、每日/每週打卡、streak 與進度查詢。採 FastAPI 分層架構，使用 SQLAlchemy 2.x 與 PostgreSQL，透過 Alembic 管理 schema；所有 mutation 以 Unit of Work 在同一交易內寫入業務資料與 immutable audit log。

認證使用 PyJWT 簽發與驗證 15 分鐘 Access Token 及 7 天 Refresh Token。Access Token 僅由 client 記憶體攜帶，Refresh Token 存於 `HttpOnly; Secure` Cookie 並以 `jti`、token family 與 rotation state 支援自動換發及重放撤銷；任何 Token 不寫入 `localStorage`。密碼使用 `pwdlib` 的 Argon2id recommended 設定。

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: Python 3.11+

**Primary Dependencies**: FastAPI, Pydantic v2, SQLAlchemy 2.x, psycopg 3, Alembic, PyJWT, pwdlib[argon2], pytest, pytest-cov, httpx

**Storage**: PostgreSQL；audit_logs 與業務資料共用同一資料庫與 transaction，測試以獨立 PostgreSQL schema 優先

**Testing**: pytest + pytest-cov；HTTPX ASGI TestClient/AsyncClient；unit tests 驗證 schemas/services，integration tests 驗證 PostgreSQL transaction、constraints、audit 與 API contract

**Target Platform**: Docker Compose 本地 Linux container（API + PostgreSQL），可部署至一般 Linux container runtime

**Project Type**: web-service / REST API

**Performance Goals**: 正常服務條件下，至少 95% 的註冊、登入、Todo/Habit CRUD 請求於 2 秒內完成；依 SC-001、SC-002 驗證

**Constraints**: mutation 的業務寫入、streak 更新與 audit insert 必須同一 SQLAlchemy Session/transaction；commit 前不得回傳成功；request log 與 audit log 分離；不含協作、分享、通知或推播

**Scale/Scope**: 第一階段單一 API 服務與單一 PostgreSQL；個人使用者、Todo、Habit、Habit check-in、進度查詢與 OpenAPI 文件；不需既有資料 migration

## Constitution Check

_GATE: Must pass before Phase 0 research. Re-check after Phase 1 design._

| Gate                        | Status | Evidence / plan                                                                                                        |
| --------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------- |
| Test-first development      | PASS   | tasks.md 先建立 acceptance、contract、service/repository failing tests，再實作                                         |
| Layered architecture        | PASS   | `app/routers`, `app/services`, `app/repositories`, `app/models`, `app/schemas`, `app/utils`；Router 不直接存取 Session |
| Meaningful test design      | PASS   | 涵蓋密碼 7/8 邊界、Daily/Weekly target、日期範圍、空結果、權限隔離、streak 中斷與重複                                  |
| Service/repository coverage | PASS   | CI 執行 pytest-cov，Service 與 Repository 各自至少 80%                                                                 |
| REST consistency            | PASS   | 統一 success/error envelope、HTTP status 與 OpenAPI contract，契約測試固定格式                                         |
| Secure-by-default identity  | PASS   | Pydantic v2、pwdlib Argon2id、PyJWT issuer/audience/type 驗證、HttpOnly Secure refresh cookie                          |
| Reuse and abstraction       | PASS   | 共用認證、clock、request context、error、transaction 與 logging helper 放入 `app/utils/`                               |
| Branching/version control   | PASS   | 使用既有 `001-todo-habit-tracker` feature branch；本規劃不 commit 或 push                                              |
| API documentation           | PASS   | FastAPI OpenAPI schema 搭配 request/response/error/auth examples，quickstart 與 contract 同步驗證                      |

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
```

**Structure Decision**: Single backend web-service project rooted at `app/`, with explicit Router -> Service -> Repository -> Model boundaries. `schemas/` owns Pydantic request/response contracts; `db/uow.py` owns transaction scope; `utils/` owns authentication, request context, errors, clock and logging helpers. Alembic and tests remain top-level. This matches the requested layered architecture and keeps public API contracts separate from persistence models.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Phase 1 design re-check: PASS. The design preserves test-first delivery, explicit Router -> Service -> Repository -> Model boundaries, Pydantic validation, JWT security, the 80% Service/Repository coverage gate, consistent REST contracts, and generated OpenAPI documentation. No constitution violation requires an exception.

No complexity exceptions.

| Violation | Why Needed | Simpler Alternative Rejected Because |
| --------- | ---------- | ------------------------------------ |
| None      | N/A        | N/A                                  |

