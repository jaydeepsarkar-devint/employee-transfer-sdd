# Deliverable 1 — Requirement / Discovery Analysis

**Feature:** Employee Internal Transfer Digital Journey (One-Point Employee Portal)
**Input:** [BRD](../../.ai-context/BRD.md)
**Status:** Draft v0.1 — awaiting Product Owner answers to open questions
**Date:** 2026-10-01

---

## 1. Business objective

Replace today's fragmented, multi-team transfer process with **one digital journey** in the One-Point Employee Portal, where:

1. An employee raises an internal transfer request (department/BU, location, role, effective date, optional reason).
2. The portal routes it through the required approvals and validations.
3. The portal **orchestrates** the downstream work (org update, payroll, IT access, facilities).
4. The employee — and every stakeholder — sees **one view of status and of who is holding the next action**.

**Measurable outcome (proposed — needs PO confirmation, Q-30):** every transfer is raised and tracked in the portal; the employee never has to contact HR/IT/Facilities to learn the status.

## 2. Primary users

| ID | User | Role in the journey | What they need |
|---|---|---|---|
| U-01 | **Employee (requester)** | Starts the request, tracks progress, may withdraw | Simple form, valid choices only, clear status and "who is it waiting on" |
| U-02 | **Current (line) manager** | Approves/rejects release of the employee | Request details, ability to approve/reject with comment |
| U-03 | **Receiving manager** (head of target dept/position) | Accepts the employee into the new team *(existence of this step is Q-02)* | Request details, approve/reject |
| U-04 | **HR (HR Business Partner / HR Ops)** | Validates eligibility, finalises the transfer, updates organisational record | Eligibility facts, approve/reject, trigger org update |
| U-05 | **Payroll team** | Updates cost centre / pay location | A task with the data they need; mark done |
| U-06 | **IT team** | Provisions new access, removes old access | A task with old/new dept & role; mark done |
| U-07 | **Facilities team** | Arranges seat/location, badge access | A task with new location & date; mark done |
| U-08 | **Portal (system / orchestrator)** | Validates, routes, creates downstream tasks, aggregates status, notifies | — |

## 3. Journey stages (happy path + failure paths)

Trigger: the employee has discussed a move with their manager *(or wants to — Q-03)* and opens **"Request internal transfer"** in the portal.

| Stage | Actor | User does | System does | Outcome / failure paths |
|---|---|---|---|---|
| S1 Open form | Employee | Opens "Internal transfer" | Authenticates (SSO); loads employee's current dept, location, role, manager from HRIS; loads selectable departments, locations, roles | Not eligible (e.g. probation, active request) → explain why, no form (Q-01, Q-09). HRIS down → "service unavailable, try later". |
| S2 Fill details | Employee | Selects department/BU, location, role; enters effective date; optional reason | Validates fields: values exist & are active; role valid for department; date within allowed window; reason ≤ max length | Invalid combination → field error. Date in past/too soon → field error (Q-05). Nothing changed vs current → error (Q-07). |
| S3 Submit | Employee | Clicks Submit (may double-click) | Re-validates server-side; checks no other active request; creates request with ID; status **Submitted / Pending current manager**; notifies next approver | Duplicate submit → one request only (idempotent). Second active request → rejected (Q-09). DB down → error, nothing saved. Notification fails → request still saved, notification retried. |
| S4 Current manager approval | Current manager | Approves / rejects (with comment) | Records decision; routes to next stage or closes as Rejected; notifies employee | Rejected → employee notified with reason (Q-13). No action for N days → reminder/escalation (Q-12). Manager absent → delegate (Q-14). |
| S5 Receiving manager approval *(if required — Q-02)* | Receiving manager | Approves / rejects | Same as S4 | Same as S4. Headcount not available → reject (Q-27). |
| S6 HR validation | HR | Reviews eligibility, approves / rejects / sends back | Records decision; on approval sets status **Approved — scheduled for <effective date>** | HR rejects → employee and managers notified (Q-28). HR needs info → sent back to employee (Q-11). |
| S7 Org update | System / HR | — | Updates employee's org record in HRIS (dept, location, role, manager) on or before effective date | HRIS update fails → retried; HR alerted; status shows "Org update pending"; downstream tasks not released until it succeeds (Q-25). |
| S8 Downstream fulfilment | Payroll, IT, Facilities | Each completes its task | Creates one task per required team (conditional — Q-16); tracks each independently | Task overdue → shown as pending with owner (Q-12). Downstream system unreachable → retry, task shows "Not yet created", ops alerted. |
| S9 Completion | System | — | When all required tasks are done → status **Completed**; confirmation sent to employee | One task stuck → request stays "In progress" with the stuck item visible. |
| S10 Track (any time) | Employee (and others with access) | Opens "My transfer request" | Shows overall status, timeline of completed steps, **list of pending actions with the stakeholder group holding each** | Employee views someone else's request → forbidden. |
| S11 Withdraw *(if allowed — Q-10)* | Employee | Withdraws request | Cancels open approvals/tasks; status **Withdrawn**; notifies stakeholders | Withdraw after org update → not allowed / needs HR (Q-10). |

### Proposed request status model (to confirm, Q-18)

```
Draft? ─▶ SUBMITTED ─▶ PENDING_CURRENT_MANAGER ─▶ PENDING_RECEIVING_MANAGER ─▶ PENDING_HR
                              │                          │                          │
                              ▼                          ▼                          ▼
                          REJECTED                   REJECTED                   REJECTED
PENDING_HR ─▶ APPROVED ─▶ IN_PROGRESS (org update + downstream tasks) ─▶ COMPLETED
Any state before IN_PROGRESS ─▶ WITHDRAWN (by employee)
```

## 4. Business rules identified (candidate — none confirmed yet)

Rules the BRD states explicitly are marked **Stated**. Everything else is a candidate rule from a working assumption and must be confirmed.

| ID | Rule | Source | Status |
|---|---|---|---|
| CR-01 | Employee can choose a new department/BU, location and role. | BRD §3 | Stated |
| CR-02 | Effective date is required. | BRD §3 | Stated |
| CR-03 | Reason is optional. | BRD §3 | Stated |
| CR-04 | Employee can see current status and pending actions with other stakeholders. | BRD §3 | Stated |
| CR-05 | Current manager must approve before HR validates. | BRD §2 (steps 2–3) | Stated (order) — Q-02 for receiving manager |
| CR-06 | HR validates eligibility. | BRD §2 step 3 | Stated — rules unknown (Q-01, Q-04) |
| CR-07 | Only one active transfer request per employee. | A-09 | Assumed |
| CR-08 | Effective date ≥ 30 calendar days and ≤ 180 days from submission. | A-05 | Assumed |
| CR-09 | At least one of department, location or role must differ from current. | A-07 | Assumed |
| CR-10 | Employees in probation or serving notice cannot apply. | A-01 | Assumed |
| CR-11 | Employee may withdraw until HR approval. | A-10 | Assumed |
| CR-12 | Rejection requires a comment from the rejecting stakeholder. | A-13 | Assumed |
| CR-13 | Payroll task always created; Facilities task only if location changes; IT task always created. | A-16 | Assumed |
| CR-14 | Request is visible only to the employee, their current manager, receiving manager, HR and the downstream team for its own task. | A-22 | Assumed |

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
| Q-01 | H | Who is eligible to request a transfer? (probation, notice period, under disciplinary/PIP, minimum tenure in current role, contract staff/interns) | Decides the S1 gate; wrong rule lets ineligible people apply or blocks valid ones | HR Policy | Permanent employees, not in probation, not serving notice. No tenure minimum. | Open |
| Q-02 | H | Does the **receiving** manager also approve? In what order vs current manager — sequential or parallel? | Shapes the whole approval chain and status model | Business (HR) | Yes. Sequential: current manager → receiving manager → HR. | Open |
| Q-03 | M | Is an offline discussion with the manager a pre-condition, or is the portal request itself the start? | Whether we need a "manager pre-agreed" checkbox | Business | Portal request is the start; no checkbox. | Open |
| Q-04 | H | What does HR "validate eligibility" mean concretely? Automated checks, manual review, or both? | Decides what the system checks vs what HR decides | HR Policy | System runs the Q-01 checks at submission; HR does a manual review and approves/rejects. | Open |
| Q-05 | H | Effective date rules: earliest (lead time)? latest? Must it be a specific day (e.g. 1st of month, payroll cut-off)? Past dates? | Test boundaries and payroll alignment | HR + Payroll | ≥ 30 days and ≤ 180 days after submission; any calendar day; no past dates. | Open |
| Q-06 | H | Can the employee pick **any** department/location/role, or only open vacancies / roles valid for that department / roles at their grade? | Drives lookup APIs and validation | HR | Any active department and location; role must belong to the selected department; no vacancy check. | Open |
| Q-07 | M | Must at least one of department, location or role change? Is a location-only transfer allowed? | Prevents empty requests; defines minimum valid request | HR | At least one must differ; location-only is allowed. | Open |
| Q-08 | L | Reason: max length? Who can see it? | Data limit and privacy | HR | Max 500 characters; visible to approvers and HR only (not downstream teams). | Open |
| Q-09 | H | Can an employee have more than one active request at a time? | Duplicate/conflicting transfers | HR | No — one active request at a time. | Open |
| Q-20 | M | Does a role change also change grade or compensation within this journey? | Compensation is a big separate process | HR / Comp & Ben | No — compensation/grade changes are out of scope; handled separately by HR. | Open |
| Q-21 | M | Are cross-legal-entity or cross-country transfers included? | They need new contracts, tax, visas | HR / Legal | Out of scope — same legal entity, same country only. | Open |
| Q-23 | M | Can a manager or HR raise a transfer on behalf of an employee? | Second entry point and permission model | Business | No — employee-initiated only (BRD). | Open |
| Q-27 | M | Is a headcount/budget check needed in the receiving department? | May block approval | Business / Finance | Receiving manager's approval implies headcount; no system check. | Open |

### 6.2 Workflow, SLAs & exceptions

| ID | Pri | Question | Why it matters | Owner | Working assumption | Status |
|---|---|---|---|---|---|---|
| Q-10 | H | Can the employee withdraw? Up to which stage? | Defines cancel rules and compensating actions | Business | Yes, until HR approval. After that only HR can cancel. | Open |
| Q-11 | M | Can the employee edit after submitting? Can an approver "send back for changes"? | Adds a rework loop | Business | No edit after submit; no send-back. Rejected → employee submits a new request. | Open |
| Q-12 | H | Approval / task SLAs: how long before a reminder, escalation or auto-expiry? | Requests getting stuck forever | Business | Reminder after 3 working days; escalation to approver's manager after 5; no auto-expiry. Values configurable. | Open |
| Q-13 | M | On rejection: is a comment mandatory? Can the employee re-apply immediately? | Fairness and UX | HR | Comment mandatory; can re-apply immediately. | Open |
| Q-14 | M | If an approver is absent, can they delegate? | Stuck approvals | Business | Use existing portal delegation if available; otherwise out of scope for v1. | Open |
| Q-15 | M | What if the employee's current manager changes, or the employee resigns, while the request is open? | Orphaned approvals | HR | Pending approval re-routes to new manager; resignation auto-withdraws the request. | Open |
| Q-24 | M | Can the effective date change after approval? By whom? | Downstream tasks are scheduled on that date | HR | Only HR can change it; downstream tasks are updated. | Open |
| Q-28 | L | If HR rejects after managers approved, who is notified? | Communication | HR | Employee, current manager and receiving manager. | Open |

### 6.3 Downstream orchestration & status view

| ID | Pri | Question | Why it matters | Owner | Working assumption | Status |
|---|---|---|---|---|---|---|
| Q-16 | H | Which downstream tasks are always required and which are conditional? (e.g. Facilities only if location changes; Payroll only if cost centre/location changes) | Defines what "Completed" means | Business + Payroll + IT + Facilities | Payroll and IT: always. Facilities: only when location changes. | Open |
| Q-17 | H | Do Payroll/IT/Facilities work inside the portal (task inbox) or in their own systems (e.g. ServiceNow, SAP) that the portal must integrate with? | Integration design and status sync | IT / Enterprise Architecture | Portal calls each team's system via an adapter and receives status back; for this build the adapters are simulated. | Open |
| Q-18 | H | What statuses and details should the employee see? Approver names or only team names? Comments? | Privacy and UX of the status view | Business + HR | Employee sees overall status, step timeline, pending actions by stakeholder role **and name for managers**, team name only for HR/Payroll/IT/Facilities. Sees rejection comment. | Open |
| Q-19 | M | Which notifications, through which channel, to whom? | Communication design | Business | Email + in-portal notification on submit, each approval/rejection, completion. | Open |
| Q-25 | H | When is the org record updated — immediately on HR approval or on the effective date? And are downstream tasks released before or after it? | Ordering of orchestration, and what happens if HRIS update fails | HR + IT | HR approval → downstream tasks created immediately (they need lead time); HRIS org update executed on effective date. Request "Completed" when org update + all tasks done. | Open |
| Q-26 | M | Audit and retention requirements for transfer records? | Compliance | HR / Legal | Every state change audited (who, when, what); retained per HR records policy (not built here). | Open |
| Q-29 | L | Is a draft (save without submit) needed? | Extra state | Business | No drafts in v1. | Open |
| Q-30 | L | How will success be measured? | Validation at Gate 2 | Business | % of transfers raised via portal; status queries to HR reduced. | Open |

### 6.4 Privacy & access

| ID | Pri | Question | Why it matters | Owner | Working assumption | Status |
|---|---|---|---|---|---|---|
| Q-22 | H | Who can see a transfer request? Should current colleagues/peers be prevented from seeing it? | A transfer request is sensitive | HR + Security | Only: the employee, current manager, receiving manager, HR, and each downstream team for its own task. | Open |
| Q-31 | M | What employee data may downstream teams see? (reason? previous role?) | Data minimisation | HR + Security | Downstream teams see only what their task needs: name, employee ID, old/new dept, location, role, effective date. Not the reason. | Open |

## 7. Assumptions list

All **assumed, not confirmed** until the Product Owner answers. Each maps 1:1 to the question with the same number.

| ID | Assumption | From | Status |
|---|---|---|---|
| A-01 | Eligible = permanent employee, not in probation, not serving notice. | Q-01 | Assumed |
| A-02 | Approval chain is sequential: current manager → receiving manager → HR. | Q-02 | Assumed |
| A-03 | The portal request is the start; offline discussion is not checked. | Q-03 | Assumed |
| A-04 | System runs eligibility checks at submission; HR does a manual review. | Q-04 | Assumed |
| A-05 | Effective date ≥ 30 and ≤ 180 calendar days after submission; never in the past. | Q-05 | Assumed |
| A-06 | Any active department & location; role must belong to the chosen department. | Q-06 | Assumed |
| A-07 | At least one of department/location/role must differ from current. | Q-07 | Assumed |
| A-08 | Reason ≤ 500 characters; hidden from downstream teams. | Q-08 | Assumed |
| A-09 | One active request per employee. | Q-09 | Assumed |
| A-10 | Withdraw allowed until HR approval. | Q-10 | Assumed |
| A-11 | No edit or send-back after submit. | Q-11 | Assumed |
| A-12 | Reminder at 3 working days, escalation at 5; configurable; no auto-expiry. | Q-12 | Assumed |
| A-13 | Rejection comment mandatory; re-apply allowed immediately. | Q-13 | Assumed |
| A-14 | Delegation out of scope for v1. | Q-14 | Assumed |
| A-15 | Manager change re-routes approval; resignation auto-withdraws. | Q-15 | Assumed |
| A-16 | Payroll & IT tasks always; Facilities only on location change. | Q-16 | Assumed |
| A-17 | Downstream systems integrated via adapters; simulated in this build. | Q-17 | Assumed |
| A-18 | Status view as described in Q-18. | Q-18 | Assumed |
| A-19 | Email + in-portal notifications at key events. | Q-19 | Assumed |
| A-20 | Compensation/grade changes out of scope. | Q-20 | Assumed |
| A-21 | Same legal entity & country only. | Q-21 | Assumed |
| A-22 | Visibility restricted per Q-22. | Q-22 | Assumed |
| A-23 | Employee-initiated only. | Q-23 | Assumed |
| A-24 | Only HR can change effective date after approval. | Q-24 | Assumed |
| A-25 | Downstream tasks released on HR approval; org update on effective date. | Q-25 | Assumed |
| A-26 | All state changes audited. | Q-26 | Assumed |
| A-27 | No headcount check; receiving manager approval implies it. | Q-27 | Assumed |
| A-28 | HR rejection notifies employee + both managers. | Q-28 | Assumed |
| A-29 | No drafts. | Q-29 | Assumed |
| A-31 | Downstream teams see minimal data, never the reason. | Q-31 | Assumed |

## 8. Business decisions vs technical decisions

Test applied (SDD guide): *"Could a non-technical business owner decide this, and would the answer change what the employee or business experiences?"* Yes → business.

### 8.1 Business decisions (need PO / HR / Security confirmation)

| ID | Decision needed | Owner | Linked Q | Status |
|---|---|---|---|---|
| BD-01 | Eligibility criteria | HR Policy | Q-01, Q-04 | Assumed (A-01, A-04) |
| BD-02 | Approval chain and order | HR | Q-02 | Assumed (A-02) |
| BD-03 | Effective date window | HR + Payroll | Q-05 | Assumed (A-05) |
| BD-04 | Selectable departments/locations/roles | HR | Q-06, Q-07 | Assumed (A-06, A-07) |
| BD-05 | One active request per employee | HR | Q-09 | Assumed (A-09) |
| BD-06 | Withdrawal rules | Business | Q-10 | Assumed (A-10) |
| BD-07 | Rejection handling (comment, re-apply) | HR | Q-13 | Assumed (A-13) |
| BD-08 | Approval/task SLAs and escalation | Business | Q-12 | Assumed (A-12) |
| BD-09 | Which downstream tasks are required when | Business + teams | Q-16 | Assumed (A-16) |
| BD-10 | Definition of "Completed" and orchestration order | HR + IT | Q-25 | Assumed (A-25) |
| BD-11 | What the employee sees in the status view | Business + HR | Q-18 | Assumed (A-18) |
| BD-12 | Who can see a request; what downstream teams see | HR + Security | Q-22, Q-31 | Assumed (A-22, A-31) |
| BD-13 | Notifications and channels | Business | Q-19 | Assumed (A-19) |
| BD-14 | Scope exclusions (comp, cross-entity, on-behalf, drafts, delegation) | Business | Q-14, Q-20, Q-21, Q-23, Q-29 | Assumed |

### 8.2 Technical decisions (engineering owns — *proposed*, finalised in the technical plan)

| ID | Proposed decision | Chosen because | Supports |
|---|---|---|---|
| TD-01 | Node.js 20 + TypeScript + Express REST API | Already installed (Node 20); typed contracts; fast to test with supertest | All FRs |
| TD-02 | Vitest + supertest for unit/API tests | Fast, TS-native, good for test-first RED/GREEN runs | Test-first |
| TD-03 | Explicit state machine for request status (allowed transitions table) | Illegal transitions (e.g. approve a withdrawn request) become impossible and unit-testable | BD-02, BD-06, BD-10 |
| TD-04 | Downstream systems behind adapter interfaces (HRIS, Payroll, ITSM, Facilities, Notifications) with in-memory fakes | Q-17 is open; adapters let the business answer change without rewrite; fakes allow failure simulation | BD-09, Q-17 |
| TD-05 | Outbox + retry for downstream calls and notifications | A failing downstream system must not lose the action or roll back an approval | BD-10, failure handling |
| TD-06 | All business limits (date window, SLA days, reason length) in config | Business answers to Q-05/Q-08/Q-12 can change without code change — guide: "build it as a setting" | BD-03, BD-08 |
| TD-07 | Idempotency key on submit | Double-click must create one request | Q-09, failure handling |
| TD-08 | Authorization check per request (owner / approver / HR / task team) on every read and write | Sensitive data; prevents viewing others' requests | BD-12 |
| TD-09 | Append-only audit event per state change, no sensitive free-text in logs | Traceable but safe | Q-26 |
| TD-10 | SQLite (or in-memory repository) for the exercise | Zero setup; repository interface lets prod swap DB | — |
| TD-11 | Minimal server-rendered/static web UI for the employee form + status view | Demo of full journey at Gate 2 without UI framework cost | KD-05 |

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

## 10. Out of scope (proposed — confirm with PO, BD-14)

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
| Approval chain (Q-02) and completion definition (Q-25) unanswered | Spec, state machine and AC all change | Get answered before Gate 1; state machine table-driven |
| Downstream integration model unknown (Q-17) | Integration rework | Adapter interfaces + fakes (TD-04) |
| Sensitive data exposure (transfer intent visible to peers) | Trust / HR risk | Strict per-request authorization (TD-08), data minimisation (Q-31) |
| Requests stuck with an approver | Poor employee experience | SLA reminders/escalation (Q-12), pending-action visibility |
| Partial failure across systems | Inconsistent records | Outbox + retry, idempotent adapters, visible per-task status (TD-05) |

## 12. What happens next

1. Product Owner answers Q-01 … Q-31 (H-priority first: Q-01, 02, 04, 05, 06, 09, 10, 12, 16, 17, 18, 22, 25).
2. Answers recorded here (status → Answered, with who and date); assumptions become confirmed business rules **BR-xx** in the spec.
3. Deliverable 2: `internal-transfer.spec.md` with FRs, NFRs and Given/When/Then AC.
