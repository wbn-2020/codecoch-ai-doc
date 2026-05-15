# CodeCoachAI V2 PRD 差距清单

Last updated: 2026-05-15

## 范围规则

正式 V2 范围唯一来源是：

`PRD/CodeCoachAI_PRD_V2_AI能力增强版.md`

如果旧聊天记录、Codex 历史输出、`V2_ROADMAP_DRAFT.md`、`PROJECT_STATE.md`、`AGENTS.md`、`NEW_CODEX_SESSION_PROMPT.md` 或之前的临时路线图与正式 V2 PRD 冲突，一律以正式 V2 PRD 为准。

此前已完成的 `/admin/**` 服务内 ADMIN 二次校验、`/inner/**` HMAC、AI 调用稳定性、项目深挖、报告学习反馈雏形，统一标记为 V2 技术基线或前置能力完成，不等于完整 V2 功能完成。

## 差距总表

| PRD 功能 | PRD 章节 | 优先级 | 当前状态 | 后端状态 | 前端状态 | 涉及服务 | 涉及数据表 | 涉及接口 | 下一步任务 |
|---|---|---|---|---|---|---|---|---|---|
| 简历文件上传 | 3.1, 5.2, 6.1, 7.1, 9, 10, 11 | P0 | 未完成 | 需新增 `codecoach-file` 或确认文件服务边界；需实现上传、元数据、鉴权、类型/大小校验 | 未完成；需新增简历上传页 | `codecoach-file`, `codecoach-resume`, `codecoach-gateway` | `file_info`, `resume_analysis_record` | `POST /resumes/upload` | 建立文件服务、上传 API、文件元数据表和简历解析记录创建流程 |
| 简历解析状态 | 3.1, 7.1, 10, 11 | P0 | 未完成 | 需实现待解析、解析中、解析成功、解析失败、待人工确认状态流转 | 未完成；需展示状态和失败原因 | `codecoach-resume`, `codecoach-ai` | `resume_analysis_record` | `GET /resumes/{id}/parse-status`, `POST /resumes/{id}/reparse` | 设计状态枚举、失败原因字段、重试入口和状态查询接口 |
| AI 简历结构化 | 3.1, 3.3, 7.1, 9, 10, 11 | P0 | 部分完成：项目深挖前置能力完成，但文件简历结构化未完成 | 需实现简历文本提取、AI 解析 Prompt、结构化结果落库、用户确认 | 未完成；需解析结果页和确认入口 | `codecoach-resume`, `codecoach-ai`, `codecoach-file` | `resume_analysis_record`, `resume_skill`, `resume_work_experience` | `GET /resumes/{id}/analysis-result`, `POST /resumes/{id}/confirm-analysis` | 完成 AI 解析 JSON 契约、字段校验、落库和确认流程 |
| AI 简历优化 | 3.1, 3.3, 7.2, 9, 10, 11 | P0 | 部分完成：项目深挖前置能力完成，正式简历优化模块未完成 | 需实现评分、风险提示、改写建议、优化记录 | 未完成；需简历优化页和记录页 | `codecoach-resume`, `codecoach-ai` | `resume_optimize_record`, `ai_task_record` | `POST /resumes/{id}/optimize`, `GET /resumes/{id}/optimize-records`, `GET /resumes/optimize-records/{recordId}` | 建立优化 Prompt、输出校验、真实经历安全边界和记录查询 |
| 行业场景面试 | 3.1, 3.2, 3.3, 7.3, 8, 9 | P0 | 部分完成：项目深挖前置能力完成，行业模板和行业面试未完成 | 需实现行业模板、行业场景、面试创建时行业上下文传递 | 未完成；需行业场景面试创建页 | `codecoach-system`, `codecoach-interview`, `codecoach-ai` | `industry_template`, `industry_scene`, `industry_question_template` | 待后端按现有面试创建接口扩展并补充行业模板管理接口 | 建立行业模板管理和面试创建行业参数，接入 AI 场景 Prompt |
| 学习计划 | 3.1, 3.3, 7.6, 8, 9, 10, 11 | P1 | 部分完成：报告内学习反馈雏形完成，正式学习计划模块未完成 | 需新增学习计划、每日任务、打卡、跳过/完成接口 | 未完成；需学习计划页和每日任务页 | `codecoach-interview`, `codecoach-ai`, `codecoach-practice` 或 `codecoach-question` | `study_plan`, `study_task`, `study_checkin` | `POST /study-plans/generate`, `GET /study-plans`, `GET /study-plans/{id}`, `GET /study-plans/{id}/tasks`, `POST /study-tasks/{id}/complete`, `POST /study-tasks/{id}/skip` | 建立学习计划模块、任务状态流转和 AI 生成契约 |
| 简答题 AI 点评 | 3.1, 3.3, 7.7, 11 | P1 | 部分完成：AI 评分/点评前置能力完成，刷题简答点评未完整按 V2 收口 | 需确认刷题回答点评接口、AI 日志和 SSE 是否复用 | 未完成或待适配；需刷题点评交互 | `codecoach-question`, `codecoach-ai`, `codecoach-practice` | `ai_task_record`, existing answer/wrong-record tables to be confirmed | 待按现有刷题接口扩展；可能复用 SSE 点评接口 | 审计现有刷题回答链路，补齐简答点评合同和日志 |
| SSE 流式输出 | 3.1, 3.3, 6.1, 7.7, 10, 11 | P1 | 未完成 | 需实现 start/chunk/progress/done/error 事件、超时、断开、限流、落库 | 未完成；需 AI 流式输出组件 | `codecoach-ai`, `codecoach-interview`, `codecoach-resume` | `ai_task_record`, possibly business records by scenario | `GET /ai/sse/interview-question`, `GET /ai/sse/interview-comment`, `GET /ai/sse/report`, `GET /ai/sse/resume-optimize`, `GET /ai/sse/study-plan` | 设计统一 SSE 事件模型和场景接入方式 |
| 用户首页增强 | 3.1, 8, 12 | P1 | 未完成 | 需提供最近面试、薄弱点、学习计划、今日推荐聚合数据 | 未完成；需首页看板增强版 | `codecoach-user`, `codecoach-interview`, `codecoach-question`, `codecoach-practice` or current practice boundary | `study_plan`, `study_task`, existing interview/report/question tables | 待新增用户首页聚合接口 | 后端先定义首页聚合 DTO，前端后续适配 |
| AI 题目生成 | 3.2, 3.3, 7.4, 9, 10, 11 | P0 | 未完成 | 需实现按技术点、难度、经验年限生成题目、答案、解析、追问、标签建议 | 未完成；需 AI 题目生成页 | `codecoach-question`, `codecoach-ai` | `question_review`, `ai_task_record`, existing question tables | `POST /admin/ai/questions/generate` | 建立生成参数、AI 输出 JSON、待审核落库流程 |
| AI 生成题审核 | 3.2, 7.4, 8, 9, 10, 11 | P0 | 未完成 | 需实现通过、驳回、编辑后通过、合并问题组 | 未完成；需 AI 生成题审核页 | `codecoach-question` | `question_review`, existing question/group/tag/category tables | `GET /admin/question-reviews`, `GET /admin/question-reviews/{id}`, `POST /admin/question-reviews/{id}/approve`, `POST /admin/question-reviews/{id}/reject` | 完成审核状态机和进入正式题库流程 |
| 行业模板管理 | 3.2, 7.3, 8, 9 | P0 | 未完成 | 需实现行业场景、业务问题、技术关注点维护 | 未完成；需行业模板管理页 | `codecoach-system`, `codecoach-interview`, `codecoach-ai` | `industry_template`, `industry_scene`, `industry_question_template` | 待新增 `/admin/industry-templates/**` 等管理接口 | 建立行业模板 CRUD、启停和面试引用关系 |
| Prompt 版本管理 | 3.2, 3.3, 7.8, 8, 9, 11 | P1 | 部分完成：V1 有 Prompt 模板管理，V2 版本化、历史、回滚、测试未完整确认 | 需新增版本表、历史、回滚、测试、调用记录关联 | 未完成；需 Prompt 模板版本页 | `codecoach-system`, `codecoach-ai` | `prompt_template_version`, `ai_task_record` | 待扩展 `/admin/prompt-templates/**` 版本接口 | 审计现有 Prompt 模板实现，补齐版本化和测试接口 |
| AI 调用日志增强 | 3.2, 11 | P1 | 部分完成：AI 稳定性和日志前置能力完成，V2 字段需核对补齐 | 需记录 token、耗时、场景、业务 ID、模板版本、失败重试入口 | 未完成或待适配；需 AI 调用日志详情页 | `codecoach-ai`, `codecoach-system` | `ai_task_record`, existing AI call log table to be confirmed | 待扩展 AI call log admin detail and retry APIs | 对齐日志表字段和管理端详情/重试入口 |
| 题库去重初版 | 3.2, 3.3, 7.5, 9, 10 | P1 | 未完成 | 需实现 normalized_title、疑似重复提示、人工确认、可选 AI 判断 | 未完成；需疑似重复题审核页 | `codecoach-question`, `codecoach-ai` | `question_duplicate_review`, existing question/group tables | `POST /admin/questions/check-duplicate`, `GET /admin/question-duplicate-reviews`, `POST /admin/question-duplicate-reviews/{id}/merge`, `POST /admin/question-duplicate-reviews/{id}/ignore` | 设计归一化规则、候选查询和审核流程 |
| 题目关系管理初版 | 3.2, 7.5, 8, 9 | P2 | 未完成 | 需支持 SAME_INTENT、FOLLOW_UP、RELATED、ADVANCED 等关系 | 未完成；需题目关系管理页 | `codecoach-question` | `question_relation` | 待新增 `/admin/question-relations/**` 管理接口 | 建立关系模型、CRUD 和题目详情展示 |
| 文件记录管理 | 3.2, 5.2, 7.1, 8, 9 | P2 | 未完成 | 需提供上传文件元数据、解析状态、归属和下载鉴权管理 | 未完成；需文件记录管理页 | `codecoach-file`, `codecoach-resume`, `codecoach-system` | `file_info`, `resume_analysis_record` | 待新增 `/admin/files/**` or equivalent file admin APIs | 后端完成文件记录查询、详情和异常状态管理 |

## 使用说明

1. 新任务必须先从本清单定位 PRD 功能、章节和优先级。
2. 标记为“部分完成”或“前置能力完成”的内容，只表示已有基础能力，不表示正式 V2 功能闭环完成。
3. 后端接口路径以正式 PRD 为基准；实现前仍需结合当前 Java 代码确认 Controller、DTO、鉴权和网关路由。
4. 当前策略是先完成后端，再完成前端，最后统一联调和 E2E。
