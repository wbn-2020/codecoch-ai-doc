# CodeCoachAI V1 Spring Cloud OpenFeign 内部接口总表

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 Spring Cloud OpenFeign 内部接口总表 |
| 项目阶段 | V1 |
| 最高依据 | 接口设计/CodeCoachAI_V1_接口设计总览.md |
| 参考文档 | PRD/CodeCoachAI_PRD_V1_微服务分层架构版.md、数据库设计/CodeCoachAI_V1_数据库设计总览.md、接口设计/CodeCoachAI_V1_认证用户接口设计.md、接口设计/CodeCoachAI_V1_题库接口设计.md、接口设计/CodeCoachAI_V1_简历接口设计.md、接口设计/CodeCoachAI_V1_面试接口设计.md、接口设计/CodeCoachAI_V1_AI接口设计.md、接口设计/CodeCoachAI_V1_系统管理接口设计.md |
| 文档用途 | 统一整理 CodeCoachAI V1 阶段所有服务间内部调用接口，明确调用方、被调用方、接口路径、请求方式、用途、权限边界、是否必须实现、失败处理建议 |

本文档中的 OpenFeign 调用均指 Spring Cloud OpenFeign 声明式 HTTP 服务间调用。

本文档只整理 `/inner/**` 服务内部接口，不整理前端直接访问接口。本文档用于后续后端微服务骨架创建、Spring Cloud OpenFeign Client 规划、服务间依赖梳理和联调顺序规划。

涉及服务：

| 服务 | 说明 |
|---|---|
| `codecoachai-auth` | 认证服务，调用 user-service 完成登录查询、注册创建用户、角色查询 |
| `codecoachai-user` | 用户服务，提供用户认证信息、用户角色、用户统计等内部接口 |
| `codecoachai-question` | 题库服务，提供面试抽题、题目详情、推荐题等内部接口 |
| `codecoachai-resume` | 简历服务，提供简历详情、项目经历、默认简历等内部接口 |
| `codecoachai-interview` | 面试服务，调用题库、简历、AI 服务完成核心面试闭环 |
| `codecoachai-ai` | AI 服务，提供面试问题生成、回答评分、追问生成、报告生成等内部接口 |
| `codecoachai-system` | 系统服务，可选调用各服务内部统计接口 |
| `codecoachai-gateway` | 统一网关，不对外暴露 `/inner/**` |

边界说明：

1. V1 不设计 Dubbo RPC。
2. V1 不设计 gRPC。
3. V1 不设计异步消息调用作为主链路。
4. V1 服务间同步调用统一使用 Spring Cloud OpenFeign。
5. `/inner/**` 接口只允许服务内部调用。
6. 前端不能访问 `/inner/**`。
7. Gateway 不对外暴露 `/inner/**`。

---

## 2. 内部接口设计原则

1. 服务间同步调用统一使用 Spring Cloud OpenFeign。
2. `/inner/**` 只允许服务内部访问。
3. Gateway 不对外暴露 `/inner/**`。
4. 前端不能直接访问 `/inner/**`。
5. 内部接口仍需进行服务间鉴权或内网隔离。
6. 内部接口调用需要透传 `traceId`。
7. 涉及用户数据时需要传递 `userId` 或从可信上下文中获取 `userId`。
8. 被调用服务必须校验数据归属，不允许调用方绕过校验。
9. Spring Cloud OpenFeign 接口返回统一响应结构。
10. 内部接口失败时，调用方需要做明确失败处理，不允许静默吞异常。
11. 核心链路接口必须实现，可选接口可以延后。
12. 内部接口仍遵守数据归属原则：一个服务不能直接操作另一个服务的数据表。

统一响应结构：

```json
{
  "code": 0,
  "message": "success",
  "data": {},
  "traceId": "optional-trace-id"
}
```

说明：

1. Spring Cloud OpenFeign Client 调用后需要识别 `code` 是否为 `0`。
2. `code != 0` 时，调用方应按业务语义转换为明确错误，而不是继续使用空数据。
3. 内部调用异常需要记录 `traceId`，便于跨服务排查。

---

## 3. 内部接口安全约束

`/inner/**` 不对公网开放。

安全约束：

1. Gateway 路由配置应排除 `/inner/**`。
2. 前端不能直接访问 `/inner/**`。
3. 如果内部调用必须经过 Gateway，需要增加内部调用标识或服务间鉴权。
4. 具体安全实现可以在后端开发阶段确定。
5. V1 可以先使用内网隔离 + Gateway 不暴露 + 简单内部调用标识。
6. 不能仅依赖“前端不调用”作为安全措施。
7. 内部接口返回敏感字段时必须限制调用方，例如 `passwordHash` 只能返回给 auth-service。
8. AI 能力接口只能由 interview-service 调用。
9. 内部接口不能绕过用户数据归属校验。

推荐服务间请求头：

| 请求头 | 是否必填 | 说明 |
|---|---:|---|
| `X-Internal-Call` | 是 | 内部调用标识，建议值为 `true` |
| `X-Service-Name` | 是 | 调用方服务名，例如 `codecoachai-interview` |
| `X-Trace-Id` | 是 | 链路追踪 ID |
| `X-User-Id` | 视场景 | 涉及用户数据归属时传递 |
| `Authorization` | 可选 | 是否透传用户 Token 由安全方案决定 |
| `X-Username` | 可选 | 当前用户名 |
| `X-Roles` | 可选 | 当前用户角色编码 |

说明：

1. `X-Internal-Call` 不能作为唯一安全措施。
2. `X-User-Id` 必须来自可信登录上下文，不允许直接信任前端随意传入的 userId。
3. 被调用服务必须根据业务校验数据归属，例如 resume-service 校验简历属于 `userId`。

---

## 4. 服务依赖关系总览

| 调用方服务 | 被调用方服务 | 调用目的 | 是否核心链路 | 说明 |
|---|---|---|---:|---|
| `codecoachai-auth` | `codecoachai-user` | 登录查询用户、注册创建用户、查询角色 | 是 | auth-service 不直接操作 user-service 数据库表 |
| `codecoachai-interview` | `codecoachai-question` | 面试抽题、查询题目详情、报告推荐题 | 是 | 报告推荐题为 V1 可选 |
| `codecoachai-interview` | `codecoachai-resume` | 查询简历详情、查询项目经历、查询默认简历 | 是 | 默认简历接口为 V1 可选 |
| `codecoachai-interview` | `codecoachai-ai` | 生成面试问题、回答评分、生成追问、生成面试报告 | 是 | AI 能力只允许 interview-service 调用 |
| `codecoachai-user` | `codecoachai-question` | 当前用户刷题、错题、收藏、掌握状态统计 | 否 | 用于 `GET /users/overview` 聚合实现，V1 第一阶段可简化 |
| `codecoachai-user` | `codecoachai-resume` | 当前用户简历数量和默认简历信息统计 | 否 | 用于 `GET /users/overview` 聚合实现，V1 第一阶段可简化 |
| `codecoachai-user` | `codecoachai-interview` | 当前用户面试统计 | 否 | 用于 `GET /users/overview` 聚合实现，V1 第一阶段可简化 |
| `codecoachai-system` | `codecoachai-user` | 首页用户统计、角色列表查询 | 否 | V1 可选，角色列表更推荐 user-service 直接对管理端提供 |
| `codecoachai-system` | `codecoachai-question` | 首页题库统计 | 否 | V1 可选 |
| `codecoachai-system` | `codecoachai-resume` | 首页简历统计 | 否 | V1 可选 |
| `codecoachai-system` | `codecoachai-interview` | 首页面试统计 | 否 | V1 可选 |
| `codecoachai-system` | `codecoachai-ai` | 首页 AI 调用统计 | 否 | V1 可选 |

---

## 5. auth-service 调用 user-service

### 5.1 GET /inner/users/by-username

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-auth` |
| 被调用方 | `codecoachai-user` |
| 请求方式 | `GET` |
| URL | `/inner/users/by-username` |
| 是否必须实现 | 是 |
| 所属链路 | 登录认证 |

用途：登录时根据 `username` 查询用户认证信息。

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `username` | `string` | 是 | 登录账号 |

响应数据：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 用户 ID |
| `username` | `string` | 登录账号 |
| `passwordHash` | `string` | 密码哈希，仅 auth-service 密码校验使用 |
| `nickname` | `string` | 用户昵称 |
| `avatarUrl` | `string` | 头像 URL |
| `email` | `string` | 邮箱 |
| `status` | `int` | 用户状态 |
| `roles` | `array<string>` | 角色编码列表 |

安全要求：

1. 仅 auth-service 可调用。
2. `passwordHash` 只能用于密码校验，不允许打印日志。
3. 前端不能访问。
4. 不允许其他服务滥用该接口获取密码哈希。

失败处理：

1. 用户不存在时 auth-service 返回登录失败。
2. 密码错误和用户不存在对前端展示建议统一为“用户名或密码错误”。
3. 账号禁用时返回账号已禁用。

### 5.2 POST /inner/users

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-auth` |
| 被调用方 | `codecoachai-user` |
| 请求方式 | `POST` |
| URL | `/inner/users` |
| 是否必须实现 | 是 |
| 所属链路 | 用户注册 |

用途：注册时创建普通用户。

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `username` | `string` | 是 | 登录账号 |
| `passwordHash` | `string` | 是 | auth-service 使用统一 PasswordEncoder 生成的密码哈希 |
| `nickname` | `string` | 否 | 昵称 |
| `email` | `string` | 否 | 邮箱 |

响应数据：

| 字段 | 类型 | 说明 |
|---|---|---|
| `userId` | `long` | 用户 ID |
| `username` | `string` | 登录账号 |
| `nickname` | `string` | 昵称 |

业务要求：

1. 默认绑定 `USER` 角色。
2. 默认启用。
3. 用户名唯一。
4. 创建用户和绑定角色在 user-service 本地事务中完成。
5. user-service 不接收明文密码。

失败处理：

1. 用户名已存在时返回用户模块错误码。
2. 创建用户失败时 auth-service 不应生成 Token。

### 5.3 GET /inner/users/{id}/roles

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-auth` |
| 被调用方 | `codecoachai-user` |
| 请求方式 | `GET` |
| URL | `/inner/users/{id}/roles` |
| 是否必须实现 | 是 |
| 所属链路 | 登录认证、权限刷新 |

用途：查询用户角色。

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 用户 ID，路径参数 |

响应数据：

| 字段 | 类型 | 说明 |
|---|---|---|
| `userId` | `long` | 用户 ID |
| `roles` | `array<string>` | 角色编码列表，例如 `USER`、`ADMIN` |

业务要求：

1. 只返回启用、未删除角色。
2. 用户不存在时返回明确错误。
3. 不返回角色菜单、按钮权限等 V1 不设计内容。

### 5.4 GET /inner/users/{id}

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-auth`、其他内部服务可选 |
| 被调用方 | `codecoachai-user` |
| 请求方式 | `GET` |
| URL | `/inner/users/{id}` |
| 是否必须实现 | 可选 |
| 所属链路 | 用户基础信息查询 |

用途：内部查询用户基础信息。

说明：

1. 不返回 `passwordHash`。
2. 不返回 Token、密码等敏感信息。
3. V1 第一版核心链路可不依赖该接口。

建议响应数据：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 用户 ID |
| `username` | `string` | 登录账号 |
| `nickname` | `string` | 昵称 |
| `avatarUrl` | `string` | 头像 URL |
| `email` | `string` | 邮箱 |
| `status` | `int` | 用户状态 |
| `roles` | `array<string>` | 角色编码列表，可选 |

---

## 6. interview-service 调用 question-service

### 6.1 POST /inner/questions/pick-for-interview

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-interview` |
| 被调用方 | `codecoachai-question` |
| 请求方式 | `POST` |
| URL | `/inner/questions/pick-for-interview` |
| 是否必须实现 | 是 |
| 所属链路 | 创建面试、阶段抽题 |

用途：创建面试或进入下一阶段时抽题。

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 当前用户 ID，用于日志、个性化排除等场景 |
| `categoryIds` | `array<long>` | 否 | 分类 ID 列表 |
| `tagIds` | `array<long>` | 否 | 标签 ID 列表 |
| `difficulty` | `string` | 否 | 难度：`EASY` / `MEDIUM` / `HARD` |
| `questionGroupId` | `long` | 否 | 指定问题组 |
| `excludeQuestionIds` | `array<long>` | 否 | 排除已问过题目 |
| `excludeGroupIds` | `array<long>` | 否 | 排除已问过问题组，避免同一考察点重复 |
| `count` | `int` | 是 | 抽题数量 |
| `stageType` | `string` | 否 | 面试阶段类型 |

响应数据：

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
| `tags` | `array<object>` | 标签列表 |

业务要求：

1. 只抽启用、未删除题目。
2. 不抽禁用分类下题目。
3. 不抽已禁用题目。
4. 支持排除已问过题目。
5. `questionGroupId` 不为空时优先从问题组内抽题。
6. 题目不足时返回实际可抽取数量，由 interview-service 决定降级或失败。
7. 该接口不做 AI 生成题目。

失败处理：

1. 抽题失败时 interview-service 应返回明确错误。
2. 题目数量不足时 interview-service 可提示用户调整条件，或按 V1 规则降级减少题量。

### 6.2 GET /inner/questions/{id}

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-interview` |
| 被调用方 | `codecoachai-question` |
| 请求方式 | `GET` |
| URL | `/inner/questions/{id}` |
| 是否必须实现 | 是 |
| 所属链路 | 面试问题快照、AI 评分上下文 |

用途：查询题目详情，用于面试问题快照和 AI 评分上下文。

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 题目 ID |

响应数据：

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
| `tags` | `array<object>` | 标签列表 |
| `status` | `int` | 题目状态 |

业务要求：

1. 默认不返回已逻辑删除题目。
2. 历史回放应依赖 interview-service 保存的题目快照。
3. 如果题目已禁用但历史面试需要回放，V1 推荐由 `interview_message` 保存当时题目快照，避免依赖实时题库数据。

### 6.3 POST /inner/questions/recommend-for-report

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-interview` |
| 被调用方 | `codecoachai-question` |
| 请求方式 | `POST` |
| URL | `/inner/questions/recommend-for-report` |
| 是否必须实现 | V1 可选 |
| 所属链路 | 面试报告推荐题 |

用途：面试报告生成后，根据薄弱点推荐题目。

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID |
| `weakKnowledgePoints` | `array<string>` | 否 | 薄弱知识点 |
| `categoryIds` | `array<long>` | 否 | 分类 ID 列表 |
| `tagIds` | `array<long>` | 否 | 标签 ID 列表 |
| `difficulty` | `string` | 否 | 难度 |
| `count` | `int` | 否 | 推荐数量 |

响应数据：

| 字段 | 类型 | 说明 |
|---|---|---|
| `questionId` | `long` | 题目 ID |
| `title` | `string` | 题目标题 |
| `categoryName` | `string` | 分类名称 |
| `difficulty` | `string` | 难度 |
| `tags` | `array<object>` | 标签列表 |
| `recommendReason` | `string` | 推荐原因 |

说明：

1. 不纳入第一版核心联调。
2. 如果实现，只返回启用、未删除题目。
3. AI 报告中也可以先返回学习方向，不强制返回具体题目。

---

## 7. interview-service 调用 resume-service

### 7.1 GET /inner/resumes/{id}

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-interview` |
| 被调用方 | `codecoachai-resume` |
| 请求方式 | `GET` |
| URL | `/inner/resumes/{id}` |
| 是否必须实现 | 是 |
| 所属链路 | 创建面试、简历上下文 |

用途：创建面试时查询简历详情。

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 简历 ID |
| `userId` | `long` | 是 | 建议作为 query 参数或请求头透传，用于校验简历归属 |
| `includeProjects` | `boolean` | 否 | 是否内嵌项目经历，默认可为 `false` |

响应数据：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 简历 ID |
| `userId` | `long` | 用户 ID |
| `resumeName` | `string` | 简历名称 |
| `targetPosition` | `string` | 求职方向 |
| `skills` | `string` | 技能栈 |
| `workSummary` | `string` | 工作经历摘要 |
| `education` | `string` | 教育经历 |
| `isDefault` | `int` | 是否默认简历 |
| `status` | `int` | 简历状态 |
| `projects` | `array<object>` | 项目经历，可选内嵌 |

业务要求：

1. 必须校验简历归属 `userId`。
2. 不返回已删除简历。
3. interview-service 创建面试时保存必要简历快照。
4. 字段以 `接口设计/CodeCoachAI_V1_简历接口设计.md` 和简历表设计为准；当前 V1 不强制 `realName`、`yearsOfExperience` 等数据库未设计字段。

失败处理：

1. 简历不存在或不属于当前用户时，interview-service 应返回简历模块错误。
2. 简历查询失败时不应继续创建面试。

### 7.2 GET /inner/resumes/{id}/projects

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-interview` |
| 被调用方 | `codecoachai-resume` |
| 请求方式 | `GET` |
| URL | `/inner/resumes/{id}/projects` |
| 是否必须实现 | 是 |
| 所属链路 | 项目深挖面试上下文 |

用途：查询简历项目经历，用于项目深挖面试上下文。

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 简历 ID |
| `userId` | `long` | 是 | 建议作为 query 参数或请求头透传，用于校验简历归属 |

响应数据：

| 字段 | 类型 | 说明 |
|---|---|---|
| `projectId` | `long` | 项目经历 ID |
| `resumeId` | `long` | 简历 ID |
| `projectName` | `string` | 项目名称 |
| `projectTime` | `string` | 项目时间 |
| `projectBackground` | `string` | 项目背景 |
| `techStack` | `string` | 技术栈 |
| `responsibility` | `string` | 个人职责 |
| `coreFeatures` | `string` | 核心功能 |
| `technicalChallenges` | `string` | 技术难点 |
| `optimizationResult` | `string` | 优化成果 |
| `extraInfo` | `string` | 补充说明 |
| `sort` | `int` | 排序值 |

业务要求：

1. 必须校验简历归属 `userId`。
2. 只返回未删除项目经历。
3. 不允许跨用户查询。
4. interview-service 创建面试时可保存项目经历快照。

### 7.3 GET /inner/users/{userId}/default-resume

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-interview` |
| 被调用方 | `codecoachai-resume` |
| 请求方式 | `GET` |
| URL | `/inner/users/{userId}/default-resume` |
| 是否必须实现 | V1 可选 |
| 所属链路 | 创建面试简历选择 |

用途：查询用户默认简历。

说明：

1. V1 推荐创建面试时前端显式传 `resumeId`。
2. 不纳入第一版核心联调。
3. 如果实现，必须只返回该 `userId` 的默认简历。
4. 如果没有默认简历，返回空或明确错误码。

建议响应数据：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 默认简历 ID，没有默认简历时可为空 |
| `userId` | `long` | 用户 ID |
| `resumeName` | `string` | 简历名称 |
| `targetPosition` | `string` | 求职方向 |
| `isDefault` | `int` | 是否默认简历 |
| `status` | `int` | 简历状态 |

---

## 8. interview-service 调用 ai-service

### 8.1 POST /inner/ai/interview/question

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-interview` |
| 被调用方 | `codecoachai-ai` |
| 请求方式 | `POST` |
| URL | `/inner/ai/interview/question` |
| 是否必须实现 | 是，但 V1 可允许在部分阶段直接使用题库题目作为问题内容 |
| 所属链路 | 开始面试、生成自然提问 |

用途：生成或润色面试问题。

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID |
| `interviewId` | `long` | 是 | 面试 ID |
| `stageId` | `long` | 是 | 阶段 ID |
| `stageType` | `string` | 否 | 阶段类型 |
| `questionId` | `long` | 否 | 题库题目 ID |
| `questionTitle` | `string` | 否 | 题目标题 |
| `questionContent` | `string` | 否 | 题目内容 |
| `referenceAnswer` | `string` | 否 | 参考答案 |
| `resumeSnapshot` | `object` | 否 | 简历快照 |
| `projectSnapshot` | `object` | 否 | 项目快照 |
| `targetPosition` | `string` | 否 | 目标岗位 |
| `difficulty` | `string` | 否 | 难度 |
| `previousMessages` | `array<object>` | 否 | 历史消息 |

响应数据：

| 字段 | 类型 | 说明 |
|---|---|---|
| `aiQuestion` | `string` | AI 提问 |
| `questionTitle` | `string` | 问题标题 |
| `questionContent` | `string` | 问题内容 |
| `suggestedAnswer` | `string` | 建议答案 |
| `promptTemplateId` | `long` | Prompt 模板 ID |
| `aiCallLogId` | `long` | AI 调用日志 ID |
| `modelName` | `string` | 模型名称 |
| `tokenUsage` | `object` | Token 用量，V1 可选 |

失败处理：

1. AI 生成失败时，interview-service 可降级使用题库原题内容。
2. 降级行为需要记录日志。
3. 如果当前阶段强依赖 AI 自然提问且无法降级，应返回明确错误。

### 8.2 POST /inner/ai/interview/evaluate

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-interview` |
| 被调用方 | `codecoachai-ai` |
| 请求方式 | `POST` |
| URL | `/inner/ai/interview/evaluate` |
| 是否必须实现 | 是 |
| 所属链路 | 提交回答、AI 评分 |

用途：对用户回答进行评分和点评。

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
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
| `resumeSnapshot` | `object` | 否 | 简历快照 |
| `previousMessages` | `array<object>` | 否 | 历史消息 |

响应数据：

| 字段 | 类型 | 说明 |
|---|---|---|
| `score` | `decimal` | 分数，建议 0-100 |
| `level` | `string` | 表现等级 |
| `comment` | `string` | 点评 |
| `advantage` | `string` | 优点 |
| `weakness` | `string` | 不足 |
| `suggestion` | `string` | 建议 |
| `needFollowUp` | `boolean` | 是否建议追问 |
| `followUpDirection` | `string` | 追问方向 |
| `promptTemplateId` | `long` | Prompt 模板 ID |
| `aiCallLogId` | `long` | AI 调用日志 ID |
| `modelName` | `string` | 模型名称 |
| `tokenUsage` | `object` | Token 用量，V1 可选 |

失败处理：

1. AI 评分失败时，interview-service 不应伪造评分。
2. 返回明确错误给前端。
3. 面试状态不应错误推进。
4. 失败原因需要可排查，ai-service 应记录 AI 调用日志。

### 8.3 POST /inner/ai/interview/follow-up

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-interview` |
| 被调用方 | `codecoachai-ai` |
| 请求方式 | `POST` |
| URL | `/inner/ai/interview/follow-up` |
| 是否必须实现 | 是 |
| 所属链路 | 动态追问 |

用途：生成动态追问。

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID |
| `interviewId` | `long` | 是 | 面试 ID |
| `stageId` | `long` | 是 | 阶段 ID |
| `sourceQuestionMessageId` | `long` | 是 | 来源问题消息 ID |
| `sourceAnswerMessageId` | `long` | 是 | 来源回答消息 ID |
| `sourceEvaluation` | `object` | 是 | 来源评分结果 |
| `questionTitle` | `string` | 否 | 原问题标题 |
| `questionContent` | `string` | 是 | 原问题内容 |
| `userAnswer` | `string` | 是 | 用户回答 |
| `followUpDirection` | `string` | 否 | 追问方向 |
| `previousFollowUpCount` | `int` | 是 | 已追问次数 |
| `maxFollowUpCount` | `int` | 是 | 最大追问次数 |
| `stageType` | `string` | 否 | 阶段类型 |

响应数据：

| 字段 | 类型 | 说明 |
|---|---|---|
| `followUpQuestion` | `string` | 追问问题 |
| `followUpReason` | `string` | 追问原因 |
| `difficulty` | `string` | 难度 |
| `promptTemplateId` | `long` | Prompt 模板 ID |
| `aiCallLogId` | `long` | AI 调用日志 ID |
| `modelName` | `string` | 模型名称 |
| `tokenUsage` | `object` | Token 用量，V1 可选 |

失败处理：

1. 追问生成失败时，interview-service 可选择进入下一题。
2. 需要记录失败原因。
3. 如果进入下一题，需要保证面试阶段状态和当前问题状态一致。

### 8.4 POST /inner/ai/interview/report

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-interview` |
| 被调用方 | `codecoachai-ai` |
| 请求方式 | `POST` |
| URL | `/inner/ai/interview/report` |
| 是否必须实现 | 是 |
| 所属链路 | 结束面试、生成报告 |

用途：生成面试报告内容。

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID |
| `interviewId` | `long` | 是 | 面试 ID |
| `interviewName` | `string` | 否 | 面试名称 |
| `targetPosition` | `string` | 否 | 目标岗位 |
| `resumeSnapshot` | `object` | 否 | 简历快照 |
| `stageResults` | `array<object>` | 是 | 阶段结果 |
| `messages` | `array<object>` | 是 | 面试消息 |
| `totalDurationSeconds` | `int` | 否 | 总耗时 |

响应数据：

| 字段 | 类型 | 说明 |
|---|---|---|
| `totalScore` | `decimal` | 总分 |
| `summary` | `string` | 总结 |
| `stageReports` | `array<object>` | 阶段报告 |
| `strengths` | `string` | 优势 |
| `weaknesses` | `string` | 不足 |
| `suggestions` | `string` | 建议 |
| `recommendedLearningPath` | `string` | 推荐学习路径，V1 可选 |
| `recommendedQuestions` | `array<object>` | 推荐题目，V1 可选 |
| `promptTemplateId` | `long` | Prompt 模板 ID |
| `aiCallLogId` | `long` | AI 调用日志 ID |
| `modelName` | `string` | 模型名称 |
| `tokenUsage` | `object` | Token 用量，V1 可选 |

失败处理：

1. 报告生成失败时，interview-service 记录 `reportStatus=FAILED`。
2. 面试主状态 V1 推荐仍可标记为 `COMPLETED`，报告状态为 `FAILED`。
3. 后续允许通过报告重试接口重新生成。
4. ai-service 必须记录失败日志。

---

## 9. system-service 可选内部统计调用

本章接口均为 V1 可选，不强制第一版实现。如果这些接口暂不实现，`GET /admin/system/overview` 可以先返回部分统计或默认值。这些接口后续可随着系统管理页完善逐步补齐。

### 9.1 GET /inner/users/statistics

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-system` |
| 被调用方 | `codecoachai-user` |
| 请求方式 | `GET` |
| URL | `/inner/users/statistics` |
| 是否必须实现 | 可选 |
| 用途 | 管理端首页用户统计 |

建议响应：

| 字段 | 类型 | 说明 |
|---|---|---|
| `userCount` | `long` | 用户总数 |
| `enabledUserCount` | `long` | 启用用户数 |
| `disabledUserCount` | `long` | 禁用用户数 |
| `todayRegisterCount` | `long` | 今日注册用户数 |

### 9.2 GET /inner/questions/statistics

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-system` |
| 被调用方 | `codecoachai-question` |
| 请求方式 | `GET` |
| URL | `/inner/questions/statistics` |
| 是否必须实现 | 可选 |
| 用途 | 管理端首页题库统计 |

建议响应：

| 字段 | 类型 | 说明 |
|---|---|---|
| `questionCount` | `long` | 题目总数 |
| `enabledQuestionCount` | `long` | 启用题目数 |
| `disabledQuestionCount` | `long` | 禁用题目数 |
| `categoryCount` | `long` | 分类数量 |
| `tagCount` | `long` | 标签数量 |
| `questionGroupCount` | `long` | 问题组数量 |

### 9.3 GET /inner/resumes/statistics

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-system` |
| 被调用方 | `codecoachai-resume` |
| 请求方式 | `GET` |
| URL | `/inner/resumes/statistics` |
| 是否必须实现 | 可选 |
| 用途 | 管理端首页简历统计 |

建议响应：

| 字段 | 类型 | 说明 |
|---|---|---|
| `resumeCount` | `long` | 简历总数 |
| `todayResumeCount` | `long` | 今日新增简历数 |

### 9.4 GET /inner/interviews/statistics

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-system` |
| 被调用方 | `codecoachai-interview` |
| 请求方式 | `GET` |
| URL | `/inner/interviews/statistics` |
| 是否必须实现 | 可选 |
| 用途 | 管理端首页面试统计 |

建议响应：

| 字段 | 类型 | 说明 |
|---|---|---|
| `interviewCount` | `long` | 面试总数 |
| `completedInterviewCount` | `long` | 已完成面试数 |
| `todayInterviewCount` | `long` | 今日新增面试数 |
| `reportFailedCount` | `long` | 报告生成失败数 |

### 9.5 GET /inner/ai/statistics

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-system` |
| 被调用方 | `codecoachai-ai` |
| 请求方式 | `GET` |
| URL | `/inner/ai/statistics` |
| 是否必须实现 | 可选 |
| 用途 | 管理端首页 AI 调用统计 |

建议响应：

| 字段 | 类型 | 说明 |
|---|---|---|
| `aiCallCount` | `long` | AI 调用总数 |
| `aiCallFailedCount` | `long` | AI 调用失败总数 |
| `todayAiCallCount` | `long` | 今日 AI 调用数 |
| `promptCount` | `long` | Prompt 模板总数 |

---

## 10. user-service 可选用户概览聚合调用

本章接口用于支持 `GET /users/overview` 的完整聚合实现，均为 V1 可选。V1 第一阶段可以先返回简化统计或默认值，不阻塞 AI 模拟面试核心闭环。

### 10.1 GET /inner/questions/users/{userId}/statistics

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-user` |
| 被调用方 | `codecoachai-question` |
| 请求方式 | `GET` |
| URL | `/inner/questions/users/{userId}/statistics` |
| 是否必须实现 | 可选 |
| 用途 | 查询当前用户刷题统计、错题统计、收藏统计 |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID，路径参数 |

建议响应：

| 字段 | 类型 | 说明 |
|---|---|---|
| `userId` | `long` | 用户 ID |
| `answeredQuestionCount` | `long` | 已答题数 |
| `wrongQuestionCount` | `long` | 错题数 |
| `favoriteQuestionCount` | `long` | 收藏题数 |
| `masteredQuestionCount` | `long` | 已掌握题数 |
| `vagueQuestionCount` | `long` | 模糊题数 |
| `unknownQuestionCount` | `long` | 未掌握题数 |

业务要求：

1. 只统计指定 `userId` 的刷题数据。
2. question-service 只访问题库、刷题、错题、收藏、掌握状态相关表。
3. user-service 调用时的 `userId` 必须来自可信登录上下文。

### 10.2 GET /inner/resumes/users/{userId}/statistics

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-user` |
| 被调用方 | `codecoachai-resume` |
| 请求方式 | `GET` |
| URL | `/inner/resumes/users/{userId}/statistics` |
| 是否必须实现 | 可选 |
| 用途 | 查询当前用户简历数量和默认简历信息 |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID，路径参数 |

建议响应：

| 字段 | 类型 | 说明 |
|---|---|---|
| `userId` | `long` | 用户 ID |
| `resumeCount` | `long` | 简历数量 |
| `defaultResumeId` | `long` | 默认简历 ID |
| `defaultResumeName` | `string` | 默认简历名称 |

业务要求：

1. 只统计指定 `userId` 的未删除简历。
2. resume-service 不直接访问 user-service 数据表。
3. 没有默认简历时，`defaultResumeId` 和 `defaultResumeName` 可为空。

### 10.3 GET /inner/interviews/users/{userId}/statistics

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-user` |
| 被调用方 | `codecoachai-interview` |
| 请求方式 | `GET` |
| URL | `/inner/interviews/users/{userId}/statistics` |
| 是否必须实现 | 可选 |
| 用途 | 查询当前用户面试统计 |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID，路径参数 |

建议响应：

| 字段 | 类型 | 说明 |
|---|---|---|
| `userId` | `long` | 用户 ID |
| `interviewCount` | `long` | 面试总数 |
| `completedInterviewCount` | `long` | 已完成面试数 |
| `averageScore` | `decimal` | 平均分 |
| `lastInterviewId` | `long` | 最近一次面试 ID |
| `lastInterviewTime` | `string` | 最近一次面试时间 |

业务要求：

1. 只统计指定 `userId` 的面试数据。
2. interview-service 不直接访问 user-service 数据表。
3. 面试统计基于 interview-service 自己保存的会话和报告数据。

### 10.4 GET /inner/users/roles

| 项目 | 内容 |
|---|---|
| 调用方 | `codecoachai-system` 可选、`codecoachai-auth` 可选 |
| 被调用方 | `codecoachai-user` |
| 请求方式 | `GET` |
| URL | `/inner/users/roles` |
| 是否必须实现 | 可选 |
| 用途 | 内部查询基础角色列表 |

建议响应：

| 字段 | 类型 | 说明 |
|---|---|---|
| `roleId` | `long` | 角色 ID |
| `roleCode` | `string` | 角色编码：`USER` / `ADMIN` |
| `roleName` | `string` | 角色名称 |
| `status` | `int` | 角色状态 |

说明：

1. 如 system-service 需要展示角色统计或基础角色信息，可以通过该接口查询。
2. system-service 不直接查询 `sys_role`、`sys_user_role`。
3. V1 管理端角色列表外部接口推荐由 user-service 直接提供 `GET /admin/roles`。

---

## 11. 内部接口实现优先级

| 优先级 | 接口 | 调用方 | 被调用方 | 是否必须实现 | 所属链路 | 说明 |
|---|---|---|---|---:|---|---|
| P0 | `GET /inner/users/by-username` | auth-service | user-service | 是 | 登录认证 | 登录查询用户认证信息 |
| P0 | `POST /inner/users` | auth-service | user-service | 是 | 用户注册 | 注册创建用户并绑定 USER 角色 |
| P0 | `GET /inner/users/{id}/roles` | auth-service | user-service | 是 | 权限认证 | 查询用户角色 |
| P0 | `POST /inner/questions/pick-for-interview` | interview-service | question-service | 是 | 创建面试 | 面试抽题 |
| P0 | `GET /inner/questions/{id}` | interview-service | question-service | 是 | 面试上下文 | 获取题目详情和参考答案 |
| P0 | `GET /inner/resumes/{id}` | interview-service | resume-service | 是 | 创建面试 | 获取简历详情 |
| P0 | `GET /inner/resumes/{id}/projects` | interview-service | resume-service | 是 | 项目深挖 | 获取项目经历 |
| P0 | `POST /inner/ai/interview/evaluate` | interview-service | ai-service | 是 | 提交回答 | AI 评分点评 |
| P0 | `POST /inner/ai/interview/follow-up` | interview-service | ai-service | 是 | 动态追问 | 生成追问 |
| P0 | `POST /inner/ai/interview/report` | interview-service | ai-service | 是 | 结束面试 | 生成报告 |
| P1 | `POST /inner/ai/interview/question` | interview-service | ai-service | 是，允许降级 | 开始面试 | 生成自然提问；可降级为题库原题 |
| P1 | `GET /inner/users/{id}` | auth-service / 其他内部服务 | user-service | 可选 | 用户基础信息 | 不返回密码哈希 |
| P1 | `POST /inner/questions/recommend-for-report` | interview-service | question-service | 可选 | 报告推荐题 | 不纳入第一版核心联调 |
| P1 | `GET /inner/users/{userId}/default-resume` | interview-service | resume-service | 可选 | 创建面试 | 推荐前端显式传 resumeId |
| P2 | `GET /inner/users/statistics` | system-service | user-service | 可选 | 管理端首页 | 用户统计 |
| P2 | `GET /inner/questions/statistics` | system-service | question-service | 可选 | 管理端首页 | 题库统计 |
| P2 | `GET /inner/resumes/statistics` | system-service | resume-service | 可选 | 管理端首页 | 简历统计 |
| P2 | `GET /inner/interviews/statistics` | system-service | interview-service | 可选 | 管理端首页 | 面试统计 |
| P2 | `GET /inner/ai/statistics` | system-service | ai-service | 可选 | 管理端首页 | AI 调用统计 |
| P2 | `GET /inner/questions/users/{userId}/statistics` | user-service | question-service | 可选 | 用户首页概览 | 刷题、错题、收藏统计 |
| P2 | `GET /inner/resumes/users/{userId}/statistics` | user-service | resume-service | 可选 | 用户首页概览 | 简历数量和默认简历 |
| P2 | `GET /inner/interviews/users/{userId}/statistics` | user-service | interview-service | 可选 | 用户首页概览 | 面试统计 |
| P2 | `GET /inner/users/roles` | system-service / auth-service | user-service | 可选 | 角色基础信息 | 内部角色列表 |

---

## 12. OpenFeign Client 规划建议

本节只做文档说明，不编写 Java 代码。Client 命名后续实现阶段可调整。

### 11.1 auth-service

建议 Spring Cloud OpenFeign Client：

| Client | 方法 | 对应接口 | 说明 |
|---|---|---|---|
| `UserFeignClient` | `getByUsername` | `GET /inner/users/by-username` | 登录查询认证信息 |
| `UserFeignClient` | `createUser` | `POST /inner/users` | 注册创建用户 |
| `UserFeignClient` | `getUserRoles` | `GET /inner/users/{id}/roles` | 查询用户角色 |
| `UserFeignClient` | `getInnerUser` | `GET /inner/users/{id}` | 可选，查询用户基础信息 |

### 12.2 user-service

建议 Spring Cloud OpenFeign Client，均为 V1 可选：

| Client | 方法 | 对应接口 | 说明 |
|---|---|---|---|
| `QuestionUserStatisticsFeignClient` | `getUserQuestionStatistics` | `GET /inner/questions/users/{userId}/statistics` | 用户刷题统计 |
| `ResumeUserStatisticsFeignClient` | `getUserResumeStatistics` | `GET /inner/resumes/users/{userId}/statistics` | 用户简历统计 |
| `InterviewUserStatisticsFeignClient` | `getUserInterviewStatistics` | `GET /inner/interviews/users/{userId}/statistics` | 用户面试统计 |

### 12.3 interview-service

建议 Spring Cloud OpenFeign Client：

| Client | 方法 | 对应接口 | 说明 |
|---|---|---|---|
| `QuestionFeignClient` | `pickForInterview` | `POST /inner/questions/pick-for-interview` | 面试抽题 |
| `QuestionFeignClient` | `getInnerQuestionDetail` | `GET /inner/questions/{id}` | 内部题目详情 |
| `QuestionFeignClient` | `recommendForReport` | `POST /inner/questions/recommend-for-report` | 可选，报告推荐题 |
| `ResumeFeignClient` | `getInnerResumeDetail` | `GET /inner/resumes/{id}` | 简历详情 |
| `ResumeFeignClient` | `getInnerResumeProjects` | `GET /inner/resumes/{id}/projects` | 项目经历 |
| `ResumeFeignClient` | `getDefaultResume` | `GET /inner/users/{userId}/default-resume` | 可选，默认简历 |
| `AiFeignClient` | `generateInterviewQuestion` | `POST /inner/ai/interview/question` | 生成自然提问 |
| `AiFeignClient` | `evaluateAnswer` | `POST /inner/ai/interview/evaluate` | 回答评分 |
| `AiFeignClient` | `generateFollowUp` | `POST /inner/ai/interview/follow-up` | 生成追问 |
| `AiFeignClient` | `generateInterviewReport` | `POST /inner/ai/interview/report` | 生成报告 |

### 12.4 system-service

建议 Spring Cloud OpenFeign Client，均为 V1 可选：

| Client | 方法 | 对应接口 | 说明 |
|---|---|---|---|
| `UserStatisticsFeignClient` | `getUserStatistics` | `GET /inner/users/statistics` | 用户统计 |
| `QuestionStatisticsFeignClient` | `getQuestionStatistics` | `GET /inner/questions/statistics` | 题库统计 |
| `ResumeStatisticsFeignClient` | `getResumeStatistics` | `GET /inner/resumes/statistics` | 简历统计 |
| `InterviewStatisticsFeignClient` | `getInterviewStatistics` | `GET /inner/interviews/statistics` | 面试统计 |
| `AiStatisticsFeignClient` | `getAiStatistics` | `GET /inner/ai/statistics` | AI 调用统计 |
| `UserRoleFeignClient` | `getInnerRoles` | `GET /inner/users/roles` | 可选，内部基础角色列表 |

说明：

1. 当前文档只用于明确服务依赖，不写代码。
2. Spring Cloud OpenFeign Client 的异常处理、请求头透传、统一返回解包建议放在 common-feign 中统一规划。
3. system-service 统计类 Client 可以后置实现。

---

## 13. 请求头透传建议

服务间调用建议透传以下请求头：

| 请求头 | 是否必填 | 说明 |
|---|---:|---|
| `Authorization` | 可选 | 是否透传用户 Token 由安全方案决定 |
| `X-User-Id` | 视场景 | 涉及用户数据归属时传递 |
| `X-Username` | 可选 | 当前用户名 |
| `X-Roles` | 可选 | 当前用户角色 |
| `X-Trace-Id` | 是 | 链路追踪 ID |
| `X-Internal-Call` | 是 | 内部调用标识 |
| `X-Service-Name` | 是 | 调用方服务名 |

说明：

1. 涉及用户数据归属时，优先显式传 `userId`。
2. `userId` 必须来自可信登录上下文，不允许前端随意传入后直接信任。
3. `traceId` 用于链路排查。
4. 内部调用标识不能作为唯一安全措施。
5. 如果使用 Sa-Token，是否透传 `Authorization` 需要结合服务间鉴权方案决定。

---

## 14. 超时、重试和降级建议

1. Spring Cloud OpenFeign 调用应配置连接超时和读取超时。
2. AI 相关调用超时时间可以比普通服务更长。
3. 用户、题库、简历等核心数据查询失败时，应返回明确错误。
4. AI 问题生成失败时，部分场景可降级使用题库原题。
5. AI 评分失败时，不应伪造评分。
6. AI 报告生成失败时，interview-service 应记录 `reportStatus=FAILED`。
7. 不建议对非幂等接口做自动重试，例如 `POST /inner/users`。
8. 查询类接口可适度重试，但 V1 可先不做复杂重试策略。
9. 降级策略必须可观测，至少记录日志。
10. AI 追问生成失败时，可以进入下一题，但必须保证面试状态一致。
11. 题目不足、简历不存在、用户禁用等业务失败不应被当作系统异常重试。

建议超时策略：

| 调用类型 | 建议处理 |
|---|---|
| 用户认证查询 | 短超时，失败直接登录失败 |
| 题库抽题和题目详情 | 普通超时，失败阻断创建面试 |
| 简历详情和项目经历 | 普通超时，失败阻断创建面试 |
| AI 评分、追问、报告 | 较长超时，失败返回明确 AI 错误 |
| 首页统计 | 短超时，可返回部分统计或默认值 |

---

## 15. DTO / VO 汇总索引

| 接口 | 请求 DTO | 响应 VO |
|---|---|---|
| `GET /inner/users/by-username` | `username` | `InnerUserAuthVO` |
| `POST /inner/users` | `InnerCreateUserRequest` | `InnerUserBasicVO`，或简化返回 `userId`、`username`、`nickname` |
| `GET /inner/users/{id}/roles` | `id` | `InnerUserRoleVO` |
| `GET /inner/users/{id}` | `id` | `InnerUserBasicVO` |
| `POST /inner/questions/pick-for-interview` | `PickInterviewQuestionRequest` | `array<PickInterviewQuestionVO>` |
| `GET /inner/questions/{id}` | `id` | `InnerQuestionDetailVO` |
| `POST /inner/questions/recommend-for-report` | `RecommendQuestionRequest` | `array<RecommendQuestionVO>` |
| `GET /inner/resumes/{id}` | `id`、`userId`、`includeProjects` | `InnerResumeDetailVO` |
| `GET /inner/resumes/{id}/projects` | `id`、`userId` | `array<InnerResumeProjectVO>` |
| `GET /inner/users/{userId}/default-resume` | `userId` | `InnerDefaultResumeVO` |
| `POST /inner/ai/interview/question` | `GenerateInterviewQuestionRequest` | `GenerateInterviewQuestionVO` |
| `POST /inner/ai/interview/evaluate` | `EvaluateAnswerRequest` | `EvaluateAnswerVO` |
| `POST /inner/ai/interview/follow-up` | `GenerateFollowUpRequest` | `GenerateFollowUpVO` |
| `POST /inner/ai/interview/report` | `GenerateInterviewReportRequest` | `GenerateInterviewReportVO` |
| `GET /inner/users/statistics` | 无 | `InnerUserStatisticsVO` |
| `GET /inner/questions/statistics` | 无 | `InnerQuestionStatisticsVO` |
| `GET /inner/resumes/statistics` | 无 | `InnerResumeStatisticsVO` |
| `GET /inner/interviews/statistics` | 无 | `InnerInterviewStatisticsVO` |
| `GET /inner/ai/statistics` | 无 | `InnerAiStatisticsVO` |
| `GET /inner/questions/users/{userId}/statistics` | `userId` | `InnerUserQuestionStatisticsVO` |
| `GET /inner/resumes/users/{userId}/statistics` | `userId` | `InnerUserResumeStatisticsVO` |
| `GET /inner/interviews/users/{userId}/statistics` | `userId` | `InnerUserInterviewStatisticsVO` |
| `GET /inner/users/roles` | 无 | `array<InnerRoleVO>` |

说明：

1. 本节只做 DTO / VO 名称索引，不重复展开所有字段。
2. 字段详情以各分模块接口详细设计文档为准。
3. `InnerUserBasicVO` 可在实现阶段结合 user-service 内部用户基础信息接口补充；核心登录链路不依赖密码哈希以外的用户详情接口。

---

## 16. 与外部接口的边界

外部接口：

1. `/auth/**` 是认证登录接口，前端可访问。
2. `/users/**` 是用户端用户资料接口，前端可访问。
3. `/questions/**` 是用户端题库和刷题接口，前端可访问。
4. `/resumes/**` 是用户端简历接口，前端可访问。
5. `/interviews/**` 是用户端面试接口，前端可访问。
6. `/admin/**` 是管理端接口，管理员前端可访问。

内部接口：

1. `/inner/**` 是服务内部接口。
2. 前端联调接口清单不应包含 `/inner/**`。
3. `/inner/**` 可在后端服务联调和单元测试中验证。
4. 管理端 Prompt 和 AI 日志接口不是 `/inner/**`，属于 `/admin/ai/**` 外部管理端接口。
5. `/inner/ai/**` 只允许 interview-service 调用，前端不允许直接访问。

---

## 17. 注意事项

1. 服务间调用统一使用 Spring Cloud OpenFeign。
2. 不使用 Dubbo、gRPC。
3. `/inner/**` 不对前端暴露。
4. Gateway 不对外暴露 `/inner/**`。
5. 内部接口仍需安全控制。
6. 数据归属由数据归属服务校验。
7. auth-service 不直接操作用户表。
8. interview-service 不直接访问题库、简历、AI 数据表。
9. AI 能力只允许 interview-service 调用。
10. system-service 统计内部接口为 V1 可选。
11. P0 内部接口必须优先保证。
12. AI 评分失败不能伪造结果。
13. 报告生成失败必须可重试。
14. 如果已有分模块文档中出现“Feign”简称，在本文档中统一理解为 Spring Cloud OpenFeign。
15. 后续实现 Spring Cloud OpenFeign Client 时，应统一处理请求头透传、超时、错误解包和日志记录。
