# TASK_LOG.md — CodeCoachAI Task Log

## 2026-05-15

### Status

V1 code is considered complete by the user.  
V1 documentation has not been updated yet.  
The user plans to continue with V2 after V1 documentation closure.

### Main decision

Use repository markdown files as durable project memory for future Codex sessions.

### Files to add

Recommended in each repository:
- `AGENTS.md`

Recommended in docs repository:
- `PROJECT_STATE.md`
- `V1_RELEASE_NOTES_DRAFT.md`
- `V2_ROADMAP_DRAFT.md`
- `NEW_CODEX_SESSION_PROMPT.md`
- `TASK_LOG.md`
- `CONTEXT_UPDATE_RULES.md`

### Reason

The existing Codex conversations for Java and Vue have become too long.  
A fresh session may lose detailed context from previous chats.  
Codex should be re-grounded from repository files instead of old chat history.

### Next action

1. Add the AGENTS.md files to each repo root.
2. Add project state and release docs to the docs repo.
3. Open a new Codex session.
4. Ask Codex to read these files first.
5. Let Codex synchronize V1 documentation before starting V2.

## 2026-05-15 — V2 roadmap correction

### Status

V1 has been completed.

During V2 planning review, the previous V2 development route was found to be based on a narrowed temporary roadmap and was not fully aligned with the official V2 PRD.

### Main decision

Use `PRD/CodeCoachAI_PRD_V2_AI能力增强版.md` as the only authoritative V2 scope source.

If old chat history, Codex historical output, `V2_ROADMAP_DRAFT.md`, `PROJECT_STATE.md`, `AGENTS.md`, `NEW_CODEX_SESSION_PROMPT.md`, or previous temporary roadmaps conflict with the official V2 PRD, the official V2 PRD wins.

### Classification correction

The previously completed work is classified as V2 technical baseline and AI interview enhancement prerequisites:

- service-side ADMIN secondary checks for `/admin/**`;
- HMAC signature protection for `/inner/**` internal calls;
- AI call stability enhancements;
- resume project deep-dive capability enhancement;
- early learning feedback in interview reports.

These items are not complete V2.

### Updated route

The V2 route is adjusted to:

1. complete the V2 backend first;
2. then complete the V2 frontend;
3. finally run full integration and E2E testing.

### Documents added or corrected

- `AGENTS.md`
- `PROJECT_STATE.md`
- `V2_ROADMAP_DRAFT.md`
- `NEW_CODEX_SESSION_PROMPT.md`
- `CodeCoachAI_V2_PRD差距清单.md`
- `CodeCoachAI_V2_后端开发路线.md`
