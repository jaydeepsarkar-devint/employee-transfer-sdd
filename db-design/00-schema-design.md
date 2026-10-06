# DB Schema Design — Internal Transfer

**Database:** SQLite (TD-10), accessed from the Express backend through a repository layer.
**Source:** [Spec v0.2](../specs/internal-transfer/internal-transfer.spec.md) · relationships in [01-relationships-erd.md](01-relationships-erd.md)
**Status:** Draft v0.1 — 2026-10-06

## 1. Conventions

| Topic | Rule |
|---|---|
| Naming | `snake_case`, singular table names |
| IDs | Portal tables: `TEXT` UUID v4. Simulated HR-system tables: `TEXT` business codes (e.g. `E1001`, `DEP-OPS`) |
| Dates | `TEXT` ISO-8601. Calendar dates as `YYYY-MM-DD` (effective date, holidays). Timestamps as UTC `YYYY-MM-DDTHH:MM:SS.sssZ` |
| Booleans | `INTEGER` 0/1 with `CHECK (x IN (0,1))` |
| Enumerations | `TEXT` with a `CHECK (... IN (...))` list matching the spec |
| Foreign keys | Always declared; `PRAGMA foreign_keys = ON` is set on every connection (SQLite has it off by default). No cascading deletes: requests and their history are never deleted (BR-25) |
| JSON | `TEXT` holding JSON, only for small snapshot/payload blobs that are never queried by field |
| Configurable limits | Values the client may change (30/180 days, 500 characters, 3/5 working days) are **not** hard-coded as DB constraints; the API enforces them from configuration (NFR-05) |
| Sensitive text | `reason` and approval `comment` exist only in their own columns; they are never copied into audit, outbox or notification rows (NFR-02, BR-22) |

## 2. Table overview

| # | Table | Owner | Purpose | Spec |
|---|---|---|---|---|
| 1 | `transfer_request` | Portal | One row per transfer request | FR-03 – FR-05, BR-08 |
| 2 | `approval_step` | Portal | One row per approval assignment (current manager, receiving manager, HR) | FR-06, FR-07, BR-03, BR-12, BR-14 |
| 3 | `downstream_task` | Portal | Payroll / IT / Facilities task per request | FR-09, FR-10, BR-15 |
| 4 | `outbox_message` | Portal | Calls to external systems, retried until they succeed (tasks, HR record update) | NFR-03, FR-11 |
| 5 | `notification` | Portal | Email and in-portal messages, also the email retry queue | FR-17, BR-24 |
| 6 | `audit_event` | Portal | Append-only history of every status change | FR-20, BR-25 |
| 7 | `team_member` | Portal | Which employees belong to HR, Payroll, IT, Facilities | FR-19, BR-21 |
| 8 | `public_holiday` | Portal | Holiday calendar for working-day counting | BR-11, A-36 |
| 9 | `hris_legal_entity` | Simulated HR system | Legal entities | BR-18 |
| 10 | `hris_location` | Simulated HR system | Locations with country | BR-05, BR-18 |
| 11 | `hris_department` | Simulated HR system | Departments, their legal entity and head | BR-03, BR-05, BR-18 |
| 12 | `hris_role` | Simulated HR system | Roles per department | BR-05 |
| 13 | `hris_employee` | Simulated HR system | Employee master: employment type, probation, notice, current placement, manager | BR-01, BR-06 |

**Why `hris_*` tables are here:** the real HR system is out of scope (OOS-07). Its data is simulated in the same SQLite file, behind the HR-system adapter (TD-04). The portal only **reads** these tables, except the simulated HR record update on the effective date (FR-11) and the test controls that simulate manager change and resignation (FR-18). When the real HR system is connected, these tables are dropped and the adapter is swapped; no portal table changes.

---

## 3. Portal tables

### 3.1 `transfer_request`

| Column | Type | Null | Rules | Purpose / spec |
|---|---|---|---|---|
| `id` | TEXT | No | PK, UUID | |
| `request_no` | TEXT | No | UNIQUE, format `TR-000001` | Human-readable number shown to users |
| `employee_id` | TEXT | No | FK → `hris_employee.id` | The requester (BR-19) |
| `idempotency_key` | TEXT | No | UNIQUE with `employee_id` | Double-submit returns the same request (FR-04, AC-28) |
| `current_department_id` | TEXT | No | FK → `hris_department.id` | Snapshot at submission (BR-06) |
| `current_location_id` | TEXT | No | FK → `hris_location.id` | Snapshot; Facilities task rule (BR-15) |
| `current_role_id` | TEXT | No | FK → `hris_role.id` | Snapshot |
| `current_manager_id` | TEXT | No | FK → `hris_employee.id` | First approver at submission |
| `target_department_id` | TEXT | No | FK → `hris_department.id` | BR-05, BR-18 |
| `target_location_id` | TEXT | No | FK → `hris_location.id` | BR-05, BR-18 |
| `target_role_id` | TEXT | No | FK → `hris_role.id` | Must belong to target department — checked by API (BR-05) |
| `receiving_manager_id` | TEXT | No | FK → `hris_employee.id` | Head of target department at submission; updated on head change (A-32, A-44) |
| `effective_date` | TEXT | No | `YYYY-MM-DD` | Window checked by API from config (BR-04); HR may change it (BR-20) |
| `reason` | TEXT | Yes | Length checked by API from config | Optional (BR-07); never copied elsewhere |
| `status` | TEXT | No | CHECK: `PENDING_CURRENT_MANAGER`, `PENDING_RECEIVING_MANAGER`, `PENDING_HR`, `IN_PROGRESS`, `COMPLETED`, `REJECTED`, `WITHDRAWN`, `CANCELLED` | Spec §5.1 |
| `eligibility_snapshot` | TEXT | No | JSON | Eligibility result at submission, e.g. `{"permanent":true,"onProbation":false,"servingNotice":false,"checkedAt":"…"}` (BR-02, AC-39) |
| `hr_update_status` | TEXT | No | CHECK: `NOT_SCHEDULED`, `SCHEDULED`, `SENDING`, `SUCCEEDED`, `FAILED`, `CANCELLED`; default `NOT_SCHEDULED` | HR record update progress (BR-16, AC-50 – AC-52) |
| `hr_updated_at` | TEXT | Yes | | When the HR record update succeeded; also blocks HR cancel (BR-28) |
| `version` | INTEGER | No | Default 1; +1 on every update | Optimistic locking: an update with a stale version changes 0 rows → conflict (NFR-04, AC-42) |
| `submitted_at` | TEXT | No | | |
| `closed_at` | TEXT | Yes | Set when status becomes terminal | |
| `updated_at` | TEXT | No | | |

**Indexes and constraints**

| Name | Definition | Why |
|---|---|---|
| `ux_request_one_active` | UNIQUE (`employee_id`) WHERE `status IN ('PENDING_CURRENT_MANAGER','PENDING_RECEIVING_MANAGER','PENDING_HR','IN_PROGRESS')` | One active request per employee, enforced even when two submissions race (BR-08, AC-25) |
| `ux_request_idempotency` | UNIQUE (`employee_id`, `idempotency_key`) | FR-04 |
| `ck_request_changes_something` | CHECK at least one of department, location, role differs from current | Last line of defence for BR-06 (API checks it first, AC-21) |
| `ix_request_hr_update_due` | (`status`, `hr_update_status`, `effective_date`) | Daily job finds requests whose effective date is today (AC-51) |

### 3.2 `approval_step`

One row per **assignment**. A step is created when the request reaches it. Re-routing (BR-14) closes the old row as `REASSIGNED` and opens a new one, so the history shows who held it and when.

| Column | Type | Null | Rules | Purpose / spec |
|---|---|---|---|---|
| `id` | TEXT | No | PK, UUID | |
| `request_id` | TEXT | No | FK → `transfer_request.id` | |
| `step_type` | TEXT | No | CHECK: `CURRENT_MANAGER`, `RECEIVING_MANAGER`, `HR` | BR-03 |
| `sequence` | INTEGER | No | CHECK 1–3, matching `step_type` | Order of the chain |
| `assignee_employee_id` | TEXT | Yes | FK → `hris_employee.id` | Named approver for manager steps |
| `assignee_team` | TEXT | Yes | CHECK: `HR` | HR step is held by the HR team, not a person (BR-23 shows "HR") |
| `status` | TEXT | No | CHECK: `PENDING`, `APPROVED`, `REJECTED`, `SKIPPED`, `REASSIGNED`, `CANCELLED` | `SKIPPED` = same person for both manager steps (A-33); `CANCELLED` = request withdrawn |
| `assigned_at` | TEXT | No | | Start of the wait for reminders (BR-11) |
| `decided_by_employee_id` | TEXT | Yes | FK → `hris_employee.id` | Who approved/rejected (HR member or delegate) |
| `decided_at` | TEXT | Yes | | |
| `comment` | TEXT | Yes | CHECK: required (non-blank) when `status = 'REJECTED'` | BR-12, AC-36; shown to employee on rejection (BR-23) |
| `reminder_sent_at` | TEXT | Yes | | Prevents a second reminder (AC-68) |
| `escalated_at` | TEXT | Yes | | Prevents a second escalation (AC-69) |

| Name | Definition | Why |
|---|---|---|
| `ck_step_assignee` | CHECK exactly one of `assignee_employee_id`, `assignee_team` is set; `assignee_team` only for `HR` | Clear owner per step |
| `ux_step_one_pending` | UNIQUE (`request_id`) WHERE `status = 'PENDING'` | A request is pending with one step at a time (BR-03) |
| `ix_step_worklist` | (`assignee_employee_id`, `status`) and (`assignee_team`, `status`) | Approver work lists (FR-06) |

### 3.3 `downstream_task`

| Column | Type | Null | Rules | Purpose / spec |
|---|---|---|---|---|
| `id` | TEXT | No | PK, UUID | |
| `request_id` | TEXT | No | FK → `transfer_request.id` | |
| `team` | TEXT | No | CHECK: `PAYROLL`, `IT`, `FACILITIES` | BR-15 |
| `status` | TEXT | No | CHECK: `NOT_SENT`, `OPEN`, `DONE`, `CANCELLED` | Spec FR-09 |
| `external_ref` | TEXT | Yes | | Ticket ID in the team's system |
| `created_at` | TEXT | No | | = HR approval time (BR-16) |
| `opened_at` | TEXT | Yes | | Delivered to team system; start of the wait for reminders (BR-11) |
| `completed_at` | TEXT | Yes | | |
| `cancelled_at` | TEXT | Yes | | BR-28 |
| `reminder_sent_at` | TEXT | Yes | | |
| `escalated_at` | TEXT | Yes | | Escalation to the team contact (A-35) |

| Name | Definition | Why |
|---|---|---|
| `ux_task_team` | UNIQUE (`request_id`, `team`) | At most one task per team per request (AC-44, AC-45) |

"Delayed" in the status view (AC-48) is not stored here: it is derived from the task's `outbox_message` having status `FAILED`.

### 3.4 `outbox_message`

Every call to an external system is written here **in the same transaction** as the decision that causes it, then sent by a background worker with retry. A failing system therefore never loses or rolls back an approval (NFR-03).

| Column | Type | Null | Rules | Purpose / spec |
|---|---|---|---|---|
| `id` | TEXT | No | PK, UUID | |
| `request_id` | TEXT | No | FK → `transfer_request.id` | |
| `task_id` | TEXT | Yes | FK → `downstream_task.id` | Set for task messages |
| `kind` | TEXT | No | CHECK: `TASK_CREATE`, `TASK_CANCEL`, `TASK_DATE_CHANGE`, `HR_RECORD_UPDATE` | FR-09, FR-11, FR-15 |
| `payload` | TEXT | No | JSON | Only BR-22 fields for tasks; never the reason |
| `status` | TEXT | No | CHECK: `PENDING`, `DONE`, `FAILED` | `FAILED` = retries exhausted → support / HR alerted (AC-48, AC-52) |
| `attempts` | INTEGER | No | Default 0 | Compared with configured max |
| `next_attempt_at` | TEXT | No | | Increasing delay between attempts |
| `last_error` | TEXT | Yes | Error code and message only; no personal data | |
| `created_at` | TEXT | No | | |
| `processed_at` | TEXT | Yes | | |

| Name | Definition | Why |
|---|---|---|
| `ck_outbox_task` | CHECK `task_id` is set when `kind` starts with `TASK_`, and null for `HR_RECORD_UPDATE` | |
| `ix_outbox_due` | (`status`, `next_attempt_at`) | Worker picks due messages |

The `HR_RECORD_UPDATE` message is created by the daily job **on** the effective date, not at approval; an HR date change (BR-20) therefore needs no outbox change.

### 3.5 `notification`

| Column | Type | Null | Rules | Purpose / spec |
|---|---|---|---|---|
| `id` | TEXT | No | PK, UUID | |
| `request_id` | TEXT | No | FK → `transfer_request.id` | |
| `recipient_employee_id` | TEXT | Yes | FK → `hris_employee.id` | Person recipient |
| `recipient_team` | TEXT | Yes | CHECK: `HR`, `PAYROLL`, `IT`, `FACILITIES`, `SUPPORT` | Team recipient; address from config (escalation contact, alerts) |
| `event` | TEXT | No | CHECK: `SUBMITTED`, `STEP_APPROVED`, `REJECTED`, `HR_APPROVED`, `WITHDRAWN`, `CANCELLED`, `DATE_CHANGED`, `COMPLETED`, `REMINDER`, `ESCALATION`, `REASSIGNED`, `HR_ALERT` | FR-16, FR-17, FR-18 |
| `channel` | TEXT | No | CHECK: `EMAIL`, `IN_PORTAL` | BR-24 |
| `status` | TEXT | No | CHECK: `QUEUED`, `SENT`, `FAILED` | Email retry (AC-77); in-portal rows are `SENT` on insert |
| `attempts` | INTEGER | No | Default 0 | |
| `next_attempt_at` | TEXT | Yes | | |
| `read_at` | TEXT | Yes | | In-portal read marker |
| `created_at` | TEXT | No | | |
| `sent_at` | TEXT | Yes | | |

| Name | Definition | Why |
|---|---|---|
| `ck_notification_recipient` | CHECK exactly one of `recipient_employee_id`, `recipient_team` is set | |
| `ix_notification_inbox` | (`recipient_employee_id`, `channel`, `read_at`) | In-portal inbox |

Message text is built from a template at send time and is not stored; emails say "see the portal" instead of including the reason or rejection comment (NFR-02).

### 3.6 `audit_event`

Append-only. Written in the same transaction as the change it records (NFR-06).

| Column | Type | Null | Rules | Purpose / spec |
|---|---|---|---|---|
| `id` | INTEGER | No | PK AUTOINCREMENT | Gives a strict order |
| `request_id` | TEXT | No | FK → `transfer_request.id` | |
| `task_id` | TEXT | Yes | FK → `downstream_task.id` | Task status changes |
| `actor_type` | TEXT | No | CHECK: `EMPLOYEE`, `SYSTEM`, `EXTERNAL_SYSTEM` | Who made the change (BR-25) |
| `actor_employee_id` | TEXT | Yes | FK → `hris_employee.id`; required when `actor_type = 'EMPLOYEE'` | |
| `action` | TEXT | No | e.g. `SUBMIT`, `APPROVE`, `REJECT`, `SKIP`, `REASSIGN`, `WITHDRAW`, `AUTO_WITHDRAW`, `CANCEL`, `CHANGE_DATE`, `TASK_STATUS`, `HR_RECORD_UPDATE`, `COMPLETE` | |
| `from_status` | TEXT | Yes | | Null on `SUBMIT` |
| `to_status` | TEXT | Yes | | |
| `details` | TEXT | Yes | JSON; never reason or comment text | e.g. old/new effective date |
| `occurred_at` | TEXT | No | | |

| Name | Definition | Why |
|---|---|---|
| `trg_audit_no_update` | BEFORE UPDATE → `RAISE(ABORT)` | Audit cannot be changed (BR-25) |
| `trg_audit_no_delete` | BEFORE DELETE → `RAISE(ABORT)` | Audit cannot be deleted (BR-25) |
| `ix_audit_request` | (`request_id`, `id`) | Timeline and audit reads |

### 3.7 `team_member`

| Column | Type | Null | Rules | Purpose / spec |
|---|---|---|---|---|
| `employee_id` | TEXT | No | FK → `hris_employee.id`; PK part | |
| `team` | TEXT | No | CHECK: `HR`, `PAYROLL`, `IT`, `FACILITIES`; PK part | Grants HR actions and task visibility (BR-21) |

### 3.8 `public_holiday`

| Column | Type | Null | Rules | Purpose / spec |
|---|---|---|---|---|
| `holiday_date` | TEXT | No | PK, `YYYY-MM-DD` | Excluded from working days (A-36, AC-70) |
| `name` | TEXT | No | | |

---

## 4. Simulated HR-system tables (`hris_*`)

### 4.1 `hris_legal_entity`

| Column | Type | Null | Rules |
|---|---|---|---|
| `id` | TEXT | No | PK |
| `name` | TEXT | No | |

### 4.2 `hris_location`

| Column | Type | Null | Rules | Spec |
|---|---|---|---|---|
| `id` | TEXT | No | PK | |
| `name` | TEXT | No | | |
| `country_code` | TEXT | No | ISO 3166-1 alpha-2 | Same-country rule (BR-18) |
| `is_active` | INTEGER | No | 0/1 | BR-05 |

### 4.3 `hris_department`

| Column | Type | Null | Rules | Spec |
|---|---|---|---|---|
| `id` | TEXT | No | PK | |
| `name` | TEXT | No | | |
| `legal_entity_id` | TEXT | No | FK → `hris_legal_entity.id` | BR-18 |
| `head_employee_id` | TEXT | Yes | FK → `hris_employee.id` | Receiving manager (A-32) |
| `is_active` | INTEGER | No | 0/1 | BR-05 |

### 4.4 `hris_role`

| Column | Type | Null | Rules | Spec |
|---|---|---|---|---|
| `id` | TEXT | No | PK | |
| `name` | TEXT | No | | |
| `department_id` | TEXT | No | FK → `hris_department.id` | Role belongs to department (BR-05) |
| `is_active` | INTEGER | No | 0/1 | |

### 4.5 `hris_employee`

| Column | Type | Null | Rules | Spec |
|---|---|---|---|---|
| `id` | TEXT | No | PK, e.g. `E1001` | |
| `full_name` | TEXT | No | | |
| `email` | TEXT | No | UNIQUE | Notifications |
| `employment_type` | TEXT | No | CHECK: `PERMANENT`, `CONTRACT`, `INTERN` | BR-01 |
| `on_probation` | INTEGER | No | 0/1 | BR-01 |
| `serving_notice` | INTEGER | No | 0/1 | BR-01 |
| `employment_status` | TEXT | No | CHECK: `ACTIVE`, `RESIGNED` | Resignation event (BR-14) |
| `department_id` | TEXT | No | FK → `hris_department.id` | Current placement; legal entity via department |
| `location_id` | TEXT | No | FK → `hris_location.id` | Country via location |
| `role_id` | TEXT | No | FK → `hris_role.id` | |
| `manager_id` | TEXT | Yes | FK → `hris_employee.id` | Current manager; manager-change event (BR-14) |

**Seeding order:** departments and employees reference each other (`head_employee_id` ↔ `department_id`). Seed legal entities and locations → departments with `head_employee_id = NULL` → roles → employees → set department heads.

---

## 5. How key rules are enforced

| Rule | API (first check, clear message) | Database (last line of defence) |
|---|---|---|
| BR-04 date window, BR-07 reason length | Yes, from config | No (configurable values) |
| BR-05 active values; role in department | Yes | FKs ensure values exist |
| BR-06 something must change | Yes (AC-21) | `ck_request_changes_something` |
| BR-08 one active request | Yes (AC-25) | `ux_request_one_active` |
| FR-04 idempotent submit | Yes | `ux_request_idempotency` |
| BR-03 one pending step at a time | Yes (state machine) | `ux_step_one_pending` |
| BR-12 rejection comment | Yes (AC-36) | `approval_step` comment CHECK |
| BR-25 audit cannot change | — | `trg_audit_no_update`, `trg_audit_no_delete` |
| NFR-04 concurrent decisions | Version check in `UPDATE … WHERE version = ?` | `version` column |

---

## Appendix — DDL (SQLite)

```sql
PRAGMA foreign_keys = ON;

-- Simulated HR system -------------------------------------------------------

CREATE TABLE hris_legal_entity (
  id   TEXT PRIMARY KEY,
  name TEXT NOT NULL
);

CREATE TABLE hris_location (
  id           TEXT PRIMARY KEY,
  name         TEXT NOT NULL,
  country_code TEXT NOT NULL,
  is_active    INTEGER NOT NULL DEFAULT 1 CHECK (is_active IN (0,1))
);

CREATE TABLE hris_department (
  id               TEXT PRIMARY KEY,
  name             TEXT NOT NULL,
  legal_entity_id  TEXT NOT NULL REFERENCES hris_legal_entity(id),
  head_employee_id TEXT REFERENCES hris_employee(id),
  is_active        INTEGER NOT NULL DEFAULT 1 CHECK (is_active IN (0,1))
);

CREATE TABLE hris_role (
  id            TEXT PRIMARY KEY,
  name          TEXT NOT NULL,
  department_id TEXT NOT NULL REFERENCES hris_department(id),
  is_active     INTEGER NOT NULL DEFAULT 1 CHECK (is_active IN (0,1))
);

CREATE TABLE hris_employee (
  id                TEXT PRIMARY KEY,
  full_name         TEXT NOT NULL,
  email             TEXT NOT NULL UNIQUE,
  employment_type   TEXT NOT NULL CHECK (employment_type IN ('PERMANENT','CONTRACT','INTERN')),
  on_probation      INTEGER NOT NULL DEFAULT 0 CHECK (on_probation IN (0,1)),
  serving_notice    INTEGER NOT NULL DEFAULT 0 CHECK (serving_notice IN (0,1)),
  employment_status TEXT NOT NULL DEFAULT 'ACTIVE' CHECK (employment_status IN ('ACTIVE','RESIGNED')),
  department_id     TEXT NOT NULL REFERENCES hris_department(id),
  location_id       TEXT NOT NULL REFERENCES hris_location(id),
  role_id           TEXT NOT NULL REFERENCES hris_role(id),
  manager_id        TEXT REFERENCES hris_employee(id)
);

-- Portal --------------------------------------------------------------------

CREATE TABLE team_member (
  employee_id TEXT NOT NULL REFERENCES hris_employee(id),
  team        TEXT NOT NULL CHECK (team IN ('HR','PAYROLL','IT','FACILITIES')),
  PRIMARY KEY (employee_id, team)
);

CREATE TABLE public_holiday (
  holiday_date TEXT PRIMARY KEY,
  name         TEXT NOT NULL
);

CREATE TABLE transfer_request (
  id                    TEXT PRIMARY KEY,
  request_no            TEXT NOT NULL UNIQUE,
  employee_id           TEXT NOT NULL REFERENCES hris_employee(id),
  idempotency_key       TEXT NOT NULL,
  current_department_id TEXT NOT NULL REFERENCES hris_department(id),
  current_location_id   TEXT NOT NULL REFERENCES hris_location(id),
  current_role_id       TEXT NOT NULL REFERENCES hris_role(id),
  current_manager_id    TEXT NOT NULL REFERENCES hris_employee(id),
  target_department_id  TEXT NOT NULL REFERENCES hris_department(id),
  target_location_id    TEXT NOT NULL REFERENCES hris_location(id),
  target_role_id        TEXT NOT NULL REFERENCES hris_role(id),
  receiving_manager_id  TEXT NOT NULL REFERENCES hris_employee(id),
  effective_date        TEXT NOT NULL,
  reason                TEXT,
  status                TEXT NOT NULL CHECK (status IN (
                          'PENDING_CURRENT_MANAGER','PENDING_RECEIVING_MANAGER','PENDING_HR',
                          'IN_PROGRESS','COMPLETED','REJECTED','WITHDRAWN','CANCELLED')),
  eligibility_snapshot  TEXT NOT NULL,
  hr_update_status      TEXT NOT NULL DEFAULT 'NOT_SCHEDULED' CHECK (hr_update_status IN (
                          'NOT_SCHEDULED','SCHEDULED','SENDING','SUCCEEDED','FAILED','CANCELLED')),
  hr_updated_at         TEXT,
  version               INTEGER NOT NULL DEFAULT 1,
  submitted_at          TEXT NOT NULL,
  closed_at             TEXT,
  updated_at            TEXT NOT NULL,
  CONSTRAINT ck_request_changes_something CHECK (
    target_department_id <> current_department_id
    OR target_location_id <> current_location_id
    OR target_role_id <> current_role_id)
);

CREATE UNIQUE INDEX ux_request_one_active ON transfer_request(employee_id)
  WHERE status IN ('PENDING_CURRENT_MANAGER','PENDING_RECEIVING_MANAGER','PENDING_HR','IN_PROGRESS');
CREATE UNIQUE INDEX ux_request_idempotency ON transfer_request(employee_id, idempotency_key);
CREATE INDEX ix_request_hr_update_due ON transfer_request(status, hr_update_status, effective_date);

CREATE TABLE approval_step (
  id                     TEXT PRIMARY KEY,
  request_id             TEXT NOT NULL REFERENCES transfer_request(id),
  step_type              TEXT NOT NULL CHECK (step_type IN ('CURRENT_MANAGER','RECEIVING_MANAGER','HR')),
  sequence               INTEGER NOT NULL CHECK (
                           (step_type = 'CURRENT_MANAGER'   AND sequence = 1) OR
                           (step_type = 'RECEIVING_MANAGER' AND sequence = 2) OR
                           (step_type = 'HR'                AND sequence = 3)),
  assignee_employee_id   TEXT REFERENCES hris_employee(id),
  assignee_team          TEXT CHECK (assignee_team IN ('HR')),
  status                 TEXT NOT NULL CHECK (status IN ('PENDING','APPROVED','REJECTED','SKIPPED','REASSIGNED','CANCELLED')),
  assigned_at            TEXT NOT NULL,
  decided_by_employee_id TEXT REFERENCES hris_employee(id),
  decided_at             TEXT,
  comment                TEXT,
  reminder_sent_at       TEXT,
  escalated_at           TEXT,
  CONSTRAINT ck_step_assignee CHECK (
    (step_type = 'HR' AND assignee_team = 'HR' AND assignee_employee_id IS NULL) OR
    (step_type <> 'HR' AND assignee_employee_id IS NOT NULL AND assignee_team IS NULL)),
  CONSTRAINT ck_step_reject_comment CHECK (
    status <> 'REJECTED' OR (comment IS NOT NULL AND length(trim(comment)) > 0))
);

CREATE UNIQUE INDEX ux_step_one_pending ON approval_step(request_id) WHERE status = 'PENDING';
CREATE INDEX ix_step_worklist_person ON approval_step(assignee_employee_id, status);
CREATE INDEX ix_step_worklist_team   ON approval_step(assignee_team, status);

CREATE TABLE downstream_task (
  id               TEXT PRIMARY KEY,
  request_id       TEXT NOT NULL REFERENCES transfer_request(id),
  team             TEXT NOT NULL CHECK (team IN ('PAYROLL','IT','FACILITIES')),
  status           TEXT NOT NULL CHECK (status IN ('NOT_SENT','OPEN','DONE','CANCELLED')),
  external_ref     TEXT,
  created_at       TEXT NOT NULL,
  opened_at        TEXT,
  completed_at     TEXT,
  cancelled_at     TEXT,
  reminder_sent_at TEXT,
  escalated_at     TEXT,
  CONSTRAINT ux_task_team UNIQUE (request_id, team)
);

CREATE TABLE outbox_message (
  id              TEXT PRIMARY KEY,
  request_id      TEXT NOT NULL REFERENCES transfer_request(id),
  task_id         TEXT REFERENCES downstream_task(id),
  kind            TEXT NOT NULL CHECK (kind IN ('TASK_CREATE','TASK_CANCEL','TASK_DATE_CHANGE','HR_RECORD_UPDATE')),
  payload         TEXT NOT NULL,
  status          TEXT NOT NULL DEFAULT 'PENDING' CHECK (status IN ('PENDING','DONE','FAILED')),
  attempts        INTEGER NOT NULL DEFAULT 0,
  next_attempt_at TEXT NOT NULL,
  last_error      TEXT,
  created_at      TEXT NOT NULL,
  processed_at    TEXT,
  CONSTRAINT ck_outbox_task CHECK (
    (kind = 'HR_RECORD_UPDATE' AND task_id IS NULL) OR
    (kind <> 'HR_RECORD_UPDATE' AND task_id IS NOT NULL))
);

CREATE INDEX ix_outbox_due ON outbox_message(status, next_attempt_at);

CREATE TABLE notification (
  id                    TEXT PRIMARY KEY,
  request_id            TEXT NOT NULL REFERENCES transfer_request(id),
  recipient_employee_id TEXT REFERENCES hris_employee(id),
  recipient_team        TEXT CHECK (recipient_team IN ('HR','PAYROLL','IT','FACILITIES','SUPPORT')),
  event                 TEXT NOT NULL CHECK (event IN (
                          'SUBMITTED','STEP_APPROVED','REJECTED','HR_APPROVED','WITHDRAWN','CANCELLED',
                          'DATE_CHANGED','COMPLETED','REMINDER','ESCALATION','REASSIGNED','HR_ALERT')),
  channel               TEXT NOT NULL CHECK (channel IN ('EMAIL','IN_PORTAL')),
  status                TEXT NOT NULL DEFAULT 'QUEUED' CHECK (status IN ('QUEUED','SENT','FAILED')),
  attempts              INTEGER NOT NULL DEFAULT 0,
  next_attempt_at       TEXT,
  read_at               TEXT,
  created_at            TEXT NOT NULL,
  sent_at               TEXT,
  CONSTRAINT ck_notification_recipient CHECK (
    (recipient_employee_id IS NOT NULL) + (recipient_team IS NOT NULL) = 1)
);

CREATE INDEX ix_notification_inbox ON notification(recipient_employee_id, channel, read_at);
CREATE INDEX ix_notification_due   ON notification(status, next_attempt_at);

CREATE TABLE audit_event (
  id                INTEGER PRIMARY KEY AUTOINCREMENT,
  request_id        TEXT NOT NULL REFERENCES transfer_request(id),
  task_id           TEXT REFERENCES downstream_task(id),
  actor_type        TEXT NOT NULL CHECK (actor_type IN ('EMPLOYEE','SYSTEM','EXTERNAL_SYSTEM')),
  actor_employee_id TEXT REFERENCES hris_employee(id),
  action            TEXT NOT NULL,
  from_status       TEXT,
  to_status         TEXT,
  details           TEXT,
  occurred_at       TEXT NOT NULL,
  CONSTRAINT ck_audit_actor CHECK (actor_type <> 'EMPLOYEE' OR actor_employee_id IS NOT NULL)
);

CREATE INDEX ix_audit_request ON audit_event(request_id, id);

CREATE TRIGGER trg_audit_no_update BEFORE UPDATE ON audit_event
BEGIN SELECT RAISE(ABORT, 'audit_event is append-only'); END;

CREATE TRIGGER trg_audit_no_delete BEFORE DELETE ON audit_event
BEGIN SELECT RAISE(ABORT, 'audit_event is append-only'); END;
```
