# Data Model

## User

- `id`: UUID primary key.
- `email`: normalized lowercase string, unique, required.
- `password_hash`: Argon2id hash, required, never exposed.
- `created_at`, `updated_at`: UTC timestamps.

Relationships: one User owns many Todos, Habits, and refresh-token states; audit logs reference the acting user without allowing normal user deletion of audit history.

## Todo

- `id`: UUID primary key.
- `user_id`: UUID foreign key to User, required and indexed.
- `title`: non-empty trimmed string, required.
- `description`: nullable string.
- `due_date`: nullable calendar date; only this field is used by completion statistics.
- `priority`: documented enum `low`, `medium`, `high`; default `medium`.
- `is_completed`: boolean; default `false`.
- `created_at`, `updated_at`: UTC timestamps.

Validation: title cannot be blank, due date must be a valid ISO date, priority must be one of the enum values, and status is boolean. Update schemas distinguish omitted fields from explicit null where applicable.

## Habit

- `id`: UUID primary key.
- `user_id`: UUID foreign key to User, required and indexed.
- `name`: non-empty trimmed string, required.
- `frequency`: enum `daily` or `weekly`, required.
- `target_count`: integer, required; daily is exactly 1, weekly is 1 through 7.
- `current_streak`, `longest_streak`: non-negative derived counters, updated with check-in mutations.
- `created_at`, `updated_at`: UTC timestamps.

State transition: create/update validates frequency and target count; check-in recomputes streak values from owned history within the same transaction; delete removes owned check-ins through cascade so no orphan history remains.

## HabitCheckIn

- `id`: UUID primary key.
- `habit_id`: UUID foreign key to Habit, required.
- `check_in_date`: UTC calendar date, required.
- `created_at`: UTC timestamp.

Constraint: unique `(habit_id, check_in_date)`. Future dates are rejected. A duplicate returns a stable conflict error and does not insert another row.

## RefreshTokenState

- `id`: UUID primary key.
- `user_id`: UUID foreign key to User.
- `family_id`: UUID identifying a rotation family.
- `token_jti_hash`: hash of the JWT jti, unique.
- `issued_at`, `expires_at`, `used_at`, `revoked_at`: UTC timestamps; used/revoked are nullable.

A successful refresh atomically marks the presented token used and persists the replacement state. Reuse of a used token revokes its family.

## AuditLog

- `id`: UUID primary key.
- `actor_user_id`: UUID, required.
- `operation`: enum `create`, `update`, `delete`, `check-in`.
- `entity_type`: controlled string such as `todo`, `habit`, `habit_check_in`.
- `entity_id`: UUID/string identifier.
- `before`: nullable JSONB snapshot.
- `after`: nullable JSONB snapshot.
- `occurred_at`: UTC timestamp, required.
- `request_id`: UUID/string, required.

Audit records are append-only, have no normal application update/delete path, and are stored in the same PostgreSQL transaction as each mutation.

## Progress Summary

A read-only response projection, not a persisted table. It contains inclusive UTC `start_date` and `end_date`, Todo `completed_count`, `total_count`, `completion_rate`, and ordered Habit check-in history. Todos without `due_date` are excluded; a zero denominator returns zero counts and rate without division by zero.
