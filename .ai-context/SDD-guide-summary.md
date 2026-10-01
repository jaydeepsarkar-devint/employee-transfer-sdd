# SDD Participant Guide — Working Summary

> Condensed from "SDD_Participant_Guide For Internal Evaluation.pdf". Used as the checklist for every artefact in `specs/internal-transfer/`.

## The chain
Business requirement → Business journey → Questions/ambiguities → Business decisions → Feature spec → Acceptance criteria → API contract → Test cases → **GATE 1** → Technical plan → Tasks → Implementation (test-first) → Testing & validation → **GATE 2** → Traceability & evidence.

No significant coding before Gate 1 is passed.

## ID conventions (must be consistent across every document)
| Prefix | Meaning |
|---|---|
| Q-xx | Open question / ambiguity |
| A-xx | Assumption (labelled "assumed, not confirmed" until confirmed) |
| BD-xx | Business decision (owner + Confirmed/Assumed status) |
| TD-xx | Technical decision (reason + rule it supports) |
| BR-xx | Business rule |
| FR-xx / NFR-xx | Functional / non-functional requirement |
| AC-xx | Acceptance criterion (Given/When/Then, one behaviour each) |
| API-xx | API contract |
| TEST-xx | Test case (positive / negative / boundary / failure / security) |
| TASK-xx | Task (objective, input/output, depends on, validation, done-when) |
| FS-xx | Failure scenario |
| SEC-xx | Security risk/control |
| DEP-xx | Dependency |
| DEV-xx | Deviation from spec (what, why, approved by) |
| AI-xx | AI usage log entry |

## Per-expectation evidence
1. **Journey & gaps** — journey table (happy + failure paths), question log (owner, impact, status, working assumption), assumptions list.
2. **Business vs technical** — two separate lists; business = owner + Confirmed/Assumed; technical = "chosen because…" + rule supported. Never hide an unanswered business question inside a technical choice (make it configurable instead).
3. **Spec** — 14 sections: name, objective, context, persona, journey, scope, out of scope, business rules, FRs, NFRs, assumptions, dependencies, open questions, AC. No vague words.
4. **API contracts + tests** — endpoint, method, auth, request, required/optional, types, validation, response, status codes, error shape, examples. Tests carry a "Comes from" column.
5. **Plan + tasks** — components, data, dependencies, order, risks; tasks 1–2 days, independently verifiable.
6. **Test-first** — RED run recorded, then GREEN; commit history shows tests before/with code.
7. **Security, integration, failure** — risk/control/test table; dependency failure behaviour; failure scenarios with tests.
8. **AI usage** — log: stage, prompt, what AI returned, decision, why; ≥1 rejected/corrected output; weak → better prompt examples.
9. **Traceability** — matrix BR→FR→AC→API→TASK→TEST→Result; at Gate 1 (to test design) and Gate 2 (with results); gaps noted and fixed.

## Evaluation weights (from BRD)
Journey 10 · Ambiguity & discovery 15 · Spec quality 20 · AC & testability 15 · Task decomposition 10 · Test-first 5 · Security & failure 5 · Traceability 20.

## Common mistakes to avoid
Happy-path only · assumptions treated as facts · business/technical mixed · vague AC · API not checked against business rules · large unverifiable tasks · undocumented AI use · matrix built at the end · silent spec changes · Gate 1 as a presentation · Gate 2 as only a demo.
