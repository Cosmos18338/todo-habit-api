# Research: Todo/Habit Tracker API

## Decision: FastAPI layered service

- **Decision**: Use FastAPI routers for HTTP concerns, services for business rules, repositories for persistence, SQLAlchemy models for storage, and Pydantic v2 schemas for external contracts.
- **Rationale**: This directly satisfies the constitution's layered architecture and keeps streak, ownership, and transaction rules independently testable.
- **Alternatives considered**: Direct database access from routers was rejected because it couples HTTP behavior to persistence and weakens authorization review.

## Decision: JWT authentication and refresh rotation

- **Decision**: Use PyJWT with an allow-listed algorithm and configured issuer/audience. Access JWTs contain `sub`, `jti`, `iat`, `exp`, and `typ=access`, expire after 15 minutes, and are returned for in-memory client use. Refresh JWTs expire after 7 days, contain a `jti`, `family_id`, and `typ=refresh`, and are sent only in an `HttpOnly; Secure` cookie. Persist a hash of the refresh `jti`/family state to support rotation, revocation, and reuse detection. Never use localStorage.
- **Rationale**: The specified JWT requirement is preserved while server-side family state makes automatic renewal and replay revocation possible. Explicit issuer, audience, expiry, and token type checks prevent token confusion.
- **Alternatives considered**: A stateless refresh JWT was rejected because it cannot reliably revoke or detect reuse. An opaque refresh token is a sound alternative, but it would not follow the supplied requirement that PyJWT sign and verify JWT credentials.

## Decision: Password security

- **Decision**: Use `pwdlib[argon2]` `PasswordHash.recommended()` and `verify_and_update`; store only the resulting Argon2id hash.
- **Rationale**: Argon2id is appropriate for password storage, and the library centralizes safe parameters. Login failures use one public message for unknown email and wrong password.
- **Alternatives considered**: Plain SHA hashing and application-managed salts are rejected. bcrypt is a compatibility fallback, not the first choice for this new service.

## Decision: Unit of Work and audit consistency

- **Decision**: Inject one SQLAlchemy 2.x Session per request. Services run mutations inside `with session.begin()`: validate, load with owner filter, capture before state, mutate, flush, insert exactly one audit row, then commit. Repositories never commit. Audit rows live in PostgreSQL beside business tables and use JSONB before/after snapshots, UTC timestamp, actor, operation, entity, entity id, and request id.
- **Rationale**: Commit succeeds or the business change and audit record both roll back, satisfying NFR-001 and NFR-002. A database unique constraint on `(habit_id, check_in_date)` is the final duplicate check under concurrency.
- **Alternatives considered**: Separate audit database, message queue, background task, or ORM-only append hooks were rejected because they cannot guarantee same-transaction persistence and exact operation metadata.

## Decision: Immutable audit protection

- **Decision**: Application role receives no UPDATE/DELETE privileges on `audit_logs`; a PostgreSQL trigger rejects update/delete attempts. Migrations use a separate owner role. Normal APIs expose no audit mutation endpoints.
- **Rationale**: ORM omission alone is not a database protection. Defense in depth makes the append-only requirement enforceable.
- **Alternatives considered**: An application-only convention was rejected because privileged SQL or an accidental bulk update could bypass it.

## Decision: Date and streak semantics

- **Decision**: Use `date` for calendar values and timezone-aware UTC `datetime` for events. Daily streaks count consecutive dates. Weekly streaks group by ISO year and ISO week and count a week only when check-ins reach `target_count`. Future check-ins are rejected; date ranges are inclusive.
- **Rationale**: UTC and ISO year/week avoid local-time and cross-year errors. Recomputing streaks from check-in history inside the mutation transaction is initially simpler and correctly handles gaps and corrections.
- **Alternatives considered**: Client timezone customization and only storing a mutable streak counter are out of scope or error-prone for backdated data.

## Decision: Request logging

- **Decision**: Middleware creates or validates a UUID request id, stores it in a ContextVar, returns it in `X-Request-ID`, and logs method, path, actor or anonymous marker, duration, and status in structured JSON using Python logging. INFO covers normal requests, WARNING covers validation/auth failures, and ERROR covers unexpected or database failures. Tokens, passwords, hashes, cookies, and authorization headers are redacted.
- **Rationale**: Application logs satisfy NFR-003 while remaining operationally separate from audit records; ContextVar preserves request-local data in async execution.
- **Alternatives considered**: Writing application logs into audit_logs was rejected because logs do not require transaction consistency and have different retention/query needs.

## Decision: Testing and migrations

- **Decision**: Use pytest, pytest-cov, HTTPX ASGI transport/TestClient, and PostgreSQL integration tests. Prefer a dedicated test schema or database with function-scoped transaction rollback; use SQLite in-memory only for narrow schema/service tests that do not depend on PostgreSQL behavior. Use Alembic migrations as the schema source of truth and review autogenerate output manually.
- **Rationale**: PostgreSQL is required to verify JSONB, timezone behavior, unique constraints, transaction rollback, and audit triggers. HTTPX integration tests prove the public contract and ownership isolation.
- **Alternatives considered**: `create_all()` and SQLite-only integration tests were rejected because they do not exercise migration drift or PostgreSQL-specific constraints.
