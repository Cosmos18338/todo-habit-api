# Feature Specification: Todo/Habit Tracker API

**Feature Branch**: `001-todo-habit-tracker`

**Created**: 2026-08-27

**Status**: Draft

**Input**: User description: "Develop a Todo/Habit Tracker API that enables users to manage tasks and habit tracking. Include user registration and login, complete Todo and Habit management, daily Habit check-ins and streak calculations, date-range queries for completion rates and check-in history, and user data isolation; the first phase excludes team collaboration, sharing, notifications, and push notifications."

## Clarifications

### Session 2026-08-28

- Q: Which date field should filter Todo records for a Todo completion-rate date range? → A: Filter by `due_date`; Todo records without a due date are excluded from date-range statistics.
- Q: How should a Weekly Habit's `target_count` affect check-ins and streak calculations? → A: At least `target_count` check-ins must be completed each week, and the streak is calculated from consecutive achieved weeks; a Weekly `target_count` must be an integer from 1 through 7, and a Daily `target_count` is fixed at 1.
- Q: Which default timezone should the service use when calculating calendar dates and calendar weeks? → A: UTC.
- Q: Which day is the first day of a calendar week? → A: Monday, following the ISO 8601 convention.
- Q: What is the minimum password validation rule for registration and login? → A: At least 8 characters.
- Q: When do JWT credentials obtained after login expire? → A: The Access Token is valid for 15 minutes and is automatically renewed with a Refresh Token valid for 7 days and stored in an `HttpOnly Secure Cookie`; no Token is stored in `localStorage`.
- Q: How should the Refresh Token be handled when a user actively logs out? → A: Revoke the current Refresh Token so subsequent renewal requests fail.
- Q: What should the SameSite attribute of the Refresh Token Cookie be? → A: `SameSite=Strict`.
- Q: What format should all error responses use? → A: `{ "error": { "code": string, "message": string, "details": array } }`.
- Q: How should a request to delete a nonexistent resource or a resource not owned by the user be handled? → A: Always treat it as a not-found resource outcome, without distinguishing nonexistence from lack of authorization.
- Q: How should concurrent check-in requests for the same Habit on the same date be handled? → A: Permit only one successful write; all others return an identifiable conflict outcome and must not create a second record.

## User Scenarios & Testing _(mandatory)_

### User Story 1 - Create an Account and Log In (Priority: P1)

New users can create a personal account with an Email and password, then log in and obtain an identity that can access personal data. When an existing user supplies invalid credentials, the system rejects the login without revealing whether the account exists.

**Why this priority**: Identity is a prerequisite for all personal-data isolation and subsequent functionality.

**Independent Test**: Can be independently verified through registration, successful login, invalid login, and duplicate Email cases, without Todo or Habit data.

**Acceptance Scenarios**:

1. **Given** an Email is not yet registered and a password has at least 8 characters, **When** the user submits registration data, **Then** the system creates the account and returns a successful result without returning the plaintext password.
2. **Given** an Email is already registered, **When** the user registers again with the same Email, **Then** the system rejects the request and returns a consistent validation-error format.
3. **Given** a user is registered, **When** the user logs in with the correct Email and password, **Then** the system returns credentials that can be used for subsequent protected operations.
4. **Given** an Email or password is invalid, **When** the user attempts to log in, **Then** the system rejects the request without indicating which credential was invalid.

---

### User Story 2 - Manage Personal Todos (Priority: P1)

Authenticated users can create, retrieve, update, and delete their own Todos, managing work with a title, description, due date, priority, and completion status.

**Why this priority**: Todo management is the product's most direct everyday value and the core workflow of the minimum viable version.

**Independent Test**: A single authenticated user can independently complete the full Todo lifecycle and verify that required fields and invalid data are rejected.

**Acceptance Scenarios**:

1. **Given** a user is authenticated and provides a valid title, **When** the user creates a Todo, **Then** the system stores the Todo, sets the default incomplete status, and returns its data.
2. **Given** a user owns a Todo, **When** the user retrieves a Todo list or an individual Todo, **Then** the system returns data that meets the query criteria.
3. **Given** a user owns a Todo, **When** the user updates the description, due date, priority, or completion status, **Then** the system updates only the specified fields and returns the updated data.
4. **Given** a user owns a Todo, **When** the user deletes it, **Then** the system deletes the data and subsequent retrievals no longer return that Todo.
5. **Given** a Todo title is empty or a priority is outside the permitted range, **When** the user submits data, **Then** the system rejects the request and identifies the field validation error.
6. **Given** a user retrieves a Todo list, **When** the user omits pagination parameters or provides valid `page` and `page_size`, **Then** the system returns paginated results using the default 20 items per page or the specified page size, with no more than 100 items per page.

---

### User Story 3 - Manage Habits and Check In Daily (Priority: P1)

Authenticated users can create, retrieve, update, and delete their own Habits, set a name, daily or weekly frequency, and target count, and check in daily after completing an activity. The system automatically calculates the current streak and longest streak from consecutive dates.

**Why this priority**: Habit tracking is the core capability that distinguishes the product from ordinary Todos and provides feedback for continued use.

**Independent Test**: A Habit can be created, checked in on consecutive days, checked in more than once, and checked in with interruptions, and the streak results can be examined independently.

**Acceptance Scenarios**:

1. **Given** a user is authenticated and provides a valid name, frequency, and positive integer target count, **When** the user creates a Habit, **Then** the system stores the Habit and returns its configuration.
2. **Given** a user owns a Habit, **When** the user checks in for the first time on a date, **Then** the system records that date and recalculates the current and longest streaks.
3. **Given** a Habit has check-ins on consecutive dates, **When** the user completes a check-in on the next consecutive date, **Then** the current streak increases and the longest streak is no less than the current streak.
4. **Given** a Habit's check-in history contains an interrupted date, **When** the user retrieves the streak, **Then** the current streak is calculated from the most recent consecutive segment and the longest streak retains the historical maximum.
5. **Given** the same Habit has already been checked in on the same date, **When** the user checks in again, **Then** the system does not create a duplicate record and returns an identifiable conflict outcome.
6. **Given** a frequency or target count does not meet the rules, **When** the user creates or updates a Habit, **Then** the system rejects the request and returns a field validation error; a Weekly target count must be an integer from 1 through 7, and a Daily target count must be 1.
7. **Given** a user retrieves a Habit list, **When** the user omits pagination parameters or provides valid `page` and `page_size`, **Then** the system returns paginated results using the default 20 items per page or the specified page size, with no more than 100 items per page.

---

### User Story 4 - View Progress and Protect Personal Data (Priority: P2)

Authenticated users can view Todo completion rates and Habit check-in history by date range and can trust that other users cannot retrieve, modify, or delete their data.

**Why this priority**: Progress review turns data into actionable information; data isolation is necessary for trust and security.

**Independent Test**: Create two users and their respective data, then separately query date ranges and another user's identifiers to verify statistics and access-denial behavior.

**Acceptance Scenarios**:

1. **Given** a user has completed and incomplete Todos in a date range, **When** the user queries the completion rate, **Then** the system returns the numerator, denominator, and completion rate for that range.
2. **Given** a user has Habit check-in records, **When** the user queries a valid date range, **Then** the system returns only the date-identifiable check-in history within that range.
3. **Given** user A owns data, **When** user B attempts to retrieve or operate on that data using any obtainable identifying information, **Then** the system denies access and does not expose the data content.
4. **Given** a date range lacks a start date, has an end date before the start date, or has an invalid date format, **When** the user queries statistics, **Then** the system rejects the request and returns a consistent validation error.

### Edge Cases

- When a user is not authenticated or credentials are invalid or expired, all protected Todo, Habit, check-in, and statistics operations are rejected.
- When an Access Token expires, the system automatically renews it with a valid Refresh Token valid for 7 days; the Refresh Token is stored in an `HttpOnly`, `Secure`, `SameSite=Strict` Cookie, and no Token may be stored in `localStorage`. After a user actively logs out, the system revokes the current Refresh Token so subsequent renewal requests fail.
- When a user attempts to operate on a nonexistent Todo, Habit, or check-in record, or one not owned by the user, the system must not return its content; deleting a nonexistent or unowned resource always returns a not-found resource outcome without distinguishing nonexistence from lack of authorization.
- A Todo due date may be empty; if present, it must be a valid date, and completion-rate calculations include only data within the query range.
- When the denominator of a Todo completion rate is zero, the system returns a clear and consistent zero-data result and does not divide by zero.
- When a Habit name is empty, the frequency is neither daily nor weekly, the Daily target count is not 1, or the Weekly target count is not an integer from 1 through 7, the system rejects the data.
- When a password has fewer than 8 characters, the system rejects the registration data.
- A check-in date must not create duplicate records; concurrent check-in requests for the same Habit on the same date permit only one successful write, while all others return an identifiable conflict outcome and must not create a second record; future-date check-ins are disallowed by default to prevent progress from being recorded in advance.
- When a date range is a single day, spans months, spans years, or is very large, results retain the same format and correctly include boundary dates.
- When a Habit is deleted, its check-in history must not become orphaned data in queries.
- When a Todo or Habit list has no data, the paginated response returns empty `items` and retains consistent pagination metadata rather than returning an error.
- When a user queries a page beyond the last page, the system returns empty `items` with correct total-count and page metadata rather than returning an error.
- When `page_size` is omitted, it defaults to 20; `page_size` may reach but not exceed the maximum per-page limit of 100, and the system rejects values above the limit or below 1 with a consistent validation error.
- When `page` is omitted, it defaults to 1; the system rejects values below 1 or non-integers with a consistent validation error.

## Requirements _(mandatory)_

### Functional Requirements

- **FR-001**: System MUST allow a person to register with a unique Email and a password of at least 8 characters and MUST reject duplicate Email registrations.
- **FR-002**: System MUST authenticate registered users with Email and password, issue an Access Token valid for 15 minutes, and MUST reject invalid credentials without revealing which credential failed.
- **FR-003**: System MUST protect all Todo, Habit, check-in, and progress operations behind authenticated user identity.
- **FR-004**: System MUST allow an authenticated user to create a Todo with a title and optional description, due date, priority, and completion status.
- **FR-005**: System MUST apply a documented default for omitted Todo fields and MUST validate title, date, priority, and status values before storing data.
- **FR-006**: System MUST allow an authenticated user to list and retrieve only their own Todo records.
- **FR-007**: System MUST allow an authenticated user to update and delete only their own Todo records.
- **FR-008**: System MUST allow an authenticated user to create a Habit with a name, Daily or Weekly frequency, and a frequency-valid target count; Daily MUST use target count 1, and Weekly MUST use an integer target count from 1 through 7.
- **FR-009**: System MUST allow an authenticated user to list, retrieve, update, and delete only their own Habit records.
- **FR-010**: System MUST allow an authenticated user to record at most one check-in per Habit per calendar date and MUST reject future-date check-ins.
- **FR-011**: System MUST calculate Daily current streak as the length of the most recent uninterrupted daily check-in sequence and longest streak as the maximum uninterrupted daily sequence; Weekly streak MUST count consecutive ISO 8601 calendar weeks, beginning on Monday, in which at least the Habit target count of check-ins was recorded.
- **FR-012**: System MUST allow users to query Todo completion statistics by an inclusive `due_date` range, exclude Todo records without a `due_date`, and return completed count, total count, and completion rate.
- **FR-013**: System MUST allow users to query their Habit check-in history by an inclusive date range with stable date ordering.
- **FR-014**: System MUST return a consistent JSON response shape, standard HTTP status semantics, and a consistent error format for public operations.
- **FR-015**: System MUST validate all external input before business rules execute, store passwords only as secure hashes, issue a Refresh Token valid for 7 days in an `HttpOnly`, `Secure`, `SameSite=Strict` Cookie for automatic Access Token renewal, never store tokens in `localStorage`, and never expose password hashes or authentication secrets in responses.
- **FR-016**: System MUST provide automatically generated OpenAPI documentation containing public request, response, error, and authentication examples.
- **FR-017**: System MUST exclude team collaboration, sharing, notifications, and push notifications from the first phase.
- **FR-018**: System MUST support pagination for Todo and Habit list queries using a positive `page` number and `page_size`; `page` MUST default to 1, `page_size` MUST default to 20, and `page_size` MUST NOT exceed the maximum of 100 items per page. The response MUST include the returned items and consistent pagination metadata including the current page, page size, total item count, and total page count. An empty list or a page beyond the last page MUST return an empty items collection with valid metadata rather than an error.
- **FR-019**: System MUST allow an authenticated user to actively log out by revoking the current Refresh Token so that subsequent renewal requests using that token fail.
- **FR-020**: System MUST return every error response in the format `{ "error": { "code": string, "message": string, "details": array } }`.
- **FR-021**: System MUST treat a request to delete or update a Todo, Habit, or check-in record that does not exist or is not owned by the authenticated user as a not-found resource outcome, without distinguishing nonexistence from lack of authorization.
- **FR-022**: System MUST allow only one concurrent check-in request for the same Habit and calendar date to create a record; all other concurrent requests for that Habit and date MUST return an identifiable conflict outcome and MUST NOT create an additional record.

### Non-Functional Requirements

- **NFR-001**: Every data-changing operation, including Todo and Habit creation, update, and deletion and Habit check-in, MUST make the successful API response and durable persistence one indivisible outcome. The system MUST NOT return success before the data is successfully persisted; if persistence fails, it MUST return an explicit error and MUST NOT leave a state where the API reports success but the data is not durable.
- **NFR-002**: Every data-changing operation MUST produce one immutable audit record containing the acting user's identifier, operation type (create, update, delete, or check-in), affected entity and identifier, values before and after the change, occurrence time in UTC, and request identifier. Audit records MUST be stored separately from ordinary application logs and MUST NOT be modified or deleted by normal application logic.
- **NFR-003**: Every API request, including non-mutating requests, MUST produce an application log containing the request identifier, the authenticated user's identifier or an anonymous marker, processing time, and result status. Key failures, including validation failures, authentication failures, and database connection failures, MUST have distinguishable severity levels such as WARNING or ERROR. Application logs serve a different purpose from the audit records in NFR-002, MAY use separate storage, and do not require transaction consistency with business data.

### Key Entities _(include if feature involves data)_

- **User**: A person with a unique Email and protected password credential who owns personal records.
- **Todo**: A user-owned task with title, optional description, due date, priority, completion status, and lifecycle timestamps.
- **Habit**: A user-owned recurring activity with name, frequency, positive target count, current streak, and longest streak.
- **Habit Check-in**: A dated record that a user completed a Habit on a calendar date, unique per Habit and date.
- **Progress Summary**: A date-bounded view containing Todo completion counts/rate and Habit check-in history.

## Success Criteria _(mandatory)_

### Measurable Outcomes

- **SC-001**: At least 95% of valid registration and login attempts complete with a clear success or failure result within 2 seconds under normal service conditions.
- **SC-002**: At least 95% of valid Todo and Habit create, read, update, and delete tasks complete within 2 seconds under normal service conditions.
- **SC-003**: 100% of automated acceptance scenarios confirm that a user cannot view or modify another user's data.
- **SC-004**: 100% of streak acceptance scenarios produce the expected current and longest streak values, including uninterrupted, interrupted, duplicate, and empty histories.
- **SC-005**: At least 90% of representative users can complete registration, create a Todo, create a Habit, record a check-in, and view progress without assistance.
- **SC-006**: 100% of public operations have discoverable documentation with valid request, response, error, and authentication examples before release.
- **SC-007**: The first-phase release contains no user-visible collaboration, sharing, notification, or push-notification behavior.
- **SC-008**: 100% of successful Todo, Habit, and Habit check-in mutation responses are preceded by verified durable persistence; 100% of simulated persistence failures return an explicit error and no successful response.
- **SC-009**: 100% of Todo, Habit, and Habit check-in mutation operations produce exactly one immutable audit record containing the required actor, operation, entity, before/after values, UTC timestamp, and request identifier, with no audit record editable or removable through normal application operations.
- **SC-010**: 100% of API request test cases produce an application log containing the request identifier, authenticated user identifier or anonymous marker, processing time, and result status; validation, authentication, and database connection failures are recorded with distinguishable severity levels, and application logging is verified independently from NFR-002 audit records.
- **SC-011**: 100% of Todo and Habit list queries return no more than 100 items in a response, use a default page size of 20 when omitted, return valid pagination metadata for empty lists and pages beyond the last page, and reject page sizes above 100 or below 1 with the standard validation error format.

## Assumptions

- Users have a stable internet connection and use one account per Email.
- Email verification, password reset, social login, and multi-factor authentication are outside the first-phase scope.
- Calendar dates and calendar weeks are interpreted in UTC; calendar weeks begin on Monday according to ISO 8601; timezone customization is outside this feature.
- Todo priority and completion status use a small documented set of values; the exact labels are finalized during planning without changing the user workflows above.
- Weekly Habit frequency records check-ins by calendar date, requires at least `target_count` check-ins per ISO 8601 calendar week beginning on Monday for that week to count toward streak, and limits `target_count` to 1 through 7; Daily Habit `target_count` is fixed at 1. Frequency-specific quota analytics beyond this streak rule are outside this feature.
- Todo completion date-range statistics include both the start date and end date and filter Todo records by `due_date`; Todo records without a `due_date` are excluded.
- The service uses the project's established authentication, validation, persistence, testing, and documentation conventions during planning.
- No existing users or records need to be migrated for the first-phase release.
