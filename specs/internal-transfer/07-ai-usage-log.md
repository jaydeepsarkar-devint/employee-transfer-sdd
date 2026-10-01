# Deliverable 7 — AI Prompts & Usage Log

AI tool: Claude Code (Claude Opus 5.5). No confidential data, passwords or real employee data were shared with the tool; all sample data is fictional.

## Usage log

| ID | Date | Stage | Prompt (summary) | What AI returned | My decision | Why |
|---|---|---|---|---|---|---|
| AI-01 | 2026-10-01 | Discovery | Store the BRD as context. Act as senior developer; read BRD + SDD guide; produce Deliverable 1 (objective, users, journey stages, rules, known decisions, open questions, assumptions, dependencies, out of scope; business vs technical split). Do not write code. | `01-discovery.md`: 8 user types, 11 journey stages with failure paths, 31 questions (prioritised, owner, assumption), 14 BD / 11 TD, 9 dependencies, 11 OOS items | _Pending my review_ | — |

## Prompt improvement examples

| Weak prompt | Better, task-oriented prompt |
|---|---|
| "Make the employee transfer portal." | "Here is the BRD for the internal transfer journey. List every question a developer needs answered before building it, grouped by eligibility, workflow, downstream orchestration and privacy. For each give why it matters, who should answer and a working assumption. Do not propose implementation. Return a table." |
