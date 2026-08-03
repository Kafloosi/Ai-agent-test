# Working in this repository

This repo specifies a multi-agent system whose core rule is that agents emit artifacts, not
narration. **That rule applies to work done in this repo, including yours.**

## Output discipline

- Emit the artifact. No preamble, no commentary between tool calls, no summary of a diff the
  reader can see, no listing options not taken.
- When reporting to the user: lead with the outcome in one sentence, then only what changes
  a decision. No recap of files touched or steps performed — the diff is the record.
- Free text only where another stage consumes it: a root cause, a failure scenario, a
  clearance condition, a stated assumption. Those are evidence, not narration, and they stay.
- Say a thing once. If it is in a doc, link it; do not restate it in chat.

## Token economy

- Read the specific part of a file you need, not the whole file.
- Prefer targeted edits over rewriting a file.
- Do not re-read a file you just wrote to confirm the write.
- Batch independent tool calls into one turn.
- Do not re-derive facts already established in this session.
- Do not spawn subagents for work you can finish in a few tool calls.

## Repository conventions

- All diagrams are mermaid, fenced as ```mermaid, and must parse.
- All schemas are JSON Schema draft 2020-12 and must load.
- Validate both before committing: mermaid parse + `json.load` over `schemas/*.json`.
- Cross-reference sections as `§N.N`; keep the numbering in `README.md`'s doc map current.
- Design docs state the rule, the rationale in one line, and the bound. Not the essay.
