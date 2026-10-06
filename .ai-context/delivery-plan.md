# Delivery Plan — Employee Internal Transfer

> Our delivery plan for the client's Internal Transfer requirement ([BRD](BRD.md)), following the SDD chain. Progress is tracked in [PROGRESS.md](PROGRESS.md).

## SDD chain

Business Requirement → Spec → Gate 1 → Plan → Tasks → Test First → Implementation → Gate 2 → Release

## Deliverables

| # | Deliverable | Content |
|---|---|---|
| 1 | Requirement / Discovery Analysis | Business objective, primary users, journey stages, business rules, known decisions, open questions, assumptions, dependencies, out-of-scope items. Business decisions kept separate from technical decisions. |
| 2 | Feature Specification + Acceptance Criteria | `.spec.md` with individually identifiable acceptance criteria |
| 3 | Spec-Derived Test Cases | |
| 4 | Technical Plan | |
| 5 | Task Decomposition | |
| 7 | AI Prompts | |
| 8 | Security Assessment | |
| 9 | Gate 1 Review | |
| 10 | Gate 2 Evidence | |

(There is no Deliverable 6.)

## Timeline

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
| Day 9 | Validation + Security + Gate 2 preparation | GREEN tests, security review, traceability | 4-5 hrs |
| Day 10 | Gate 2 + Final Presentation | Final artefact chain + demo + 10-15 min walkthrough | 2.5-3 hrs |

## Milestones

- **Milestone 1 — Discovery & Specification.** No coding yet. Output: BRD → Spec → AC → API Contract → Test Cases.
- **Milestone 2 — Gate 1.** A real peer review of the spec, not a presentation.
- **Milestone 3 — Plan → Tasks.**
- **Milestone 4 — Implementation & Gate 2.** Task → Prompt → Test (RED) → Implementation → Test (GREEN) → Review.

Estimated effort: 30-35 hours.
