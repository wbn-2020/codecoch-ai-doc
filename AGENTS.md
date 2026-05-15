# AGENTS.md — CodeCoachAI Documentation

## Project identity

This repository is the documentation repository of CodeCoachAI, an AI interview training platform.

Current project phase:
- V1 code is considered functionally complete.
- Documentation is not yet synchronized with the final V1 code.
- The immediate priority is V1 documentation closure before V2 development.
- After V1 documentation is frozen, use this repository as the durable project memory for future Codex sessions.

Related repositories:
- Frontend: https://github.com/wbn-2020/codecoch-ai-vue.git
- Backend: https://github.com/wbn-2020/codecoch-ai-java.git
- Docs: https://github.com/wbn-2020/codecoch-ai-doc.git

## Documentation role

This repo is the project memory center.

Keep these documents current:
- `PROJECT_STATE.md` — current phase, completed work, open risks
- `V1_RELEASE_NOTES_DRAFT.md` — final V1 summary
- `V2_ROADMAP_DRAFT.md` — V2 planning
- API contract docs — actual backend/frontend interface agreement
- `TASK_LOG.md` — major task history and decisions
- `NEW_CODEX_SESSION_PROMPT.md` — prompt to start a clean Codex session

## V1 documentation closure checklist

Before declaring V1 final, confirm and document:

1. V1 scope
2. Final frontend route list
3. Final backend service/module list
4. Final API contract
5. Database table list
6. User-side flow
7. Admin-side flow
8. AI mock mode vs real model mode
9. Known limitations
10. V1 acceptance test checklist
11. V2 backlog

## Non-negotiable rules

1. Do not write docs based only on old PRD assumptions. Align with actual code.
2. If code and docs conflict, mark the conflict and ask the user whether to update docs or code.
3. Use "V1 completed", "V1 pending verification", and "V2 planned" labels clearly.
4. Keep docs concise enough for Codex to read in new sessions.
5. When AGENTS.md becomes too long, keep this file short and reference dedicated markdown files.

## Required response format for Codex

When completing a documentation task, report:

1. Documents modified
2. Source of truth used
3. Important corrections made
4. Remaining outdated areas
5. Next documentation task
