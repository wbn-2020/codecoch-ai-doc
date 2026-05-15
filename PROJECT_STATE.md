# PROJECT_STATE.md — CodeCoachAI Current State

Last updated: 2026-05-15

## 1. Project overview

CodeCoachAI is a Java + Vue AI interview training platform.

The project is designed as a portfolio-grade full-stack AI application for a Java developer. It should demonstrate:
- Java backend architecture
- Vue frontend engineering
- Microservice design
- API contract design
- AI application integration
- Admin/user-side business flows
- Data modeling and operational logging

Repositories:
- Frontend: https://github.com/wbn-2020/codecoch-ai-vue.git
- Backend: https://github.com/wbn-2020/codecoch-ai-java.git
- Docs: https://github.com/wbn-2020/codecoch-ai-doc.git

## 2. Current phase

Current status:
- V1 code: functionally completed by the user.
- V1 docs: not yet updated.
- Next goal: close V1 documentation and prepare V2 planning.
- Do not start V2 coding before V1 documentation is aligned with the final code.

Recommended label:
- `V1.0-RC` before documentation closure.
- `V1.0` only after docs, acceptance checklist, and release notes are complete.

## 3. V1 core scope

V1 includes:

### User side
- Register/login/logout
- User profile
- Question browsing
- Question answering
- Favorites
- Wrong records
- Resume management
- Project experience management
- Interview creation
- Interview room
- Answer submission
- AI scoring
- AI follow-up question
- Interview ending
- Interview report
- Interview history

### Admin side
- Dashboard
- User/role management
- Question management
- Category management
- Tag management
- Group management
- Prompt template management
- AI call log management
- System configuration

### Backend capabilities
- Gateway routing
- Auth/token handling
- User service
- Question service
- Resume service
- Interview service
- AI service
- System/admin service
- MySQL persistence
- Redis/token support
- Nacos service registration
- API docs

## 4. V1 exclusions

Do not treat the following as V1 requirements:
- File upload / MinIO
- Resume parsing from PDF/Word
- MQ-based async workflow
- Elasticsearch
- Vector search / embedding
- Voice interview
- Learning plan generation
- AI-generated question-bank expansion
- Advanced analytics dashboard
- Payment/subscription
- Enterprise/team features

## 5. Known V1 closure tasks

Before V1 final tag:

1. Update documentation to match final code.
2. Freeze actual API contracts.
3. Confirm no old API paths remain in frontend Network requests.
4. Confirm normal user cannot access admin APIs/pages.
5. Confirm users cannot access other users' private data.
6. Confirm mock AI mode and real AI mode are both documented.
7. Confirm frontend timeout strategy supports real AI calls.
8. Produce V1 acceptance report.
9. Create V1 release notes.
10. Tag the final V1 version after verification.

## 6. Development rules for new Codex sessions

Every new Codex session must first read:
1. Root `AGENTS.md`
2. This `PROJECT_STATE.md`
3. `V1_RELEASE_NOTES_DRAFT.md`
4. `V2_ROADMAP_DRAFT.md`
5. The relevant repo code before making changes

Do not rely on old chat history.
The markdown files in this docs repo are the durable project memory.
