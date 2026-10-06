# Deliverable 1 — Requirement / Discovery Analysis

**Feature:** Employee Internal Transfer Digital Journey (One-Point Employee Portal)
**Input:** [BRD](../../.ai-context/BRD.md)
**Status:** Draft v0.3 — all 31 questions answered by the client (2026-10-06); ready for the spec
**Date:** 2026-10-01

---

## 1. Business objective

Replace today's fragmented, multi-team transfer process with **one digital journey** in the One-Point Employee Portal, where:

1. An employee raises an internal transfer request (department/BU, location, role, effective date, optional reason).
2. The portal routes it through the required approvals and validations.
3. The portal **orchestrates** the downstream work (org update, payroll, IT access, facilities).
4. The employee — and every stakeholder — sees **one view of status and of who is holding the next action**.

**Measurable outcome (confirmed, Q-30):** every transfer is raised and tracked in the portal; the employee never has to contact HR/IT/Facilities to learn the status.

## 2. Primary users

| ID | User | Role in the journey | What they need |
|---|---|---|---|
| U-01 | **Employee (requester)** | Starts the request, tracks progress, may withdraw | Simple form, valid choices only, clear status and "who is it waiting on" |
| U-02 | **Current (line) manager** | Approves/rejects release of the employee | Request details, ability to approve/reject with comment |
| U-03 | **Receiving manager** (head of target dept/position) | Accepts the employee into the new team (confirmed, Q-02) | Request details, approve/reject |
| U-04 | **HR (HR Business Partner / HR Ops)** | Validates eligibility, finalises the transfer, updates organisational record | Eligibility facts, approve/reject, trigger org update |
| U-05 | **Payroll team** | Updates cost centre / pay location | A task with the data they need; mark done |
| U-06 | **IT team** | Provisions new access, removes old access | A task with old/new dept & role; mark done |
| U-07 | **Facilities team** | Arranges seat/location, badge access | A task with new location & date; mark done |
| U-08 | **Portal (system / orchestrator)** | Validates, routes, creates downstream tasks, aggregates status, notifies | — |

## 3. Journey stages (happy path + failure paths)

Trigger: the employee opens **"Request internal transfer"** in the portal. No prior manager discussion is checked (Q-03).

| Stage | Actor | User does | System does | Outcome / failure paths |
|---|---|---|---|---|
| S1 Open form | Employee | Opens "Internal transfer" | Authenticates (SSO); loads employee's current dept, location, role, manager from HRIS; loads selectable departments, locations, roles | Not eligible (e.g. probation, active request) → explain why, no form (Q-01, Q-09). HRIS down → "service unavailable, try later". |
| S2 Fill details | Employee | Selects department/BU, location, role; enters effective date; optional reason | Validates fields: values exist & are active; role valid for department; date within allowed window; reason ≤ max length | Invalid combination → field error. Date in past/too soon → field error (Q-05). Nothing changed vs current → error (Q-07). |
| S3 Submit | Employee | Clicks Submit (may double-click) | Re-validates server-side; checks no other active request; creates request with ID; status **Submitted / Pending current manager**; notifies next approver | Duplicate submit → one request only (idempotent). Second active request → rejected (Q-09). DB down → error, nothing saved. Notification fails → request still saved, notification retried. |
| S4 Current manager approval | Current manager | Approves / rejects (with comment) | Records decision; routes to next stage or closes as Rejected; notifies employee | Rejected → employee notified with reason (Q-13). No action for 3 working days → reminder; 5 working days → escalation to approver's manager; no auto-expiry; values configurable (Q-12). Manager absent → existing portal delegation if available, else out of scope (Q-14). Manager changes → pending approval moves to new manager; employee resigns → request auto-withdrawn (Q-15). |
| S5 Receiving manager approval (Q-02) | Receiving manager | Approves / rejects | Same as S4 | Same as S4. No system headcount check; approval confirms headcount (Q-27). |
| S6 HR validation | HR | Reviews eligibility, approves / rejects | Records decision; on approval sets status **Approved — scheduled for <effective date>** | HR rejects → employee and both managers notified (Q-28). No send-back; rejected employee submits a new request (Q-11). |
| S7 Org update | System / HR | — | Updates employee's org record in HRIS (dept, location, role, manager) on or before effective date | HRIS update fails → retried; HR alerted; status shows "Org update pending". Org update runs on the effective date; downstream tasks are not blocked by it — they are released at HR approval (Q-25). |
| S8 Downstream fulfilment | Payroll, IT, Facilities | Each completes its task | On HR approval, creates one task per required team (Payroll and IT always; Facilities only on location change — Q-16, Q-25); tracks each independently | Task overdue → shown as pending with owner; reminder/escalation per Q-12. Downstream system unreachable → retry, task shows "Not yet created", ops alerted. |
| S9 Completion | System | — | When the org update and all required tasks are done → status **Completed**; confirmation sent to employee (Q-25) | One task stuck → request stays "In progress" with the stuck item visible. |
| S10 Track (any time) | Employee (and others with access) | Opens "My transfer request" | Shows overall status, timeline of completed steps, **list of pending actions with the stakeholder group holding each** | Employee views someone else's request → forbidden. |
| S11 Withdraw (Q-10) | Employee | Withdraws request | Cancels open approvals/tasks; status **Withdrawn**; notifies stakeholders | Withdraw after HR approval → not allowed; only HR can cancel (Q-10). |

### Request status model (confirmed)

```
SUBMITTED ─▶ PENDING_CURRENT_MANAGER ─▶ PENDING_RECEIVING_MANAGER ─▶ PENDING_HR
                              │                          │                          │
                              ▼                          ▼                          ▼
                          REJECTED                   REJECTED                   REJECTED
PENDING_HR ─▶ APPROVED ─▶ IN_PROGRESS (org update + downstream tasks) ─▶ COMPLETED
Any state before APPROVED ─▶ WITHDRAWN (by employee, or automatically on resignation)
APPROVED / IN_PROGRESS ─▶ CANCELLED (by HR only)
```

## 4. Business rules identified

Rules the BRD states explicitly are marked **Stated**. Rules confirmed by the client on 2026-10-06 are marked **Confirmed**. These become BR-xx in the spec.

| ID | Rule | Source | Status |
|---|---|---|---|
| CR-01 | Employee can choose a new department/BU, location and role. | BRD §3 | Stated |
| CR-02 | Effective date is required. | BRD §3 | Stated |
| CR-03 | Reason is optional. | BRD §3 | Stated |
| CR-04 | Employee can see current status and pending actions with other stakeholders. | BRD §3 | Stated |
| CR-05 | Current manager must approve before HR validates. | BRD §2 (steps 2–3), Q-02 | Stated; order current → receiving → HR confirmed |
| CR-06 | HR validates eligibility. | BRD §2 step 3, Q-01, Q-04 | Stated; rules confirmed |
| CR-07 | Only one active transfer request per employee. | A-09 | Confirmed |
| CR-08 | Effective date ≥ 30 calendar days and ≤ 180 days from submission. | A-05 | Confirmed |
| CR-09 | At least one of department, location or role must differ from current. | A-07 | Confirmed |
| CR-10 | Employees in probation or serving notice cannot apply. | A-01 | Confirmed |
| CR-11 | Employee may withdraw until HR approval. | A-10 | Confirmed |
| CR-12 | Rejection requires a comment from the rejecting stakeholder. | A-13 | Confirmed |
| CR-13 | Payroll task always created; Facilities task only if location changes; IT task always created. | A-16 | Confirmed |
| CR-14 | Request is visible only to the employee, their current manager, receiving manager, HR and the downstream team for its own task. | A-22 | Confirmed |

## 5. Known decisions (from the BRD — no confirmation needed)

| ID | Decision |
|---|---|
| KD-01 | The journey lives inside the existing One-Point Employee Portal (no new app). |
| KD-02 | The **employee** initiates the request. |
| KD-03 | Inputs: department/BU, location, role/position, effective date, optional reason. |
| KD-04 | The portal orchestrates downstream activities (org update, payroll, IT, facilities). |
| KD-05 | The employee gets a single view of progress, including actions pending with others. |
| KD-06 | Stakeholders involved: manager, HR, payroll, IT, facilities. |

## 6. Open questions log

Priority: **H** = blocks the spec/AC; **M** = changes behaviour but has a safe default; **L** = can be decided later.
Status: Open → Answered / Assumed.

### 6.1 Eligibility & request rules

| ID | Pri | Question | Why it matters | Owner | Working assumption | Status |
|---|---|---|---|---|---|---|
| Q-01 | H | Who is eligible to request a transfer? (probation, notice period, under disciplinary/PIP, minimum tenure in current role, contract staff/interns) | Decides the S1 gate; wrong rule lets ineligible people apply or blocks valid ones | HR Policy | Permanent employees, not in probation, not serving notice. No tenure minimum. | Answered — client, 2026-10-06 |
| Q-02 | H | Does the **receiving** manager also approve? In what order vs current manager — sequential or parallel? | Shapes the whole approval chain and status model | Business (HR) | Yes. Sequential: current manager → receiving manager → HR. | Answered — client, 2026-10-06 |
| Q-03 | M | Is an offline discussion with the manager a pre-condition, or is the portal request itself the start? | Whether we need a "manager pre-agreed" checkbox | Business | Portal request is the start; no checkbox. | Answered — client, 2026-10-06 |
| Q-04 | H | What does HR "validate eligibility" mean concretely? Automated checks, manual review, or both? | Decides what the system checks vs what HR decides | HR Policy | System runs the Q-01 checks at submission; HR does a manual review and approves/rejects. | Answered — client, 2026-10-06 |
| Q-05 | H | Effective date rules: earliest (lead time)? latest? Must it be a specific day (e.g. 1st of month, payroll cut-off)? Past dates? | Test boundaries and payroll alignment | HR + Payroll | ≥ 30 days and ≤ 180 days after submission; any calendar day; no past dates. | Answered — client, 2026-10-06 |
| Q-06 | H | Can the employee pick **any** department/location/role, or only open vacancies / roles valid for that department / roles at their grade? | Drives lookup APIs and validation | HR | Any active department and location; role must belong to the selected department; no vacancy check. | Answered — client, 2026-10-06 |
| Q-07 | M | Must at least one of department, location or role change? Is a location-only transfer allowed? | Prevents empty requests; defines minimum valid request | HR | At least one must differ; location-only is allowed. | Answered — client, 2026-10-06 |
| Q-08 | L | Reason: max length? Who can see it? | Data limit and privacy | HR | Max 500 characters; visible to approvers and HR only (not downstream teams). | Answered — client, 2026-10-06 |
| Q-09 | H | Can an employee have more than one active request at a time? | Duplicate/conflicting transfers | HR | No — one active request at a time. | Answered — client, 2026-10-06 |
| Q-20 | M | Does a role change also change grade or compensation within this journey? | Compensation is a big separate process | HR / Comp & Ben | No — compensation/grade changes are out of scope; handled separately by HR. | Answered — client, 2026-10-06 |
| Q-21 | M | Are cross-legal-entity or cross-country transfers included? | They need new contracts, tax, visas | HR / Legal | Out of scope — same legal entity, same country only. | Answered — client, 2026-10-06 |
| Q-23 | M | Can a manager or HR raise a transfer on behalf of an employee? | Second entry point and permission model | Business | No — employee-initiated only (BRD). | Answered — client, 2026-10-06 |
| Q-27 | M | Is a headcount/budget check needed in the receiving department? | May block approval | Business / Finance | Receiving manager's approval implies headcount; no system check. | Answered — client, 2026-10-06 |

### 6.2 Workflow, SLAs & exceptions

| ID | Pri | Question | Why it matters | Owner | Working assumption | Status |
|---|---|---|---|---|---|---|
| Q-10 | H | Can the employee withdraw? Up to which stage? | Defines cancel rules and compensating actions | Business | Yes, until HR approval. After that only HR can cancel. | Answered — client, 2026-10-06 |
| Q-11 | M | Can the employee edit after submitting? Can an approver "send back for changes"? | Adds a rework loop | Business | No edit after submit; no send-back. Rejected → employee submits a new request. | Answered — client, 2026-10-06 |
| Q-12 | H | Approval / task SLAs: how long before a reminder, escalation or auto-expiry? | Requests getting stuck forever | Business | Reminder after 3 working days; escalation to approver's manager after 5; no auto-expiry. Values configurable. | Answered — client, 2026-10-06 |
| Q-13 | M | On rejection: is a comment mandatory? Can the employee re-apply immediately? | Fairness and UX | HR | Comment mandatory; can re-apply immediately. | Answered — client, 2026-10-06 |
| Q-14 | M | If an approver is absent, can they delegate? | Stuck approvals | Business | Use existing portal delegation if available; otherwise out of scope for v1. | Answered — client, 2026-10-06 |
| Q-15 | M | What if the employee's current manager changes, or the employee resigns, while the request is open? | Orphaned approvals | HR | Pending approval re-routes to new manager; resignation auto-withdraws the request. | Answered — client, 2026-10-06 |
| Q-24 | M | Can the effective date change after approval? By whom? | Downstream tasks are scheduled on that date | HR | Only HR can change it; downstream tasks are updated. | Answered — client, 2026-10-06 |
| Q-28 | L | If HR rejects after managers approved, who is notified? | Communication | HR | Employee, current manager and receiving manager. | Answered — client, 2026-10-06 |

### 6.3 Downstream orchestration & status view

| ID | Pri | Question | Why it matters | Owner | Working assumption | Status |
|---|---|---|---|---|---|---|
| Q-16 | H | Which downstream tasks are always required and which are conditional? (e.g. Facilities only if location changes; Payroll only if cost centre/location changes) | Defines what "Completed" means | Business + Payroll + IT + Facilities | Payroll and IT: always. Facilities: only when location changes. | Answered — client, 2026-10-06 |
| Q-17 | H | Do Payroll/IT/Facilities work inside the portal (task inbox) or in their own systems (e.g. ServiceNow, SAP) that the portal must integrate with? | Integration design and status sync | IT / Enterprise Architecture | Portal calls each team's system via an adapter and receives status back; for this build the adapters are simulated. | Answered — client, 2026-10-06 |
| Q-18 | H | What statuses and details should the employee see? Approver names or only team names? Comments? | Privacy and UX of the status view | Business + HR | Employee sees overall status, step timeline, pending actions by stakeholder role **and name for managers**, team name only for HR/Payroll/IT/Facilities. Sees rejection comment. | Answered — client, 2026-10-06 |
| Q-19 | M | Which notifications, through which channel, to whom? | Communication design | Business | Email + in-portal notification on submit, each approval/rejection, completion. | Answered — client, 2026-10-06 |
| Q-25 | H | When is the org record updated — immediately on HR approval or on the effective date? And are downstream tasks released before or after it? | Ordering of orchestration, and what happens if HRIS update fails | HR + IT | HR approval → downstream tasks created immediately (they need lead time); HRIS org update executed on effective date. Request "Completed" when org update + all tasks done. | Answered — client, 2026-10-06 |
| Q-26 | M | Audit and retention requirements for transfer records? | Compliance | HR / Legal | Every state change audited (who, when, what); retained per HR records policy (not built here). | Answered — client, 2026-10-06 |
| Q-29 | L | Is a draft (save without submit) needed? | Extra state | Business | No drafts in v1. | Answered — client, 2026-10-06 |
| Q-30 | L | How will success be measured? | Validation at Gate 2 | Business | % of transfers raised via portal; status queries to HR reduced. | Answered — client, 2026-10-06 |

### 6.4 Privacy & access

| ID | Pri | Question | Why it matters | Owner | Working assumption | Status |
|---|---|---|---|---|---|---|
| Q-22 | H | Who can see a transfer request? Should current colleagues/peers be prevented from seeing it? | A transfer request is sensitive | HR + Security | Only: the employee, current manager, receiving manager, HR, and each downstream team for its own task. | Answered — client, 2026-10-06 |
| Q-31 | M | What employee data may downstream teams see? (reason? previous role?) | Data minimisation | HR + Security | Downstream teams see only what their task needs: name, employee ID, old/new dept, location, role, effective date. Not the reason. | Answered — client, 2026-10-06 |

## 7. Assumptions list

Each maps 1:1 to the question with the same number. All confirmed by the client on 2026-10-06.

| ID | Assumption | From | Status |
|---|---|---|---|
| A-01 | Eligible = permanent employee, not in probation, not serving notice. | Q-01 | Confirmed |
| A-02 | Approval chain is sequential: current manager → receiving manager → HR. | Q-02 | Confirmed |
| A-03 | The portal request is the start; offline discussion is not checked. | Q-03 | Confirmed |
| A-04 | System runs eligibility checks at submission; HR does a manual review. | Q-04 | Confirmed |
| A-05 | Effective date ≥ 30 and ≤ 180 calendar days after submission; never in the past. | Q-05 | Confirmed |
| A-06 | Any active department & location; role must belong to the chosen department. | Q-06 | Confirmed |
| A-07 | At least one of department/location/role must differ from current. | Q-07 | Confirmed |
| A-08 | Reason ≤ 500 characters; hidden from downstream teams. | Q-08 | Confirmed |
| A-09 | One active request per employee. | Q-09 | Confirmed |
| A-10 | Withdraw allowed until HR approval. | Q-10 | Confirmed |
| A-11 | No edit or send-back after submit. | Q-11 | Confirmed |
| A-12 | Reminder at 3 working days, escalation at 5; configurable; no auto-expiry. | Q-12 | Confirmed |
| A-13 | Rejection comment mandatory; re-apply allowed immediately. | Q-13 | Confirmed |
| A-14 | Delegation out of scope for v1. | Q-14 | Confirmed |
| A-15 | Manager change re-routes approval; resignation auto-withdraws. | Q-15 | Confirmed |
| A-16 | Payroll & IT tasks always; Facilities only on location change. | Q-16 | Confirmed |
| A-17 | Downstream systems integrated via adapters; simulated in this build. | Q-17 | Confirmed |
| A-18 | Status view as described in Q-18. | Q-18 | Confirmed |
| A-19 | Email + in-portal notifications at key events. | Q-19 | Confirmed |
| A-20 | Compensation/grade changes out of scope. | Q-20 | Confirmed |
| A-21 | Same legal entity & country only. | Q-21 | Confirmed |
| A-22 | Visibility restricted per Q-22. | Q-22 | Confirmed |
| A-23 | Employee-initiated only. | Q-23 | Confirmed |
| A-24 | Only HR can change effective date after approval. | Q-24 | Confirmed |
| A-25 | Downstream tasks released on HR approval; org update on effective date. | Q-25 | Confirmed |
| A-26 | All state changes audited. | Q-26 | Confirmed |
| A-27 | No headcount check; receiving manager approval implies it. | Q-27 | Confirmed |
| A-28 | HR rejection notifies employee + both managers. | Q-28 | Confirmed |
| A-29 | No drafts. | Q-29 | Confirmed |
| A-31 | Downstream teams see minimal data, never the reason. | Q-31 | Confirmed |

## 8. Business decisions vs technical decisions

Test applied (SDD guide): *"Could a non-technical business owner decide this, and would the answer change what the employee or business experiences?"* Yes → business.

### 8.1 Business decisions

| ID | Decision needed | Owner | Linked Q | Status |
|---|---|---|---|---|
| BD-01 | Eligibility criteria | HR Policy | Q-01, Q-04 | Confirmed |
| BD-02 | Approval chain and order | HR | Q-02 | Confirmed |
| BD-03 | Effective date window | HR + Payroll | Q-05 | Confirmed |
| BD-04 | Selectable departments/locations/roles | HR | Q-06, Q-07 | Confirmed |
| BD-05 | One active request per employee | HR | Q-09 | Confirmed |
| BD-06 | Withdrawal rules | Business | Q-10 | Confirmed |
| BD-07 | Rejection handling (comment, re-apply) | HR | Q-13 | Confirmed |
| BD-08 | Approval/task SLAs and escalation | Business | Q-12 | Confirmed |
| BD-09 | Which downstream tasks are required when | Business + teams | Q-16 | Confirmed |
| BD-10 | Definition of "Completed" and orchestration order | HR + IT | Q-25 | Confirmed |
| BD-11 | What the employee sees in the status view | Business + HR | Q-18 | Confirmed |
| BD-12 | Who can see a request; what downstream teams see | HR + Security | Q-22, Q-31 | Confirmed |
| BD-13 | Notifications and channels | Business | Q-19 | Confirmed |
| BD-14 | Scope exclusions (comp, cross-entity, on-behalf, drafts, delegation) | Business | Q-14, Q-20, Q-21, Q-23, Q-29 | Confirmed |

### 8.2 Technical decisions (engineering owns — *proposed*, finalised in the technical plan)

| ID | Proposed decision | Chosen because | Supports |
|---|---|---|---|
| TD-01 | **Backend:** Node.js 20 + TypeScript + Express REST API | Stack chosen by the team (2026-10-06); Node 20 already installed; typed contracts; fast to test with supertest | All FRs |
| TD-02 | Vitest + supertest for backend unit/API tests; Vitest + React Testing Library for frontend components | Fast, TS-native, good for test-first RED/GREEN runs | Test-first |
| TD-03 | Explicit state machine for request status (allowed transitions table) | Illegal transitions (e.g. approve a withdrawn request) become impossible and unit-testable | BD-02, BD-06, BD-10 |
| TD-04 | Downstream systems behind adapter interfaces (HRIS, Payroll, ITSM, Facilities, Notifications) with in-memory fakes | Q-17 confirmed: teams use their own systems, simulated in this release; fakes allow failure simulation | BD-09, Q-17 |
| TD-05 | Outbox + retry for downstream calls and notifications | A failing downstream system must not lose the action or roll back an approval | BD-10, failure handling |
| TD-06 | All business limits (date window, SLA days, reason length) in config | Business answers to Q-05/Q-08/Q-12 can change without code change — guide: "build it as a setting" | BD-03, BD-08 |
| TD-07 | Idempotency key on submit | Double-click must create one request | Q-09, failure handling |
| TD-08 | Authorization check per request (owner / approver / HR / task team) on every read and write | Sensitive data; prevents viewing others' requests | BD-12 |
| TD-09 | Append-only audit event per state change, no sensitive free-text in logs | Traceable but safe | Q-26 |
| TD-10 | **Database:** SQLite (via `better-sqlite3`) behind a repository interface; schema in [`db-design/`](../../db-design/00-schema-design.md) | Stack chosen by the team (2026-10-06); file-based, zero setup, data survives restarts for the Gate 2 demo; partial unique indexes and triggers enforce key rules (BR-08, BR-12, BR-25); repository interface lets production swap the DB | BR-08, BR-25, NFR-04 |
| TD-11 | **Frontend:** Next.js + TypeScript, calling the Express API (no business rules in the frontend; the API re-validates everything) | Stack chosen by the team (2026-10-06); typed UI for the form, status view and approver work lists | KD-05, FR-01, FR-06, FR-13 |

> Rule followed: no technical decision above answers an open business question. Where a business value is unknown (date window, SLA, which tasks), the technical decision is to make it **configurable** (TD-04, TD-06).

## 9. Dependencies

| ID | Dependency | Used for | If unavailable / slow (initial view — detailed in security & plan) |
|---|---|---|---|
| DEP-01 | Identity / SSO (portal login) | Who the user is, their roles | No access; portal handles login |
| DEP-02 | HRIS — employee master | Current dept/location/role/manager, employment status (probation/notice) | Cannot open form or submit → 503 "try later"; nothing saved |
| DEP-03 | HRIS — org structure & positions | Department, location, role lists; role-to-department mapping | Form can't load lists → 503 |
| DEP-04 | HRIS — org update | Apply the transfer on effective date | Retry; HR alerted; status "Org update pending" |
| DEP-05 | Payroll system | Payroll task | Task creation queued & retried; shown "pending creation" |
| DEP-06 | ITSM (IT service management) | Access provision/removal task | Same as DEP-05 |
| DEP-07 | Facilities system | Seat/location/badge task | Same as DEP-05 |
| DEP-08 | Notification service (email + in-portal) | All notifications | Queued & retried; never blocks the workflow |
| DEP-09 | Audit store | State-change audit | Written in same transaction as state change |

## 10. Out of scope

Confirmed via the linked questions. OOS-05, OOS-08, OOS-10 and OOS-11 were confirmed later through spec question Q-42 (2026-10-06).

| ID | Item | Reason |
|---|---|---|
| OOS-01 | Compensation, grade or salary changes | Separate Comp & Ben process (Q-20) |
| OOS-02 | Cross-legal-entity / international transfers | New contract, tax, immigration (Q-21) |
| OOS-03 | Manager- or HR-initiated transfers | BRD says employee initiates (Q-23) |
| OOS-04 | Internal job postings / vacancy search / recruitment | Different journey (Q-06) |
| OOS-05 | Bulk / reorganisation transfers | Different volume and approvals |
| OOS-06 | Approval delegation management | Q-14 — use existing portal capability if any |
| OOS-07 | Real integration with production Payroll/ITSM/Facilities/HRIS | Adapters with simulated systems in this build (Q-17) |
| OOS-08 | Relocation allowance, travel, housing | Separate policy |
| OOS-09 | Draft requests, editing after submit | Q-11, Q-29 |
| OOS-10 | Mobile app; native UI redesign of the portal | Portal shell already exists |
| OOS-11 | Reporting/analytics dashboards | Not in BRD |

## 11. Top risks spotted during discovery

| Risk | Impact | Mitigation |
|---|---|---|
| Downstream tasks start before the org update (Q-25), so a task may finish while the HRIS update later fails | Records out of step between systems | Org update retried and HR alerted; request stays In progress until the update succeeds (TD-05) |
| Real downstream system interfaces not yet known (Q-17 answered: own systems, simulated now) | Integration rework when real systems connect | Adapter interfaces + fakes (TD-04) |
| Sensitive data exposure (transfer intent visible to peers) | Trust / HR risk | Strict per-request authorization (TD-08), data minimisation (Q-31) |
| Requests stuck with an approver | Poor employee experience | Reminder at 3 / escalation at 5 working days, configurable (Q-12, TD-06); pending-action visibility |
| Partial failure across systems | Inconsistent records | Outbox + retry, idempotent adapters, visible per-task status (TD-05) |

## 12. What happens next

1. ~~Client answers Q-01 … Q-31~~ — received 2026-10-06 and recorded above. All proposed answers were accepted.
2. ~~Chase Q-12 and Q-25~~ — answered 2026-10-06; proposed answers accepted.
3. Confirmed rules (CR-xx) become business rules **BR-xx** in the spec.
4. Deliverable 2: [`internal-transfer.spec.md`](internal-transfer.spec.md) — draft v0.1 written 2026-10-06. Questions raised while writing it continue as Q-32 – Q-44 in spec Section 13.
