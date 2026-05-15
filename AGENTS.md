# AGENTS.md — CodeCoachAI Documentation

## Project identity

This repository is the documentation repository and durable project memory center of CodeCoachAI, an AI interview training platform.

Related repositories:
- Frontend: https://github.com/wbn-2020/codecoch-ai-vue.git
- Backend: https://github.com/wbn-2020/codecoch-ai-java.git
- Docs: https://github.com/wbn-2020/codecoch-ai-doc.git

## Current phase

- V1 has been completed.
- CodeCoachAI is moving toward the official V2 PRD implementation.
- The current implementation already contains a group of V2 technical baseline and AI interview enhancement prerequisites:
  - service-side ADMIN secondary checks for `/admin/**`;
  - HMAC signature protection for `/inner/**` internal calls;
  - AI call stability improvements;
  - resume project deep-dive enhancement;
  - early learning feedback in interview reports.
- These completed items are not the full V2 PRD.

Do not describe "security baseline + AI stability + project deep dive + learning feedback prototype" as complete V2.

## V2 source of truth

The only authoritative source for V2 scope is:

`PRD/CodeCoachAI_PRD_V2_AI能力增强版.md`

If any of the following conflict with the official V2 PRD, the official V2 PRD wins:

1. `V2_ROADMAP_DRAFT.md`
2. `PROJECT_STATE.md`
3. `AGENTS.md`
4. `NEW_CODEX_SESSION_PROMPT.md`
5. old chat history
6. Codex historical output
7. previous temporary roadmaps

`V2_ROADMAP_DRAFT.md` is only valid when it is aligned with the official V2 PRD.

## Documentation role

Keep these documents current:
- `PROJECT_STATE.md` — current phase, completed baseline, open V2 gaps, execution order
- `PRD/CodeCoachAI_PRD_V2_AI能力增强版.md` — official V2 scope
- `CodeCoachAI_V2_PRD差距清单.md` — V2 PRD gap list
- `CodeCoachAI_V2_后端开发路线.md` — backend-first implementation route
- `V2_ROADMAP_DRAFT.md` — V2 roadmap aligned with the official PRD
- API contract docs — actual backend/frontend interface agreement
- `TASK_LOG.md` — major task history and decisions
- `NEW_CODEX_SESSION_PROMPT.md` — prompt to start a clean Codex session

## V2 development rules

Every V2 development task must state:

1. corresponding PRD section;
2. priority: P0 / P1 / P2;
3. affected backend services;
4. affected database tables;
5. affected interfaces;
6. whether frontend is affected.

Current execution strategy:

1. complete the V2 backend first;
2. then complete the V2 frontend;
3. finally run full integration and E2E testing.

## Non-negotiable rules

1. Do not write V2 docs based on old roadmap assumptions.
2. Do not rely on old chat history as V2 scope evidence.
3. Do not narrow or expand V2 scope without explicitly comparing against the official V2 PRD.
4. If code and docs conflict, mark the conflict and ask whether to update docs or code.
5. Keep docs concise enough for Codex to read in new sessions.

## Required response format for Codex

When completing a documentation or planning task, report:

1. documents modified;
2. source of truth used;
3. important corrections made;
4. remaining outdated areas;
5. next task.
