# CodeCoachAI V1 题库接口设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 题库接口设计 |
| 项目阶段 | V1 |
| 最高依据 | 接口设计/CodeCoachAI_V1_接口设计总览.md |
| 参考文档 | PRD/CodeCoachAI_PRD_V1_微服务分层架构版.md、数据库设计/CodeCoachAI_V1_数据库设计总览.md、数据库设计/CodeCoachAI_V1_题库表设计.md、数据库设计/CodeCoachAI_V1_刷题表设计.md |
| 文档用途 | 详细设计 question-service 在 V1 阶段的用户端、管理端和内部 Feign 接口 |

本文档只设计 CodeCoachAI V1 题库相关接口，不提前设计 V2 / V3 功能。

涉及服务：

| 服务 | 说明 |
|---|---|
| `codecoachai-question` | 题库、分类、标签、问题组、刷题、收藏、错题、掌握状态的数据归属服务 |
| `codecoachai-interview` | 通过 `/inner/questions/**` 内部接口抽题和查询题目详情 |
| `codecoachai-gateway` | 对外统一入口，不对外暴露 `/inner/**` |
| `codecoachai-auth` / `codecoachai-user` | 提供登录态、用户上下文和角色权限 |

---

## 2. 模块职责

question-service 负责：

1. 用户端题库查询。
2. 题目详情查询。
3. 用户提交答案。
4. 用户答题记录保存。
5. 错题记录维护。
6. 收藏题目。
7. 掌握状态维护。
8. 管理端题目维护。
9. 管理端分类维护。
10. 管理端标签维护。
11. 管理端问题组维护。
12. 为 interview-service 提供面试抽题内部接口。

question-service 不负责：

| 能力 | V1 处理方式 |
|---|---|
| AI 生成题目 | V1 不设计 AI 题目批量生成接口 |
| AI 评分 | 刷题提交答案不调用 AI；AI 面试评分由 interview-service 调用 ai-service |
| 面试流程编排 | 归属 interview-service |
| 简历数据 | 归属 resume-service |
| Elasticsearch 搜索 | V1 使用 MySQL 条件查询 |
| 文件导入导出 | V1 不设计 Excel 导入导出和文件上传 |
| 独立刷题服务 | V1 不拆 practice-service，刷题能力归 question-service |

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

用户端和管理端接口统一使用：

```text
Authorization: Bearer {token}
```

### 3.4 用户端权限

用户端接口要求：

1. 必须登录。
2. 只能操作当前登录用户自己的答题记录、收藏记录、错题记录和掌握状态。
3. 只能访问已启用、未删除的题目、分类、标签、问题组。
4. 不允许通过用户端接口查看禁用题目。

### 3.5 管理端权限

管理端接口统一使用 `/admin` 前缀，要求：

1. 必须登录。
2. 必须具备 `ADMIN` 角色。
3. 管理员可以查看启用和禁用数据。
4. 管理端默认不返回已逻辑删除数据。

### 3.6 数据归属校验

| 数据 | 归属规则 |
|---|---|
| `user_question_record` | 只能由当前用户创建和查询自己的记录 |
| `wrong_question` | 只能查询当前用户自己的错题 |
| `favorite_question` | 只能查询和修改当前用户自己的收藏 |
| `user_question_mastery` | 只能查询和修改当前用户自己的掌握状态 |
| `question` / `question_category` / `question_tag` / `question_group` | 管理端维护，用户端只读启用数据 |

### 3.7 逻辑删除规则

V1 使用 `deleted` 字段进行逻辑删除：

| 值 | 说明 |
|---|---|
| `0` | 未删除 |
| `1` | 已删除 |

查询规则：

1. 用户端只查询 `deleted = 0` 的数据。
2. 管理端默认只查询 `deleted = 0` 的数据。
3. 删除题目、分类、标签、问题组均为逻辑删除。
4. 历史答题记录、收藏记录、错题记录建议保留，不因题目删除而物理删除。

### 3.8 启用 / 禁用规则

| 数据 | 用户端行为 | 管理端行为 |
|---|---|---|
| 题目禁用 | 不可见，不可答题，不参与面试抽题 | 可查看、可重新启用 |
| 分类禁用 | 分类不可见，该分类下题目不在用户端展示 | 可查看、可重新启用 |
| 标签禁用 | 标签筛选不可见 | 可查看、可重新启用 |
| 问题组禁用 | 创建面试时不可选，不参与指定问题组抽题 | 可查看、可重新启用 |

### 3.9 `/inner/**` 内部接口安全约束

`/inner/**` 接口只允许服务内部 Feign 调用：

1. Gateway 不对外暴露 `/inner/**`。
2. 前端不能直接访问 `/inner/questions/**`。
3. interview-service 只能通过 `/inner/questions/**` 获取抽题和题目详情能力。
4. 如果内部调用经过 Gateway，需要内部调用标识或服务间鉴权。
5. 内部接口仍需遵守业务规则，不能随意绕过启用、删除、题目归属等校验。

---

## 4. 枚举设计

### 4.1 QuestionDifficulty

| 值 | 枚举名 | 说明 |
|---|---|---|
| `EASY` | 简单 | 基础题 |
| `MEDIUM` | 中等 | 常规面试题 |
| `HARD` | 困难 | 深度题或场景题 |

### 4.2 QuestionStatus

| 值 | 枚举名 | 说明 |
|---|---|---|
| `1` | `ENABLED` | 启用 |
| `0` | `DISABLED` | 禁用 |

### 4.3 MasteryStatus

| 值 | 说明 |
|---|---|
| `MASTERED` | 已掌握 |
| `VAGUE` | 模糊 |
| `UNKNOWN` | 未掌握 |

### 4.4 AnswerResult

| 值 | 说明 |
|---|---|
| `CORRECT` | 答对 |
| `WRONG` | 答错 |
| `PARTIAL` | 部分正确 |
| `UNKNOWN` | 未判定 |

说明：V1 用户刷题提交答案可以先由用户自评或简单保存，不做 AI 判分。AI 面试评分归 interview-service 调用 ai-service 完成。

---

## 5. 用户端接口设计

### 5.1 GET /questions

接口说明：题目列表。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/questions` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `question`、`question_category`、`question_tag`、`question_tag_relation`、`favorite_question`、`user_question_record`、`user_question_mastery` |

查询参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 标题关键字 |
| `categoryId` | `long` | 否 | 分类 ID |
| `tagIds` | `array<long>` | 否 | 标签 ID 列表 |
| `difficulty` | `string` | 否 | `EASY` / `MEDIUM` / `HARD` |
| `masteryStatus` | `string` | 否 | `MASTERED` / `VAGUE` / `UNKNOWN` |
| `favoriteOnly` | `boolean` | 否 | 是否只看收藏 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 题目 ID |
| `title` | `string` | 题目标题 |
| `categoryId` | `long` | 分类 ID |
| `categoryName` | `string` | 分类名称 |
| `difficulty` | `string` | 难度 |
| `tags` | `array<TagVO>` | 标签列表 |
| `favorite` | `boolean` | 当前用户是否收藏 |
| `masteryStatus` | `string` | 当前用户掌握状态 |
| `answered` | `boolean` | 当前用户是否答过 |
| `createdAt` | `string` | 创建时间 |

业务规则：

1. 只查询启用、未删除题目。
2. 只查询启用、未删除分类下的题目。
3. 支持 MySQL 条件查询，V1 不使用 Elasticsearch。
4. 普通用户只能看到用户端可见题目。
5. 是否收藏、是否答过、掌握状态需要基于当前用户计算。

### 5.2 GET /questions/{id}

接口说明：题目详情。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/questions/{id}` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `question`、`question_category`、`question_tag`、`question_tag_relation`、`favorite_question`、`user_question_record`、`user_question_mastery` |

路径参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 题目 ID |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 题目 ID |
| `title` | `string` | 题目标题 |
| `content` | `string` | 题目内容 |
| `answer` | `string` | 参考答案，对应数据库 `reference_answer` |
| `analysis` | `string` | 答案解析 |
| `category` | `QuestionCategoryVO` | 分类信息 |
| `tags` | `array<QuestionTagVO>` | 标签列表 |
| `difficulty` | `string` | 难度 |
| `favorite` | `boolean` | 当前用户是否收藏 |
| `masteryStatus` | `string` | 当前用户掌握状态 |
| `lastAnswer` | `string` | 当前用户最近一次答案 |
| `lastAnswerResult` | `string` | 当前用户最近一次答题结果 |

业务规则：

1. 只允许查看启用、未删除题目。
2. 普通用户不能查看禁用题目。
3. 返回参考答案和解析，用于刷题学习。
4. 记录是否收藏、掌握状态、最近一次答题信息。

### 5.3 POST /questions/{id}/answers

接口说明：提交答案。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/questions/{id}/answers` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `user_question_record`、`wrong_question`、`user_question_mastery`、`question` |

路径参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 题目 ID |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userAnswer` | `string` | 是 | 用户答案 |
| `selfResult` | `string` | 否 | `CORRECT` / `WRONG` / `PARTIAL` / `UNKNOWN` |
| `masteryStatus` | `string` | 否 | `MASTERED` / `VAGUE` / `UNKNOWN` |

响应参数：

| 字段 | 类型 | 说明 |
|---|---|---|
| `recordId` | `long` | 答题记录 ID |
| `questionId` | `long` | 题目 ID |
| `answerResult` | `string` | 答题结果 |
| `masteryStatus` | `string` | 掌握状态 |
| `wrongRecordGenerated` | `boolean` | 是否产生或更新错题记录 |
| `answeredAt` | `string` | 答题时间 |

业务规则：

1. 题目必须存在、启用且未删除。
2. 保存 `user_question_record`。
3. 如果 `selfResult` 为 `WRONG` 或 `PARTIAL`，写入或更新 `wrong_question`，`source_type` 建议为 `MANUAL`。
4. 如果 `selfResult` 为 `CORRECT`，V1 推荐保留历史答题记录，并将当前用户对应 `wrong_question.status` 更新为 `0`，使其不再出现在活跃错题列表。
5. 如果传入 `masteryStatus`，写入或更新 `user_question_mastery`。
6. V1 不调用 AI 评分。
7. 用户只能提交自己的答题记录。

### 5.4 POST /questions/{id}/favorite

接口说明：收藏题目。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/questions/{id}/favorite` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `favorite_question`、`question` |

业务规则：

1. 当前用户收藏题目。
2. 重复收藏保持幂等，返回成功。
3. 题目必须存在、启用且未删除。

### 5.5 DELETE /questions/{id}/favorite

接口说明：取消收藏。

| 项目 | 内容 |
|---|---|
| 请求方式 | `DELETE` |
| URL | `/questions/{id}/favorite` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `favorite_question` |

业务规则：

1. 当前用户取消收藏。
2. 未收藏时也返回成功，保持幂等。
3. V1 可使用逻辑删除或物理删除收藏记录，接口表现保持一致。

### 5.6 GET /questions/favorites

接口说明：收藏列表。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/questions/favorites` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `favorite_question`、`question`、`question_category`、`question_tag` |

查询参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 标题关键字 |
| `categoryId` | `long` | 否 | 分类 ID |
| `difficulty` | `string` | 否 | 难度 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

响应字段参考 `GET /questions`。

业务规则：

1. 只返回当前用户收藏。
2. 只返回未删除题目。
3. 如果题目已禁用，V1 建议用户端不展示。

### 5.7 GET /questions/wrong-records

接口说明：错题列表。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/questions/wrong-records` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `wrong_question`、`question`、`question_category`、`question_tag`、`user_question_record`、`user_question_mastery` |

查询参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 标题关键字 |
| `categoryId` | `long` | 否 | 分类 ID |
| `difficulty` | `string` | 否 | 难度 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `wrongRecordId` | `long` | 错题记录 ID |
| `questionId` | `long` | 题目 ID |
| `title` | `string` | 题目标题 |
| `categoryName` | `string` | 分类名称 |
| `difficulty` | `string` | 难度 |
| `tags` | `array<QuestionTagVO>` | 标签列表 |
| `lastAnswer` | `string` | 最近一次答案 |
| `lastAnswerResult` | `string` | 最近一次答题结果 |
| `wrongCount` | `int` | 错误或部分正确次数 |
| `lastWrongAt` | `string` | 最近错题时间 |
| `masteryStatus` | `string` | 掌握状态 |

业务规则：

1. 只返回当前用户错题。
2. 错题来源于 `POST /questions/{id}/answers`。
3. 默认只返回 `wrong_question.status = 1` 且未删除的错题。
4. V1 不设计复杂错题复习计划。

### 5.8 PUT /questions/{id}/mastery

接口说明：修改掌握状态。

| 项目 | 内容 |
|---|---|
| 请求方式 | `PUT` |
| URL | `/questions/{id}/mastery` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `user_question_mastery`、`wrong_question`、`question` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `masteryStatus` | `string` | 是 | `MASTERED` / `VAGUE` / `UNKNOWN` |

响应参数：

| 字段 | 类型 | 说明 |
|---|---|---|
| `questionId` | `long` | 题目 ID |
| `masteryStatus` | `string` | 掌握状态 |
| `updatedAt` | `string` | 更新时间 |

业务规则：

1. 只能修改当前用户自己的掌握状态。
2. 题目必须存在且未删除。
3. `masteryStatus` 只能是 `MASTERED` / `VAGUE` / `UNKNOWN`。
4. V1 建议当状态改为 `MASTERED` 时，将活跃错题状态更新为已移出；当状态为 `VAGUE` 或 `UNKNOWN` 时，可写入或恢复错题记录。

---

## 6. 管理端题目接口设计

### 6.1 GET /admin/questions

接口说明：管理端题目分页列表。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/admin/questions` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 关联表 | `question`、`question_category`、`question_group`、`question_tag`、`question_tag_relation` |

查询参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 标题关键字 |
| `categoryId` | `long` | 否 | 分类 ID |
| `tagId` | `long` | 否 | 标签 ID |
| `difficulty` | `string` | 否 | 难度 |
| `status` | `int` | 否 | `1` 启用，`0` 禁用 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 题目 ID |
| `title` | `string` | 题目标题 |
| `categoryName` | `string` | 分类名称 |
| `groupId` | `long` | 问题组 ID |
| `groupTitle` | `string` | 问题组标准标题 |
| `difficulty` | `string` | 难度 |
| `status` | `int` | 状态 |
| `tags` | `array<QuestionTagVO>` | 标签列表 |
| `createdAt` | `string` | 创建时间 |
| `updatedAt` | `string` | 更新时间 |

业务规则：

1. 管理员可查看启用和禁用题目。
2. 不返回已逻辑删除数据，除非后续单独设计回收站，V1 不做回收站。

### 6.2 POST /admin/questions

接口说明：新增题目。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/admin/questions` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 关联表 | `question`、`question_tag_relation` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `title` | `string` | 是 | 题目标题 |
| `content` | `string` | 是 | 题目内容 |
| `answer` | `string` | 是 | 参考答案，对应数据库 `reference_answer` |
| `analysis` | `string` | 否 | 答案解析 |
| `categoryId` | `long` | 是 | 分类 ID |
| `groupId` | `long` | 是 | 问题组 ID，V1 题目必须绑定问题组 |
| `difficulty` | `string` | 是 | `EASY` / `MEDIUM` / `HARD` |
| `tagIds` | `array<long>` | 否 | 标签 ID 列表 |
| `status` | `int` | 否 | `1` 启用，`0` 禁用，默认 `1` |

业务规则：

1. 分类必须存在且未删除。
2. 问题组必须存在且未删除。
3. 标签必须存在且未删除。
4. 新增题目时维护 `question_tag_relation`。
5. 默认启用。
6. 仅管理员可操作。

### 6.3 PUT /admin/questions/{id}

接口说明：修改题目。

| 项目 | 内容 |
|---|---|
| 请求方式 | `PUT` |
| URL | `/admin/questions/{id}` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 关联表 | `question`、`question_tag_relation` |

请求参数同 `POST /admin/questions`。

业务规则：

1. 题目必须存在且未删除。
2. 修改题目基础信息。
3. 修改标签关系。
4. 修改问题组时，只更新 `question.group_id`，不新增额外关系表。
5. 仅管理员可操作。

### 6.4 PUT /admin/questions/{id}/status

接口说明：启用 / 禁用题目。

| 项目 | 内容 |
|---|---|
| 请求方式 | `PUT` |
| URL | `/admin/questions/{id}/status` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 关联表 | `question` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `status` | `int` | 是 | `1` 启用，`0` 禁用 |

业务规则：

1. 题目必须存在且未删除。
2. 禁用后用户端不可见。
3. 禁用题目不应被面试抽题接口选中。
4. 管理端仍可查看。

### 6.5 DELETE /admin/questions/{id}

接口说明：删除题目。

| 项目 | 内容 |
|---|---|
| 请求方式 | `DELETE` |
| URL | `/admin/questions/{id}` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 关联表 | `question`、`question_tag_relation` |

业务规则：

1. 逻辑删除题目。
2. 删除后用户端不可见。
3. 删除后不应被面试抽题接口选中。
4. V1 建议保留历史答题记录、收藏记录、错题记录和面试记录，但不再展示为可刷题目。

---

## 7. 管理端分类接口设计

### 7.1 分类接口列表

| 接口名称 | 方法 | URL | 说明 |
|---|---|---|---|
| 分类列表 | `GET` | `/admin/question-categories` | 查询分类列表 |
| 新增分类 | `POST` | `/admin/question-categories` | 创建分类 |
| 修改分类 | `PUT` | `/admin/question-categories/{id}` | 编辑分类 |
| 启用 / 禁用分类 | `PUT` | `/admin/question-categories/{id}/status` | 修改分类状态 |
| 删除分类 | `DELETE` | `/admin/question-categories/{id}` | 逻辑删除分类 |

分类字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 分类 ID |
| `name` | `string` | 分类名称，对应数据库 `category_name` |
| `code` | `string` | 分类编码，对应数据库 `category_code` |
| `parentId` | `long` | 父分类 ID，可选，一级分类为 `0` |
| `sort` | `int` | 排序值，对应数据库 `sort_order` |
| `status` | `int` | 状态 |
| `description` | `string` | 描述，V1 可选展示字段 |
| `createdAt` | `string` | 创建时间 |
| `updatedAt` | `string` | 更新时间 |

业务规则：

1. 分类名称建议同级唯一。
2. 禁用分类后，用户端分类不可见。
3. 禁用分类后，V1 建议题目列表不展示该分类下题目。
4. 删除分类前如果存在未删除题目，V1 建议不允许删除，提示先迁移或删除题目。
5. V1 可先支持一级分类，`parentId` 作为树形分类预留。

---

## 8. 管理端标签接口设计

### 8.1 标签接口列表

| 接口名称 | 方法 | URL | 说明 |
|---|---|---|---|
| 标签列表 | `GET` | `/admin/question-tags` | 查询标签列表 |
| 新增标签 | `POST` | `/admin/question-tags` | 创建标签 |
| 修改标签 | `PUT` | `/admin/question-tags/{id}` | 编辑标签 |
| 启用 / 禁用标签 | `PUT` | `/admin/question-tags/{id}/status` | 修改标签状态 |
| 删除标签 | `DELETE` | `/admin/question-tags/{id}` | 逻辑删除标签 |

标签字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 标签 ID |
| `name` | `string` | 标签名称，对应数据库 `tag_name` |
| `code` | `string` | 标签编码，对应数据库 `tag_code` |
| `status` | `int` | 状态 |
| `description` | `string` | 描述，V1 可选展示字段 |
| `createdAt` | `string` | 创建时间 |
| `updatedAt` | `string` | 更新时间 |

业务规则：

1. 标签名称唯一。
2. 禁用标签后用户端筛选不展示。
3. 删除标签前如果有关联题目，V1 建议不允许删除，提示先解除关联。
4. 管理端仍可查看禁用标签。

---

## 9. 管理端问题组接口设计

### 9.1 问题组接口列表

| 接口名称 | 方法 | URL | 说明 |
|---|---|---|---|
| 问题组列表 | `GET` | `/admin/question-groups` | 查询问题组列表 |
| 新增问题组 | `POST` | `/admin/question-groups` | 创建问题组 |
| 修改问题组 | `PUT` | `/admin/question-groups/{id}` | 编辑问题组 |
| 启用 / 禁用问题组 | `PUT` | `/admin/question-groups/{id}/status` | 修改问题组状态 |
| 删除问题组 | `DELETE` | `/admin/question-groups/{id}` | 逻辑删除问题组 |

问题组字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 问题组 ID |
| `name` | `string` | 问题组名称，对应数据库 `canonical_title` |
| `description` | `string` | 考察意图说明 |
| `categoryId` | `long` | 主分类 ID，对应数据库 `main_category_id` |
| `knowledgePoint` | `string` | 主知识点，对应数据库 `main_knowledge_point` |
| `difficulty` | `string` | 难度 |
| `status` | `int` | 状态 |
| `questionCount` | `int` | 组内题目数量 |
| `questionIds` | `array<long>` | 组内题目 ID 列表，接口操作字段 |
| `createdAt` | `string` | 创建时间 |
| `updatedAt` | `string` | 更新时间 |

业务规则：

1. 问题组用于归并同一考察意图下的不同问法，也可在创建面试时作为固定题目范围或面试题集合。
2. V1 数据库设计没有单独的问题组和题目关系表，问题组与题目关系通过 `question.group_id` 维护。
3. `questionIds` 是接口操作字段，新增或修改问题组时可用于批量把题目归入该问题组。
4. V1 一个题目只归属一个问题组。
5. `questionIds` 对应题目必须存在且未删除。
6. 禁用问题组后前端创建面试时不可选。
7. 禁用问题组后不影响历史面试记录。
8. 删除问题组使用逻辑删除。
9. 删除问题组前如果存在未删除题目，V1 建议不允许删除，提示先迁移题目或解除归属。

---

## 10. 内部 Feign 接口设计

### 10.1 POST /inner/questions/pick-for-interview

调用方：interview-service。

用途：创建面试或进入下一阶段时抽题。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/inner/questions/pick-for-interview` |
| 是否对前端暴露 | 否 |
| 关联表 | `question`、`question_category`、`question_group`、`question_tag`、`question_tag_relation` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID，用于上下文和日志 |
| `categoryIds` | `array<long>` | 否 | 分类 ID 列表 |
| `tagIds` | `array<long>` | 否 | 标签 ID 列表 |
| `difficulty` | `string` | 否 | 难度 |
| `questionGroupId` | `long` | 否 | 指定问题组 ID |
| `excludeQuestionIds` | `array<long>` | 否 | 排除已问过题目 |
| `excludeGroupIds` | `array<long>` | 否 | 排除已问过的问题组，V1 面试去重推荐使用 |
| `count` | `int` | 是 | 抽题数量 |
| `stageType` | `string` | 否 | 面试阶段类型 |

响应参数：

| 字段 | 类型 | 说明 |
|---|---|---|
| `questionId` | `long` | 题目 ID |
| `groupId` | `long` | 问题组 ID |
| `title` | `string` | 题目标题 |
| `content` | `string` | 题目内容 |
| `answer` | `string` | 参考答案 |
| `analysis` | `string` | 答案解析 |
| `categoryName` | `string` | 分类名称 |
| `difficulty` | `string` | 难度 |
| `tags` | `array<QuestionTagVO>` | 标签列表 |

业务规则：

1. 只抽取启用、未删除题目。
2. 不抽取禁用分类下题目。
3. 不抽取已禁用题目。
4. 支持排除已经问过的题目。
5. V1 面试去重推荐优先排除已使用的 `group_id`。
6. 如果 `questionGroupId` 不为空，优先从问题组内抽题。
7. 如果题目不足，返回实际可抽取数量，并由 interview-service 决定是否降级。
8. 该接口不做 AI 生成题目。

### 10.2 GET /inner/questions/{id}

调用方：interview-service。

用途：获取面试问题详情、参考答案、解析，用于 AI 评分上下文。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/inner/questions/{id}` |
| 是否对前端暴露 | 否 |
| 关联表 | `question`、`question_category`、`question_group`、`question_tag` |

响应参数：

| 字段 | 类型 | 说明 |
|---|---|---|
| `questionId` | `long` | 题目 ID |
| `groupId` | `long` | 问题组 ID |
| `title` | `string` | 题目标题 |
| `content` | `string` | 题目内容 |
| `answer` | `string` | 参考答案 |
| `analysis` | `string` | 答案解析 |
| `categoryName` | `string` | 分类名称 |
| `difficulty` | `string` | 难度 |
| `tags` | `array<QuestionTagVO>` | 标签列表 |
| `status` | `int` | 题目状态 |

业务规则：

1. 内部接口可以查询题目详情。
2. 默认仍不返回已逻辑删除题目。
3. 如果题目禁用但历史面试需要回放，V1 推荐由 `interview_message` 保存当时题目快照，避免依赖禁用题目。

### 10.3 POST /inner/questions/recommend-for-report

调用方：interview-service。

用途：面试报告生成后，根据薄弱点推荐题目。

V1 定位：

1. 该接口 V1 可选。
2. 不纳入第一版前后端联调清单。
3. 如果实现，只返回启用、未删除题目。
4. 不调用 AI 生成新题，仅基于现有题库按分类、标签、难度进行推荐。

---

## 11. DTO / VO 设计

### 11.1 用户端 DTO / VO

#### QuestionQueryRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 标题关键字 |
| `categoryId` | `long` | 否 | 分类 ID |
| `tagIds` | `array<long>` | 否 | 标签 ID 列表 |
| `difficulty` | `string` | 否 | 难度 |
| `masteryStatus` | `string` | 否 | 掌握状态 |
| `favoriteOnly` | `boolean` | 否 | 是否只看收藏 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

#### QuestionListVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 题目 ID |
| `title` | `string` | 是 | 题目标题 |
| `categoryId` | `long` | 是 | 分类 ID |
| `categoryName` | `string` | 是 | 分类名称 |
| `difficulty` | `string` | 是 | 难度 |
| `tags` | `array<QuestionTagVO>` | 否 | 标签列表 |
| `favorite` | `boolean` | 是 | 是否收藏 |
| `masteryStatus` | `string` | 否 | 掌握状态 |
| `answered` | `boolean` | 是 | 是否答过 |
| `createdAt` | `string` | 是 | 创建时间 |

#### QuestionDetailVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 题目 ID |
| `title` | `string` | 是 | 题目标题 |
| `content` | `string` | 是 | 题目内容 |
| `answer` | `string` | 是 | 参考答案 |
| `analysis` | `string` | 否 | 答案解析 |
| `category` | `QuestionCategoryVO` | 是 | 分类 |
| `tags` | `array<QuestionTagVO>` | 否 | 标签列表 |
| `difficulty` | `string` | 是 | 难度 |
| `favorite` | `boolean` | 是 | 是否收藏 |
| `masteryStatus` | `string` | 否 | 掌握状态 |
| `lastAnswer` | `string` | 否 | 最近答案 |
| `lastAnswerResult` | `string` | 否 | 最近答题结果 |

#### SubmitQuestionAnswerRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userAnswer` | `string` | 是 | 用户答案 |
| `selfResult` | `string` | 否 | 自评结果 |
| `masteryStatus` | `string` | 否 | 掌握状态 |

#### SubmitQuestionAnswerVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `recordId` | `long` | 是 | 答题记录 ID |
| `questionId` | `long` | 是 | 题目 ID |
| `answerResult` | `string` | 是 | 答题结果 |
| `masteryStatus` | `string` | 否 | 掌握状态 |
| `wrongRecordGenerated` | `boolean` | 是 | 是否生成或更新错题 |
| `answeredAt` | `string` | 是 | 答题时间 |

#### FavoriteQuestionVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `favoriteId` | `long` | 是 | 收藏记录 ID |
| `questionId` | `long` | 是 | 题目 ID |
| `title` | `string` | 是 | 题目标题 |
| `categoryName` | `string` | 是 | 分类名称 |
| `difficulty` | `string` | 是 | 难度 |
| `tags` | `array<QuestionTagVO>` | 否 | 标签列表 |
| `createdAt` | `string` | 是 | 收藏时间 |

#### WrongQuestionQueryRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 标题关键字 |
| `categoryId` | `long` | 否 | 分类 ID |
| `difficulty` | `string` | 否 | 难度 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

#### WrongQuestionVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `wrongRecordId` | `long` | 是 | 错题记录 ID |
| `questionId` | `long` | 是 | 题目 ID |
| `title` | `string` | 是 | 题目标题 |
| `categoryName` | `string` | 是 | 分类名称 |
| `difficulty` | `string` | 是 | 难度 |
| `tags` | `array<QuestionTagVO>` | 否 | 标签列表 |
| `lastAnswer` | `string` | 否 | 最近答案 |
| `lastAnswerResult` | `string` | 否 | 最近答题结果 |
| `wrongCount` | `int` | 是 | 错误或部分正确次数 |
| `lastWrongAt` | `string` | 否 | 最近错题时间 |
| `masteryStatus` | `string` | 否 | 掌握状态 |

#### UpdateMasteryRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `masteryStatus` | `string` | 是 | `MASTERED` / `VAGUE` / `UNKNOWN` |

#### UpdateMasteryVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `questionId` | `long` | 是 | 题目 ID |
| `masteryStatus` | `string` | 是 | 掌握状态 |
| `updatedAt` | `string` | 是 | 更新时间 |

### 11.2 管理端 DTO / VO

#### AdminQuestionQueryRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 标题关键字 |
| `categoryId` | `long` | 否 | 分类 ID |
| `tagId` | `long` | 否 | 标签 ID |
| `difficulty` | `string` | 否 | 难度 |
| `status` | `int` | 否 | 状态 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

#### AdminQuestionPageVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 题目 ID |
| `title` | `string` | 是 | 题目标题 |
| `categoryName` | `string` | 是 | 分类名称 |
| `groupId` | `long` | 是 | 问题组 ID |
| `groupTitle` | `string` | 否 | 问题组标准标题 |
| `difficulty` | `string` | 是 | 难度 |
| `status` | `int` | 是 | 状态 |
| `tags` | `array<QuestionTagVO>` | 否 | 标签列表 |
| `createdAt` | `string` | 是 | 创建时间 |
| `updatedAt` | `string` | 是 | 更新时间 |

#### AdminQuestionSaveRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `title` | `string` | 是 | 题目标题 |
| `content` | `string` | 是 | 题目内容 |
| `answer` | `string` | 是 | 参考答案 |
| `analysis` | `string` | 否 | 答案解析 |
| `categoryId` | `long` | 是 | 分类 ID |
| `groupId` | `long` | 是 | 问题组 ID |
| `difficulty` | `string` | 是 | 难度 |
| `tagIds` | `array<long>` | 否 | 标签 ID 列表 |
| `status` | `int` | 否 | 状态，默认启用 |

#### UpdateQuestionStatusRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `status` | `int` | 是 | `1` 启用，`0` 禁用 |

#### QuestionCategoryVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 分类 ID |
| `name` | `string` | 是 | 分类名称 |
| `code` | `string` | 否 | 分类编码 |
| `parentId` | `long` | 是 | 父分类 ID |
| `sort` | `int` | 是 | 排序值 |
| `status` | `int` | 是 | 状态 |
| `description` | `string` | 否 | 描述 |
| `createdAt` | `string` | 是 | 创建时间 |
| `updatedAt` | `string` | 是 | 更新时间 |

#### SaveQuestionCategoryRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `name` | `string` | 是 | 分类名称 |
| `code` | `string` | 是 | 分类编码 |
| `parentId` | `long` | 否 | 父分类 ID，默认 `0` |
| `sort` | `int` | 否 | 排序值 |
| `status` | `int` | 否 | 状态，默认启用 |
| `description` | `string` | 否 | 描述 |

#### UpdateCategoryStatusRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `status` | `int` | 是 | `1` 启用，`0` 禁用 |

#### QuestionTagVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 标签 ID |
| `name` | `string` | 是 | 标签名称 |
| `code` | `string` | 否 | 标签编码 |
| `status` | `int` | 是 | 状态 |
| `description` | `string` | 否 | 描述 |
| `createdAt` | `string` | 是 | 创建时间 |
| `updatedAt` | `string` | 是 | 更新时间 |

#### SaveQuestionTagRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `name` | `string` | 是 | 标签名称 |
| `code` | `string` | 否 | 标签编码 |
| `status` | `int` | 否 | 状态，默认启用 |
| `description` | `string` | 否 | 描述 |

#### UpdateTagStatusRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `status` | `int` | 是 | `1` 启用，`0` 禁用 |

#### QuestionGroupVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 问题组 ID |
| `name` | `string` | 是 | 问题组名称 |
| `description` | `string` | 否 | 考察意图说明 |
| `categoryId` | `long` | 是 | 主分类 ID |
| `knowledgePoint` | `string` | 否 | 主知识点 |
| `difficulty` | `string` | 是 | 难度 |
| `status` | `int` | 是 | 状态 |
| `questionCount` | `int` | 是 | 题目数量 |
| `questionIds` | `array<long>` | 否 | 组内题目 ID |
| `createdAt` | `string` | 是 | 创建时间 |
| `updatedAt` | `string` | 是 | 更新时间 |

#### SaveQuestionGroupRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `name` | `string` | 是 | 问题组名称 |
| `canonicalAnswer` | `string` | 否 | 标准答案 |
| `categoryId` | `long` | 是 | 主分类 ID |
| `knowledgePoint` | `string` | 否 | 主知识点 |
| `difficulty` | `string` | 是 | 难度 |
| `description` | `string` | 否 | 考察意图说明 |
| `status` | `int` | 否 | 状态，默认启用 |
| `questionIds` | `array<long>` | 否 | 归入该组的题目 ID |

#### UpdateGroupStatusRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `status` | `int` | 是 | `1` 启用，`0` 禁用 |

### 11.3 内部 Feign DTO / VO

#### PickInterviewQuestionRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID |
| `categoryIds` | `array<long>` | 否 | 分类 ID 列表 |
| `tagIds` | `array<long>` | 否 | 标签 ID 列表 |
| `difficulty` | `string` | 否 | 难度 |
| `questionGroupId` | `long` | 否 | 指定问题组 |
| `excludeQuestionIds` | `array<long>` | 否 | 排除题目 ID |
| `excludeGroupIds` | `array<long>` | 否 | 排除问题组 ID |
| `count` | `int` | 是 | 抽题数量 |
| `stageType` | `string` | 否 | 面试阶段类型 |

#### PickInterviewQuestionVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `questionId` | `long` | 是 | 题目 ID |
| `groupId` | `long` | 是 | 问题组 ID |
| `title` | `string` | 是 | 题目标题 |
| `content` | `string` | 是 | 题目内容 |
| `answer` | `string` | 是 | 参考答案 |
| `analysis` | `string` | 否 | 答案解析 |
| `categoryName` | `string` | 是 | 分类名称 |
| `difficulty` | `string` | 是 | 难度 |
| `tags` | `array<QuestionTagVO>` | 否 | 标签列表 |

#### InnerQuestionDetailVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `questionId` | `long` | 是 | 题目 ID |
| `groupId` | `long` | 是 | 问题组 ID |
| `title` | `string` | 是 | 题目标题 |
| `content` | `string` | 是 | 题目内容 |
| `answer` | `string` | 是 | 参考答案 |
| `analysis` | `string` | 否 | 答案解析 |
| `categoryName` | `string` | 是 | 分类名称 |
| `difficulty` | `string` | 是 | 难度 |
| `tags` | `array<QuestionTagVO>` | 否 | 标签列表 |
| `status` | `int` | 是 | 题目状态 |

#### RecommendQuestionRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID |
| `weakKnowledgePoints` | `array<string>` | 否 | 薄弱知识点 |
| `categoryIds` | `array<long>` | 否 | 分类 ID |
| `tagIds` | `array<long>` | 否 | 标签 ID |
| `difficulty` | `string` | 否 | 难度 |
| `count` | `int` | 否 | 推荐数量 |

#### RecommendQuestionVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `questionId` | `long` | 是 | 题目 ID |
| `title` | `string` | 是 | 题目标题 |
| `categoryName` | `string` | 是 | 分类名称 |
| `difficulty` | `string` | 是 | 难度 |
| `tags` | `array<QuestionTagVO>` | 否 | 标签列表 |
| `recommendReason` | `string` | 否 | 推荐原因 |

---

## 12. 参数校验规则

| 参数 | 校验规则 |
|---|---|
| `pageNo` | 最小 1 |
| `pageSize` | 最大 100 |
| `title` | 不能为空，最大 200 |
| `content` | 不能为空 |
| `answer` | 不能为空 |
| `difficulty` | 必须是 `EASY` / `MEDIUM` / `HARD` |
| `status` | 只能是 `0` / `1` |
| `masteryStatus` | 必须是 `MASTERED` / `VAGUE` / `UNKNOWN` |
| `selfResult` | 必须是 `CORRECT` / `WRONG` / `PARTIAL` / `UNKNOWN` |
| `tagIds` | 不能为空时必须全部存在且未删除 |
| `categoryId` | 必须存在且未删除 |
| `groupId` | 必须存在且未删除 |
| `questionIds` | 不能为空时必须全部存在且未删除 |
| `count` | 最小 1，最大建议 20 |
| `keyword` | 最大 100 |
| `name` | 分类、标签、问题组名称不能为空，最大 100 |

---

## 13. 错误码设计

题库模块错误码范围：`43000-43999`。

| code | message | 说明 |
|---:|---|---|
| `43001` | 题目不存在 | 题目不存在或已删除 |
| `43002` | 题目已禁用 | 用户端访问或抽题命中禁用题目 |
| `43003` | 分类不存在 | 分类不存在或已删除 |
| `43004` | 分类已禁用 | 用户端访问禁用分类 |
| `43005` | 标签不存在 | 标签不存在或已删除 |
| `43006` | 标签已禁用 | 用户端筛选或绑定禁用标签 |
| `43007` | 问题组不存在 | 问题组不存在或已删除 |
| `43008` | 问题组已禁用 | 创建面试或抽题使用禁用问题组 |
| `43009` | 掌握状态非法 | `masteryStatus` 不合法 |
| `43010` | 题目难度非法 | `difficulty` 不合法 |
| `43011` | 题目已收藏 | 非幂等场景下重复收藏 |
| `43012` | 收藏记录不存在 | 查询或取消收藏时记录不存在 |
| `43013` | 错题记录不存在 | 查询错题详情时记录不存在 |
| `43014` | 题目数量不足 | 内部抽题可用题目不足 |
| `43015` | 分类下存在题目，不能删除 | 删除分类前需要迁移或删除题目 |
| `43016` | 标签下存在题目，不能删除 | 删除标签前需要解除题目关联 |
| `43017` | 问题组下存在题目，不能删除或需要先解除关系 | 删除问题组前需要迁移题目 |
| `43018` | 答题结果非法 | `selfResult` 不合法 |
| `43019` | 题目不可访问 | 当前用户端不可访问该题目 |

---

## 14. 数据库表关系

本模块涉及：

| 表名 | 说明 |
|---|---|
| `question_category` | 题目分类 |
| `question_tag` | 题目标签 |
| `question_group` | 问题组 |
| `question` | 题目 |
| `question_tag_relation` | 题目标签关系 |
| `user_question_record` | 用户答题记录 |
| `wrong_question` | 错题记录 |
| `favorite_question` | 收藏题目 |
| `user_question_mastery` | 用户题目掌握状态 |

关系说明：

1. question-service 是以上表的数据归属服务。
2. interview-service 不直接访问题库表。
3. 用户提交答案产生 `user_question_record`。
4. 错题记录由 `wrong_question` 维护。
5. 收藏记录由 `favorite_question` 维护。
6. 掌握状态由 `user_question_mastery` 维护。
7. 题目与标签通过 `question_tag_relation` 维护多对多关系。
8. 题目与问题组通过 `question.group_id` 维护，V1 不新增问题组题目关系表。

---

## 15. 前端页面对应关系

| 前端页面 | 关联接口 |
|---|---|
| 题库列表页 | `GET /questions`、`POST /questions/{id}/favorite`、`DELETE /questions/{id}/favorite`、`PUT /questions/{id}/mastery` |
| 题目详情页 | `GET /questions/{id}`、`POST /questions/{id}/answers`、收藏和掌握状态接口 |
| 错题本页 | `GET /questions/wrong-records`、`GET /questions/{id}`、`PUT /questions/{id}/mastery` |
| 收藏题目页 | `GET /questions/favorites`、取消收藏接口 |
| 管理端题目管理页 | `/admin/questions/**` |
| 管理端分类管理页 | `/admin/question-categories/**` |
| 管理端标签管理页 | `/admin/question-tags/**` |
| 管理端问题组管理页 | `/admin/question-groups/**` |
| 创建面试页 | 间接使用问题组和 `/inner/questions/pick-for-interview` 抽题能力 |

---

## 16. 注意事项

1. 用户端只展示启用、未删除题目。
2. 管理端接口必须具备 `ADMIN` 角色。
3. 用户答题记录、收藏、错题、掌握状态都必须绑定当前用户。
4. `POST /questions/{id}/answers` 不调用 AI。
5. AI 面试评分归 interview-service 调用 ai-service。
6. `/inner/questions/**` 不对前端暴露。
7. 面试抽题只通过 `/inner/questions/pick-for-interview`。
8. V1 不使用 Elasticsearch。
9. V1 不做 AI 题目生成。
10. V1 不做题目导入导出。
11. V1 不做刷题计划。
12. 问题组与题目关系按现有数据库设计通过 `question.group_id` 维护，不新增关系表。
