# BRD — Employee Internal Transfer Digital Journey

> Source: assignment brief, stored verbatim (HTML entities cleaned). This is the business requirement input for the SDD chain. Do not edit; record interpretations in `docs/`.

## 1. Objective

This exercise evaluates your ability to apply INT's Specification-Driven Development/Delivery (SDD) methodology to an enterprise employee digital journey.

The focus is not only on implementation.

You are expected to demonstrate how you:

- Understand a business journey.
- Identify ambiguity and missing decisions.
- Convert business requirements into a precise specification.
- Define testable acceptance criteria.
- Define API contracts.
- Design the technical approach.
- Decompose the work into independently verifiable tasks.
- Apply test-first development.
- Handle integration, security and failure scenarios.
- Maintain traceability from requirement to implementation.

The expected SDD chain is:

Business Requirement → Spec → Gate 1 → Plan → Tasks → Test First → Implementation → Gate 2 → Release

## 2. Business Context

The organisation has a One-Point Employee Portal through which employees can access HR, payroll, IT, learning, facilities and other employee services.

Currently, an employee requesting an internal transfer needs to interact with multiple teams and systems.

For example:

1. Employee discusses the transfer with the manager.
2. Manager confirms the transfer.
3. HR validates eligibility.
4. Employee's organisational information is updated.
5. Payroll may need to be updated.
6. IT may need to provision or remove access.
7. Facilities may need to arrange the employee's new location.
8. Employee receives confirmation.

The organisation wants to provide a single digital journey through the One-Point Employee Portal.

## 3. Business Requirement

An employee should be able to initiate an Internal Transfer Request from the One-Point Employee Portal.

The employee should be able to:

- Select the proposed new department/business unit.
- Select the proposed new location.
- Select the proposed role/job position.
- Provide an effective date.
- Provide an optional reason for the transfer.
- Submit the request.
- View the current status of the request.
- View actions that are pending with other stakeholders.

The organisation wants the portal to orchestrate the downstream activities and provide the employee with a single view of progress.

## 4. Developer Assignment

Using the INT SDD methodology, produce the following artefacts.

- **Deliverable 1 — Requirement / Discovery Analysis.** Identify: business objective, primary users, journey stages, business rules, known decisions, open questions, assumptions, dependencies, out-of-scope items. Clearly distinguish: Business decision vs Technical decision.
- **Deliverable 2 — Feature Specification** in .md file and Acceptance Criteria [Define individually identifiable acceptance criteria]
- **Deliverable 3 — Spec-Derived Test Cases**
- **Deliverable 4 — Technical Plan**
- **Deliverable 5 — Task Decomposition**
- **Deliverable 7 — AI Prompts**
- **Deliverable 8 — Security Assessment**
- **Deliverable 9 — Gate 1 Review**
- **Deliverable 10 — Gate 2 Evidence**

(Note: the brief has no Deliverable 6.)

## 5. Evaluation Criteria

| Area | Weight |
|---|---|
| Business journey understanding | 10% |
| Ambiguity & discovery | 15% |
| Specification quality | 20% |
| Acceptance criteria & testability | 15% |
| Task decomposition | 10% |
| Test-first approach | 5% |
| Security & failure handling | 5% |
| SDD traceability | 20% |
| **Total** | **100%** |

## 6. Recommended 10-day timeline

| Day | Activity | Expected Output | Approx. Effort |
|---|---|---|---|
| Day 1 | Understand requirement + Discovery | Business understanding, questions, assumptions, open decisions | 2.5-3 hrs |
| Day 2 | BRD interpretation + SDD Spec | BRD interpretation + initial .spec.md | 3-4 hrs |
| Day 3 | Acceptance Criteria + API Contract + Test Cases | Complete specification + AC + API + spec-derived tests | 3-4 hrs |
| Day 4 | Gate 1 Peer Review | Review comments + revised/approved spec | 1.5-2 hrs |
| Day 5 | Technical Plan + Architecture | .plan.md, integration approach, data model, failure handling, ADR candidates | 3-4 hrs |
| Day 6 | Task Decomposition + AI Prompts | .tasks.md, task-to-AC mapping, implementation prompts | 2.5-3 hrs |
| Day 7 | Test-first implementation | RED tests + first implementation increment | 4-5 hrs |
| Day 8 | Guided implementation | Remaining core implementation + integration | 4-5 hrs |
| Day 9 | Validation + Security + Gate 2 preparation | GREEN tests, security review, traceability | 4–5 hrs |
| Day 10 | Gate 2 + Final Presentation | Final artefact chain + demo + 10–15 min walkthrough | 2.5-3 hrs |

## 7. Milestones

Assessment is segregated into 4 major milestones:

- **Milestone 1 — Discovery & Specification: Days 1–2.** The developer should demonstrate that they don't start coding immediately. By the end of Day 3, they should have: BRD → Spec → AC → API Contract → Test Cases
- **Milestone 2 — Gate 1: Day 3.** This should be a real peer review, not merely a presentation.
- **Milestone 3 — Plan → Tasks: Days 4-5**
- **Milestone 4 — Implementation & Gate 2: Days 6-8.** We would require: Task → Prompt → Test (RED) → Implementation → Test (GREEN) → Review

Effort expectation of approximately 30-35 hours total, spread over 8 working days.
