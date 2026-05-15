# CodeCoachAI V2 后端开发路线

Last updated: 2026-05-15

## 范围规则

正式 V2 范围唯一来源是：

`PRD/CodeCoachAI_PRD_V2_AI能力增强版.md`

本路线服务于“先完成后端，再做前端，最后统一联调 / E2E 测试”。如果本文件与正式 V2 PRD 冲突，以正式 V2 PRD 为准。

每个后端任务开始前必须声明：

1. 对应 PRD 章节；
2. 优先级 P0 / P1 / P2；
3. 涉及服务；
4. 涉及表；
5. 涉及接口；
6. 是否需要新增模块；
7. 是否影响前端；
8. 编译验证要求；
9. 后续测试点。

## 后端阶段路线

| 阶段 | 优先级 | 对应 PRD 章节 | 目标 | 涉及服务 | 涉及表 | 涉及接口 | 是否需要新增模块 | 是否影响前端 | 编译验证要求 | 后续测试点 |
|---|---|---|---|---|---|---|---|---|---|---|
| A0：PRD 对齐与后端缺口清单 | P0 | 1, 3, 5, 7, 9, 10, 11, 12, 13 | 建立后端实现基准，确认已完成、部分完成、未完成 | all backend services | all V2 tables | all V2 interfaces | 否 | 间接影响，形成前端合同基准 | 不要求编译；需要静态扫描 Controller/DTO/entity/SQL | 输出后端缺口清单，确认任务边界 |
| A1：`codecoach-file` / 简历文件上传 | P0 | 3.1, 5.2, 6.1, 7.1, 9, 10, 11 | 支持 PDF/Word/Markdown/TXT 上传和文件元数据 | `codecoach-file`, `codecoach-resume`, `codecoach-gateway` | `file_info`, `resume_analysis_record` | `POST /resumes/upload` | 是，新增或确认 `codecoach-file` | 是，后续需要简历上传页 | `mvn -q -DskipTests package` or targeted module compile | 上传类型/大小校验、鉴权、元数据、用户隔离 |
| A2：简历解析记录与状态流转 | P0 | 3.1, 7.1, 10, 11 | 建立解析记录、状态机、失败原因、重试入口 | `codecoach-resume`, `codecoach-file` | `resume_analysis_record` | `GET /resumes/{id}/parse-status`, `POST /resumes/{id}/reparse` | 否 | 是，前端需要状态展示 | backend compile/package | 待解析、解析中、成功、失败、待确认、重试 |
| A3：AI 简历结构化解析 | P0 | 3.1, 3.3, 7.1, 9, 10, 11 | 提取简历文本并调用 AI 生成结构化 JSON | `codecoach-resume`, `codecoach-ai`, `codecoach-file` | `resume_analysis_record`, `resume_skill`, `resume_work_experience`, existing resume tables | `GET /resumes/{id}/analysis-result`, `POST /resumes/{id}/confirm-analysis` | 否 | 是，前端需要解析结果页 | backend compile/package; AI JSON parser tests if available | AI 成功/失败、JSON 校验、结构化落库、用户确认 |
| A4：AI 简历优化 | P0 | 3.1, 3.3, 7.2, 9, 10, 11 | 输出评分、风险、改写建议和可追问问题 | `codecoach-resume`, `codecoach-ai` | `resume_optimize_record`, `ai_task_record` | `POST /resumes/{id}/optimize`, `GET /resumes/{id}/optimize-records`, `GET /resumes/optimize-records/{recordId}` | 否 | 是，前端需要简历优化页 | backend compile/package | 真实经历边界、优化记录、失败重试、权限隔离 |
| A5：行业模板与行业场景面试 | P0 | 3.1, 3.2, 3.3, 7.3, 8, 9 | 支持行业模板维护，并让面试创建和 AI 提问使用行业上下文 | `codecoach-system`, `codecoach-interview`, `codecoach-ai` | `industry_template`, `industry_scene`, `industry_question_template` | 待新增 `/admin/industry-templates/**`; 扩展面试创建接口 | 否 | 是，前端需要行业模板管理和行业面试创建 | backend compile/package | 行业 CRUD、启停、面试创建带行业、AI Prompt 上下文 |
| A6：AI 题目生成与审核 | P0 | 3.2, 3.3, 7.4, 8, 9, 10, 11 | 管理员生成题目，进入审核后再入正式题库 | `codecoach-question`, `codecoach-ai` | `question_review`, `ai_task_record`, existing question/category/tag/group tables | `POST /admin/ai/questions/generate`, `GET /admin/question-reviews`, `GET /admin/question-reviews/{id}`, `POST /admin/question-reviews/{id}/approve`, `POST /admin/question-reviews/{id}/reject` | 否 | 是，前端需要生成和审核页 | backend compile/package | 生成参数、审核状态机、通过入库、驳回、编辑后通过 |
| A7：题库去重初版 | P1 | 3.2, 3.3, 7.5, 9, 10 | 实现 normalized_title 和疑似重复题审核 | `codecoach-question`, `codecoach-ai` | `question_duplicate_review`, existing question/group tables | `POST /admin/questions/check-duplicate`, `GET /admin/question-duplicate-reviews`, `POST /admin/question-duplicate-reviews/{id}/merge`, `POST /admin/question-duplicate-reviews/{id}/ignore` | 否 | 是，前端需要重复题审核页 | backend compile/package | 归一化、候选查询、AI 判断、合并/忽略 |
| A8：正式学习计划模块 | P1 | 3.1, 3.3, 7.6, 8, 9, 10, 11 | 生成 7/14/30 天计划、每日任务和打卡 | `codecoach-interview`, `codecoach-ai`, `codecoach-question` or `codecoach-practice` | `study_plan`, `study_task`, `study_checkin` | `POST /study-plans/generate`, `GET /study-plans`, `GET /study-plans/{id}`, `GET /study-plans/{id}/tasks`, `POST /study-tasks/{id}/complete`, `POST /study-tasks/{id}/skip` | 否 | 是，前端需要学习计划和每日任务页 | backend compile/package | 计划生成、任务状态、完成/跳过、用户隔离 |
| A9：Prompt 模板版本管理 | P1 | 3.2, 3.3, 7.8, 8, 9, 11 | 支持版本、历史、回滚、测试和调用关联 | `codecoach-system`, `codecoach-ai` | `prompt_template_version`, `ai_task_record` | 待扩展 `/admin/prompt-templates/**` 版本接口 | 否 | 是，前端需要 Prompt 版本页 | backend compile/package | 新增版本、启停、回滚、测试、调用记录 |
| A10：SSE 流式输出 | P1 | 3.1, 3.3, 6.1, 7.7, 10, 11 | 对 AI 长耗时输出提供 SSE 流式体验 | `codecoach-ai`, `codecoach-interview`, `codecoach-resume` | `ai_task_record`, scenario business tables | `GET /ai/sse/interview-question`, `GET /ai/sse/interview-comment`, `GET /ai/sse/report`, `GET /ai/sse/resume-optimize`, `GET /ai/sse/study-plan` | 否 | 是，需要流式输出组件 | backend compile/package; targeted SSE smoke test | start/chunk/progress/done/error、超时、断开、落库、限流 |
| A11：后端最终静态收口 | P0 | 10, 11, 12, 13 | 对齐服务、表、接口、DTO、鉴权、日志和错误处理 | all V2 backend services | all V2 tables | all V2 backend interfaces | 否 | 是，冻结前端适配合同 | full backend package; targeted smoke tests | 路由、鉴权、用户隔离、AI 失败、日志、接口合同 |

## 阶段执行要求

### A0：PRD 对齐与后端缺口清单

- 先静态扫描后端 Controller、DTO、entity、mapper、SQL、Feign、gateway route。
- 输出“已有 / 部分已有 / 未实现”清单。
- 不修改业务代码，除非用户明确进入实现阶段。

### A1：`codecoach-file` / 简历文件上传

- 文件上传先用本地文件存储或轻量对象存储，V3 再统一升级 MinIO。
- 文件记录必须绑定用户和业务类型。
- 上传接口必须校验文件类型、大小和访问权限。

### A2：简历解析记录与状态流转

- 状态至少覆盖：待解析、解析中、解析成功、解析失败、待人工确认。
- 失败必须记录原因。
- 重试必须防止越权和重复执行。

### A3：AI 简历结构化解析

- AI 输出必须有 JSON 契约和格式校验。
- 解析结果应落到结构化表或可查询结构中。
- 用户确认后才能作为正式简历结构化数据使用。

### A4：AI 简历优化

- 必须明确“基于真实经历优化表达”的安全边界。
- 不生成虚假项目、虚假公司、虚假数据或夸大职责。
- 优化记录可查询、可追溯。

### A5：行业模板与行业场景面试

- 行业模板由后台维护。
- 面试创建时可以选择行业方向。
- AI 提问和追问必须能读取行业上下文。

### A6：AI 题目生成与审核

- AI 生成题不能直接进入正式题库。
- 题目、答案、解析、追问题、标签建议、分类建议需要进入审核记录。
- 审核通过后才写入正式题库。

### A7：题库去重初版

- V2 不做 Embedding 向量检索和全自动合并。
- 先实现文本归一化、疑似重复提示和人工确认。
- AI 判断只能作为辅助，不能自动删除或合并。

### A8：正式学习计划模块

- 学习计划应基于面试报告、错题、薄弱知识点、目标岗位、经验年限和可用学习时间。
- 每日任务要有状态流转。
- 生成失败要允许重新生成。

### A9：Prompt 模板版本管理

- Prompt 修改必须保留历史版本。
- 支持回滚和模板测试。
- AI 调用日志需要关联模板版本。

### A10：SSE 流式输出

- SSE 事件至少包括 start、chunk、progress、done、error。
- 必须处理连接超时、用户主动断开、异常关闭、生成内容落库和接口限流。

### A11：后端最终静态收口

- 对照 `CodeCoachAI_V2_PRD差距清单.md` 和正式 PRD。
- 确认 V2 后端服务、表、接口、DTO、鉴权、日志、错误码和 AI 失败策略。
- 后端收口后再进入前端 V2 适配。
