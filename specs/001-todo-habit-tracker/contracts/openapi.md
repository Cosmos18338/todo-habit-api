# Public API Contract

All endpoints return JSON. Success responses use `{ "data": ..., "meta": ... }`; errors use `{ "error": { "code": "...", "message": "...", "details": [...] }, "request_id": "..." }`. Validation occurs before service rules. Resource ownership failures use `404` without disclosing whether another user's resource exists.

## Authentication

- `POST /auth/register` -> `201`: body `{email, password}`; returns public User data.
- `POST /auth/login` -> `200`: body `{email, password}`; returns an access token response and sets refresh cookie. Wrong email/password both return `401` with `AUTHENTICATION_FAILED`.
- `POST /auth/refresh` -> `200`: reads refresh cookie, rotates it, sets replacement cookie, and returns a new 15-minute access token. Invalid, expired, revoked, or reused tokens return `401`.
- `POST /auth/logout` -> `204`: revokes the current refresh family and clears the cookie.

The access token is sent as `Authorization: Bearer <token>` and is never persisted in localStorage. The refresh cookie is `HttpOnly; Secure`, scoped to `/auth/refresh`, and has a seven-day maximum age.

## Todo resources

- `POST /todos` -> `201`: `{title, description?, due_date?, priority?, is_completed?}`.
- `GET /todos?page=1&page_size=20` -> `200`: paginated list of the authenticated user's Todos. `page` is a positive integer defaulting to `1`; `page_size` is a positive integer defaulting to `20` and capped at `100`.
- `GET /todos/{todo_id}` -> `200`: one owned Todo.
- `PATCH /todos/{todo_id}` -> `200`: partial update of supplied fields.
- `DELETE /todos/{todo_id}` -> `204`: deletes one owned Todo and writes one audit record.

Defaults: priority `medium`, `is_completed=false`, nullable description/due date. Invalid title, date, priority, or status returns `422` with `VALIDATION_ERROR`.

Paginated Todo list responses use `data.items` for the current page and include `meta.page`, `meta.page_size`, `meta.total_count`, and `meta.has_next_page`. An empty list or a page beyond the last page returns `200` with an empty `items` array and valid metadata. A `page_size` below `1` or above `100`, or a `page` below `1`, returns `422` with `VALIDATION_ERROR`.

## Habit resources

- `POST /habits` -> `201`: `{name, frequency, target_count}`.
- `GET /habits?page=1&page_size=20` -> `200`: paginated list of the authenticated user's Habits. `page` is a positive integer defaulting to `1`; `page_size` is a positive integer defaulting to `20` and capped at `100`.
- `GET /habits/{habit_id}` -> `200`: one owned Habit.
- `PATCH /habits/{habit_id}` -> `200`: partial update with cross-field validation.
- `DELETE /habits/{habit_id}` -> `204`: deletes the Habit and dependent check-ins.
- `POST /habits/{habit_id}/check-ins` -> `201`: `{check_in_date}`; rejects future dates and duplicates.
- `GET /habits/{habit_id}/check-ins?start_date=YYYY-MM-DD&end_date=YYYY-MM-DD` -> `200`: inclusive ascending history.

Daily target count must equal `1`; Weekly target count must be an integer from `1` through `7`. Duplicate check-ins return `409` with `CHECK_IN_ALREADY_EXISTS`.

Paginated Habit list responses use `data.items` for the current page and include `meta.page`, `meta.page_size`, `meta.total_count`, and `meta.has_next_page`. An empty list or a page beyond the last page returns `200` with an empty `items` array and valid metadata. A `page_size` below `1` or above `100`, or a `page` below `1`, returns `422` with `VALIDATION_ERROR`.

## Progress

- `GET /progress/todos?start_date=YYYY-MM-DD&end_date=YYYY-MM-DD` -> `200`: `{start_date, end_date, completed_count, total_count, completion_rate}`. Filter is inclusive `due_date`; null due dates are excluded.
- `GET /progress/habits/{habit_id}/check-ins?start_date=YYYY-MM-DD&end_date=YYYY-MM-DD` -> `200`: ordered check-in history for the owned Habit.

Missing dates, invalid ISO dates, or `end_date < start_date` return `422` with `INVALID_DATE_RANGE`.

## Standard statuses and examples

- `200` successful read/update/authentication.
- `201` successful creation.
- `204` successful deletion/logout with no body.
- `401` missing/invalid authentication.
- `404` missing or not-owned resource.
- `409` duplicate email, duplicate check-in, or refresh reuse conflict.
- `422` schema or date/business input validation.
- `500` persistence or unexpected server failure; response contains no internal database detail.

FastAPI's generated OpenAPI schema must include these operation summaries, security requirements, request/response models, and examples for success and standard errors.
