# Deliverable 9 — Gate 1 Review

**Gate 1 status:** ⏳ PENDING REVIEW
<!-- Allowed values: ⏳ PENDING REVIEW · 🔁 CHANGES REQUESTED · ✅ PASSED -->

| Field | Value |
|---|---|
| Artefacts under review | [Discovery v0.3](01-discovery.md) · [Spec v0.2](internal-transfer.spec.md) · [DB design v0.1](../../db-design/00-schema-design.md) |
| Author | Jaydeep Sarkar |
| Reviewer | _name_ |
| Review requested | 2026-10-06 |
| Review completed | _date_ |
| Pull request | _link_ |

Rule: no implementation code is written until this file says **✅ PASSED** and the reviewer has approved the pull request.

## 1. Checklist (reviewer fills)

| # | Check | Result (Yes / No / N/A) | Note |
|---|---|---|---|
| C-01 | Business objective and journey are clear and match the BRD | | |
| C-02 | Every open question has an answer with who and when; no assumption is presented as fact | | |
| C-03 | Business decisions are kept separate from technical decisions | | |
| C-04 | Every business rule (BR) has a source (BRD section or answered question) | | |
| C-05 | Requirements and acceptance criteria use no vague words (e.g. "fast", "appropriate", "user-friendly") | | |
| C-06 | Every FR / NFR has at least one acceptance criterion (spec coverage table) | | |
| C-07 | Each acceptance criterion tests one behaviour and has exact values (boundaries 29/30/180/181 days, 500/501 characters) | | |
| C-08 | Failure, security and permission cases are covered, not only the happy path | | |
| C-09 | Status model and allowed transitions are complete; illegal moves are refused | | |
| C-10 | Out-of-scope items are listed and confirmed | | |
| C-11 | DB design supports the business rules (one active request, audit cannot change, rejection comment) | | |
| C-12 | API contract matches the business rules and acceptance criteria | | |
| C-13 | Test cases trace to acceptance criteria ("Comes from" column) | | |
| C-14 | Traceability matrix BR → FR → AC → API → TEST exists | | |

## 2. Review comments

| ID | Location (file § / ID) | Comment | Severity (Blocker / Major / Minor) | Author response | Change made (file, version) | Status (Open / Resolved / Rejected with reason) |
|---|---|---|---|---|---|---|
| RC-01 | | | | | | |

## 3. Decision

| Field | Value |
|---|---|
| Decision | _PASSED / CHANGES REQUESTED_ |
| Conditions (if any) | |
| Approved spec version | _e.g. Spec v0.3_ |
| Reviewer sign-off | _name, date_ |

After **PASSED**: the spec is frozen at the approved version. Any later change is logged in `deviation-log.md` (DEV-xx) with what changed, why and who approved it.
