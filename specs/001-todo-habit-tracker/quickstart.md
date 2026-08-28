# Quickstart Validation

## Prerequisites

- Docker Desktop with Compose.
- Python 3.11+ and a virtual environment for local test commands.
- A generated `JWT_SECRET_KEY` supplied through environment configuration.

## Start the development environment

```powershell
docker compose up --build -d
alembic upgrade head
```

The API is available at `http://localhost:8000`. OpenAPI is available at `/docs` and `/redoc`.

## Run the focused test suite

```powershell
pytest -q
pytest --cov=app/services --cov=app/repositories --cov-fail-under=80
```

Integration tests must use a dedicated PostgreSQL test schema/database and must not use a developer database.

## End-to-end scenarios

1. Register with an 8-character password and confirm the response has no password hash.
2. Log in with valid credentials; confirm a 15-minute access JWT is returned and the refresh JWT is only set as an `HttpOnly; Secure` cookie with a 7-day expiry.
3. Call a protected Todo endpoint without a token, with an expired token, and with a valid token; verify consistent authentication errors and success behavior.
4. Create, update, list, retrieve, and delete a Todo. Repeat using a second user and verify the first user's resource is not disclosed.
5. Create Daily and Weekly Habits; verify Daily target `1`, Weekly targets `1` and `7`, and rejection at invalid boundaries.
6. Record consecutive and interrupted Daily check-ins, then verify current and longest streaks. Record enough Weekly check-ins to cross the target threshold and verify ISO-week streaks.
7. Repeat one check-in for the same habit/date and verify a conflict response and exactly one database row.
8. Query an inclusive Todo due-date range and Habit history range, including a zero-result and invalid-range case.
9. Create enough Todos or Habits to span multiple pages, query `GET /todos?page=1&page_size=20` and `GET /habits?page=1&page_size=20`, and verify `meta.page`, `meta.page_size`, `meta.total_count`, and `meta.has_next_page` are present.
10. Query a page beyond the final page, such as `GET /todos?page=999&page_size=20`, and verify it returns `200` with an empty `data.items` array and valid pagination metadata rather than an error. Also verify `page_size=100` is accepted while `page_size=101` is rejected with `422`.
11. Force a flush or commit failure and verify no business row or audit row remains while the API returns an explicit error.
12. Inspect captured logs and verify every request has request id, actor/anonymous marker, duration, and status; verify validation/auth/database failures have distinct severity.

Contract details and response/error shapes are defined in [contracts/openapi.md](contracts/openapi.md). Entity constraints are defined in [data-model.md](data-model.md).
