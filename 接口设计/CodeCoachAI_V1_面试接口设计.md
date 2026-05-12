# CodeCoachAI V1 面试接口设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 面试接口设计 |
| 项目阶段 | V1 |
| 最高依据 | 接口设计/CodeCoachAI_V1_接口设计总览.md |
| 参考文档 | PRD/CodeCoachAI_PRD_V1_微服务分层架构版.md、数据库设计/CodeCoachAI_V1_数据库设计总览.md、数据库设计/CodeCoachAI_V1_面试表设计.md、数据库设计/CodeCoachAI_V1_题库表设计.md、数据库设计/CodeCoachAI_V1_简历表设计.md、接口设计/CodeCoachAI_V1_题库接口设计.md、接口设计/CodeCoachAI_V1_简历接口设计.md |
| 文档用途 | 详细设计 interview-service 在 V1 阶段的 AI 模拟面试核心闭环接口 |

本文档只设计 CodeCoachAI V1 面试相关接口，不设计语音面试、视频面试、SSE 流式输出、WebSocket 实时通信、异步任务队列等后续能力。

涉及服务：

| 服务 | 说明 |
|---|---|
| `codecoachai-interview` | 面试会话、阶段、消息、报告的数据归属服务 |
| `codecoachai-question` | 被 interview-service 通过 `/inner/questions/**` 内部调用 |
| `codecoachai-resume` | 被 interview-service 通过 `/inner/resumes/**` 内部调用 |
| `codecoachai-ai` | 被 interview-service 通过 `/inner/ai/**` 内部调用 |
| `codecoachai-gateway` | 对外统一入口，不对外暴露 `/inner/**` |
| `codecoachai-auth` / `codecoachai-user` | 提供认证、登录态和当前用户上下文 |

字段说明：

1. 本文档优先贴合 `数据库设计/CodeCoachAI_V1_面试表设计.md`。
2. 数据库当前面试状态使用 `NOT_STARTED` 表示“已创建、未开始”，因此本文档不使用 `CREATED` 作为实际落库状态。
3. `POST /interviews/{id}/report/retry` 是为满足报告生成失败允许重试的 V1 补充接口，已同步补充到接口总览。
4. 当前面试表设计没有显式 `resume_snapshot_json`、`question_snapshot_json`、`interview_config_json` 字段；V1 接口仍要求保存必要快照，实现前建议对面试表设计做小修订，或至少通过 `interview_message.question_content`、`ai_comment`、`knowledge_points`、`interview_report.report_content` 等字段保存回放所需信息。

---

## 2. 模块职责

interview-service 负责：

1. 创建面试会话。
2. 生成面试阶段。
3. 开始面试。
4. 获取当前问题。
5. 保存用户回答。
6. 调用 ai-service 评分。
7. 调用 ai-service 生成动态追问。
8. 控制下一题、下一阶段、结束面试。
9. 保存面试消息。
10. 保存阶段状态。
11. 结束面试。
12. 调用 ai-service 生成面试报告。
13. 查询面试历史。
14. 查询面试详情。
15. 查询面试报告。

interview-service 不负责：

| 能力 | V1 处理方式 |
|---|---|
| 用户认证 | 由 auth-service / gateway / common-security 提供 |
| 题库管理 | 归属 question-service |
| 简历维护 | 归属 resume-service |
| 直接访问题库表 | 不允许，只能调用 `/inner/questions/**` |
| 直接访问简历表 | 不允许，只能调用 `/inner/resumes/**` |
| 大模型底层调用 | 归属 ai-service |
| Prompt 模板管理 | 归属 ai-service |
| 语音、视频、SSE、WebSocket | V1 不设计 |
| 管理端面试管理 | V1 暂不设计管理端接口 |

---

## 3. 统一约定

### 3.1 统一响应结构

所有接口使用接口总览中定义的统一响应结构：

```json
{
  "code": 0,
  "message": "success",
  "data": {},
  "traceId": "optional-trace-id"
}
```

### 3.2 分页结构

分页请求统一使用：

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `pageNo` | `int` | `1` | 当前页码，最小 1 |
| `pageSize` | `int` | `10` | 每页数量，最大 100 |

分页响应统一使用：

| 字段 | 类型 | 说明 |
|---|---|---|
| `records` | `array` | 当前页数据 |
| `total` | `long` | 总记录数 |
| `pageNo` | `int` | 当前页码 |
| `pageSize` | `int` | 每页数量 |
| `pages` | `long` | 总页数 |

### 3.3 Token 请求头格式

用户端面试接口统一使用：

```text
Authorization: Bearer {token}
```

### 3.4 当前登录用户上下文

interview-service 从登录态或 Gateway 透传信息中获取当前用户上下文。

当前用户上下文至少包含：

| 字段 | 说明 |
|---|---|
| `userId` | 当前登录用户 ID |
| `username` | 登录账号 |
| `roles` | 角色编码列表 |

用户端所有 `/interviews/**` 接口必须基于当前登录用户 `userId` 做数据归属校验。前端不能传 `userId` 创建或查询其他用户面试。

### 3.5 数据归属校验

| 数据 | 归属规则 |
|---|---|
| `interview_session` | 只能由当前用户创建和查询自己的面试 |
| `interview_stage` | 必须归属于当前用户自己的面试会话 |
| `interview_message` | 必须归属于当前用户自己的面试会话 |
| `interview_report` | 必须归属于当前用户自己的面试会话和用户 ID |

### 3.6 面试会话状态流转

数据库设计中的面试状态：

```text
NOT_STARTED -> IN_PROGRESS -> COMPLETED
```

辅助状态：

| 状态 | 说明 |
|---|---|
| `WAITING_ANSWER` | 等待用户回答，V1 可作为内部细分状态 |
| `AI_EVALUATING` | AI 评分中，V1 同步接口中可作为短暂状态 |
| `REPORT_GENERATING` | 报告生成中，V1 同步生成时可作为短暂状态 |
| `CANCELED` | 已取消，V1 可选 |
| `FAILED` | 面试流程失败，V1 用于不可恢复异常 |

说明：

1. 创建面试后状态为 `NOT_STARTED`，等价于接口草案中的“已创建未开始”。
2. 开始面试后状态为 `IN_PROGRESS`。
3. 面试完成后状态为 `COMPLETED`。
4. 报告生成失败时，V1 推荐面试主状态仍为 `COMPLETED`，报告状态为 `FAILED`，允许后续重试。

### 3.7 面试阶段状态流转

阶段状态建议：

```text
NOT_STARTED -> IN_PROGRESS -> COMPLETED
```

说明：

1. 创建面试时生成阶段，阶段初始状态为 `NOT_STARTED`。
2. 开始面试时，第一个阶段变为 `IN_PROGRESS`。
3. 当前阶段题目全部完成后，当前阶段变为 `COMPLETED`。
4. 进入下一阶段时，新阶段变为 `IN_PROGRESS`。

### 3.8 面试消息保存规则

`interview_message` 用于保存 AI 问题、用户回答、AI 点评和追问关系。

保存规则：

1. 每一道正式问题保存一条 AI 消息，`role = AI`，`is_follow_up = 0`。
2. 每一次用户回答保存一条用户消息，`role = USER`，并关联当前问题消息。
3. 每一次 AI 评分点评可以保存到用户回答消息的 `ai_comment`、`score`，也可以额外保存一条 `EVALUATION` 类型 AI 消息；V1 推荐保存到回答消息，减少消息数量。
4. 每一次追问保存一条 AI 消息，`role = AI`，`is_follow_up = 1`，`parent_message_id` 指向主问题或上一轮追问。
5. 每个主问题最多追问 2 次，由 interview-service 基于 `parent_message_id` 统计。
6. 必须保存必要的问题快照，例如 `question_id`、`group_id`、`question_content`、参考答案摘要或评分上下文摘要。

### 3.9 AI 调用失败处理规则

AI 调用失败包括提问、评分、追问、报告生成失败。

处理规则：

1. ai-service 负责记录 `ai_call_log`。
2. interview-service 需要保存当前面试状态，不应产生错误的下一步动作。
3. 评分失败时，`POST /interviews/{id}/answer` 返回明确失败响应，前端可提示用户重试提交或稍后再试。
4. 追问生成失败时，可以降级为 `NEXT_QUESTION`，但必须在响应或日志中明确失败原因；V1 推荐先返回失败，避免状态错乱。
5. 报告生成失败时，记录失败状态和失败原因，并允许重试。

### 3.10 报告生成失败处理规则

V1 报告可同步生成。

规则：

1. 报告生成成功后，写入或更新 `interview_report`。
2. 报告生成失败时，`interview_report.status` 记录为 `FAILED`，并保存失败原因。
3. 面试主状态推荐保持 `COMPLETED`，表示问答流程已完成。
4. 用户可通过 `POST /interviews/{id}/report/retry` 重试生成报告。
5. 重复调用完成接口或重试接口时，不应生成多份有效报告。

### 3.11 `/inner/**` 内部接口调用约束

1. 前端不能访问 `/inner/ai/**`。
2. 前端不能访问 `/inner/questions/**`。
3. 前端不能访问 `/inner/resumes/**`。
4. Gateway 不对外暴露 `/inner/**`。
5. interview-service 通过 OpenFeign 调用内部接口。
6. 如果内部调用经过 Gateway，需要内部调用标识或服务间鉴权。
7. 内部调用仍需透传用户上下文或明确传入 `userId`，不能绕过数据归属校验。

### 3.12 非流式响应约定

V1 不做流式输出，所有接口以普通 HTTP JSON 响应为准。AI 评分、追问、报告生成接口可能耗时较长，前端需要显示加载状态。

---

## 4. 枚举设计

### 4.1 InterviewStatus

以数据库设计为准：

| 值 | 说明 |
|---|---|
| `NOT_STARTED` | 已创建，未开始 |
| `IN_PROGRESS` | 进行中 |
| `WAITING_ANSWER` | 等待用户回答，V1 可作为内部细分状态 |
| `AI_EVALUATING` | AI 评估中，V1 可作为内部短暂状态 |
| `REPORT_GENERATING` | 报告生成中，V1 可作为内部短暂状态 |
| `COMPLETED` | 已完成 |
| `CANCELED` | 已取消，V1 可选 |
| `FAILED` | 面试失败 |

说明：用户需求中提到的 `CREATED` 在 V1 数据库中对应 `NOT_STARTED`。

### 4.2 InterviewMode

| 值 | 说明 |
|---|---|
| `TECHNICAL_BASIC` | 八股文技术面试 |
| `PROJECT_DEEP_DIVE` | 简历项目深挖面试 |
| `COMPREHENSIVE` | 综合模拟面试 |

### 4.3 InterviewStageType

数据库当前未单独设计 `stage_type` 字段，V1 可用 `stage_name` 和 `focus_points` 表达阶段类型。接口层建议使用以下枚举生成阶段：

| 值 | 说明 |
|---|---|
| `BASIC` | 基础知识 |
| `PROJECT` | 项目深挖 |
| `SCENARIO` | 场景题 |
| `SYSTEM_DESIGN` | 系统设计，V1 可选 |
| `SUMMARY` | 总结，V1 可选 |

### 4.4 InterviewStageStatus

| 值 | 说明 |
|---|---|
| `NOT_STARTED` | 未开始 |
| `IN_PROGRESS` | 进行中 |
| `COMPLETED` | 已完成 |

### 4.5 InterviewMessageRole

| 值 | 说明 |
|---|---|
| `AI` | AI 提问或点评 |
| `USER` | 用户回答 |
| `SYSTEM` | 系统消息，V1 可选 |

### 4.6 InterviewMessageType

数据库当前未单独设计 `message_type` 字段，接口层可根据 `role`、`is_follow_up`、`user_answer`、`ai_comment` 推导。

| 值 | 说明 |
|---|---|
| `QUESTION` | 正式问题 |
| `ANSWER` | 用户回答 |
| `FOLLOW_UP` | 追问 |
| `EVALUATION` | AI 评分点评 |
| `REPORT` | 报告内容，V1 可选 |

### 4.7 NextAction

| 值 | 说明 |
|---|---|
| `FOLLOW_UP` | 继续追问 |
| `NEXT_QUESTION` | 进入下一题 |
| `NEXT_STAGE` | 进入下一阶段 |
| `FINISH` | 结束面试 |

### 4.8 ReportStatus

`interview_report.status` 用于表达报告生成状态。当前数据库 SQL 默认值为 `COMPLETED`，为了接口语义清晰，V1 接口推荐统一使用以下状态；实现前建议同步小修订数据库默认值。

| 值 | 说明 |
|---|---|
| `NOT_GENERATED` | 未生成 |
| `GENERATING` | 生成中，V1 同步生成时可选 |
| `GENERATED` | 已生成 |
| `FAILED` | 生成失败 |

状态边界：

1. `interview_session.status` 表示面试流程状态，例如 `NOT_STARTED`、`IN_PROGRESS`、`COMPLETED`。
2. `interview_report.status` 表示报告生成状态，例如 `NOT_GENERATED`、`GENERATED`、`FAILED`。
3. 不要把 `COMPLETED` 同时用于面试状态和报告状态，避免语义混乱。

---

## 5. 用户端接口设计

### 5.1 POST /interviews

接口说明：创建面试。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/interviews` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `interview_session`、`interview_stage`、`interview_message` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `resumeId` | `long` | 是 | 简历 ID |
| `questionGroupId` | `long` | 否 | 指定问题组 ID |
| `interviewName` | `string` | 否 | 面试名称，当前数据库无独立字段，V1 可由前端展示或由后端根据岗位和时间生成 |
| `interviewMode` | `string` | 否 | `TECHNICAL_BASIC` / `PROJECT_DEEP_DIVE` / `COMPREHENSIVE`，默认 `COMPREHENSIVE` |
| `targetPosition` | `string` | 否 | 目标岗位 |
| `experienceLevel` | `string` | 否 | 经验年限 |
| `industryDirection` | `string` | 否 | 行业方向 |
| `difficulty` | `string` | 否 | `EASY` / `MEDIUM` / `HARD`，默认 `MEDIUM` |
| `interviewerStyle` | `string` | 否 | 面试官风格 |
| `stageTypes` | `array<string>` | 否 | 阶段类型，例如 `BASIC`、`PROJECT` |
| `questionCount` | `int` | 否 | 题目数量 |
| `config` | `object` | 否 | 扩展配置，例如每题最大追问次数 |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `interviewId` | `long` | 面试 ID |
| `status` | `string` | 面试状态，创建后为 `NOT_STARTED` |
| `reportStatus` | `string` | 报告状态，创建后为 `NOT_GENERATED` |
| `stageList` | `array<InterviewStageVO>` | 阶段列表 |
| `createdAt` | `string` | 创建时间 |

业务规则：

1. 必须绑定当前登录用户。
2. `resumeId` 必须属于当前用户。
3. 如果传 `questionGroupId`，必须是启用、未删除的问题组。
4. 创建面试时调用 resume-service 内部接口查询简历详情和项目经历。
5. 创建面试时调用 question-service 内部接口抽题。
6. 创建面试时应保存必要的简历快照和题目快照。
7. 创建后状态为 `NOT_STARTED`。
8. V1 推荐创建时生成阶段和预选题目，但正式提问可在 start 或 current 阶段生成。
9. V1 不调用 AI 生成整场面试计划，避免复杂化。

### 5.2 POST /interviews/{id}/start

接口说明：开始面试。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/interviews/{id}/start` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `interview_session`、`interview_stage`、`interview_message` |

路径参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 面试 ID |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `interviewId` | `long` | 面试 ID |
| `status` | `string` | 面试状态，开始后为 `IN_PROGRESS` |
| `currentStage` | `InterviewStageVO` | 当前阶段 |
| `currentQuestion` | `CurrentInterviewQuestionVO` | 当前问题 |
| `startedAt` | `string` | 开始时间 |

业务规则：

1. 只能开始当前用户自己的面试。
2. 只有 `NOT_STARTED` 状态可以开始。
3. 开始后状态变为 `IN_PROGRESS`。
4. 第一个阶段变为 `IN_PROGRESS`。
5. 返回第一道问题。
6. 当前问题可以来自题库快照，也可以结合 ai-service 生成自然表达形式。
7. 必须保存 AI 提问消息或题目消息到 `interview_message`。

### 5.3 GET /interviews/{id}/current

接口说明：获取当前问题。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/interviews/{id}/current` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `interview_session`、`interview_stage`、`interview_message` |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `interviewId` | `long` | 面试 ID |
| `status` | `string` | 面试状态 |
| `currentStage` | `InterviewStageVO` | 当前阶段 |
| `currentQuestion` | `CurrentInterviewQuestionVO` | 当前待回答问题 |
| `progress` | `InterviewProgressVO` | 面试进度 |
| `lastEvaluation` | `InterviewEvaluationVO` | 最近一次评分，可选 |

`currentQuestion` 建议包含：

| 字段 | 类型 | 说明 |
|---|---|---|
| `messageId` | `long` | 当前问题消息 ID |
| `questionId` | `long` | 题库题目 ID，可选 |
| `questionTitle` | `string` | 问题标题 |
| `questionContent` | `string` | 问题内容 |
| `questionType` | `string` | 问题类型 |
| `isFollowUp` | `boolean` | 是否追问 |
| `stageId` | `long` | 阶段 ID |
| `stageType` | `string` | 阶段类型 |

业务规则：

1. 只能查询当前用户自己的面试。
2. 面试必须处于 `IN_PROGRESS`。
3. 如果没有当前问题，需要根据状态返回下一步动作或提示。
4. 不重复生成问题。
5. 只返回当前待回答的问题。

### 5.4 POST /interviews/{id}/answer

接口说明：提交回答。这是 V1 面试核心接口。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/interviews/{id}/answer` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `interview_session`、`interview_stage`、`interview_message` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `messageId` | `long` | 是 | 当前问题消息 ID |
| `answerContent` | `string` | 是 | 用户回答内容 |
| `answerDurationSeconds` | `int` | 否 | 回答耗时秒数 |
| `clientSubmitTime` | `string` | 否 | 客户端提交时间 |

响应字段必须包含：

| 字段 | 类型 | 说明 |
|---|---|---|
| `interviewId` | `long` | 面试 ID |
| `answerMessageId` | `long` | 用户回答消息 ID |
| `evaluation` | `InterviewEvaluationVO` | AI 评分点评 |
| `nextAction` | `string` | 下一步动作 |
| `nextQuestion` | `NextQuestionVO` | 下一题或追问题，不存在时为空 |
| `currentStage` | `InterviewStageVO` | 当前阶段 |
| `interviewStatus` | `string` | 面试状态 |
| `progress` | `InterviewProgressVO` | 面试进度 |

`evaluation` 建议包含：

| 字段 | 类型 | 说明 |
|---|---|---|
| `score` | `decimal` | 本轮得分 |
| `level` | `string` | 表现等级 |
| `comment` | `string` | 点评 |
| `advantage` | `string` | 优点 |
| `weakness` | `string` | 不足 |
| `suggestion` | `string` | 建议 |

`nextAction` 必须是：

```text
FOLLOW_UP
NEXT_QUESTION
NEXT_STAGE
FINISH
```

`nextQuestion` 如果存在，建议包含：

| 字段 | 类型 | 说明 |
|---|---|---|
| `messageId` | `long` | 下一问题消息 ID |
| `questionId` | `long` | 题库题目 ID，可选 |
| `questionTitle` | `string` | 问题标题 |
| `questionContent` | `string` | 问题内容 |
| `isFollowUp` | `boolean` | 是否追问 |
| `stageId` | `long` | 阶段 ID |
| `stageType` | `string` | 阶段类型 |

业务规则：

1. 只能给当前用户自己的面试提交回答。
2. 面试必须处于 `IN_PROGRESS`。
3. `messageId` 必须是当前待回答的问题。
4. 保存用户回答到 `interview_message`。
5. 调用 ai-service 的 `/inner/ai/interview/evaluate` 进行评分和点评。
6. 保存 AI 评分点评结果。
7. interview-service 根据评分、阶段配置、追问次数、题目数量决定 `nextAction`。
8. 如果需要追问，调用 `/inner/ai/interview/follow-up` 生成追问。
9. 如果进入下一题，从当前阶段题目池取下一题。
10. 如果进入下一阶段，更新阶段状态并返回新阶段第一题。
11. 如果全部完成，`nextAction` 返回 `FINISH`。
12. AI 评分失败时，需要给出明确失败响应，不应生成错误的下一步状态。
13. V1 不做流式输出。
14. 该接口只负责保存用户回答、调用 AI 评分、生成追问或下一步动作。
15. 该接口不直接生成面试报告。
16. 当 `nextAction = FINISH` 时，前端应继续调用 `POST /interviews/{id}/finish`，由结束接口统一完成报告生成。

### 5.5 POST /interviews/{id}/finish

接口说明：结束面试。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/interviews/{id}/finish` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `interview_session`、`interview_stage`、`interview_report` |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `interviewId` | `long` | 面试 ID |
| `status` | `string` | 面试状态 |
| `reportStatus` | `string` | 报告状态 |
| `reportId` | `long` | 报告 ID，可选 |
| `finishedAt` | `string` | 结束时间 |
| `message` | `string` | 提示信息 |

业务规则：

1. 只能结束当前用户自己的面试。
2. 面试必须处于 `IN_PROGRESS`，或者最近一次 `nextAction` 已经是 `FINISH`。
3. 结束后调用 ai-service 的 `/inner/ai/interview/report` 生成报告。
4. V1 可同步生成报告。
5. 报告生成成功后，`interview_session.status` 更新为 `COMPLETED`，`interview_report.status` 更新为 `GENERATED`，并保存报告内容。
6. 报告生成失败时，记录失败原因，`interview_report.status` 更新为 `FAILED`。
7. V1 推荐报告失败时面试主状态仍标记 `COMPLETED`，表示问答流程已结束；报告状态为 `FAILED`，允许后续重试报告生成。
8. 该接口不应重复生成多份有效报告，重复调用需要幂等处理。

### 5.6 POST /interviews/{id}/report/retry

接口说明：报告生成失败后重试。

这是为满足“报告生成失败允许重试”的 V1 补充接口，已同步补充到 `接口设计/CodeCoachAI_V1_接口设计总览.md`。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/interviews/{id}/report/retry` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `interview_session`、`interview_report` |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `interviewId` | `long` | 面试 ID |
| `reportStatus` | `string` | 报告状态 |
| `reportId` | `long` | 报告 ID |
| `message` | `string` | 提示信息 |

业务规则：

1. 只能重试当前用户自己的面试报告。
2. 只有 `reportStatus` 为 `FAILED` 或 `NOT_GENERATED` 时允许重试。
3. 调用 ai-service 的 `/inner/ai/interview/report` 重新生成报告。
4. 成功后更新 `reportStatus` 为 `GENERATED`。
5. 失败时记录失败原因。
6. 不重复生成多份有效报告。
7. 只允许当前用户操作自己的面试报告。
8. 该接口不纳入创建面试核心流程。

### 5.7 GET /interviews

接口说明：面试历史列表。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/interviews` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `interview_session`、`interview_stage`、`interview_message`、`interview_report` |

查询参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `status` | `string` | 否 | 面试状态 |
| `reportStatus` | `string` | 否 | 报告状态 |
| `keyword` | `string` | 否 | 面试名称、目标岗位关键字 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `interviewId` | `long` | 面试 ID |
| `interviewName` | `string` | 面试名称，可由目标岗位、模式和创建时间生成 |
| `resumeName` | `string` | 简历名称，来自面试快照或创建时记录 |
| `targetPosition` | `string` | 目标岗位 |
| `status` | `string` | 面试状态 |
| `reportStatus` | `string` | 报告状态 |
| `totalScore` | `decimal` | 总分 |
| `stageCount` | `int` | 阶段数 |
| `questionCount` | `int` | 问题数 |
| `startedAt` | `string` | 开始时间 |
| `finishedAt` | `string` | 结束时间 |
| `createdAt` | `string` | 创建时间 |

业务规则：

1. 只返回当前用户自己的面试。
2. 按 `createdAt` 或 `updatedAt` 倒序。
3. 不返回详细消息内容。
4. 支持基础筛选。

### 5.8 GET /interviews/{id}

接口说明：面试详情。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/interviews/{id}` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `interview_session`、`interview_stage`、`interview_message`、`interview_report` |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `interviewId` | `long` | 面试 ID |
| `interviewName` | `string` | 面试名称 |
| `status` | `string` | 面试状态 |
| `reportStatus` | `string` | 报告状态 |
| `resumeSnapshot` | `ResumeSnapshotVO` | 简历快照 |
| `stages` | `array<InterviewStageVO>` | 阶段列表 |
| `messages` | `array<InterviewMessageVO>` | 消息列表 |
| `createdAt` | `string` | 创建时间 |
| `startedAt` | `string` | 开始时间 |
| `finishedAt` | `string` | 结束时间 |

`stages` 建议包含：

| 字段 | 类型 | 说明 |
|---|---|---|
| `stageId` | `long` | 阶段 ID |
| `stageType` | `string` | 阶段类型 |
| `stageName` | `string` | 阶段名称 |
| `status` | `string` | 阶段状态 |
| `score` | `decimal` | 阶段得分 |
| `sort` | `int` | 阶段顺序 |

`messages` 建议包含：

| 字段 | 类型 | 说明 |
|---|---|---|
| `messageId` | `long` | 消息 ID |
| `stageId` | `long` | 阶段 ID |
| `role` | `string` | `AI` / `USER` / `SYSTEM` |
| `messageType` | `string` | 消息类型 |
| `questionId` | `long` | 题目 ID，可空 |
| `content` | `string` | 问题内容或用户回答 |
| `score` | `decimal` | 得分 |
| `evaluation` | `InterviewEvaluationVO` | AI 评分点评 |
| `createdAt` | `string` | 创建时间 |

业务规则：

1. 只能查看当前用户自己的面试。
2. 用于面试详情页和历史回放。
3. 应优先展示 interview-service 保存的快照和消息，不依赖题库实时数据。
4. 不展示 AI 内部 Prompt。
5. 不展示敏感日志。

### 5.9 GET /interviews/{id}/report

接口说明：查看面试报告。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/interviews/{id}/report` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `interview_session`、`interview_report` |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `reportId` | `long` | 报告 ID |
| `interviewId` | `long` | 面试 ID |
| `reportStatus` | `string` | 报告状态 |
| `totalScore` | `decimal` | 总分 |
| `summary` | `string` | 总结 |
| `stageReports` | `array<StageReportVO>` | 阶段报告 |
| `strengths` | `string` | 优势 |
| `weaknesses` | `string` | 不足 |
| `suggestions` | `string` | 复习建议 |
| `recommendedQuestions` | `array<object>` | 推荐题目，可选 |
| `generatedAt` | `string` | 生成时间 |
| `failedReason` | `string` | 失败原因，可选 |

业务规则：

1. 只能查看当前用户自己的报告。
2. 如果报告未生成，返回明确状态。
3. 如果报告生成失败，返回 `FAILED` 和失败提示。
4. 不直接调用 ai-service 重新生成，重试使用 `POST /interviews/{id}/report/retry`。
5. 报告内容来自 `interview_report` 表。

---

## 6. 内部依赖接口说明

本章不重复设计其他服务接口，只说明 interview-service 对其他服务的调用关系。

### 6.1 调用 question-service

使用接口：

| 方法 | URL | 用途 |
|---|---|---|
| `POST` | `/inner/questions/pick-for-interview` | 面试抽题 |
| `GET` | `/inner/questions/{id}` | 获取题目详情和参考答案 |
| `POST` | `/inner/questions/recommend-for-report` | 报告推荐题，V1 可选 |

说明：

1. 用于抽题、获取题目详情、报告推荐题。
2. interview-service 不直接访问题库表。
3. 抽题只抽启用、未删除题目。
4. 创建面试或开始面试时需要保存题目快照。

### 6.2 调用 resume-service

使用接口：

| 方法 | URL | 用途 |
|---|---|---|
| `GET` | `/inner/resumes/{id}` | 查询简历详情 |
| `GET` | `/inner/resumes/{id}/projects` | 查询项目经历 |
| `GET` | `/inner/users/{userId}/default-resume` | 查询默认简历，V1 可选 |

说明：

1. 用于创建面试时获取简历上下文。
2. interview-service 不直接访问简历表。
3. 必须校验简历归属。
4. 创建面试时保存必要简历快照。

### 6.3 调用 ai-service

使用接口：

| 方法 | URL | 用途 |
|---|---|---|
| `POST` | `/inner/ai/interview/question` | 生成自然面试问题 |
| `POST` | `/inner/ai/interview/evaluate` | 回答评分和点评 |
| `POST` | `/inner/ai/interview/follow-up` | 生成动态追问 |
| `POST` | `/inner/ai/interview/report` | 生成面试报告 |

说明：

1. AI 能力只能由 interview-service 内部调用。
2. 前端不直接访问 ai-service 能力接口。
3. AI 调用失败时需要记录失败信息。
4. AI 调用日志由 ai-service 负责记录。

---

## 7. DTO / VO 设计

### 7.1 创建与启动

#### CreateInterviewRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `resumeId` | `long` | 是 | 简历 ID |
| `questionGroupId` | `long` | 否 | 问题组 ID |
| `interviewName` | `string` | 否 | 面试名称，数据库当前无独立字段 |
| `interviewMode` | `string` | 否 | 面试模式 |
| `targetPosition` | `string` | 否 | 目标岗位 |
| `experienceLevel` | `string` | 否 | 经验年限 |
| `industryDirection` | `string` | 否 | 行业方向 |
| `difficulty` | `string` | 否 | 难度 |
| `interviewerStyle` | `string` | 否 | 面试官风格 |
| `stageTypes` | `array<string>` | 否 | 阶段类型 |
| `questionCount` | `int` | 否 | 题目数量 |
| `config` | `object` | 否 | 扩展配置 |

#### CreateInterviewVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `interviewId` | `long` | 是 | 面试 ID |
| `status` | `string` | 是 | 面试状态 |
| `reportStatus` | `string` | 是 | 报告状态 |
| `stageList` | `array<InterviewStageVO>` | 是 | 阶段列表 |
| `createdAt` | `string` | 是 | 创建时间 |

#### InterviewStageVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `stageId` | `long` | 是 | 阶段 ID |
| `stageType` | `string` | 否 | 阶段类型，数据库可由 `stageName` 推导 |
| `stageName` | `string` | 是 | 阶段名称 |
| `stageOrder` | `int` | 是 | 阶段顺序 |
| `expectedQuestionCount` | `int` | 是 | 预计问题数 |
| `actualQuestionCount` | `int` | 是 | 实际问题数 |
| `focusPoints` | `string` | 否 | 考察重点 |
| `status` | `string` | 是 | 阶段状态 |
| `stageScore` | `decimal` | 否 | 阶段得分 |

#### StartInterviewVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `interviewId` | `long` | 是 | 面试 ID |
| `status` | `string` | 是 | 面试状态 |
| `currentStage` | `InterviewStageVO` | 是 | 当前阶段 |
| `currentQuestion` | `CurrentInterviewQuestionVO` | 是 | 当前问题 |
| `startedAt` | `string` | 是 | 开始时间 |

#### CurrentInterviewQuestionVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `messageId` | `long` | 是 | 问题消息 ID |
| `questionId` | `long` | 否 | 题库题目 ID |
| `groupId` | `long` | 否 | 问题组 ID |
| `questionTitle` | `string` | 否 | 问题标题 |
| `questionContent` | `string` | 是 | 问题内容 |
| `questionType` | `string` | 否 | 问题类型 |
| `isFollowUp` | `boolean` | 是 | 是否追问 |
| `stageId` | `long` | 是 | 阶段 ID |
| `stageType` | `string` | 否 | 阶段类型 |

### 7.2 答题与流程控制

#### SubmitInterviewAnswerRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `messageId` | `long` | 是 | 当前问题消息 ID |
| `answerContent` | `string` | 是 | 用户回答 |
| `answerDurationSeconds` | `int` | 否 | 回答耗时 |
| `clientSubmitTime` | `string` | 否 | 客户端提交时间 |

#### SubmitInterviewAnswerVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `interviewId` | `long` | 是 | 面试 ID |
| `answerMessageId` | `long` | 是 | 回答消息 ID |
| `evaluation` | `InterviewEvaluationVO` | 是 | AI 评分点评 |
| `nextAction` | `string` | 是 | 下一步动作 |
| `nextQuestion` | `NextQuestionVO` | 否 | 下一题或追问题 |
| `currentStage` | `InterviewStageVO` | 是 | 当前阶段 |
| `interviewStatus` | `string` | 是 | 面试状态 |
| `progress` | `InterviewProgressVO` | 是 | 面试进度 |

#### InterviewEvaluationVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `score` | `decimal` | 是 | 本轮得分 |
| `level` | `string` | 否 | 表现等级 |
| `comment` | `string` | 是 | AI 点评 |
| `advantage` | `string` | 否 | 优点 |
| `weakness` | `string` | 否 | 不足 |
| `suggestion` | `string` | 否 | 建议 |
| `knowledgePoints` | `string` | 否 | 涉及知识点 |
| `followUpSuggested` | `boolean` | 否 | AI 是否建议追问 |
| `followUpReason` | `string` | 否 | 追问原因 |

#### NextQuestionVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `messageId` | `long` | 是 | 下一问题消息 ID |
| `questionId` | `long` | 否 | 题库题目 ID |
| `groupId` | `long` | 否 | 问题组 ID |
| `questionTitle` | `string` | 否 | 问题标题 |
| `questionContent` | `string` | 是 | 问题内容 |
| `isFollowUp` | `boolean` | 是 | 是否追问 |
| `stageId` | `long` | 是 | 阶段 ID |
| `stageType` | `string` | 否 | 阶段类型 |

#### InterviewProgressVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `currentStageOrder` | `int` | 是 | 当前阶段序号 |
| `totalStageCount` | `int` | 是 | 总阶段数 |
| `currentQuestionIndex` | `int` | 是 | 当前题序号 |
| `totalQuestionCount` | `int` | 是 | 总题数 |
| `answeredQuestionCount` | `int` | 是 | 已回答题数 |
| `followUpCount` | `int` | 是 | 当前主问题追问次数 |
| `maxFollowUpCount` | `int` | 是 | 最大追问次数，V1 默认 2 |

### 7.3 结束与报告

#### FinishInterviewVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `interviewId` | `long` | 是 | 面试 ID |
| `status` | `string` | 是 | 面试状态 |
| `reportStatus` | `string` | 是 | 报告状态 |
| `reportId` | `long` | 否 | 报告 ID |
| `finishedAt` | `string` | 是 | 结束时间 |
| `message` | `string` | 是 | 提示信息 |

#### RetryReportVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `interviewId` | `long` | 是 | 面试 ID |
| `reportStatus` | `string` | 是 | 报告状态 |
| `reportId` | `long` | 否 | 报告 ID |
| `message` | `string` | 是 | 提示信息 |

#### InterviewReportVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `reportId` | `long` | 否 | 报告 ID |
| `interviewId` | `long` | 是 | 面试 ID |
| `reportStatus` | `string` | 是 | 报告状态 |
| `totalScore` | `decimal` | 否 | 总分 |
| `summary` | `string` | 否 | 总结，对应 `report_content` 或结构化摘要 |
| `stageReports` | `array<StageReportVO>` | 否 | 阶段报告 |
| `strengths` | `string` | 否 | 优势，对应 `strength_summary` |
| `weaknesses` | `string` | 否 | 问题总结，对应 `problem_summary` |
| `suggestions` | `string` | 否 | 复习建议，对应 `review_suggestions` |
| `weakKnowledgePoints` | `string` | 否 | 薄弱知识点 |
| `projectExpressionProblems` | `string` | 否 | 项目表达问题 |
| `recommendedQuestions` | `array<object>` | 否 | 推荐题目 |
| `generatedAt` | `string` | 否 | 生成时间 |
| `failedReason` | `string` | 否 | 失败原因，当前表无独立字段，V1 可写入 `report_content` 或后续补字段 |

#### StageReportVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `stageId` | `long` | 是 | 阶段 ID |
| `stageName` | `string` | 是 | 阶段名称 |
| `stageType` | `string` | 否 | 阶段类型 |
| `score` | `decimal` | 否 | 阶段得分 |
| `summary` | `string` | 否 | 阶段总结 |
| `weaknesses` | `string` | 否 | 阶段问题 |
| `suggestions` | `string` | 否 | 阶段建议 |

### 7.4 历史与详情

#### InterviewQueryRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `status` | `string` | 否 | 面试状态 |
| `reportStatus` | `string` | 否 | 报告状态 |
| `keyword` | `string` | 否 | 关键字 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

#### InterviewListVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `interviewId` | `long` | 是 | 面试 ID |
| `interviewName` | `string` | 否 | 面试名称 |
| `interviewMode` | `string` | 是 | 面试模式 |
| `resumeName` | `string` | 否 | 简历名称 |
| `targetPosition` | `string` | 否 | 目标岗位 |
| `status` | `string` | 是 | 面试状态 |
| `reportStatus` | `string` | 是 | 报告状态 |
| `totalScore` | `decimal` | 否 | 总分 |
| `stageCount` | `int` | 是 | 阶段数 |
| `questionCount` | `int` | 是 | 问题数 |
| `startedAt` | `string` | 否 | 开始时间 |
| `finishedAt` | `string` | 否 | 结束时间 |
| `createdAt` | `string` | 是 | 创建时间 |

#### InterviewDetailVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `interviewId` | `long` | 是 | 面试 ID |
| `interviewName` | `string` | 否 | 面试名称 |
| `interviewMode` | `string` | 是 | 面试模式 |
| `targetPosition` | `string` | 否 | 目标岗位 |
| `experienceLevel` | `string` | 否 | 经验年限 |
| `industryDirection` | `string` | 否 | 行业方向 |
| `difficulty` | `string` | 是 | 难度 |
| `interviewerStyle` | `string` | 否 | 面试官风格 |
| `status` | `string` | 是 | 面试状态 |
| `reportStatus` | `string` | 是 | 报告状态 |
| `resumeSnapshot` | `ResumeSnapshotVO` | 否 | 简历快照 |
| `stages` | `array<InterviewStageVO>` | 是 | 阶段列表 |
| `messages` | `array<InterviewMessageVO>` | 是 | 消息列表 |
| `createdAt` | `string` | 是 | 创建时间 |
| `startedAt` | `string` | 否 | 开始时间 |
| `finishedAt` | `string` | 否 | 结束时间 |

#### InterviewMessageVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `messageId` | `long` | 是 | 消息 ID |
| `stageId` | `long` | 是 | 阶段 ID |
| `questionId` | `long` | 否 | 题目 ID |
| `groupId` | `long` | 否 | 问题组 ID |
| `role` | `string` | 是 | 角色 |
| `messageType` | `string` | 是 | 消息类型 |
| `content` | `string` | 是 | 展示内容 |
| `questionContent` | `string` | 否 | AI 问题内容 |
| `userAnswer` | `string` | 否 | 用户回答 |
| `aiComment` | `string` | 否 | AI 点评 |
| `score` | `decimal` | 否 | 得分 |
| `isFollowUp` | `boolean` | 是 | 是否追问 |
| `parentMessageId` | `long` | 否 | 父消息 ID |
| `followUpReason` | `string` | 否 | 追问原因 |
| `knowledgePoints` | `string` | 否 | 知识点 |
| `createdAt` | `string` | 是 | 创建时间 |

#### ResumeSnapshotVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `resumeId` | `long` | 否 | 简历 ID |
| `resumeName` | `string` | 否 | 简历名称 |
| `targetPosition` | `string` | 否 | 求职方向 |
| `skills` | `string` | 否 | 技能栈 |
| `workSummary` | `string` | 否 | 工作经历摘要 |
| `education` | `string` | 否 | 教育经历 |
| `projects` | `array<object>` | 否 | 项目经历快照 |

### 7.5 内部上下文对象

#### InterviewQuestionSnapshotVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `questionId` | `long` | 否 | 题库题目 ID |
| `groupId` | `long` | 否 | 问题组 ID |
| `title` | `string` | 否 | 题目标题 |
| `content` | `string` | 是 | 题目内容 |
| `answer` | `string` | 否 | 参考答案 |
| `analysis` | `string` | 否 | 解析 |
| `categoryName` | `string` | 否 | 分类名称 |
| `difficulty` | `string` | 否 | 难度 |
| `tags` | `array<object>` | 否 | 标签快照 |

#### InterviewResumeSnapshotVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `resumeId` | `long` | 是 | 简历 ID |
| `resumeName` | `string` | 是 | 简历名称 |
| `targetPosition` | `string` | 否 | 求职方向 |
| `skills` | `string` | 否 | 技能栈 |
| `workSummary` | `string` | 否 | 工作经历摘要 |
| `education` | `string` | 否 | 教育经历 |
| `projects` | `array<object>` | 否 | 项目经历快照 |

#### AiEvaluateContextDTO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `interviewId` | `long` | 是 | 面试 ID |
| `stageId` | `long` | 是 | 阶段 ID |
| `questionMessageId` | `long` | 是 | 问题消息 ID |
| `questionSnapshot` | `InterviewQuestionSnapshotVO` | 是 | 问题快照 |
| `resumeSnapshot` | `InterviewResumeSnapshotVO` | 否 | 简历快照 |
| `answerContent` | `string` | 是 | 用户回答 |
| `historyMessages` | `array<InterviewMessageVO>` | 否 | 历史问答摘要 |

#### AiReportContextDTO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `interviewId` | `long` | 是 | 面试 ID |
| `userId` | `long` | 是 | 用户 ID |
| `session` | `object` | 是 | 面试会话摘要 |
| `resumeSnapshot` | `InterviewResumeSnapshotVO` | 否 | 简历快照 |
| `stages` | `array<InterviewStageVO>` | 是 | 阶段列表 |
| `messages` | `array<InterviewMessageVO>` | 是 | 问答消息 |
| `totalScore` | `decimal` | 否 | 总分 |

---

## 8. 参数校验规则

| 参数 | 校验规则 |
|---|---|
| `pageNo` | 最小 1 |
| `pageSize` | 最大 100 |
| `resumeId` | 必填，且必须属于当前用户 |
| `questionGroupId` | 可选，传入时必须存在且启用 |
| `interviewName` | 最大 100 |
| `interviewMode` | 必须是 `TECHNICAL_BASIC` / `PROJECT_DEEP_DIVE` / `COMPREHENSIVE` |
| `targetPosition` | 最大 100 |
| `experienceLevel` | 最大 32 |
| `industryDirection` | 最大 128 |
| `difficulty` | 必须是 `EASY` / `MEDIUM` / `HARD` |
| `interviewerStyle` | 最大 64 |
| `stageTypes` | 不能为空时必须是合法阶段类型 |
| `questionCount` | 最小 1，最大建议 30 |
| `answerContent` | 不能为空，按数据库 `text` 字段控制，接口建议最大 10000 |
| `messageId` | 必填 |
| `messageId` 归属 | 必须属于当前面试且是当前待回答问题 |
| `answerDurationSeconds` | 不能小于 0 |
| `finish` | 只能在合法状态下调用 |
| `report retry` | 只能在 `reportStatus` 为 `FAILED` 或 `NOT_GENERATED` 时调用 |

---

## 9. 错误码设计

面试模块错误码范围：`45000-45999`。

| code | message | 说明 |
|---:|---|---|
| `45001` | 面试不存在 | 面试 ID 不存在或已删除 |
| `45002` | 无权访问该面试 | 面试不属于当前用户 |
| `45003` | 面试状态不允许操作 | 当前状态不允许执行该动作 |
| `45004` | 面试已开始 | 重复开始面试 |
| `45005` | 面试未开始 | 当前操作要求面试已开始 |
| `45006` | 面试已结束 | 已完成面试不允许继续回答 |
| `45007` | 当前问题不存在 | 找不到当前待回答问题 |
| `45008` | 回答内容不能为空 | `answerContent` 为空 |
| `45009` | 当前问题不匹配 | `messageId` 不是当前待回答问题 |
| `45010` | 阶段不存在 | 阶段 ID 不存在 |
| `45011` | 阶段状态异常 | 阶段状态不符合流转规则 |
| `45012` | 简历快照生成失败 | 调用 resume-service 或保存快照失败 |
| `45013` | 题目快照生成失败 | 调用 question-service 或保存快照失败 |
| `45014` | 抽题失败 | 内部抽题接口失败 |
| `45015` | 题目数量不足 | 可用题目不足 |
| `45016` | AI 评分失败 | ai-service 评分失败 |
| `45017` | AI 追问生成失败 | ai-service 追问失败 |
| `45018` | 报告不存在 | 报告记录不存在 |
| `45019` | 报告生成失败 | ai-service 报告生成失败 |
| `45020` | 报告状态不允许重试 | 当前报告状态不能重试 |
| `45021` | 下一题生成失败 | 无法生成下一题或下一阶段问题 |
| `45022` | 面试配置非法 | 创建面试配置不合法 |

复用通用错误码：

| code | message | 说明 |
|---:|---|---|
| `40001` | 参数校验失败 | 请求参数不合法 |
| `41000` | 未登录 | 未携带 Token 或 Token 无效 |
| `41003` | 无访问权限 | 非法访问或权限不足 |
| `43000-43999` | 题库模块错误 | 抽题、题目、问题组相关错误 |
| `44000-44999` | 简历模块错误 | 简历归属、简历不存在等错误 |
| `46000-46999` | AI 模块错误 | AI 调用、Prompt 等错误 |

---

## 10. 数据库表关系

本模块涉及：

| 表名 | 说明 |
|---|---|
| `interview_session` | 面试会话 |
| `interview_stage` | 面试阶段 |
| `interview_message` | 面试消息 |
| `interview_report` | 面试报告 |

关系说明：

1. interview-service 是这些表的数据归属服务。
2. 用户与 `interview_session` 是一对多关系。
3. `interview_session` 与 `interview_stage` 是一对多关系。
4. `interview_session` 与 `interview_message` 是一对多关系。
5. `interview_session` 与 `interview_report` 是一对一关系，当前数据库通过 `uk_interview_report_session_id` 约束保证一个会话最多一份有效报告。
6. `interview_message` 保存 AI 提问、用户回答、AI 点评、追问关系。
7. `interview_report` 保存最终报告。
8. `interview_session` 建议保存必要简历快照字段或 `resume_snapshot_json`。
9. `interview_message` 建议保存必要题目快照字段或 `question_snapshot_json`。
10. `interview_session` 或关联配置字段建议保存 `interview_config_json`，用于记录创建面试时的题量、阶段、难度、追问次数等配置。
11. 如果暂不新增显式快照 JSON 字段，至少需要确保 `interview_message` / `interview_report` 保存历史回放需要的题目内容、参考答案摘要、评分上下文、知识点等信息。
12. 历史面试回放不应强依赖题库和简历的实时数据。

关系示意：

```text
interview_session
  ├── interview_stage
  ├── interview_message
  └── interview_report
```

---

## 11. 面试流程说明

V1 面试主流程：

```text
用户创建面试
  ↓
选择简历和问题组
  ↓
interview-service 调用 resume-service 获取简历
  ↓
interview-service 调用 question-service 抽题
  ↓
创建 interview_session 和 interview_stage
  ↓
开始面试
  ↓
AI 提问或返回题库问题
  ↓
用户回答
  ↓
AI 评分
  ↓
判断下一步动作
  ↓
追问 / 下一题 / 下一阶段 / 结束
  ↓
生成报告
  ↓
查看报告
```

会话状态流转：

```text
NOT_STARTED -> IN_PROGRESS -> COMPLETED
```

异常或可选状态：

```text
IN_PROGRESS -> CANCELED
IN_PROGRESS -> FAILED
```

报告状态流转：

```text
NOT_GENERATED -> GENERATED
NOT_GENERATED -> FAILED
FAILED -> GENERATED
```

下一步动作判断：

| nextAction | 触发场景 |
|---|---|
| `FOLLOW_UP` | 回答错误、回答过浅、需要结合项目继续追问，且追问次数未超过上限 |
| `NEXT_QUESTION` | 当前主问题完成，当前阶段仍有题 |
| `NEXT_STAGE` | 当前阶段完成，仍有后续阶段 |
| `FINISH` | 所有阶段完成或用户主动结束 |

---

## 12. 前端页面对应关系

| 前端页面 | 关联接口 |
|---|---|
| 创建面试页 | `POST /interviews` |
| AI 面试房间页 | `POST /interviews/{id}/start`、`GET /interviews/{id}/current`、`POST /interviews/{id}/answer`、`POST /interviews/{id}/finish` |
| 面试历史页 | `GET /interviews` |
| 面试详情页 | `GET /interviews/{id}` |
| 面试报告页 | `GET /interviews/{id}/report` |
| 报告失败重试 | `POST /interviews/{id}/report/retry` |

说明：

1. 创建面试页调用 `POST /interviews`。
2. AI 面试房间页调用开始、当前问题、提交回答、结束面试接口。
3. 面试历史页调用 `GET /interviews`。
4. 面试详情页调用 `GET /interviews/{id}`。
5. 面试报告页调用 `GET /interviews/{id}/report`。
6. 报告失败时可调用 `POST /interviews/{id}/report/retry`。

---

## 13. 注意事项

1. interview-service 负责编排面试流程。
2. AI 能力不直接暴露给前端。
3. 前端不访问 `/inner/ai/**`。
4. 前端不访问 `/inner/questions/**`。
5. 前端不访问 `/inner/resumes/**`。
6. 用户只能操作自己的面试。
7. 创建面试时必须保存简历和题目必要快照。
8. 提交回答接口必须返回 AI 评分、点评、`nextAction`、下一题或追问题。
9. `nextAction` 必须包含 `FOLLOW_UP`、`NEXT_QUESTION`、`NEXT_STAGE`、`FINISH`。
10. V1 不做 SSE 流式输出。
11. V1 不做语音视频。
12. V1 不做 WebSocket 实时通信。
13. V1 不做异步任务队列。
14. V1 报告可同步生成。
15. 报告生成失败必须记录失败状态，并允许重试。
16. 面试历史回放应基于 interview-service 保存的数据。
17. `POST /interviews/{id}/report/retry` 是 V1 补充接口，已同步补充到接口总览。
18. 当前数据库设计对报告状态和快照字段仍偏简化，正式实现前建议做一次小范围表设计修订。
