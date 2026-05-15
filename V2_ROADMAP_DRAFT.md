# V2_ROADMAP_DRAFT.md — CodeCoachAI V2 Roadmap

Last updated: 2026-05-15

## Scope rule

The only authoritative source for V2 scope is:

`PRD/CodeCoachAI_PRD_V2_AI能力增强版.md`

If this roadmap conflicts with the official V2 PRD, the official V2 PRD wins. This file is an execution roadmap, not a separate scope source.

Previously completed work such as `/admin/**` service-side ADMIN checks, `/inner/**` HMAC signatures, AI stability, project deep-dive enhancement, and report learning feedback prototype is classified as V2 technical baseline and AI interview enhancement prerequisites. It is not complete V2.

## V2 goal

V2 upgrades CodeCoachAI from a V1 interview closed loop into a more complete AI interview training product:

```text
resume upload or manual resume input
  -> AI resume parsing and risk identification
  -> industry scenario interview
  -> AI deep-dive based on question bank, resume, and industry template
  -> report weak-point extraction
  -> learning plan generation
  -> practice, review, and next interview
```

## Execution strategy

1. Complete V2 backend first.
2. Complete V2 frontend after backend contracts are stable.
3. Run full integration and E2E testing at the end.

## Roadmap

| Stage | Priority | Goal | Main PRD sections | Main output |
|---|---|---|---|---|
| A0 | P0 | V2 PRD alignment and gap list | 1, 3, 5, 7, 9, 10, 12, 13 | `CodeCoachAI_V2_PRD差距清单.md`, backend route |
| A1 | P0 | `codecoach-file` and resume file upload | 3.1, 5.2, 6.1, 7.1, 9, 10, 11 | file metadata, upload API, type/size checks |
| A2 | P0 | AI resume structured parsing | 3.1, 3.3, 7.1, 9, 10, 11 | parse record, status flow, structured JSON |
| A3 | P0 | AI resume optimization | 3.1, 3.3, 7.2, 9, 10, 11 | optimization record, scoring, risk and rewrite advice |
| A4 | P0 | industry templates and industry scenario interviews | 3.1, 3.2, 3.3, 7.3, 8, 9 | industry template model, scenario interview contract |
| A5 | P0 | AI question generation and review | 3.2, 3.3, 7.4, 9, 10, 11 | generated-question review workflow |
| A6 | P1 | initial question-bank deduplication | 3.2, 3.3, 7.5, 9, 10 | normalized title, duplicate review, merge/ignore |
| A7 | P1 | formal learning plan module | 3.1, 3.3, 7.6, 8, 9, 10, 11 | study plan, study task, check-in |
| A8 | P1 | Prompt template version management | 3.2, 7.8, 9, 11 | prompt versions, rollback, test, call records |
| A9 | P1 | SSE streaming output | 3.1, 3.3, 7.7, 10, 11 | AI stream endpoints and persistence rules |
| A10 | P2 | `codecoach-practice` split, only if necessary | 5.2, 13 | practice service boundary decision and migration plan |
| A11 | P0 | backend final static closure | 10, 11, 12 | backend route/table/interface audit, compile pass |
| A12 | P0/P1/P2 | frontend V2 adaptation | 8, 10, 12 | V2 user/admin pages and API integration |
| A13 | P0 | full integration and E2E testing | 12 | V2 acceptance report |

## Stage details

### A0 — V2 PRD alignment and gap list

- Confirm each V2 PRD function, priority, service, table, interface, and frontend impact.
- Mark completed baseline work as "partial" or "prerequisite completed", not full completion.
- Output the gap list and backend-first route.

### A1 — `codecoach-file` and resume file upload

- Add or confirm `codecoach-file`.
- Support PDF, Word, Markdown, and TXT upload.
- Persist `file_info`.
- Expose resume upload and file metadata management contracts.
- Validate file type, size, owner, and download authorization.

### A2 — AI resume structured parsing

- Create resume parse records and parse status flow.
- Extract text from uploaded files.
- Call AI resume parsing Prompt.
- Save structured result and support user confirmation.

### A3 — AI resume optimization

- Generate resume score, role match, project expression advice, risk warning, rewrite advice, and interview follow-up risks.
- Enforce the boundary that optimization is based on real experience only.
- Persist optimization records.

### A4 — Industry templates and industry scenario interviews

- Add industry template, industry scene, and industry question template models.
- Support industries such as e-commerce, financial payment, online education, SaaS, content community, logistics/ERP, and general admin systems.
- Let interview creation and AI prompts consume industry context.

### A5 — AI question generation and review

- Admin generates questions by technology point, type, difficulty, experience level, and quantity.
- Generated questions enter a review workflow before entering the official question bank.
- Support approve, reject, edit-then-approve, and merge into question group.

### A6 — Question-bank deduplication initial version

- Generate normalized titles.
- Find suspicious duplicates by category, keywords, and normalized title.
- Optionally call AI to judge same intent.
- Support manual merge, new question, follow-up marking, or ignore.

### A7 — Formal learning plan module

- Generate 7-day, 14-day, 30-day, and custom plans.
- Create daily tasks based on report, wrong records, weak points, target role, and available time.
- Support task completion, skip, and check-in.

### A8 — Prompt template version management

- Add template versions, history, rollback, version note, and test capability.
- Cover interview, report, resume parsing, resume optimization, question generation, learning plan, and dedup prompts.

### A9 — SSE streaming output

- Support streaming for interview question, answer comment, report, resume optimization, study plan, and question generation.
- Implement start/chunk/progress/done/error events.
- Handle timeout, disconnect, exception closure, persistence, and rate limiting.

### A10 — `codecoach-practice` split, only if necessary

- The official PRD lists `codecoach-practice` as a V2 service.
- Decide based on current backend coupling and migration risk.
- If the split is too disruptive, document the decision and keep the boundary prepared for later extraction.

### A11 — Backend final static closure

- Audit backend services, DTOs, routes, tables, Feign calls, auth, and logs against the official V2 PRD.
- Run backend compile/package checks.
- Produce backend closure notes.

### A12 — Frontend V2 adaptation

- Implement user pages: resume upload, parse result, resume optimization, scenario interview creation, learning plan, daily task, SSE display, enhanced home dashboard.
- Implement admin pages: AI question generation, generated-question review, duplicate review, question relation management, industry template management, Prompt version management, file records, AI call log details.

### A13 — Full integration and E2E testing

- Verify the V2 acceptance flow from resume upload to learning plan and repeated training.
- Verify admin AI question generation, review, dedup, Prompt versioning, and file record management.
- Produce the final V2 integration/E2E report.
