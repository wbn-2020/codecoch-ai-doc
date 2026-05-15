# PROJECT_STATE.md — CodeCoachAI Current State

Last updated: 2026-05-15

## 1. Project overview

CodeCoachAI is a Java + Vue AI interview training platform.

The project is designed as a portfolio-grade full-stack AI application for a Java developer. It should demonstrate:
- Java backend architecture;
- Vue frontend engineering;
- microservice design;
- API contract design;
- AI application integration;
- admin/user-side business flows;
- data modeling and operational logging.

Repositories:
- Frontend: https://github.com/wbn-2020/codecoch-ai-vue.git
- Backend: https://github.com/wbn-2020/codecoch-ai-java.git
- Docs: https://github.com/wbn-2020/codecoch-ai-doc.git

## 2. Current phase

Current status:

1. V1 has been completed.
2. CodeCoachAI is now preparing for the full official V2 PRD implementation.
3. The official and only V2 scope source is `PRD/CodeCoachAI_PRD_V2_AI能力增强版.md`.
4. Existing project memory and old Codex outputs must be corrected when they conflict with the official V2 PRD.

The following items have already been completed and should be classified as V2 technical baseline and AI interview enhancement prerequisites:

- service-side ADMIN secondary checks for `/admin/**`;
- HMAC signature protection for `/inner/**` internal calls;
- AI call stability enhancements;
- resume project deep-dive capability enhancement;
- early learning feedback in interview reports.

These items are not the full V2 PRD. They are prerequisites and partial enhancements that support later V2 delivery.

## 3. V2 source of truth

The only authoritative V2 scope source is:

`PRD/CodeCoachAI_PRD_V2_AI能力增强版.md`

If the following content conflicts with the official V2 PRD, the official V2 PRD wins:

1. `V2_ROADMAP_DRAFT.md`
2. `PROJECT_STATE.md`
3. `AGENTS.md`
4. `NEW_CODEX_SESSION_PROMPT.md`
5. old chat history
6. Codex historical output
7. previous temporary roadmaps

## 4. V1 completed scope

V1 includes the completed core closed loop:

### User side
- register/login/logout;
- user profile;
- question browsing;
- question answering;
- favorites and wrong records;
- manual resume/project experience management;
- interview creation and interview room;
- answer submission;
- AI scoring and follow-up questions;
- interview ending, report, and history.

### Admin side
- dashboard;
- user/role management;
- question management;
- category/tag/group management;
- prompt template management;
- AI call log management;
- system configuration.

### Backend capabilities
- gateway routing;
- auth/token handling;
- user service;
- question service;
- resume service;
- interview service;
- AI service;
- system/admin service;
- MySQL persistence;
- Redis/token support;
- Nacos service registration.

## 5. Official V2 gap summary

The core product capabilities in the official V2 PRD have not been fully implemented yet.

The backend main line must continue according to the official PRD, including:

1. `codecoach-file` and resume upload;
2. resume parse records and parse status flow;
3. AI resume structured parsing;
4. AI resume optimization;
5. industry templates and industry scenario interviews;
6. AI question generation and review;
7. initial question-bank deduplication;
8. formal learning plan module;
9. Prompt template version management;
10. SSE streaming output;
11. final backend static closure.

For details, use:
- `CodeCoachAI_V2_PRD差距清单.md`
- `CodeCoachAI_V2_后端开发路线.md`
- `V2_ROADMAP_DRAFT.md`

## 6. V2 execution route

The V2 execution route is:

1. complete the V2 backend first;
2. then complete the V2 frontend;
3. finally run full integration and E2E testing.

Do not start frontend-wide V2 adaptation before backend contracts are stable, except for small contract confirmation or prototype checks explicitly requested by the user.

## 7. Development rules for new Codex sessions

Every new Codex session for V2 must first read:

1. root `AGENTS.md`;
2. this `PROJECT_STATE.md`;
3. `PRD/CodeCoachAI_PRD_V2_AI能力增强版.md`;
4. `CodeCoachAI_V2_PRD差距清单.md`;
5. `CodeCoachAI_V2_后端开发路线.md`.

Do not rely on old chat history.
The markdown files in this docs repo are the durable project memory.
