# Feature Specification: Todo/Habit Tracker API

**Feature Branch**: `001-todo-habit-tracker`

**Created**: 2026-08-27

**Status**: Draft

**Input**: User description: "開發一個 Todo/Habit Tracker API，讓使用者管理待辦事項與習慣追蹤。包含使用者註冊與登入、Todo 與 Habit 的完整管理、Habit 每日打卡與 streak 計算、依日期範圍查詢完成率與打卡歷史，以及使用者資料隔離；第一階段不包含團隊協作、分享、通知或推播。"

## Clarifications

### Session 2026-08-28

- Q: Todo 完成率的日期範圍應該依哪個日期欄位篩選 Todo？ → A: 依 `due_date` 篩選；沒有到期日的 Todo 不納入日期範圍統計。
- Q: Weekly Habit 的 `target_count` 應如何影響打卡與 streak 計算？ → A: 每週需完成至少 `target_count` 次，streak 以連續達成週數計算；Weekly 的 `target_count` 必須是 1 至 7 的整數，Daily 的 `target_count` 固定為 1。
- Q: 服務計算日曆日與日曆週時，應採用哪個預設時區？ → A: UTC。
- Q: 註冊與登入使用的密碼最低驗證規則應為何？ → A: 至少 8 個字元。
- Q: 登入後取得的 JWT 身分憑證應在多久後過期？ → A: Access Token 效期 15 分鐘，搭配效期 7 天、儲存於 `HttpOnly Secure Cookie` 的 Refresh Token 自動換發；任何 Token 都不儲存於 `localStorage`。

## User Scenarios & Testing _(mandatory)_

### User Story 1 - 建立帳號並登入 (Priority: P1)

新使用者可以使用 Email 與密碼建立個人帳號，之後登入並取得可存取個人資料的身分。既有使用者使用錯誤憑證時，系統拒絕登入且不洩漏帳號是否存在。

**Why this priority**: 身分識別是所有個人資料隔離與後續功能的前提。

**Independent Test**: 可透過註冊、成功登入、錯誤登入與重複 Email 案例獨立驗證，不需要 Todo 或 Habit 資料。

**Acceptance Scenarios**:

1. **Given** Email 尚未註冊且密碼至少 8 個字元，**When** 使用者提交註冊資料，**Then** 系統建立帳號並回傳成功結果，且不回傳明文密碼。
2. **Given** Email 已註冊，**When** 使用者再次以相同 Email 註冊，**Then** 系統拒絕請求並回傳一致的驗證錯誤格式。
3. **Given** 使用者已註冊，**When** 使用者提交正確 Email 與密碼登入，**Then** 系統回傳可用於後續受保護操作的身分憑證。
4. **Given** Email 或密碼錯誤，**When** 使用者嘗試登入，**Then** 系統拒絕請求且不指出是哪一項憑證錯誤。

---

### User Story 2 - 管理個人 Todo (Priority: P1)

已登入使用者可以建立、查詢、更新與刪除自己的 Todo，並以標題、描述、到期日、優先級與完成狀態管理工作。

**Why this priority**: Todo 管理是產品最直接的日常價值，也是最小可用版本的核心工作流程。

**Independent Test**: 可用單一已登入使用者獨立完成 Todo 的完整生命週期，並驗證必要欄位與無效資料被拒絕。

**Acceptance Scenarios**:

1. **Given** 使用者已登入且提供有效標題，**When** 使用者建立 Todo，**Then** 系統儲存 Todo，設定預設未完成狀態，並回傳其資料。
2. **Given** 使用者擁有 Todo，**When** 使用者查詢 Todo 清單或單筆 Todo，**Then** 系統回傳符合其條件的資料。
3. **Given** 使用者擁有 Todo，**When** 使用者更新描述、到期日、優先級或完成狀態，**Then** 系統只更新指定欄位並回傳更新後資料。
4. **Given** 使用者擁有 Todo，**When** 使用者刪除該 Todo，**Then** 系統刪除資料且後續查詢不再回傳該 Todo。
5. **Given** Todo 標題為空或優先級不在允許範圍，**When** 使用者提交資料，**Then** 系統拒絕請求並指出欄位驗證錯誤。
6. **Given** 使用者查詢 Todo 清單，**When** 使用者未提供分頁參數或提供有效的 `page` 與 `page_size`，**Then** 系統依預設每頁 20 筆或指定的每頁筆數回傳分頁結果，且每頁最多 100 筆。

---

### User Story 3 - 管理 Habit 並每日打卡 (Priority: P1)

已登入使用者可以建立、查詢、更新與刪除自己的 Habit，設定名稱、每日或每週頻率與目標次數，並在完成行動後每日打卡。系統依連續日期自動計算目前 streak 與最長 streak。

**Why this priority**: Habit 追蹤是區別於一般 Todo 的核心功能，能提供持續使用的回饋。

**Independent Test**: 可建立一個 Habit、連續多日打卡、重複打卡、間斷打卡，並單獨檢查 streak 結果。

**Acceptance Scenarios**:

1. **Given** 使用者已登入且提供有效名稱、頻率與正整數目標次數，**When** 使用者建立 Habit，**Then** 系統儲存 Habit 並回傳其設定。
2. **Given** 使用者擁有 Habit，**When** 使用者在某日期首次打卡，**Then** 系統記錄該日期且重新計算目前與最長 streak。
3. **Given** Habit 在連續日期已有打卡，**When** 使用者完成下一個連續日期的打卡，**Then** 目前 streak 增加，最長 streak 不低於目前 streak。
4. **Given** Habit 的打卡歷史中有中斷日期，**When** 使用者查詢 streak，**Then** 目前 streak 從最近一次連續區段計算，最長 streak 保留歷史最高值。
5. **Given** 同一 Habit 同一日期已打卡，**When** 使用者再次打卡，**Then** 系統不產生重複紀錄，並回傳可辨識的衝突結果。
6. **Given** 頻率或目標次數不符合規則，**When** 使用者建立或更新 Habit，**Then** 系統拒絕請求並回傳欄位驗證錯誤；Weekly 的目標次數必須是 1 至 7 的整數，Daily 的目標次數必須為 1。
7. **Given** 使用者查詢 Habit 清單，**When** 使用者未提供分頁參數或提供有效的 `page` 與 `page_size`，**Then** 系統依預設每頁 20 筆或指定的每頁筆數回傳分頁結果，且每頁最多 100 筆。

---

### User Story 4 - 檢視進度並保護個人資料 (Priority: P2)

已登入使用者可以依日期範圍檢視 Todo 完成率與 Habit 打卡歷史，並確信其他使用者無法查詢、修改或刪除自己的資料。

**Why this priority**: 進度回顧讓資料轉化為可行動的資訊；隔離資料則是信任與安全的必要條件。

**Independent Test**: 建立兩個使用者及各自資料，分別查詢日期範圍與他人識別碼，驗證統計結果與存取拒絕行為。

**Acceptance Scenarios**:

1. **Given** 使用者在日期範圍內有已完成與未完成 Todo，**When** 使用者查詢完成率，**Then** 系統回傳該範圍內的分子、分母與完成率。
2. **Given** 使用者有 Habit 打卡紀錄，**When** 使用者查詢有效日期範圍，**Then** 系統只回傳該範圍內按日期可辨識的打卡歷史。
3. **Given** 使用者 A 擁有資料，**When** 使用者 B 嘗試以任何可取得的識別資訊查詢或操作該資料，**Then** 系統拒絕存取且不洩漏資料內容。
4. **Given** 日期範圍缺少起始日、結束日早於起始日，或日期格式無效，**When** 使用者查詢統計資料，**Then** 系統拒絕請求並回傳一致的驗證錯誤。

### Edge Cases

- 未登入、憑證無效或憑證過期時，所有受保護的 Todo、Habit、打卡與統計操作都被拒絕。
- Access Token 過期時，系統使用有效的 7 天 Refresh Token 自動換發新的 15 分鐘 Access Token；Refresh Token 儲存於 HttpOnly Secure Cookie，任何 Token 不得儲存於 localStorage。
- 使用者嘗試操作不存在或不屬於自己的 Todo、Habit 或打卡紀錄時，系統不得回傳其內容。
- Todo 到期日可為空；若有值，必須是有效日期，且完成率計算只納入查詢範圍內的資料。
- Todo 完成率的分母為零時，系統回傳明確且一致的零資料結果，不執行除以零。
- Habit 名稱為空、頻率不是每日或每週、Daily 目標次數不是 1、Weekly 目標次數不在 1 至 7 或不是整數時，系統拒絕資料。
- 密碼少於 8 個字元時，系統拒絕註冊資料。
- 打卡日期不可產生重複紀錄；未來日期是否可打卡採用不允許的預設，以避免提前灌入進度。
- 日期範圍為單日、跨月、跨年與很大的範圍時，結果仍使用相同格式並正確包含邊界日期。
- 刪除 Habit 時，其打卡歷史不得在查詢中成為孤兒資料。
- Todo 或 Habit 清單沒有資料時，分頁回應回傳空的 `items`，並保留一致的分頁 metadata，不回傳錯誤。
- 使用者查詢超出最後一頁的頁碼時，系統回傳空的 `items` 與正確的總筆數及頁碼 metadata，不回傳錯誤。
- `page_size` 未提供時預設為 20；`page_size` 可達到但不可超過最大單頁上限 100，超過上限或小於 1 時系統拒絕請求並回傳一致的驗證錯誤。
- `page` 未提供時預設為 1；小於 1 或不是整數時系統拒絕請求並回傳一致的驗證錯誤。

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
- **FR-011**: System MUST calculate Daily current streak as the length of the most recent uninterrupted daily check-in sequence and longest streak as the maximum uninterrupted daily sequence; Weekly streak MUST count consecutive calendar weeks in which at least the Habit target count of check-ins was recorded.
- **FR-012**: System MUST allow users to query Todo completion statistics by an inclusive `due_date` range, exclude Todo records without a `due_date`, and return completed count, total count, and completion rate.
- **FR-013**: System MUST allow users to query their Habit check-in history by an inclusive date range with stable date ordering.
- **FR-014**: System MUST return a consistent JSON response shape, standard HTTP status semantics, and a consistent error format for public operations.
- **FR-015**: System MUST validate all external input before business rules execute, store passwords only as secure hashes, issue a Refresh Token valid for 7 days in an HttpOnly Secure Cookie for automatic Access Token renewal, never store tokens in localStorage, and never expose password hashes or authentication secrets in responses.
- **FR-016**: System MUST provide automatically generated OpenAPI documentation containing public request, response, error, and authentication examples.
- **FR-017**: System MUST exclude team collaboration, sharing, notifications, and push notifications from the first phase.
- **FR-018**: System MUST support pagination for Todo and Habit list queries using a positive `page` number and `page_size`; `page` MUST default to 1, `page_size` MUST default to 20, and `page_size` MUST NOT exceed the maximum of 100 items per page. The response MUST include the returned items and consistent pagination metadata including the current page, page size, total item count, and total page count. An empty list or a page beyond the last page MUST return an empty items collection with valid metadata rather than an error.

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
- Calendar dates and calendar weeks are interpreted in UTC; timezone customization is outside this feature.
- Todo priority and completion status use a small documented set of values; the exact labels are finalized during planning without changing the user workflows above.
- Weekly Habit frequency records check-ins by calendar date, requires at least `target_count` check-ins per calendar week for that week to count toward streak, and limits `target_count` to 1 through 7; Daily Habit `target_count` is fixed at 1. Frequency-specific quota analytics beyond this streak rule are outside this feature.
- Todo completion date-range statistics include both the start date and end date and filter Todo records by `due_date`; Todo records without a `due_date` are excluded.
- The service uses the project's established authentication, validation, persistence, testing, and documentation conventions during planning.
- No existing users or records need to be migrated for the first-phase release.
