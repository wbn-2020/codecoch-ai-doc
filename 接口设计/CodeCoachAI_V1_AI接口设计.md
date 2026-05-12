# CodeCoachAI V1 AI 接口设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 AI 接口设计 |
| 项目阶段 | V1 |
| 最高依据 | 接口设计/CodeCoachAI_V1_接口设计总览.md |
| 参考文档 | PRD/CodeCoachAI_PRD_V1_微服务分层架构版.md、数据库设计/CodeCoachAI_V1_数据库设计总览.md、数据库设计/CodeCoachAI_V1_AI表设计.md、数据库设计/CodeCoachAI_V1_面试表设计.md、接口设计/CodeCoachAI_V1_面试接口设计.md |
| 文档用途 | 详细设计 ai-service 在 V1 阶段的 AI 内部能力接口、Prompt 模板管理接口、AI 调用日志查询接口 |

本文档只设计 CodeCoachAI V1 AI 相关接口，不设计 AI 题目批量生成、AI 生成题审核、简历 AI 优化、SSE 流式输出、WebSocket、异步 AI 任务队列等后续能力。

涉及服务：

| 服务 | 说明 |
|---|---|
| `codecoachai-ai` | AI 能力封装、Prompt 模板、AI 调用日志的数据归属服务 |
| `codecoachai-interview` | AI 能力接口的内部调用方 |
| `codecoachai-gateway` | 对外统一入口，不对外暴露 `/inner/**` |
| `codecoachai-auth` / `codecoachai-user` | 提供认证与管理员权限 |
| `codecoachai-system` | 可选，用于读取 AI 超时时间、模型名称等系统配置 |

字段说明：

1. 本文档优先贴合 `数据库设计/CodeCoachAI_V1_AI表设计.md`。
2. `prompt_template` 当前使用 `template_name`、`template_code`、`scene`、`template_content`、`variable_description` 字段，不强制拆分 `systemPrompt` 和 `userPromptTemplate`。
3. `ai_call_log` 当前使用 `scene`、`business_id`、`request_content`、`response_content`、`cost_time_ms` 字段，V1 数据库不强制保存 token 用量。
4. token 用量、provider、promptVersion 等字段在接口中可作为可选展示字段，正式实现前如需落库，应先小范围修订 AI 表设计。

---

## 2. 模块职责

ai-service 负责：

1. AI 面试问题生成。
2. AI 回答评分与点评。
3. AI 动态追问生成。
4. AI 面试报告生成。
5. Prompt 模板管理。
6. Prompt 变量说明。
7. AI 调用日志记录。
8. AI 调用日志管理端查询。
9. AI 调用失败信息记录。

ai-service 不负责：

| 能力 | V1 处理方式 |
|---|---|
| 用户认证 | 由 auth-service / gateway / common-security 负责 |
| 题库管理 | 归属 question-service |
| 简历维护 | 归属 resume-service |
| 面试流程编排 | 归属 interview-service |
| 面试状态流转 | 归属 interview-service |
| 保存 `interview_message` | 归属 interview-service |
| 保存 `interview_report` | 归属 interview-service |
| 前端 AI 能力接口 | V1 不提供，AI 能力只通过 `/inner/ai/**` 内部调用 |
| 流式输出 | V1 不设计 |
| 异步任务队列 | V1 不设计 |
| AI 题目批量生成 | V1 不设计 |
| 简历 AI 优化 | V1 不设计 |

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

### 3.2 Token 请求头格式

管理端接口统一使用：

```text
Authorization: Bearer {token}
```

说明：

1. `/admin/ai/**` 接口需要登录并具备 `ADMIN` 角色。
2. `/inner/ai/**` 仅服务内部 Feign 调用，不由前端携带用户 Token 直接访问。

### 3.3 管理端权限

管理端 Prompt 和 AI 日志接口统一使用 `/admin/ai` 前缀。

要求：

1. 必须登录。
2. 必须具备 `ADMIN` 角色。
3. 不返回 API Key、用户密码、Token 等敏感信息。
4. 日志中的请求内容和响应内容需要按需脱敏。

### 3.4 `/inner/**` 内部接口安全约束

`/inner/ai/**` 只允许 interview-service 调用。

约束：

1. Gateway 不对外暴露 `/inner/**`。
2. 前端不能直接访问 `/inner/ai/**`。
3. `/inner/ai/**` 仅允许服务内部 Feign 调用。
4. 如果内部调用经过 Gateway，需要内部调用标识或服务间鉴权。
5. AI 能力接口不负责校验用户是否拥有某场面试，数据归属由 interview-service 负责。
6. ai-service 可以接收 interview-service 传入的 `userId`、`interviewId`、`stageId`、`messageId` 等上下文字段，用于日志记录和排查问题。

### 3.5 AI 请求响应 JSON 格式

AI 能力接口统一使用 JSON 请求和 JSON 响应。

约定：

1. 请求体必须包含业务上下文字段。
2. 响应体尽量结构化，便于 interview-service 直接保存和判断。
3. AI 原始响应可以记录到 `ai_call_log.response_content`。
4. AI 解析后的结构化结果返回给 interview-service。
5. AI 响应解析失败时，不返回伪造结果。

### 3.6 AI 调用超时处理

V1 AI 调用为同步 HTTP 调用。

规则：

1. 超时时间可从配置文件或 system-service 读取。
2. AI 调用超时后记录 `ai_call_log.status = TIMEOUT` 或 `FAILED`。
3. 接口返回明确 AI 超时错误码。
4. interview-service 根据错误决定是否允许重试或终止当前流程。

### 3.7 AI 调用失败处理

规则：

1. AI 调用失败时，ai-service 返回明确错误码和错误信息。
2. AI 调用失败不能返回伪造评分、伪造追问或伪造报告。
3. AI 供应商异常、网络异常、响应解析失败都需要记录日志。
4. interview-service 决定面试流程如何处理。

### 3.8 AI 日志记录规则

每次 AI 调用都需要写入 `ai_call_log`，成功和失败都记录。

建议记录：

| 字段 | 对应数据库字段 |
|---|---|
| 用户 ID | `user_id` |
| 调用场景 | `scene` |
| 模型名称 | `model_name` |
| Prompt 模板 ID | `prompt_template_id` |
| 业务 ID | `business_id`，例如面试 ID |
| 请求内容 | `request_content` |
| 响应内容 | `response_content` |
| 调用耗时 | `cost_time_ms` |
| 调用状态 | `status` |
| 失败原因 | `error_message` |

说明：V1 数据库暂不记录 token 用量和 provider 字段，接口可作为可选展示字段，后续按需要扩展表结构。

### 3.9 Prompt 模板启用规则

1. AI 能力接口只使用已启用、未删除的 Prompt 模板。
2. 同一 AI 场景建议只有一个启用模板。
3. 如果某场景没有启用模板，对应 AI 能力接口返回 Prompt 不存在或未启用错误。
4. Prompt 模板禁用后，不能被新的 AI 调用选中。

### 3.10 敏感信息脱敏规则

1. API Key 不允许通过接口返回。
2. API Key 不允许写入普通日志。
3. API Key 不应存储在 `prompt_template` 或 `ai_call_log` 中。
4. 请求内容中的手机号、邮箱、Token 等敏感字段建议脱敏后记录。
5. 管理端日志详情可展示 Prompt 和 AI 响应，但仍需隐藏密钥和认证信息。

---

## 4. 枚举设计

### 4.1 PromptType

以数据库 `prompt_template.scene` 字段为准，V1 推荐使用以下场景值：

| 值 | 说明 |
|---|---|
| `INTERVIEW_QUESTION_GENERATE` | 面试问题生成 |
| `PROJECT_DEEP_DIVE_QUESTION` | 项目深挖问题生成 |
| `INTERVIEW_ANSWER_EVALUATE` | 回答评分点评 |
| `INTERVIEW_FOLLOW_UP_GENERATE` | 动态追问 |
| `INTERVIEW_REPORT_GENERATE` | 面试报告生成 |

接口层可将用户更易理解的 `INTERVIEW_QUESTION`、`ANSWER_EVALUATE`、`FOLLOW_UP`、`INTERVIEW_REPORT` 映射到以上数据库场景值，但落库建议统一使用 `scene` 枚举。

### 4.2 PromptStatus

| 值 | 枚举名 | 说明 |
|---|---|---|
| `1` | `ENABLED` | 启用 |
| `0` | `DISABLED` | 禁用 |

### 4.3 AiCallType

以数据库 `ai_call_log.scene` 字段为准，V1 推荐使用：

| 值 | 说明 |
|---|---|
| `INTERVIEW_QUESTION_GENERATE` | 生成面试问题 |
| `PROJECT_DEEP_DIVE_QUESTION` | 生成项目深挖问题 |
| `INTERVIEW_ANSWER_EVALUATE` | 回答评分 |
| `INTERVIEW_FOLLOW_UP_GENERATE` | 生成追问 |
| `INTERVIEW_REPORT_GENERATE` | 生成报告 |

### 4.4 AiCallStatus

| 值 | 说明 |
|---|---|
| `SUCCESS` | 调用成功 |
| `FAILED` | 调用失败 |
| `TIMEOUT` | 调用超时 |

说明：当前数据库注释为 `SUCCESS/FAILED`，如需要独立表达超时，V1 可将 `TIMEOUT` 作为 `FAILED` 的一种错误原因，或在实现前小修订枚举说明。

### 4.5 AiProvider

| 值 | 说明 |
|---|---|
| `OPENAI_COMPATIBLE` | OpenAI 兼容接口 |
| `DASHSCOPE` | 通义千问，可选 |
| `DEEPSEEK` | DeepSeek，可选 |
| `MOCK` | 本地模拟，开发环境可选 |

说明：

1. V1 文档不强制绑定具体供应商。
2. 实现阶段可以优先使用 OpenAI Compatible 形式，便于切换模型。
3. `MOCK` 只用于本地开发和演示兜底，不作为真实 AI 能力。
4. 当前数据库未设计 `provider` 字段，V1 可从配置中读取，不强制落库。

---

## 5. 内部 AI 能力接口设计

### 5.1 POST /inner/ai/interview/question

接口说明：生成面试问题。

调用方：interview-service。

用途：根据面试阶段、题目快照、简历上下文，生成或润色 AI 提问内容。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/inner/ai/interview/question` |
| 是否对前端暴露 | 否 |
| 关联表 | `prompt_template`、`ai_call_log` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID，用于日志记录 |
| `interviewId` | `long` | 是 | 面试 ID，对应 `business_id` |
| `stageId` | `long` | 是 | 阶段 ID |
| `stageType` | `string` | 否 | 阶段类型 |
| `questionId` | `long` | 否 | 题库题目 ID |
| `questionTitle` | `string` | 否 | 题目标题 |
| `questionContent` | `string` | 否 | 题目内容 |
| `referenceAnswer` | `string` | 否 | 参考答案 |
| `resumeSnapshot` | `AiResumeSnapshotDTO` | 否 | 简历快照 |
| `projectSnapshot` | `AiProjectSnapshotDTO` | 否 | 项目快照 |
| `targetPosition` | `string` | 否 | 目标岗位 |
| `difficulty` | `string` | 否 | 难度 |
| `previousMessages` | `array<AiPreviousMessageDTO>` | 否 | 历史消息摘要 |
| `promptType` | `string` | 否 | 默认 `INTERVIEW_QUESTION_GENERATE` |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `aiQuestion` | `string` | AI 生成后的提问内容 |
| `questionTitle` | `string` | 问题标题 |
| `questionContent` | `string` | 问题内容 |
| `suggestedAnswer` | `string` | 建议答案，可选 |
| `promptTemplateId` | `long` | Prompt 模板 ID |
| `aiCallLogId` | `long` | AI 调用日志 ID |
| `modelName` | `string` | 模型名称 |
| `tokenUsage` | `AiTokenUsageVO` | Token 用量，可选，V1 可不落库 |

业务规则：

1. 仅 interview-service 内部调用。
2. 如果传入题库题目，AI 可以基于题目生成更自然的面试提问。
3. 如果不传题库题目，V1 不建议完全由 AI 自由生成题目，除非面试流程明确允许。
4. 该接口不保存 `interview_message`，消息保存由 interview-service 完成。
5. 该接口必须记录 AI 调用日志。
6. Prompt 模板必须启用。
7. AI 调用失败返回 AI 模块错误码，由 interview-service 决定是否降级。

### 5.2 POST /inner/ai/interview/evaluate

接口说明：回答评分。

调用方：interview-service。

用途：根据题目、参考答案、用户回答、上下文进行评分和点评。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/inner/ai/interview/evaluate` |
| 是否对前端暴露 | 否 |
| 关联表 | `prompt_template`、`ai_call_log` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID |
| `interviewId` | `long` | 是 | 面试 ID |
| `stageId` | `long` | 是 | 阶段 ID |
| `questionMessageId` | `long` | 是 | 问题消息 ID |
| `answerMessageId` | `long` | 否 | 回答消息 ID，创建回答消息后可传 |
| `stageType` | `string` | 否 | 阶段类型 |
| `questionId` | `long` | 否 | 题库题目 ID |
| `questionTitle` | `string` | 否 | 题目标题 |
| `questionContent` | `string` | 是 | 问题内容 |
| `referenceAnswer` | `string` | 否 | 参考答案 |
| `analysis` | `string` | 否 | 解析 |
| `userAnswer` | `string` | 是 | 用户回答 |
| `answerDurationSeconds` | `int` | 否 | 回答耗时 |
| `resumeSnapshot` | `AiResumeSnapshotDTO` | 否 | 简历快照 |
| `previousMessages` | `array<AiPreviousMessageDTO>` | 否 | 历史消息摘要 |
| `promptType` | `string` | 否 | 默认 `INTERVIEW_ANSWER_EVALUATE` |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `score` | `decimal` | 分数，建议 0-100 |
| `level` | `string` | 表现等级 |
| `comment` | `string` | 点评 |
| `advantage` | `string` | 优点 |
| `weakness` | `string` | 不足 |
| `suggestion` | `string` | 建议 |
| `needFollowUp` | `boolean` | AI 是否建议追问 |
| `followUpDirection` | `string` | 追问方向，可选 |
| `promptTemplateId` | `long` | Prompt 模板 ID |
| `aiCallLogId` | `long` | AI 调用日志 ID |
| `modelName` | `string` | 模型名称 |
| `tokenUsage` | `AiTokenUsageVO` | Token 用量，可选 |

业务规则：

1. 评分建议使用 0-100 分。
2. `level` 可以返回 `EXCELLENT` / `GOOD` / `NORMAL` / `WEAK`。
3. `needFollowUp` 只表示 AI 建议是否追问，最终是否追问由 interview-service 决定。
4. ai-service 不负责推进下一题、下一阶段或结束面试。
5. 该接口不保存 `interview_message`。
6. 该接口必须记录 AI 调用日志。
7. AI 评分失败时返回明确错误，不返回伪造评分。

### 5.3 POST /inner/ai/interview/follow-up

接口说明：生成动态追问。

调用方：interview-service。

用途：根据用户回答、评分结果和追问方向生成追问问题。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/inner/ai/interview/follow-up` |
| 是否对前端暴露 | 否 |
| 关联表 | `prompt_template`、`ai_call_log` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID |
| `interviewId` | `long` | 是 | 面试 ID |
| `stageId` | `long` | 是 | 阶段 ID |
| `sourceQuestionMessageId` | `long` | 是 | 来源问题消息 ID |
| `sourceAnswerMessageId` | `long` | 是 | 来源回答消息 ID |
| `sourceEvaluation` | `EvaluateAnswerVO` | 是 | 来源评分结果 |
| `questionTitle` | `string` | 否 | 原问题标题 |
| `questionContent` | `string` | 是 | 原问题内容 |
| `userAnswer` | `string` | 是 | 用户回答 |
| `followUpDirection` | `string` | 否 | 追问方向 |
| `previousFollowUpCount` | `int` | 是 | 已追问次数 |
| `maxFollowUpCount` | `int` | 是 | 最大追问次数 |
| `stageType` | `string` | 否 | 阶段类型 |
| `promptType` | `string` | 否 | 默认 `INTERVIEW_FOLLOW_UP_GENERATE` |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `followUpQuestion` | `string` | 追问问题 |
| `followUpReason` | `string` | 追问原因 |
| `difficulty` | `string` | 追问难度 |
| `promptTemplateId` | `long` | Prompt 模板 ID |
| `aiCallLogId` | `long` | AI 调用日志 ID |
| `modelName` | `string` | 模型名称 |
| `tokenUsage` | `AiTokenUsageVO` | Token 用量，可选 |

业务规则：

1. 仅 interview-service 内部调用。
2. 是否允许追问由 interview-service 根据阶段配置、追问次数和评分结果决定。
3. ai-service 只生成追问内容。
4. 该接口不保存 `interview_message`。
5. 该接口必须记录 AI 调用日志。
6. 追问内容应围绕原问题和用户回答，不应偏离当前面试阶段。

### 5.4 POST /inner/ai/interview/report

接口说明：生成面试报告。

调用方：interview-service。

用途：根据整场面试的阶段、问题、回答、评分和简历快照生成报告内容。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/inner/ai/interview/report` |
| 是否对前端暴露 | 否 |
| 关联表 | `prompt_template`、`ai_call_log` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID |
| `interviewId` | `long` | 是 | 面试 ID |
| `interviewName` | `string` | 否 | 面试名称 |
| `targetPosition` | `string` | 否 | 目标岗位 |
| `resumeSnapshot` | `AiResumeSnapshotDTO` | 否 | 简历快照 |
| `stageResults` | `array<AiStageResultDTO>` | 是 | 阶段结果 |
| `messages` | `array<AiReportMessageDTO>` | 是 | 面试消息摘要 |
| `totalDurationSeconds` | `int` | 否 | 总耗时 |
| `promptType` | `string` | 否 | 默认 `INTERVIEW_REPORT_GENERATE` |

`stageResults` 建议包含：

| 字段 | 类型 | 说明 |
|---|---|---|
| `stageId` | `long` | 阶段 ID |
| `stageType` | `string` | 阶段类型 |
| `stageName` | `string` | 阶段名称 |
| `score` | `decimal` | 阶段得分 |
| `questionCount` | `int` | 题目数量 |
| `summary` | `string` | 阶段摘要 |

`messages` 建议包含：

| 字段 | 类型 | 说明 |
|---|---|---|
| `stageId` | `long` | 阶段 ID |
| `questionContent` | `string` | 问题内容 |
| `userAnswer` | `string` | 用户回答 |
| `score` | `decimal` | 得分 |
| `evaluationComment` | `string` | 评分点评 |
| `weakness` | `string` | 不足 |
| `suggestion` | `string` | 建议 |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `totalScore` | `decimal` | 总分 |
| `summary` | `string` | 总结 |
| `stageReports` | `array<AiStageReportVO>` | 阶段报告 |
| `strengths` | `string` | 优势 |
| `weaknesses` | `string` | 不足 |
| `suggestions` | `string` | 复习建议 |
| `recommendedLearningPath` | `string` | 推荐学习路径，可选 |
| `recommendedQuestions` | `array<object>` | 推荐题目，可选 |
| `promptTemplateId` | `long` | Prompt 模板 ID |
| `aiCallLogId` | `long` | AI 调用日志 ID |
| `modelName` | `string` | 模型名称 |
| `tokenUsage` | `AiTokenUsageVO` | Token 用量，可选 |

业务规则：

1. 仅 interview-service 内部调用。
2. ai-service 只生成报告内容，不保存 `interview_report`。
3. `interview_report` 由 interview-service 保存。
4. 报告生成失败时返回明确错误信息。
5. 该接口必须记录 AI 调用日志。
6. V1 可同步生成报告，不设计异步任务。
7. `recommendedQuestions` 可以只返回推荐方向，不强制调用 question-service 推荐题目。

---

## 6. 管理端 Prompt 模板接口设计

### 6.1 GET /admin/ai/prompts

接口说明：Prompt 模板分页列表。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/admin/ai/prompts` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 关联表 | `prompt_template` |

查询参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 模板名称、模板编码关键字 |
| `promptType` | `string` | 否 | Prompt 类型，对应 `scene` |
| `status` | `int` | 否 | 状态 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 模板 ID |
| `promptName` | `string` | 模板名称，对应 `template_name` |
| `templateCode` | `string` | 模板编码 |
| `promptType` | `string` | Prompt 类型，对应 `scene` |
| `version` | `string` | 版本，当前表未设计，V1 可由 `template_code` 或描述表达 |
| `status` | `int` | 状态 |
| `description` | `string` | 描述，当前表未设计，可从变量说明或扩展字段展示 |
| `updatedAt` | `string` | 更新时间 |
| `createdAt` | `string` | 创建时间 |

业务规则：

1. 仅管理员可访问。
2. 不返回 API Key。
3. 支持按类型、状态、关键字筛选。
4. 默认按 `updatedAt` 倒序。
5. 列表页不返回完整 Prompt 内容，避免内容过长。

### 6.2 GET /admin/ai/prompts/{id}

接口说明：Prompt 模板详情。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/admin/ai/prompts/{id}` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 关联表 | `prompt_template` |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 模板 ID |
| `promptName` | `string` | 模板名称 |
| `templateCode` | `string` | 模板编码 |
| `promptType` | `string` | Prompt 类型，对应 `scene` |
| `templateContent` | `string` | 模板内容，对应 `template_content` |
| `systemPrompt` | `string` | 可选展示字段，V1 可不拆分 |
| `userPromptTemplate` | `string` | 可选展示字段，V1 可不拆分 |
| `variables` | `string` | 变量说明，对应 `variable_description` |
| `version` | `string` | 版本，V1 可选展示字段 |
| `status` | `int` | 状态 |
| `description` | `string` | 描述，V1 可选展示字段 |
| `createdAt` | `string` | 创建时间 |
| `updatedAt` | `string` | 更新时间 |

业务规则：

1. 仅管理员可访问。
2. 可查看 Prompt 模板内容。
3. 变量说明用于前端管理展示和后端渲染校验。
4. 不返回 API Key。

### 6.3 POST /admin/ai/prompts

接口说明：新增 Prompt 模板。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/admin/ai/prompts` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 关联表 | `prompt_template` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `promptName` | `string` | 是 | 模板名称，对应 `template_name` |
| `templateCode` | `string` | 是 | 模板编码，对应 `template_code` |
| `promptType` | `string` | 是 | Prompt 类型，对应 `scene` |
| `templateContent` | `string` | 是 | 模板内容 |
| `systemPrompt` | `string` | 否 | 可选，若前端拆分编辑，后端可合并到 `templateContent` |
| `userPromptTemplate` | `string` | 否 | 可选，若前端拆分编辑，后端可合并到 `templateContent` |
| `variables` | `string` | 否 | 变量说明，对应 `variable_description` |
| `version` | `string` | 否 | 版本，V1 可不落库 |
| `status` | `int` | 是 | 状态 |
| `description` | `string` | 否 | 描述，V1 可不落库 |

业务规则：

1. 仅管理员可操作。
2. `promptType` 必须合法。
3. `templateCode` 必须唯一。
4. 同一 `promptType` 可以有多个模板，但同一时间建议只有一个启用模板。
5. 如果新增模板时 `status = 1`，V1 推荐自动禁用同类型其他模板，确保同一场景只有一个主启用模板。
6. Prompt 内容不能为空。

### 6.4 PUT /admin/ai/prompts/{id}

接口说明：修改 Prompt 模板。

| 项目 | 内容 |
|---|---|
| 请求方式 | `PUT` |
| URL | `/admin/ai/prompts/{id}` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 关联表 | `prompt_template` |

请求参数同 `POST /admin/ai/prompts`。

业务规则：

1. 仅管理员可操作。
2. 已存在模板才允许修改。
3. 修改启用模板需要谨慎，V1 可以允许直接修改，但应记录 `updatedAt`。
4. 更严谨的版本管理可后续增强，V1 不做复杂版本发布流程。

### 6.5 PUT /admin/ai/prompts/{id}/status

接口说明：启用 / 禁用 Prompt 模板。

| 项目 | 内容 |
|---|---|
| 请求方式 | `PUT` |
| URL | `/admin/ai/prompts/{id}/status` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 关联表 | `prompt_template` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `status` | `int` | 是 | `1` 启用，`0` 禁用 |

业务规则：

1. 仅管理员可操作。
2. 启用时需要确保同一 `promptType` 下只有一个主启用模板。
3. V1 推荐启用当前模板并禁用同类型其他模板。
4. 禁用后，AI 能力接口不能选择该模板。
5. 如果某 `promptType` 没有启用模板，对应 AI 能力接口应返回 Prompt 不存在或未启用错误。

---

## 7. 管理端 AI 调用日志接口设计

### 7.1 GET /admin/ai/logs

接口说明：AI 调用日志分页列表。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/admin/ai/logs` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 关联表 | `ai_call_log` |

查询参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 否 | 用户 ID |
| `interviewId` | `long` | 否 | 面试 ID，对应 `business_id` |
| `callType` | `string` | 否 | 调用类型，对应 `scene` |
| `status` | `string` | 否 | 调用状态 |
| `modelName` | `string` | 否 | 模型名称 |
| `startTime` | `string` | 否 | 开始时间 |
| `endTime` | `string` | 否 | 结束时间 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 日志 ID |
| `userId` | `long` | 用户 ID |
| `interviewId` | `long` | 面试 ID，由 `business_id` 转换 |
| `callType` | `string` | 调用类型 |
| `status` | `string` | 调用状态 |
| `modelName` | `string` | 模型名称 |
| `promptTemplateId` | `long` | Prompt 模板 ID |
| `latencyMs` | `long` | 调用耗时，对应 `cost_time_ms` |
| `inputTokens` | `int` | 输入 token，可选 |
| `outputTokens` | `int` | 输出 token，可选 |
| `totalTokens` | `int` | 总 token，可选 |
| `errorMessage` | `string` | 错误信息，可截断 |
| `createdAt` | `string` | 创建时间 |

业务规则：

1. 仅管理员可访问。
2. 默认按 `createdAt` 倒序。
3. 列表页不返回完整 Prompt 和完整响应，避免内容过长。
4. 错误信息可以截断展示。
5. token 用量字段 V1 可为空。

### 7.2 GET /admin/ai/logs/{id}

接口说明：AI 调用日志详情。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/admin/ai/logs/{id}` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 关联表 | `ai_call_log`、`prompt_template` |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 日志 ID |
| `userId` | `long` | 用户 ID |
| `interviewId` | `long` | 面试 ID，由 `business_id` 转换 |
| `stageId` | `long` | 阶段 ID，可从 `request_content` 解析 |
| `messageId` | `long` | 消息 ID，可从 `request_content` 解析 |
| `callType` | `string` | 调用类型，对应 `scene` |
| `status` | `string` | 调用状态 |
| `provider` | `string` | AI 供应商，V1 可选 |
| `modelName` | `string` | 模型名称 |
| `promptTemplateId` | `long` | Prompt 模板 ID |
| `promptVersion` | `string` | Prompt 版本，V1 可选 |
| `requestParams` | `string` | 请求参数，对应 `request_content` |
| `promptContent` | `string` | 渲染后 Prompt，可包含于 `request_content` |
| `responseContent` | `string` | 响应内容 |
| `errorMessage` | `string` | 失败原因 |
| `latencyMs` | `long` | 调用耗时 |
| `inputTokens` | `int` | 输入 token，可选 |
| `outputTokens` | `int` | 输出 token，可选 |
| `totalTokens` | `int` | 总 token，可选 |
| `createdAt` | `string` | 创建时间 |

业务规则：

1. 仅管理员可访问。
2. 可查看请求、Prompt 和响应内容，用于调试。
3. 不返回 API Key。
4. 对可能包含隐私的字段进行必要脱敏。
5. 如果日志内容较长，V1 可以完整返回，后续再做折叠展示或大字段优化。

---

## 8. DTO / VO 设计

### 8.1 内部 AI 能力 DTO / VO

#### GenerateInterviewQuestionRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID |
| `interviewId` | `long` | 是 | 面试 ID |
| `stageId` | `long` | 是 | 阶段 ID |
| `stageType` | `string` | 否 | 阶段类型 |
| `questionId` | `long` | 否 | 题目 ID |
| `questionTitle` | `string` | 否 | 题目标题 |
| `questionContent` | `string` | 否 | 题目内容 |
| `referenceAnswer` | `string` | 否 | 参考答案 |
| `resumeSnapshot` | `AiResumeSnapshotDTO` | 否 | 简历快照 |
| `projectSnapshot` | `AiProjectSnapshotDTO` | 否 | 项目快照 |
| `targetPosition` | `string` | 否 | 目标岗位 |
| `difficulty` | `string` | 否 | 难度 |
| `previousMessages` | `array<AiPreviousMessageDTO>` | 否 | 历史消息 |
| `promptType` | `string` | 否 | Prompt 类型 |

#### GenerateInterviewQuestionVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `aiQuestion` | `string` | 是 | AI 提问 |
| `questionTitle` | `string` | 否 | 问题标题 |
| `questionContent` | `string` | 是 | 问题内容 |
| `suggestedAnswer` | `string` | 否 | 建议答案 |
| `promptTemplateId` | `long` | 是 | Prompt 模板 ID |
| `aiCallLogId` | `long` | 是 | AI 调用日志 ID |
| `modelName` | `string` | 否 | 模型名称 |
| `tokenUsage` | `AiTokenUsageVO` | 否 | Token 用量 |

#### EvaluateAnswerRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID |
| `interviewId` | `long` | 是 | 面试 ID |
| `stageId` | `long` | 是 | 阶段 ID |
| `questionMessageId` | `long` | 是 | 问题消息 ID |
| `answerMessageId` | `long` | 否 | 回答消息 ID |
| `stageType` | `string` | 否 | 阶段类型 |
| `questionId` | `long` | 否 | 题目 ID |
| `questionTitle` | `string` | 否 | 题目标题 |
| `questionContent` | `string` | 是 | 问题内容 |
| `referenceAnswer` | `string` | 否 | 参考答案 |
| `analysis` | `string` | 否 | 解析 |
| `userAnswer` | `string` | 是 | 用户回答 |
| `answerDurationSeconds` | `int` | 否 | 回答耗时 |
| `resumeSnapshot` | `AiResumeSnapshotDTO` | 否 | 简历快照 |
| `previousMessages` | `array<AiPreviousMessageDTO>` | 否 | 历史消息 |
| `promptType` | `string` | 否 | Prompt 类型 |

#### EvaluateAnswerVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `score` | `decimal` | 是 | 分数 |
| `level` | `string` | 否 | 表现等级 |
| `comment` | `string` | 是 | 点评 |
| `advantage` | `string` | 否 | 优点 |
| `weakness` | `string` | 否 | 不足 |
| `suggestion` | `string` | 否 | 建议 |
| `needFollowUp` | `boolean` | 是 | 是否建议追问 |
| `followUpDirection` | `string` | 否 | 追问方向 |
| `promptTemplateId` | `long` | 是 | Prompt 模板 ID |
| `aiCallLogId` | `long` | 是 | AI 调用日志 ID |
| `modelName` | `string` | 否 | 模型名称 |
| `tokenUsage` | `AiTokenUsageVO` | 否 | Token 用量 |

#### GenerateFollowUpRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID |
| `interviewId` | `long` | 是 | 面试 ID |
| `stageId` | `long` | 是 | 阶段 ID |
| `sourceQuestionMessageId` | `long` | 是 | 来源问题消息 ID |
| `sourceAnswerMessageId` | `long` | 是 | 来源回答消息 ID |
| `sourceEvaluation` | `EvaluateAnswerVO` | 是 | 来源评分 |
| `questionTitle` | `string` | 否 | 原问题标题 |
| `questionContent` | `string` | 是 | 原问题内容 |
| `userAnswer` | `string` | 是 | 用户回答 |
| `followUpDirection` | `string` | 否 | 追问方向 |
| `previousFollowUpCount` | `int` | 是 | 已追问次数 |
| `maxFollowUpCount` | `int` | 是 | 最大追问次数 |
| `stageType` | `string` | 否 | 阶段类型 |
| `promptType` | `string` | 否 | Prompt 类型 |

#### GenerateFollowUpVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `followUpQuestion` | `string` | 是 | 追问问题 |
| `followUpReason` | `string` | 否 | 追问原因 |
| `difficulty` | `string` | 否 | 难度 |
| `promptTemplateId` | `long` | 是 | Prompt 模板 ID |
| `aiCallLogId` | `long` | 是 | AI 调用日志 ID |
| `modelName` | `string` | 否 | 模型名称 |
| `tokenUsage` | `AiTokenUsageVO` | 否 | Token 用量 |

#### GenerateInterviewReportRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID |
| `interviewId` | `long` | 是 | 面试 ID |
| `interviewName` | `string` | 否 | 面试名称 |
| `targetPosition` | `string` | 否 | 目标岗位 |
| `resumeSnapshot` | `AiResumeSnapshotDTO` | 否 | 简历快照 |
| `stageResults` | `array<AiStageResultDTO>` | 是 | 阶段结果 |
| `messages` | `array<AiReportMessageDTO>` | 是 | 面试消息 |
| `totalDurationSeconds` | `int` | 否 | 总耗时 |
| `promptType` | `string` | 否 | Prompt 类型 |

#### GenerateInterviewReportVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `totalScore` | `decimal` | 是 | 总分 |
| `summary` | `string` | 是 | 总结 |
| `stageReports` | `array<AiStageReportVO>` | 否 | 阶段报告 |
| `strengths` | `string` | 否 | 优势 |
| `weaknesses` | `string` | 否 | 不足 |
| `suggestions` | `string` | 否 | 建议 |
| `recommendedLearningPath` | `string` | 否 | 推荐学习路径 |
| `recommendedQuestions` | `array<object>` | 否 | 推荐题目 |
| `promptTemplateId` | `long` | 是 | Prompt 模板 ID |
| `aiCallLogId` | `long` | 是 | AI 调用日志 ID |
| `modelName` | `string` | 否 | 模型名称 |
| `tokenUsage` | `AiTokenUsageVO` | 否 | Token 用量 |

#### AiTokenUsageVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `inputTokens` | `int` | 否 | 输入 token |
| `outputTokens` | `int` | 否 | 输出 token |
| `totalTokens` | `int` | 否 | 总 token |

### 8.2 Prompt 管理 DTO / VO

#### AdminPromptQueryRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 模板名称或编码关键字 |
| `promptType` | `string` | 否 | Prompt 类型 |
| `status` | `int` | 否 | 状态 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

#### AdminPromptPageVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 模板 ID |
| `promptName` | `string` | 是 | 模板名称 |
| `templateCode` | `string` | 是 | 模板编码 |
| `promptType` | `string` | 是 | Prompt 类型 |
| `version` | `string` | 否 | 版本 |
| `status` | `int` | 是 | 状态 |
| `description` | `string` | 否 | 描述 |
| `updatedAt` | `string` | 是 | 更新时间 |
| `createdAt` | `string` | 是 | 创建时间 |

#### AdminPromptDetailVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 模板 ID |
| `promptName` | `string` | 是 | 模板名称 |
| `templateCode` | `string` | 是 | 模板编码 |
| `promptType` | `string` | 是 | Prompt 类型 |
| `templateContent` | `string` | 是 | 模板内容 |
| `systemPrompt` | `string` | 否 | 系统提示词，可选展示字段 |
| `userPromptTemplate` | `string` | 否 | 用户提示词模板，可选展示字段 |
| `variables` | `string` | 否 | 变量说明 |
| `version` | `string` | 否 | 版本 |
| `status` | `int` | 是 | 状态 |
| `description` | `string` | 否 | 描述 |
| `createdAt` | `string` | 是 | 创建时间 |
| `updatedAt` | `string` | 是 | 更新时间 |

#### SavePromptRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `promptName` | `string` | 是 | 模板名称 |
| `templateCode` | `string` | 是 | 模板编码 |
| `promptType` | `string` | 是 | Prompt 类型 |
| `templateContent` | `string` | 是 | 模板内容 |
| `systemPrompt` | `string` | 否 | 系统提示词，可选 |
| `userPromptTemplate` | `string` | 否 | 用户提示词模板，可选 |
| `variables` | `string` | 否 | 变量说明 |
| `version` | `string` | 否 | 版本 |
| `status` | `int` | 是 | 状态 |
| `description` | `string` | 否 | 描述 |

#### UpdatePromptStatusRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `status` | `int` | 是 | `1` 启用，`0` 禁用 |

### 8.3 AI 日志 DTO / VO

#### AdminAiLogQueryRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 否 | 用户 ID |
| `interviewId` | `long` | 否 | 面试 ID |
| `callType` | `string` | 否 | 调用类型 |
| `status` | `string` | 否 | 调用状态 |
| `modelName` | `string` | 否 | 模型名称 |
| `startTime` | `string` | 否 | 开始时间 |
| `endTime` | `string` | 否 | 结束时间 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

#### AdminAiLogPageVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 日志 ID |
| `userId` | `long` | 否 | 用户 ID |
| `interviewId` | `long` | 否 | 面试 ID |
| `callType` | `string` | 是 | 调用类型 |
| `status` | `string` | 是 | 调用状态 |
| `modelName` | `string` | 否 | 模型名称 |
| `promptTemplateId` | `long` | 否 | Prompt 模板 ID |
| `latencyMs` | `long` | 否 | 调用耗时 |
| `inputTokens` | `int` | 否 | 输入 token |
| `outputTokens` | `int` | 否 | 输出 token |
| `totalTokens` | `int` | 否 | 总 token |
| `errorMessage` | `string` | 否 | 错误信息 |
| `createdAt` | `string` | 是 | 创建时间 |

#### AdminAiLogDetailVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 日志 ID |
| `userId` | `long` | 否 | 用户 ID |
| `interviewId` | `long` | 否 | 面试 ID |
| `stageId` | `long` | 否 | 阶段 ID |
| `messageId` | `long` | 否 | 消息 ID |
| `callType` | `string` | 是 | 调用类型 |
| `status` | `string` | 是 | 调用状态 |
| `provider` | `string` | 否 | AI 供应商 |
| `modelName` | `string` | 否 | 模型名称 |
| `promptTemplateId` | `long` | 否 | Prompt 模板 ID |
| `promptVersion` | `string` | 否 | Prompt 版本 |
| `requestParams` | `string` | 否 | 请求参数 |
| `promptContent` | `string` | 否 | 渲染后 Prompt |
| `responseContent` | `string` | 否 | 响应内容 |
| `errorMessage` | `string` | 否 | 错误信息 |
| `latencyMs` | `long` | 否 | 调用耗时 |
| `inputTokens` | `int` | 否 | 输入 token |
| `outputTokens` | `int` | 否 | 输出 token |
| `totalTokens` | `int` | 否 | 总 token |
| `createdAt` | `string` | 是 | 创建时间 |

### 8.4 上下文对象

#### AiResumeSnapshotDTO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `resumeId` | `long` | 否 | 简历 ID |
| `resumeName` | `string` | 否 | 简历名称 |
| `targetPosition` | `string` | 否 | 求职方向 |
| `skills` | `string` | 否 | 技能栈 |
| `workSummary` | `string` | 否 | 工作经历摘要 |
| `education` | `string` | 否 | 教育经历 |
| `projects` | `array<AiProjectSnapshotDTO>` | 否 | 项目经历 |

#### AiProjectSnapshotDTO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `projectName` | `string` | 否 | 项目名称 |
| `projectTime` | `string` | 否 | 项目时间 |
| `projectBackground` | `string` | 否 | 项目背景 |
| `techStack` | `string` | 否 | 技术栈 |
| `responsibility` | `string` | 否 | 个人职责 |
| `coreFeatures` | `string` | 否 | 核心功能 |
| `technicalChallenges` | `string` | 否 | 技术难点 |
| `optimizationResult` | `string` | 否 | 优化成果 |
| `extraInfo` | `string` | 否 | 补充说明 |

#### AiPreviousMessageDTO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `role` | `string` | 是 | `AI` / `USER` |
| `content` | `string` | 是 | 消息内容 |
| `score` | `decimal` | 否 | 得分 |
| `createdAt` | `string` | 否 | 创建时间 |

#### AiStageResultDTO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `stageId` | `long` | 是 | 阶段 ID |
| `stageType` | `string` | 否 | 阶段类型 |
| `stageName` | `string` | 是 | 阶段名称 |
| `score` | `decimal` | 否 | 阶段得分 |
| `questionCount` | `int` | 是 | 题目数量 |
| `summary` | `string` | 否 | 阶段摘要 |

#### AiReportMessageDTO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `stageId` | `long` | 是 | 阶段 ID |
| `questionContent` | `string` | 是 | 问题内容 |
| `userAnswer` | `string` | 是 | 用户回答 |
| `score` | `decimal` | 否 | 得分 |
| `evaluationComment` | `string` | 否 | 点评 |
| `weakness` | `string` | 否 | 不足 |
| `suggestion` | `string` | 否 | 建议 |

#### AiStageReportVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `stageId` | `long` | 是 | 阶段 ID |
| `stageType` | `string` | 否 | 阶段类型 |
| `stageName` | `string` | 是 | 阶段名称 |
| `score` | `decimal` | 否 | 阶段得分 |
| `summary` | `string` | 否 | 阶段总结 |
| `weaknesses` | `string` | 否 | 阶段不足 |
| `suggestions` | `string` | 否 | 阶段建议 |

---

## 9. 参数校验规则

| 参数 | 校验规则 |
|---|---|
| `pageNo` | 最小 1 |
| `pageSize` | 最大 100 |
| `promptName` | 不能为空，最大 128 |
| `templateCode` | 不能为空，最大 128，且唯一 |
| `promptType` | 必须合法，对应 `scene` |
| `templateContent` | 不能为空 |
| `systemPrompt` | 如果前端拆分传入，不能为空字符串 |
| `userPromptTemplate` | 如果前端拆分传入，不能为空字符串 |
| `variables` | 建议为合法 JSON 或清晰的变量说明文本 |
| `status` | 只能是 `0` / `1` |
| `userAnswer` | 不能为空，按数据库 `LONGTEXT` 或业务限制控制，接口建议最大 10000 |
| `questionContent` | 评分和追问场景不能为空；问题生成场景可为空但必须有题目、简历或阶段上下文 |
| `interviewId` | AI 能力接口必填 |
| `stageId` | 评分、追问场景下建议必填 |
| `messageId` | 评分、追问场景下建议必填，具体字段为 `questionMessageId`、`answerMessageId` |
| `previousFollowUpCount` | 不能小于 0 |
| `maxFollowUpCount` | 不能小于 0 |
| `callType` | 必须合法，对应 `scene` |
| `startTime` / `endTime` | 格式 `yyyy-MM-dd HH:mm:ss` |
| `endTime` | 不应早于 `startTime` |

---

## 10. 错误码设计

AI 模块错误码范围：`46000-46999`。

| code | message | 说明 |
|---:|---|---|
| `46001` | AI 调用失败 | 大模型调用失败 |
| `46002` | AI 调用超时 | 调用超过超时时间 |
| `46003` | AI 响应解析失败 | 响应无法解析为结构化结果 |
| `46004` | Prompt 模板不存在 | 未找到对应模板 |
| `46005` | Prompt 模板未启用 | 模板被禁用或无启用模板 |
| `46006` | Prompt 类型非法 | `promptType` 或 `scene` 非法 |
| `46007` | Prompt 内容为空 | 模板内容为空 |
| `46008` | Prompt 变量缺失 | 渲染必需变量缺失 |
| `46009` | AI 供应商配置缺失 | 未配置供应商 |
| `46010` | AI 模型配置缺失 | 未配置模型名称或模型参数 |
| `46011` | AI 调用日志不存在 | 日志 ID 不存在 |
| `46012` | Prompt 状态非法 | status 非 0/1 |
| `46013` | 同类型 Prompt 启用模板冲突 | 同一 scene 存在多个启用模板 |
| `46014` | 报告生成失败 | 生成面试报告失败 |
| `46015` | 追问生成失败 | 生成追问失败 |
| `46016` | 回答评分失败 | 回答评分失败 |
| `46017` | 问题生成失败 | 生成面试问题失败 |
| `46018` | Prompt 模板编码已存在 | templateCode 重复 |
| `46019` | AI 请求上下文缺失 | 内部调用缺少必要上下文 |

复用通用错误码：

| code | message | 说明 |
|---:|---|---|
| `40001` | 参数校验失败 | 请求参数不合法 |
| `41000` | 未登录 | 管理端接口未登录 |
| `41003` | 无访问权限 | 非管理员访问管理端接口 |
| `45000-45999` | 面试模块错误 | interview-service 处理流程时透传或转换 |

---

## 11. 数据库表关系

本模块涉及：

| 表名 | 说明 |
|---|---|
| `prompt_template` | Prompt 模板 |
| `ai_call_log` | AI 调用日志 |

关系说明：

1. ai-service 是这些表的数据归属服务。
2. `prompt_template` 用于保存不同类型 AI 能力的 Prompt 模板。
3. `ai_call_log` 用于记录 AI 请求、响应、状态、耗时、模型、失败原因等。
4. interview-service 不直接访问 `prompt_template` 和 `ai_call_log` 表。
5. Prompt 模板启用状态影响 `/inner/ai/**` 能力接口。
6. AI 调用日志由 ai-service 在每次调用 AI 模型时写入。
7. API Key 不应存储在 `prompt_template` 或 `ai_call_log` 中，建议通过配置中心、环境变量或 system_config 管理，具体实现阶段再定。

关系示意：

```text
prompt_template
  └── ai_call_log.prompt_template_id
```

---

## 12. AI 调用流程说明

### 12.1 回答评分流程

```text
interview-service 提交评分上下文
  ↓
ai-service 查询启用的 INTERVIEW_ANSWER_EVALUATE Prompt
  ↓
渲染 Prompt 变量
  ↓
调用大模型
  ↓
解析 AI 响应为结构化评分
  ↓
记录 ai_call_log
  ↓
返回评分结果给 interview-service
  ↓
interview-service 保存评分结果并决定 nextAction
```

### 12.2 追问生成流程

```text
interview-service 判断需要追问
  ↓
ai-service 查询启用的 INTERVIEW_FOLLOW_UP_GENERATE Prompt
  ↓
渲染上下文
  ↓
调用大模型
  ↓
解析追问内容
  ↓
记录 ai_call_log
  ↓
返回追问给 interview-service
  ↓
interview-service 保存追问消息
```

### 12.3 报告生成流程

```text
interview-service 结束面试
  ↓
整理整场面试上下文
  ↓
ai-service 查询启用的 INTERVIEW_REPORT_GENERATE Prompt
  ↓
调用大模型生成报告
  ↓
解析报告结构
  ↓
记录 ai_call_log
  ↓
返回报告内容
  ↓
interview-service 保存 interview_report
```

---

## 13. Prompt 变量建议

### 13.1 INTERVIEW_QUESTION_GENERATE

建议支持变量：

| 变量名 | 说明 |
|---|---|
| `targetPosition` | 目标岗位 |
| `stageType` | 阶段类型 |
| `difficulty` | 难度 |
| `questionTitle` | 题目标题 |
| `questionContent` | 题目内容 |
| `referenceAnswer` | 参考答案 |
| `resumeSummary` | 简历摘要 |
| `projectSummary` | 项目摘要 |

### 13.2 INTERVIEW_ANSWER_EVALUATE

建议支持变量：

| 变量名 | 说明 |
|---|---|
| `questionContent` | 问题内容 |
| `referenceAnswer` | 参考答案 |
| `userAnswer` | 用户回答 |
| `stageType` | 阶段类型 |
| `targetPosition` | 目标岗位 |
| `resumeSummary` | 简历摘要 |
| `previousMessages` | 历史问答摘要 |

### 13.3 INTERVIEW_FOLLOW_UP_GENERATE

建议支持变量：

| 变量名 | 说明 |
|---|---|
| `questionContent` | 原问题 |
| `userAnswer` | 用户回答 |
| `evaluationComment` | AI 点评 |
| `weakness` | 暴露的问题 |
| `followUpDirection` | 追问方向 |
| `previousFollowUpCount` | 已追问次数 |

### 13.4 INTERVIEW_REPORT_GENERATE

建议支持变量：

| 变量名 | 说明 |
|---|---|
| `targetPosition` | 目标岗位 |
| `resumeSummary` | 简历摘要 |
| `stageResults` | 阶段结果 |
| `qaRecords` | 问答记录 |
| `totalDuration` | 面试总时长 |
| `averageScore` | 平均得分 |
| `weaknessSummary` | 薄弱点摘要 |

变量规则：

1. 变量命名应保持稳定。
2. Prompt 模板缺少必需变量时，应返回 Prompt 变量缺失错误。
3. 变量渲染失败应记录 AI 调用失败日志。
4. `variable_description` 字段用于记录变量说明和管理端展示。

---

## 14. 前端页面对应关系

| 前端页面 | 关联接口 |
|---|---|
| AI 面试房间页 | 不直接调用 ai-service，由 interview-service 间接调用 |
| 面试报告页 | 不直接调用 ai-service，由 interview-service 间接调用 |
| Prompt 模板管理页 | `GET /admin/ai/prompts`、`GET /admin/ai/prompts/{id}`、`POST /admin/ai/prompts`、`PUT /admin/ai/prompts/{id}`、`PUT /admin/ai/prompts/{id}/status` |
| AI 调用日志页 | `GET /admin/ai/logs`、`GET /admin/ai/logs/{id}` |

说明：

1. 用户端前端永远不直接访问 `/inner/ai/**`。
2. 管理端只访问 Prompt 管理和日志查询接口。
3. AI 能力接口只给 interview-service 内部使用。

---

## 15. 注意事项

1. ai-service 只提供 AI 能力，不编排面试流程。
2. `/inner/ai/**` 不对前端暴露。
3. 前端不能直接调用 AI 评分、追问、报告接口。
4. Prompt 模板和 AI 日志只允许管理员访问。
5. AI 调用失败必须记录失败日志。
6. AI 调用失败不能返回伪造结果。
7. API Key 不允许返回前端。
8. API Key 不允许写入普通日志。
9. V1 不做 SSE。
10. V1 不做 WebSocket。
11. V1 不做异步任务。
12. V1 不做 AI 题目批量生成。
13. V1 不做 AI 生成题审核。
14. V1 不做简历 AI 优化。
15. V1 报告生成由 interview-service 发起，ai-service 只返回报告内容。
16. `interview_report` 由 interview-service 保存，不由 ai-service 保存。
17. 当前 AI 表设计不保存 token 用量、provider、Prompt 版本历史；如实现需要，应先小范围修订数据库设计。
