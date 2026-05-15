# CONTEXT_UPDATE_RULES.md — How to Keep Codex Memory Current

## Purpose

This file defines how to keep project memory usable across new Codex sessions.

## Rule 1 — AGENTS.md is for stable rules

Put stable instructions in `AGENTS.md`, such as:
- Project identity
- Tech stack
- Build commands
- Coding rules
- Security rules
- API contract rules
- Required response format

Do not put long daily task history in `AGENTS.md`.

## Rule 2 — PROJECT_STATE.md is for current project status

Update `PROJECT_STATE.md` when:
- A version is completed
- A new phase starts
- V1/V2 scope changes
- A major decision is made
- A major risk is discovered or resolved

## Rule 3 — TASK_LOG.md is for chronological history

Update `TASK_LOG.md` after each major milestone:
- What was done
- What was decided
- What remains
- Where the code/docs changed

## Rule 4 — Release notes are for version closure

Use `V1_RELEASE_NOTES_DRAFT.md` to collect:
- Final V1 scope
- Completed features
- Acceptance checklist
- Known limitations
- Release decision

After release, rename or copy to:
- `V1_RELEASE_NOTES.md`

## Rule 5 — Roadmap is for future work

Use `V2_ROADMAP_DRAFT.md` for:
- V2 feature candidates
- Priority
- Dependencies
- Exclusions
- Implementation phases

Do not mix V2 plans into V1 release docs.

## Rule 6 — New session startup process

At the start of every new Codex session, ask Codex to read:
1. `AGENTS.md`
2. `PROJECT_STATE.md`
3. Relevant release notes
4. Relevant roadmap
5. Actual source code

Then ask for an analysis first.
Do not ask it to modify code immediately.

## Rule 7 — Keep docs short enough to be read

If a file becomes too long:
- Keep a short summary at the top.
- Move details into a dedicated document.
- Add links/references from the summary file.
