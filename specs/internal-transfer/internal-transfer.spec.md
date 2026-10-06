# Deliverable 2 — Feature Specification: Employee Internal Transfer

**Spec ID:** SPEC-IT-001
**Version:** 0.2 — Ready for Gate 1
**Date:** 2026-10-06
**Inputs:** [BRD](../../.ai-context/BRD.md) · [Discovery v0.3](01-discovery.md) (all Q-01 … Q-31 answered by the client on 2026-10-06)
**Status:** Ready for Gate 1. All business questions answered: Q-01 – Q-31 (discovery) and Q-32 – Q-44 (Section 13), confirmed by the client on 2026-10-06.

ID rules: every rule, requirement and criterion below has an ID. `BR` = business rule, `FR` / `NFR` = functional / non-functional requirement, `AC` = acceptance criterion, `Q` / `A` = question / assumption (continuing the discovery numbering). Changes after Gate 1 go to `deviation-log.md` (DEV-xx).

---

## 1. Feature name

**Internal Transfer Request** in the One-Point Employee Portal.

## 2. Objective

Let an employee raise an internal transfer request in the portal and follow it to completion in one place. The portal routes the request through the required approvals, sends the follow-up work to Payroll, IT and Facilities, updates the employee's HR record, and shows the employee at every point what is done and who has the next action.

**Success measures (Q-30):**
- Share of internal transfers raised through the portal (target to be set by the client after go-live).
- Number of transfer-status questions sent to HR, compared with before go-live.

## 3. Context

Today a transfer needs the employee to deal separately with their manager, HR, Payroll, IT and Facilities (BRD §2). The portal already exists and already provides employee services and login (KD-01). This feature adds one journey to it. Payroll, IT and Facilities keep working in their own systems; the portal connects to them. In this release those connections are simulated (Q-17).

## 4. Personas

| ID | Persona | What they do in this feature |
|---|---|---|
| U-01 | Employee (requester) | Raises the request, tracks it, may withdraw it before HR approval |
| U-02 | Current manager | First approver: approves or rejects |
| U-03 | Receiving manager | Second approver: approves or rejects taking the employee into the target department |
| U-04 | HR | Third approver: reviews eligibility, approves or rejects; after approval may change the effective date or cancel |
| U-05 | Payroll team | Completes the payroll task in its own system |
| U-06 | IT team | Completes the access task in its own system |
| U-07 | Facilities team | Completes the location task in its own system (only when location changes) |
| U-08 | Portal (system) | Validates, routes, creates tasks, updates the HR record, reminds, escalates, notifies, records audit |

## 5. Journey

| Step | Actor | What happens | Rules |
|---|---|---|---|
| J1 | Employee | Opens "Request internal transfer". The portal checks eligibility and shows the form with current details filled in, or explains why they cannot apply. | BR-01, BR-08 |
| J2 | Employee | Selects department, location, role; enters effective date; optionally enters a reason; submits. | BR-04 – BR-07, BR-17, BR-18 |
| J3 | Portal | Re-checks every rule, saves the request, sets it to *Pending current manager*, notifies the current manager. | BR-02, BR-03, BR-08 |
| J4 | Current manager | Approves (→ receiving manager) or rejects with a comment (→ *Rejected*). | BR-03, BR-12 |
| J5 | Receiving manager | Approves (→ HR) or rejects with a comment (→ *Rejected*). | BR-03, BR-12, BR-26 |
| J6 | HR | Reviews eligibility; approves (→ *In progress*) or rejects with a comment (→ *Rejected*). | BR-02, BR-12, BR-24 |
| J7 | Portal | On HR approval, creates the Payroll and IT tasks, and the Facilities task if the location changes, and sends each to its team's system. | BR-15, BR-16, BR-22 |
| J8 | Teams | Complete their tasks; their systems report completion back to the portal. | BR-15 |
| J9 | Portal | On the effective date, updates the employee's HR record. | BR-16 |
| J10 | Portal | When the HR record is updated and all tasks are done, sets *Completed* and notifies the employee. | BR-16 |
| J11 | Employee | At any time, views status, timeline and pending actions. | BR-23 |
| J12 | Employee | May withdraw until HR approves. After that only HR can cancel. | BR-09 |
| — | Portal | Throughout: reminders and escalations, manager-change and resignation handling, notifications, audit. | BR-11, BR-14, BR-24, BR-25 |

### 5.1 Request status model

| Status | Meaning | Shown to employee as |
|---|---|---|
| `PENDING_CURRENT_MANAGER` | Waiting for the current manager | "Waiting for your manager, <name>" |
| `PENDING_RECEIVING_MANAGER` | Waiting for the receiving manager | "Waiting for the receiving manager, <name>" |
| `PENDING_HR` | Waiting for HR review | "Waiting for HR" |
| `IN_PROGRESS` | HR approved; tasks and/or HR record update outstanding | "Approved — transfer in progress" |
| `COMPLETED` | HR record updated and all tasks done (terminal) | "Completed" |
| `REJECTED` | Rejected by an approver (terminal) | "Rejected by <role>" + comment |
| `WITHDRAWN` | Withdrawn by the employee, or automatically on resignation (terminal) | "Withdrawn" |
| `CANCELLED` | Cancelled by HR after approval (terminal) | "Cancelled by HR" |

Allowed transitions (anything not listed is refused):

| From | To | By |
|---|---|---|
| *(new)* | `PENDING_CURRENT_MANAGER` | Employee submits |
| `PENDING_CURRENT_MANAGER` | `PENDING_RECEIVING_MANAGER` | Current manager approves |
| `PENDING_CURRENT_MANAGER` | `PENDING_HR` | Current manager approves **and** receiving manager is the same person (A-33) |
| `PENDING_RECEIVING_MANAGER` | `PENDING_HR` | Receiving manager approves |
| `PENDING_HR` | `IN_PROGRESS` | HR approves |
| `IN_PROGRESS` | `COMPLETED` | Portal, when BR-16 completion condition is met |
| any `PENDING_*` | `REJECTED` | The approver the request is pending with |
| any `PENDING_*` | `WITHDRAWN` | Employee, or portal on resignation |
| `IN_PROGRESS` | `CANCELLED` | HR, only before the HR record update (A-37) |

*Refinement of the discovery model:* discovery listed `SUBMITTED` and `APPROVED` as separate states. Neither is ever a resting state — submission goes straight to `PENDING_CURRENT_MANAGER`, and HR approval creates the tasks immediately (Q-25) — so both are removed. *Active* request = any status that is not terminal.

## 6. Scope

In this release:

1. Eligibility check and request form (department, location, role, effective date, optional reason).
2. Submission with full server-side validation and protection against double submission.
3. Sequential approval: current manager → receiving manager → HR, with approve / reject.
4. A work list for each approver showing requests pending with them.
5. Downstream tasks for Payroll, IT and Facilities, sent to simulated team systems, with status received back.
6. HR record update on the effective date, with retry.
7. Status view: overall status, timeline, pending actions.
8. Withdraw (employee), cancel and effective-date change (HR).
9. Reminders and escalations.
10. Handling of manager change and resignation while a request is open.
11. Email and in-portal notifications.
12. Per-request access control, data minimisation, audit trail.

## 7. Out of scope

| ID | Item | Source |
|---|---|---|
| OOS-01 | Grade, pay or compensation changes | Q-20 |
| OOS-02 | Transfers across legal entities or countries | Q-21 |
| OOS-03 | Requests raised by a manager or HR on the employee's behalf | Q-23 |
| OOS-04 | Vacancy search, job postings, recruitment | Q-06 |
| OOS-05 | Bulk or reorganisation transfers | Q-42 |
| OOS-06 | Building delegation; existing portal delegation is used if present | Q-14 |
| OOS-07 | Connections to the real HRIS, Payroll, IT and Facilities systems (simulated in this release) | Q-17 |
| OOS-08 | Relocation allowance, travel, housing | Q-42 |
| OOS-09 | Drafts; editing a request after submission; "send back for changes" | Q-11, Q-29 |
| OOS-10 | Mobile app; redesign of the portal | Q-42 |
| OOS-11 | Reporting and analytics dashboards | Q-42 |
| OOS-12 | Headcount or budget checks | Q-27 |
| OOS-13 | Record retention and deletion (existing HR records policy applies) | Q-26 |

## 8. Business rules

All rules are confirmed by the client (2026-10-06). A-32 onward in the Source column are spec-stage assumptions the client confirmed (Section 13).

| ID | Rule | Source |
|---|---|---|
| BR-01 | Only a **permanent** employee who is **not on probation** and **not serving notice** may request a transfer. There is no minimum time in the current role. | Q-01 |
| BR-02 | The portal checks BR-01 when the form is opened and again at submission, and stores the check result on the request. HR then reviews and approves or rejects. | Q-04 |
| BR-03 | Approvals are sequential: current manager → receiving manager → HR. A step cannot be acted on before the previous step is approved. The receiving manager is the head of the target department in the HR system (A-32). If the receiving manager is the same person as the current manager, that person approves once (A-33). | Q-02, A-32, A-33 |
| BR-04 | Effective date is required; it must be **at least 30 and at most 180 calendar days** after the submission date; any calendar day is allowed; past dates are never allowed. 30 and 180 are configuration values. Dates are calendar dates in the organisation's configured time zone (A-41). | Q-05, A-41 |
| BR-05 | Selected department and location must be **active**. The selected role must **belong to the selected department**. There is no vacancy check. | Q-06 |
| BR-06 | At least one of department, location or role must differ from the employee's current value. A location-only change is allowed. | Q-07 |
| BR-07 | Reason is optional, at most **500 characters** (configuration value). It is visible to the employee, both managers and HR only. | Q-08 |
| BR-08 | An employee may have **one active request** at a time. After a request reaches a terminal status, a new one may be submitted immediately. | Q-09, Q-13 |
| BR-09 | The employee may withdraw while the request is in any `PENDING_*` status. After HR approval only HR can cancel. | Q-10 |
| BR-10 | A submitted request cannot be edited by the employee, and approvers cannot send it back. To change it, the employee withdraws (if allowed) or waits for rejection and submits a new request. | Q-11 |
| BR-11 | If a request or task waits with the same approver or team for **3 working days**, the portal sends a reminder; at **5 working days** it escalates — to the approver's manager for manager approvals, and to the team's configured escalation contact for HR review and for tasks (A-35). There is no automatic expiry. 3 and 5 are configuration values. Working days = Monday–Friday excluding configured public holidays (A-36). | Q-12, A-35, A-36 |
| BR-12 | Rejecting at any approval step requires a non-empty comment. | Q-13 |
| BR-13 | If the portal has approval delegation, a delegate may act for an absent approver; this feature builds no delegation of its own. | Q-14 |
| BR-14 | If the employee's current manager changes in the HR system while the request is `PENDING_CURRENT_MANAGER`, the pending approval moves to the new manager. If the head of the target department changes while `PENDING_RECEIVING_MANAGER`, it moves to the new head (A-44). If the employee resigns while the request is in a `PENDING_*` status, the request is withdrawn automatically; if they resign while `IN_PROGRESS`, HR is alerted to decide whether to cancel (A-38). | Q-15, A-38, A-44 |
| BR-15 | On HR approval the portal creates a **Payroll** task and an **IT** task for every request, and a **Facilities** task only when the location changes. Teams can only mark a task done (A-40). | Q-16, A-40 |
| BR-16 | Tasks are created **at HR approval**. The HR record (department, location, role, manager) is updated **on the effective date**. The request is **Completed** when the HR record update has succeeded **and** all its tasks are done. | Q-25 |
| BR-17 | The journey does not change grade or pay. | Q-20 |
| BR-18 | Only departments of the employee's current legal entity and locations in the employee's current country can be selected. | Q-21 |
| BR-19 | Only the employee can raise their own request. | Q-23 |
| BR-20 | After HR approval only HR can change the effective date; the new date must not be in the past and must be at most 180 days from the day of the change (A-39). Open tasks and the scheduled HR record update follow the new date. | Q-24, A-39 |
| BR-21 | A request can be seen only by: the employee, the current manager, the receiving manager, HR, and each downstream team for its own task. Anyone else gets the same response as for a request that does not exist. | Q-22 |
| BR-22 | A downstream team receives only: employee name, employee ID, current and new department, current and new location, current and new role, effective date. Never the reason. | Q-31 |
| BR-23 | The employee's status view shows: overall status, a timeline of completed steps with dates, and every pending action with who holds it — managers by name; HR, Payroll, IT and Facilities by team name only. A rejected request shows the rejection comment. | Q-18 |
| BR-24 | Notifications go by email and in-portal. Events and recipients are in FR-17. | Q-19, Q-28 |
| BR-25 | Every status change and every task status change is recorded: who, when, from-status, to-status. Audit records cannot be changed or deleted through the portal. | Q-26 |
| BR-26 | There is no headcount or budget check; the receiving manager's approval confirms headcount. | Q-27 |
| BR-27 | There are no drafts; a request exists only once it is submitted. | Q-29 |
| BR-28 | HR can cancel an `IN_PROGRESS` request only before the HR record update has succeeded. Open tasks are cancelled in the team systems. | Q-10, A-37 |
| BR-29 | HR cannot approve a request whose effective date is already in the past; HR must first set a new date (rules of BR-20) (A-34). | A-34 |

## 9. Functional requirements

| ID | Requirement | Rules |
|---|---|---|
| FR-01 | When the employee opens the transfer page, the portal shows the form with current department, location, role and manager (read-only) if the employee is eligible and has no active request; otherwise it shows the reason they cannot apply and, if there is an active request, a link to it. | BR-01, BR-02, BR-08 |
| FR-02 | The portal provides selectable lists: active departments of the employee's legal entity, active locations in the employee's country, and active roles of the selected department. | BR-05, BR-18 |
| FR-03 | The portal accepts a submission only if every rule in BR-01, BR-04 – BR-08, BR-18 passes on the server. On failure it returns every failing field with a message; nothing is saved. | BR-01, BR-04 – BR-08, BR-18 |
| FR-04 | Repeated submission of the same request (same idempotency key from the same employee) creates one request and returns that request each time. | BR-08 |
| FR-05 | On submission the portal creates the request with a unique ID, status `PENDING_CURRENT_MANAGER`, the eligibility check result and the submission time. | BR-02, BR-03 |
| FR-06 | Each approver has a work list of requests currently pending with them, and can approve or reject a request pending with them. | BR-03, BR-12 |
| FR-07 | Status moves only along the allowed transitions in Section 5.1. A decision by someone the request is not pending with, or on a request no longer at that step, is refused and changes nothing. | BR-03, BR-09, BR-28 |
| FR-08 | HR sees, for each request pending with HR, the eligibility result recorded at submission and the employee's eligibility now. | BR-02 |
| FR-09 | On HR approval the portal creates the tasks required by BR-15, sends each to its team's system with the data in BR-22, and records each task's status (`NOT_SENT`, `OPEN`, `DONE`, `CANCELLED`). | BR-15, BR-16, BR-22 |
| FR-10 | The portal accepts task status updates from team systems and shows them in the status view. | BR-15, BR-23 |
| FR-11 | On the effective date the portal sends the HR record update (new department, location, role, and receiving manager as new manager). Failures are retried; HR is alerted on failure. | BR-16 |
| FR-12 | The portal sets `COMPLETED` and notifies the employee as soon as the HR record update has succeeded and all tasks are `DONE`. | BR-16 |
| FR-13 | The employee's status view shows the content in BR-23. | BR-23 |
| FR-14 | The employee can withdraw a request in any `PENDING_*` status. | BR-09 |
| FR-15 | HR can cancel an `IN_PROGRESS` request before the HR record update, and can change its effective date. | BR-20, BR-28 |
| FR-16 | The portal sends reminders and escalations per BR-11. | BR-11 |
| FR-17 | The portal sends email and in-portal notifications: **submitted** → employee, current manager; **approved at a step** → employee, next approver; **rejected by a manager** → employee; **rejected by HR** → employee, current manager, receiving manager; **HR approved** → employee, both managers; **withdrawn / cancelled** → employee, every approver who has acted or is pending; **effective date changed** → employee, both managers; **completed** → employee, both managers. | BR-24 |
| FR-18 | The portal reacts to HR-system events for manager change and resignation per BR-14. | BR-14 |
| FR-19 | Every read and write checks that the caller is allowed per BR-21. | BR-21, BR-22 |
| FR-20 | The portal records the audit entries required by BR-25. | BR-25 |

## 10. Non-functional requirements

| ID | Requirement | Source |
|---|---|---|
| NFR-01 | **Authentication:** every endpoint requires the portal's existing login; unauthenticated calls get 401. | DEP-01 |
| NFR-02 | **Privacy:** the reason text and rejection comments are never written to application logs and never sent to downstream systems. | BR-07, BR-22 |
| NFR-03 | **Reliability:** if a team system, the HR system update or the notification service fails, the decision that triggered it is kept (never rolled back), the call is retried with increasing delay up to a configured number of attempts, and after the last attempt support is alerted and the item is shown as delayed. | TD-05, Q-25 |
| NFR-04 | **Consistency:** two decisions on the same request at the same time result in exactly one being applied; the other gets a conflict response and changes nothing. | BR-03 |
| NFR-05 | **Configurability:** date window (30/180), reason length (500), reminder and escalation days (3/5), public-holiday calendar, time zone, retry count and the Facilities-task rule are configuration values changed without code changes. | Q-05, Q-08, Q-12, TD-06 |
| NFR-06 | **Audit integrity:** audit entries are written in the same transaction as the change they record. | BR-25 |
| NFR-07 | **Response time:** opening the form, submitting and opening the status view each complete within 2 seconds at the 95th percentile with simulated downstream systems (A-43). | A-43 |

## 11. Assumptions

Discovery assumptions A-01 … A-31 are all confirmed (see [Discovery §7](01-discovery.md)). The assumptions below were made while writing this spec and **confirmed by the client on 2026-10-06** (Section 13).

| ID | Assumption | Question | Status |
|---|---|---|---|
| A-32 | The receiving manager is the head of the target department as recorded in the HR system. | Q-32 | Confirmed |
| A-33 | If the receiving manager is the same person as the current manager (e.g. location-only or role-only change in the same department), that person approves once and the request goes to HR. | Q-33 | Confirmed |
| A-34 | If the effective date has passed while the request was still waiting for approval, HR must set a new date before approving. | Q-34 | Confirmed |
| A-35 | Escalation for an overdue HR review or task goes to a configured escalation contact for each team (HR, Payroll, IT, Facilities). | Q-35 | Confirmed |
| A-36 | Working days are Monday–Friday excluding public holidays in a configured calendar. | Q-36 | Confirmed |
| A-37 | HR cannot cancel once the HR record update has succeeded; reversing a completed move is a new transfer request. | Q-37 | Confirmed |
| A-38 | Resignation while `IN_PROGRESS` does not withdraw automatically; HR is alerted and decides whether to cancel. | Q-38 | Confirmed |
| A-39 | A new effective date set by HR must not be in the past and at most 180 days from the day of the change. | Q-39 | Confirmed |
| A-40 | Downstream teams can only mark a task done; problems are sorted out with HR outside the portal. | Q-40 | Confirmed |
| A-41 | Dates are calendar dates in one organisation-wide time zone, set in configuration. | Q-41 | Confirmed |
| A-43 | 2-second, 95th-percentile response target for form, submit and status view. | Q-43 | Confirmed |
| A-44 | If the head of the target department changes while the request is pending with them, the approval moves to the new head (same as Q-15 for the current manager). | Q-44 | Confirmed |
## 12. Dependencies

| ID | Dependency | Used for | Behaviour if unavailable |
|---|---|---|---|
| DEP-01 | Portal login (SSO) | Identity and roles | Portal login handles it; feature not reachable |
| DEP-02 | HR system — employee data | Current details, employment type, probation, notice | Form not shown, submission refused with "service unavailable, try later"; nothing saved |
| DEP-03 | HR system — org structure | Departments, locations, roles, department heads | Same as DEP-02 |
| DEP-04 | HR system — record update | Apply the transfer on the effective date | Retried; HR alerted; request stays `IN_PROGRESS` (NFR-03) |
| DEP-05 | Payroll system (simulated) | Payroll task | Task `NOT_SENT`, retried (NFR-03) |
| DEP-06 | IT service system (simulated) | IT task | Same as DEP-05 |
| DEP-07 | Facilities system (simulated) | Facilities task | Same as DEP-05 |
| DEP-08 | Notification service | Email and in-portal messages | Queued and retried; never blocks a status change |
| DEP-09 | HR-system events | Manager change, resignation | Events processed when received; a delayed event delays re-routing only |
| DEP-10 | Portal delegation (if it exists) | Delegate approvals | Without it, only the named approver can act |

## 13. Questions raised while writing the spec

All answered by the client on 2026-10-06: every assumption shown was accepted. The Affects column shows what would change if an answer is revised later.

| ID | Question | Answer (client, 2026-10-06) | Affects |
|---|---|---|---|
| Q-32 | Who is the receiving manager — the head of the target department in the HR system, or someone else (e.g. the manager of the target position)? | A-32 | BR-03, AC-30, AC-31 |
| Q-33 | When the current and receiving manager are the same person, should they approve once or twice? | A-33 (once) | BR-03, AC-33 |
| Q-34 | If the effective date passes while the request is still waiting for approval, what happens? | A-34 (HR sets a new date before approving) | BR-29, AC-43 |
| Q-35 | Who receives the escalation for an overdue HR review or Payroll, IT or Facilities task? | A-35 (team's escalation contact) | BR-11, AC-69 |
| Q-36 | Which calendar defines "working days" for reminders (weekends, public holidays, which country)? | A-36 | BR-11, AC-67 – AC-70 |
| Q-37 | Can HR cancel after the HR record has been updated? | A-37 (no) | BR-28, AC-62 |
| Q-38 | If the employee resigns after HR approval, should the request be withdrawn automatically? | A-38 (no; HR decides) | BR-14, AC-75 |
| Q-39 | What limits apply when HR changes the effective date after approval? | A-39 | BR-20, AC-65, AC-66 |
| Q-40 | Can a downstream team refuse or fail a task in the portal, or only complete it? | A-40 (only complete) | BR-15, FR-09 |
| Q-41 | Which time zone defines "today" and the effective date? | A-41 (one organisation-wide zone) | BR-04, NFR-05 |
| Q-42 | Please confirm out-of-scope items OOS-05, OOS-08, OOS-10, OOS-11. | Out of scope | Section 7 |
| Q-43 | Is there a response-time target for the portal? | A-43 (2 s at 95th percentile) | NFR-07 |
| Q-44 | If the head of the target department changes while the request is waiting for them, should it move to the new head? | A-44 (yes) | BR-14, AC-73 |

## 14. Acceptance criteria

Each criterion tests one behaviour. Sample data is fictional. "Today" is the submission date. Default configuration values are used (30/180 days, 500 characters, 3/5 working days).

**Background used in examples:** employee *Asha Rao (E1001)*, permanent, not on probation, not serving notice; current department *Finance*, location *Kolkata*, role *Financial Analyst*, manager *Ravi Sen*. Target department *Operations* is headed by *Meera Das*.

### 14.1 Eligibility and form (FR-01, FR-02)

| ID | Given | When | Then | Traces to |
|---|---|---|---|---|
| AC-01 | Asha is eligible and has no active request | she opens the transfer page | the form is shown with Finance, Kolkata, Financial Analyst and Ravi Sen shown read-only | FR-01, BR-01 |
| AC-02 | Asha is on probation | she opens the transfer page | no form is shown; the message says employees on probation cannot request a transfer | FR-01, BR-01 |
| AC-03 | Asha is serving notice | she opens the transfer page | no form is shown; the message says employees serving notice cannot request a transfer | FR-01, BR-01 |
| AC-04 | Asha's employment type is contract (not permanent) | she opens the transfer page | no form is shown; the message says only permanent employees can request a transfer | FR-01, BR-01 |
| AC-05 | Asha has a request in `PENDING_HR` | she opens the transfer page | no form is shown; the message says she already has an active request and links to it | FR-01, BR-08 |
| AC-06 | The HR system is unavailable | Asha opens the transfer page | the message says the service is temporarily unavailable; no form is shown | FR-01, DEP-02 |
| AC-07 | Department *Legacy Ops* is inactive | Asha views the department list | *Legacy Ops* is not in the list | FR-02, BR-05 |
| AC-08 | Department *Group Treasury* belongs to another legal entity | Asha views the department list | *Group Treasury* is not in the list | FR-02, BR-18 |
| AC-09 | Location *Singapore* is in another country | Asha views the location list | *Singapore* is not in the list | FR-02, BR-18 |
| AC-10 | Asha selected *Operations* | she views the role list | only active roles of *Operations* are listed | FR-02, BR-05 |

### 14.2 Submission and validation (FR-03 – FR-05)

| ID | Given | When | Then | Traces to |
|---|---|---|---|---|
| AC-11 | Valid input: *Operations*, *Kolkata*, *Operations Analyst*, effective date today + 45 days, no reason | Asha submits | a request is created with a unique ID and status `PENDING_CURRENT_MANAGER`, and the eligibility result is stored on it | FR-05, BR-02 |
| AC-12 | Any one of department, location, role or effective date is missing | Asha submits | the submission is refused with an error naming the missing field; nothing is saved | FR-03 |
| AC-13 | Effective date is today + 29 days | Asha submits | refused: effective date must be at least 30 days after today | FR-03, BR-04 |
| AC-14 | Effective date is today + 30 days | Asha submits | accepted | FR-03, BR-04 |
| AC-15 | Effective date is today + 180 days | Asha submits | accepted | FR-03, BR-04 |
| AC-16 | Effective date is today + 181 days | Asha submits | refused: effective date must be at most 180 days after today | FR-03, BR-04 |
| AC-17 | Effective date is yesterday | Asha submits | refused: effective date cannot be in the past | FR-03, BR-04 |
| AC-18 | Role *Store Manager* does not belong to *Operations* | Asha submits *Operations* + *Store Manager* | refused: role does not belong to the selected department | FR-03, BR-05 |
| AC-19 | A request names an inactive department or location (sent directly to the API) | it is submitted | refused: department/location is not active | FR-03, BR-05 |
| AC-20 | A request names a department of another legal entity or a location in another country (sent directly to the API) | it is submitted | refused: not allowed for this employee | FR-03, BR-18 |
| AC-21 | Department, location and role are all equal to Asha's current values | she submits | refused: at least one of department, location or role must change | FR-03, BR-06 |
| AC-22 | Only the location differs (*Finance*, *Pune*, *Financial Analyst*) | Asha submits | accepted | FR-03, BR-06 |
| AC-23 | Reason has exactly 500 characters | Asha submits | accepted | FR-03, BR-07 |
| AC-24 | Reason has 501 characters | Asha submits | refused: reason must be at most 500 characters | FR-03, BR-07 |
| AC-25 | Asha has a request in `PENDING_CURRENT_MANAGER` | she submits another request | refused with a conflict: one active request at a time; no new request is created | FR-03, BR-08 |
| AC-26 | Asha's last request is `REJECTED` (or `WITHDRAWN`, `CANCELLED`, `COMPLETED`) | she submits a new valid request | accepted | FR-03, BR-08 |
| AC-27 | Asha became ineligible (now serving notice) after opening the form | she submits | refused: not eligible; nothing is saved | FR-03, BR-02 |
| AC-28 | Asha submits a valid request | she submits it again with the same idempotency key | no second request is created; the response returns the first request | FR-04 |
| AC-29 | The portal's database fails during submission | Asha submits | she gets an error; no request is saved and no notification is sent | FR-03, NFR-03 |

### 14.3 Approvals (FR-06 – FR-08)

| ID | Given | When | Then | Traces to |
|---|---|---|---|---|
| AC-30 | Asha's request was just submitted | Ravi Sen opens his work list | the request is in it; Ravi has received a notification | FR-06, FR-17 |
| AC-31 | Request is `PENDING_CURRENT_MANAGER` | Ravi approves | status becomes `PENDING_RECEIVING_MANAGER`; Meera Das (head of *Operations*) gets it in her work list and is notified; Asha is notified | FR-06, BR-03 |
| AC-32 | Request is `PENDING_RECEIVING_MANAGER` | Meera approves | status becomes `PENDING_HR`; HR is notified; Asha is notified | FR-06, BR-03 |
| AC-33 | Asha requests a location-only change, so the receiving manager is Ravi | Ravi approves | status becomes `PENDING_HR` directly; Ravi is not asked a second time | FR-07, BR-03 (A-33) |
| AC-34 | Request is `PENDING_CURRENT_MANAGER` | Meera tries to approve | refused: not pending with her; status unchanged | FR-07, BR-03 |
| AC-35 | Request is `PENDING_RECEIVING_MANAGER` | a colleague with no role on the request tries to approve | refused as if the request did not exist; status unchanged | FR-19, BR-21 |
| AC-36 | Any approver rejects with an empty comment | they submit the rejection | refused: comment is required; status unchanged | FR-06, BR-12 |
| AC-37 | Request is `PENDING_CURRENT_MANAGER` | Ravi rejects with comment "Critical quarter-end, revisit in Q1" | status becomes `REJECTED`; no further approval step is created; Asha is notified and sees the comment | FR-06, BR-12, BR-23 |
| AC-38 | Request is `PENDING_RECEIVING_MANAGER` | Meera rejects with a comment | status becomes `REJECTED`; Asha is notified and sees the comment | FR-06, BR-12 |
| AC-39 | Request is `PENDING_HR` | HR opens it | HR sees the eligibility result recorded at submission and Asha's eligibility now | FR-08, BR-02 |
| AC-40 | Request is `PENDING_HR`, effective date not in the past | HR approves | status becomes `IN_PROGRESS`; tasks are created (AC-44 / AC-45); Asha, Ravi and Meera are notified | FR-09, BR-16 |
| AC-41 | Request is `PENDING_HR` | HR rejects with a comment | status becomes `REJECTED`; Asha, Ravi and Meera are notified | FR-17, BR-24 |
| AC-42 | Two people act on the same request at the same moment (e.g. Ravi approves while Asha withdraws) | both actions reach the portal | exactly one is applied; the other gets a conflict response and changes nothing | NFR-04 |
| AC-43 | Request is `PENDING_HR` and the effective date is yesterday | HR approves without changing the date | refused: set a new effective date first | BR-29 (A-34) |

### 14.4 Downstream tasks, HR record update, completion (FR-09 – FR-12)

| ID | Given | When | Then | Traces to |
|---|---|---|---|---|
| AC-44 | Location changes (Kolkata → Pune) | HR approves | Payroll, IT and Facilities tasks are created and sent | FR-09, BR-15 |
| AC-45 | Location does not change | HR approves | Payroll and IT tasks are created; no Facilities task | FR-09, BR-15 |
| AC-46 | A task is sent to a team system | the payload is inspected | it contains only name, employee ID, current/new department, location, role and effective date; it does not contain the reason | FR-09, BR-22 |
| AC-47 | The IT system is unavailable | HR approves | the request is still `IN_PROGRESS` (approval not rolled back); the IT task is `NOT_SENT` and retried | FR-09, NFR-03 |
| AC-48 | The IT task has failed on every configured retry | the last retry fails | support is alerted; the status view shows the IT task as delayed | NFR-03 |
| AC-49 | The Payroll task is `OPEN` | the Payroll system reports it done | the task becomes `DONE` and the status view shows it done | FR-10 |
| AC-50 | Request is `IN_PROGRESS`, effective date is tomorrow | the daily run executes today | no HR record update is sent | FR-11, BR-16 |
| AC-51 | Request is `IN_PROGRESS`, effective date is today | the daily run executes | the HR record update is sent with *Operations*, the new location, the new role and Meera Das as manager | FR-11, BR-16 |
| AC-52 | The HR record update fails | it is retried until the last attempt | HR is alerted; the status view shows "HR record update pending"; the request is not `COMPLETED` | FR-11, NFR-03 |
| AC-53 | All tasks are `DONE` but the effective date is in the future | — | the request stays `IN_PROGRESS` | FR-12, BR-16 |
| AC-54 | HR record update has succeeded and the last open task becomes `DONE` | the update is received | status becomes `COMPLETED`; Asha, Ravi and Meera are notified | FR-12, BR-16 |

### 14.5 Status view and access (FR-13, FR-19)

| ID | Given | When | Then | Traces to |
|---|---|---|---|---|
| AC-55 | Request is `PENDING_RECEIVING_MANAGER` | Asha opens her status view | she sees status "Waiting for the receiving manager, Meera Das", and the timeline shows "Submitted" and "Approved by your manager, Ravi Sen" with dates | FR-13, BR-23 |
| AC-56 | Request is `PENDING_HR` | Asha opens her status view | the pending action is shown as "HR", without any HR person's name | FR-13, BR-23 |
| AC-57 | Request is `IN_PROGRESS` with Payroll done, IT open, Facilities open | Asha opens her status view | she sees each task with its team name and status; IT and Facilities listed as pending | FR-13, BR-23 |
| AC-58 | Another employee, Vikram, is not on Asha's request | he requests Asha's request by its ID | he gets the same "not found" response as for a non-existent ID | FR-19, BR-21 |
| AC-59 | The IT team user views its task | the task is shown | it shows only BR-22 data; the reason and other teams' tasks are not visible | FR-19, BR-22 |
| AC-60 | The request is submitted | Asha tries to change the department, date or reason | refused: a submitted request cannot be edited | BR-10 |

### 14.6 Withdraw, cancel, change date (FR-14, FR-15)

| ID | Given | When | Then | Traces to |
|---|---|---|---|---|
| AC-61 | Request is `PENDING_RECEIVING_MANAGER` | Asha withdraws | status becomes `WITHDRAWN`; it leaves Meera's work list; Ravi and Meera are notified | FR-14, BR-09 |
| AC-62 | Request is `IN_PROGRESS` and the HR record has been updated | HR tries to cancel | refused: cannot cancel after the HR record update | FR-15, BR-28 (A-37) |
| AC-63 | Request is `IN_PROGRESS` | Asha tries to withdraw | refused: contact HR to cancel after approval | FR-14, BR-09 |
| AC-64 | Request is `IN_PROGRESS`, HR record not yet updated, IT and Facilities tasks open | HR cancels | status becomes `CANCELLED`; open tasks are cancelled in team systems; the HR record update is not sent; Asha, Ravi and Meera are notified | FR-15, BR-28 |
| AC-65 | Request is `IN_PROGRESS` | HR changes the effective date to today + 60 days | the date is updated; open tasks and the scheduled HR record update use the new date; Asha, Ravi and Meera are notified | FR-15, BR-20 |
| AC-66 | Request is `IN_PROGRESS` | HR changes the effective date to yesterday | refused: date cannot be in the past | FR-15, BR-20 (A-39) |

### 14.7 Reminders, escalation, manager change, resignation (FR-16, FR-18)

| ID | Given | When | Then | Traces to |
|---|---|---|---|---|
| AC-67 | Request has been with Ravi for 2 working days | the daily reminder run executes | no reminder is sent | FR-16, BR-11 |
| AC-68 | Request has been with Ravi for 3 working days | the daily reminder run executes | Ravi gets a reminder | FR-16, BR-11 |
| AC-69 | Request has been with Ravi for 5 working days | the daily reminder run executes | the request is escalated to Ravi's manager; for an overdue task, to that team's escalation contact | FR-16, BR-11 (A-35) |
| AC-70 | Request reached Ravi on a Friday | the run executes on the following Monday | the wait is counted as 1 working day (Saturday and Sunday not counted) | FR-16, BR-11 (A-36) |
| AC-71 | Request has been with Ravi for 30 working days | the daily run executes | the request is still `PENDING_CURRENT_MANAGER` (no automatic expiry) | FR-16, BR-11 |
| AC-72 | Request is `PENDING_CURRENT_MANAGER`; HR system reports Asha's manager changed from Ravi to Kiran | the event is processed | the request moves to Kiran's work list; Kiran is notified; Ravi can no longer act on it | FR-18, BR-14 |
| AC-73 | Request is `PENDING_RECEIVING_MANAGER`; head of *Operations* changes from Meera to Arjun | the event is processed | the request moves to Arjun's work list; Arjun is notified | FR-18, BR-14 (A-44) |
| AC-74 | Request is `PENDING_HR`; HR system reports Asha resigned | the event is processed | status becomes `WITHDRAWN`; Ravi, Meera and HR are notified | FR-18, BR-14 |
| AC-75 | Request is `IN_PROGRESS`; HR system reports Asha resigned | the event is processed | status stays `IN_PROGRESS`; HR is alerted to decide on cancellation | FR-18, BR-14 (A-38) |

### 14.8 Notifications, audit, configuration (FR-17, FR-20, NFR)

| ID | Given | When | Then | Traces to |
|---|---|---|---|---|
| AC-76 | Asha submits a valid request | the request is saved | Asha and Ravi each receive an email and an in-portal notification | FR-17, BR-24 |
| AC-77 | The notification service is down | Ravi approves | the approval is saved and the status changes; the notification is queued and sent when the service recovers | FR-17, NFR-03 |
| AC-78 | Any status change or task status change happens | the audit trail is read | it has an entry with actor, timestamp, from-status and to-status | FR-20, BR-25 |
| AC-79 | Asha's request includes a reason | application logs and audit entries for the request are read | the reason text does not appear | NFR-02 |
| AC-80 | An unauthenticated caller | calls any transfer endpoint | gets 401; no data is returned | NFR-01 |
| AC-81 | Configuration sets the minimum notice to 45 days | Asha submits with effective date today + 40 days | refused: at least 45 days; no code change was needed | NFR-05 |

---

## Coverage check

| Requirement | Acceptance criteria |
|---|---|
| FR-01 | AC-01 – AC-06 |
| FR-02 | AC-07 – AC-10 |
| FR-03 | AC-12 – AC-27, AC-29 |
| FR-04 | AC-28 |
| FR-05 | AC-11 |
| FR-06 | AC-30 – AC-32, AC-36 – AC-38 |
| FR-07 | AC-33, AC-34 |
| FR-08 | AC-39 |
| FR-09 | AC-40, AC-44 – AC-47 |
| FR-10 | AC-49 |
| FR-11 | AC-50 – AC-52 |
| FR-12 | AC-53, AC-54 |
| FR-13 | AC-55 – AC-57 |
| FR-14 | AC-61, AC-63 |
| FR-15 | AC-62, AC-64 – AC-66 |
| FR-16 | AC-67 – AC-71 |
| FR-17 | AC-30, AC-41, AC-76, AC-77 |
| FR-18 | AC-72 – AC-75 |
| FR-19 | AC-35, AC-58, AC-59 |
| FR-20 | AC-78 |
| NFR-01 | AC-80 |
| NFR-02 | AC-79 |
| NFR-03 | AC-29, AC-47, AC-48, AC-52, AC-77 |
| NFR-04 | AC-42 |
| NFR-05 | AC-81 |
| NFR-06 | Verified by design review and an integration test at Gate 2 |
| NFR-07 | Verified by a timing test at Gate 2 |

Every FR has at least one AC. API contracts (endpoints, payloads, status codes) come next in the API contract, mapped to these AC.
