# Deliverable 7 — AI Prompts & Usage Log

AI tool: Claude Code (Claude Opus 5.5). No confidential data, passwords or real employee data were shared with the tool; all sample data is fictional.

## Usage log

| ID | Date | Stage | Prompt (summary) | What AI returned | My decision | Why |
|---|---|---|---|---|---|---|
| AI-01 | 2026-10-01 | Discovery | Store the BRD as context. Act as senior developer; read BRD + SDD guide; produce Deliverable 1 (objective, users, journey stages, rules, known decisions, open questions, assumptions, dependencies, out of scope; business vs technical split). Do not write code. | `01-discovery.md`: 8 user types, 11 journey stages with failure paths, 31 questions (prioritised, owner, assumption), 14 BD / 11 TD, 9 dependencies, 11 OOS items | _Pending my review_ | — |
| AI-02 | 2026-10-06 | Discovery | Draft the client questions (Q-01 … Q-31) as a message I can send, each with a proposed answer; then record the client's replies in the discovery document. | Client-ready question list; discovery updated: 29 questions Answered, then Q-12 and Q-25 on follow-up (all 31 answered); assumptions, rules and business decisions marked Confirmed; journey stages corrected (no send-back, no headcount check, withdraw only before HR approval) | _Pending my review_ | — |
| AI-03 | 2026-10-06 | Specification | Write Deliverable 2 from the confirmed discovery: the 14 spec sections, numbered BR/FR/NFR, Given/When/Then AC (one behaviour each, boundary values), and flag anything still ambiguous instead of guessing. | `internal-transfer.spec.md` v0.1: 29 BR, 20 FR, 7 NFR, 81 AC, coverage table; simplified status model (dropped SUBMITTED/APPROVED); 13 new questions Q-32 – Q-44 with assumptions A-32 – A-44 | _Pending my review_ | — |
| AI-04 | 2026-10-06 | Technical design | Change stack to Express + TypeScript backend, Next.js + TypeScript frontend, SQLite; design the DB schema (tables and relationships) in a separate db-design folder. | TD-01/02/10/11 updated; db-design/00-schema-design.md (13 tables, constraints, DDL) and 01-relationships-erd.md (Mermaid ERD, 25 relationships, table changes per journey step); DDL run on SQLite with 14 constraint checks passing | _Pending my review_ | — |

## Prompt improvement examples

| Weak prompt | Better, task-oriented prompt |
|---|---|
| "Make the employee transfer portal." | "Here is the BRD for the internal transfer journey. List every question a developer needs answered before building it, grouped by eligibility, workflow, downstream orchestration and privacy. For each give why it matters, who should answer and a working assumption. Do not propose implementation. Return a table." |
