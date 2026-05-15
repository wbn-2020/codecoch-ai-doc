# V2_ROADMAP_DRAFT.md — CodeCoachAI V2 Roadmap Draft

## V2 goal

V2 should improve CodeCoachAI from a "core demo closed loop" into a stronger AI interview training product.

V2 should focus on:
- More realistic AI interview ability
- Better resume/project understanding
- Better report quality
- Better learning feedback loop
- Better admin/configuration experience
- More stable engineering quality

## V2 principles

1. Do not break V1 stable flows.
2. Every V2 feature must have a clear business value.
3. Avoid adding infrastructure only for appearance.
4. Prefer small, verifiable iterations.
5. Keep frontend/backend/API/docs synchronized.
6. For every AI feature, define prompt, input, output JSON, timeout, error handling, and logging.

## Recommended V2 phases

### V2.1 — V1 documentation and engineering hardening

Goal: make V1 maintainable before adding new features.

Tasks:
- Synchronize API docs with actual code.
- Produce final V1 acceptance report.
- Clean obsolete frontend API compatibility code.
- Standardize DTO naming.
- Improve error messages.
- Improve permission test cases.
- Improve README startup instructions.
- Tag V1 release.

### V2.2 — Real AI interview enhancement

Goal: make AI interview behavior more valuable.

Tasks:
- Standardize AI prompt templates.
- Define structured JSON output for scoring/follow-up/report.
- Add prompt versioning.
- Add real model provider configuration.
- Add AI timeout and retry strategy.
- Improve AI call logs: provider, model, scenario, prompt version, token usage, latency, error reason.
- Add fallback behavior when AI call fails.

### V2.3 — Resume/project deep-dive capability

Goal: make interviews based on the user's resume/project experience.

Tasks:
- Improve project experience data model if needed.
- Generate project-specific questions from resume/project content.
- Add project deep-dive question types:
  - project background
  - technical architecture
  - database design
  - difficult problems
  - optimization
  - troubleshooting
  - trade-off decisions
- Add project-based report section.

### V2.4 — Learning feedback loop

Goal: transform reports into actionable improvement.

Tasks:
- Weak knowledge point extraction.
- Recommended question list based on weak areas.
- Review plan generation.
- Wrong record aggregation by knowledge point.
- Practice progress dashboard.

### V2.5 — Optional advanced features

Only start these after V2.1–V2.4 are stable:
- Resume PDF/Word upload and parsing
- Embedding/vector retrieval for resume/question matching
- Voice interview
- Async report generation with MQ
- Advanced analytics dashboard

## V2 first task recommendation

The first V2 task should not be a new feature.

Recommended first task:
"Synchronize V1 documentation with current code and produce V1 release notes, API contract, and acceptance checklist."

After that, start:
"Refactor AI output into structured JSON contracts and improve real model mode stability."
