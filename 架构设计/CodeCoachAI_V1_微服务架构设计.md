# CodeCoachAI V1 微服务架构设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 微服务架构设计 |
| 项目阶段 | V1 |
| 后端项目 | CodeCoachAI-java |
| 前端项目 | CodeCoachAI-vue |
| 最高依据 | PRD/CodeCoachAI_PRD_V1_微服务分层架构版.md |
| 文档用途 | 指导 V1 微服务拆分、服务边界、模块职责、数据归属、后续接口与数据库设计 |

本文档只描述 CodeCoachAI V1 阶段的微服务架构设计。

如果其他 PRD、功能清单或后续版本规划与 V1 微服务分层架构版 PRD 存在冲突，以 V1 微服务分层架构版 PRD 为准。

V1 不提前实现 V2 / V3 功能，不引入文件服务、搜索服务、任务服务、消息队列、MinIO、Elasticsearch、完整数据看板、完整操作日志等后续能力。

---

## 2. V1 项目定位

CodeCoachAI V1 是一个面向 Java 求职者的个人作品集项目，核心定位是：

> 基于 Spring Cloud Alibaba 微服务架构的 AI Java 面试训练平台。

V1 的产品目标不是构建完整商业系统，而是跑通 AI 模拟面试核心闭环，并在第一版确定微服务骨架。

V1 重点展示以下能力：

1. 用户注册登录与基础权限控制。
2. 管理员维护题库、分类、标签、问题组。
3. 用户进行基础刷题、收藏、错题和掌握状态管理。
4. 用户手动维护简历和项目经历。
5. 用户创建 AI 模拟面试。
6. 后端通过面试阶段和状态机控制 AI 面试流程。
7. AI 根据题库、简历和面试配置提问。
8. AI 对用户回答评分、点评并生成动态追问。
9. 系统控制每个主问题最多追问 2 次。
10. 面试结束后生成结构化面试报告。
11. 管理员维护 Prompt 模板并查看 AI 调用日志。

V1 的技术目标是：

1. 采用 Maven 多模块微服务工程结构。
2. 使用 Spring Cloud Alibaba 确定服务拆分和基础通信方式。
3. 使用 Nacos 做服务注册与配置管理。
4. 使用 Spring Cloud Gateway 作为统一入口。
5. 使用 Spring Cloud OpenFeign 完成服务间调用。
6. 使用 Sa-Token 或 JWT 完成登录认证。
7. 使用 MyBatis-Plus 操作 MySQL。
8. 使用 Redis 做登录态、权限缓存、分类标签缓存、AI 调用限流和面试上下文缓存。
9. 通过独立 AI 服务统一封装大模型调用。
10. 每个业务服务内部统一采用 controller / service / service.impl / mapper / domain 分层。

---

## 3. V1 技术架构总览

### 3.1 总体架构

```text
CodeCoachAI-vue
  ↓
codecoachai-gateway
  ↓
auth / user / question / resume / interview / ai / system
  ↓
MySQL / Redis / Nacos / 大模型 API
```

V1 采用轻量微服务架构：

1. 前端所有请求统一进入 Gateway。
2. Gateway 负责路由、跨域、Token 基础校验和用户信息透传。
3. 认证、用户、题库、简历、面试、AI、系统管理按领域拆分为独立服务。
4. 服务间调用统一使用 Spring Cloud OpenFeign。
5. 每个业务服务负责自己的数据表，不做跨服务强事务。
6. AI 调用统一经过 codecoachai-ai，业务服务不直接调用大模型 API。
7. MySQL 作为 V1 主存储。
8. Redis 只用于 V1 明确需要的缓存、登录态、限流和可选上下文缓存。
9. Nacos 作为服务注册和配置管理基础设施。

### 3.2 V1 技术栈

| 类型 | 技术 |
|---|---|
| 开发语言 | Java 17 |
| 基础框架 | Spring Boot 3 |
| 微服务框架 | Spring Cloud Alibaba |
| 注册与配置 | Nacos |
| 网关 | Spring Cloud Gateway |
| 服务调用 | Spring Cloud OpenFeign |
| 限流熔断 | Sentinel，V1 可先接入基础依赖和简单规则 |
| ORM | MyBatis-Plus |
| 数据库 | MySQL 8 |
| 缓存 | Redis |
| 登录认证 | Sa-Token 或 JWT，V1 建议优先 Sa-Token |
| 接口文档 | Knife4j / Swagger |
| 参数校验 | Jakarta Validation |
| 工具库 | Lombok、Hutool |
| AI 接入 | 大模型 Chat API |
| 前端 | Vue 3 + Vite + TypeScript + Element Plus |

### 3.3 V1 架构原则

1. V1 直接确定微服务骨架，避免后续从单体重构。
2. 服务数量控制在可开发范围内，不一次性拆出完整目标架构。
3. 核心闭环优先，工程能力按 V1 必要范围接入。
4. 业务流程由后端服务控制，AI 只负责生成问题、评分、点评、追问和报告内容。
5. AI 调用必须经过统一 AI 服务，便于记录日志、统一 Prompt 和后续扩展模型供应商。
6. V1 避免复杂跨服务强一致事务，优先使用单服务本地事务。
7. 数据归属按服务领域划分，避免多个服务直接写同一类业务表。
8. 公共能力下沉到 common 模块，业务服务保持职责清晰。

---

## 4. V1 微服务模块列表

V1 后端项目建议采用以下模块结构：

```text
CodeCoachAI-java
├── codecoachai-common
│   ├── common-core
│   ├── common-web
│   ├── common-security
│   ├── common-mybatis
│   ├── common-redis
│   ├── common-feign
│   ├── common-ai
│   └── common-log
│
├── codecoachai-gateway
├── codecoachai-auth
├── codecoachai-user
├── codecoachai-question
├── codecoachai-resume
├── codecoachai-interview
├── codecoachai-ai
├── codecoachai-system
└── pom.xml
```

V1 微服务清单：

| 服务 | 类型 | V1 是否必需 | 说明 |
|---|---|---:|---|
| codecoachai-gateway | 网关服务 | 是 | 前端统一入口 |
| codecoachai-auth | 认证服务 | 是 | 注册、登录、Token |
| codecoachai-user | 用户服务 | 是 | 用户资料、角色关系 |
| codecoachai-question | 题库服务 | 是 | 题库、刷题、错题、收藏、问题组 |
| codecoachai-resume | 简历服务 | 是 | 简历和项目经历 |
| codecoachai-interview | 面试服务 | 是 | 面试主流程、状态机、报告 |
| codecoachai-ai | AI 服务 | 是 | AIClient、Prompt、AI 日志 |
| codecoachai-system | 系统服务 | 是 | 系统配置、管理端首页简化统计、可选菜单查询 |
| codecoachai-common | 公共模块 | 是 | 公共能力复用 |

---

## 5. 每个服务的职责边界

## 5.1 codecoachai-gateway

### 职责

1. 作为前端请求统一入口。
2. 根据路由规则转发到各业务服务。
3. 处理跨域请求。
4. 做 Token 基础校验。
5. 将用户 ID、角色等登录上下文透传给下游服务。
6. 提供基础限流预留。
7. 统一处理网关层异常响应。

### 不负责

1. 不处理具体业务逻辑。
2. 不直接访问业务数据库。
3. 不直接调用 AI 模型。
4. 不承担完整 RBAC 业务判断，服务内部仍需做权限校验。

---

## 5.2 codecoachai-auth

### 职责

1. 用户注册。
2. 用户登录。
3. 用户退出。
4. Token 生成。
5. Token 校验。
6. 获取当前登录用户信息。
7. 登录失败处理。
8. 管理员登录入口。

### 主要调用关系

1. 调用 user-service 查询用户账号、密码、状态和角色。
2. 使用 Redis 保存或校验登录态，取决于认证方案。

### 不负责

1. 不维护用户详细资料。
2. 不维护题库、简历、面试等业务数据。
3. 不直接管理 Prompt 和 AI 调用日志。

---

## 5.3 codecoachai-user

### 职责

1. 用户基础信息管理。
2. 查询当前用户资料。
3. 修改昵称、头像、邮箱等基础资料。
4. 维护用户角色关系。
5. 提供用户基础学习概览。
6. 向 auth-service 提供用户查询接口。

### 核心实体

```text
SysUser
SysRole
SysUserRole
```

### 不负责

1. 不处理登录 Token 生成。
2. 不保存用户简历内容。
3. 不保存题库答题记录。
4. 不生成面试报告。

---

## 5.4 codecoachai-question

### 职责

1. 题目管理。
2. 分类管理。
3. 标签管理。
4. 问题组管理。
5. 基础刷题。
6. 用户答题记录。
7. 错题本。
8. 收藏题目。
9. 掌握状态。
10. 面试抽题接口。
11. 按 question_group 控制同类题去重。

### 核心实体

```text
Question
QuestionCategory
QuestionTag
QuestionTagRelation
QuestionGroup
UserQuestionRecord
WrongQuestion
FavoriteQuestion
UserQuestionMastery
```

### V1 重点规则

1. 管理员新增题目时必须选择已有问题组或新建问题组。
2. 同一场练习或面试中避免重复抽取相同 group_id 的题目。
3. V1 问题组由管理员手动维护。
4. V1 不做 Embedding 语义去重。
5. V1 不做 AI 自动判重。
6. V1 可保留 normalized_title 字段，为后续去重做准备。

### 不负责

1. 不直接生成 AI 面试问题。
2. 不保存简历数据。
3. 不控制面试状态机。
4. 不生成面试报告。

---

## 5.5 codecoachai-resume

### 职责

1. 简历手动录入。
2. 简历列表。
3. 简历详情。
4. 简历编辑。
5. 简历删除。
6. 设置默认简历。
7. 项目经历维护。
8. 为 interview-service 提供简历查询接口。

### 核心实体

```text
Resume
ResumeProject
```

### V1 简历范围

V1 简历只支持手动录入，不支持文件上传解析。

简历基础字段包括：

1. 简历名称。
2. 求职方向。
3. 技能栈。
4. 工作经历摘要。
5. 教育经历。
6. 默认简历标识。
7. 状态。

项目经历字段包括：

1. 项目名称。
2. 项目时间。
3. 项目背景。
4. 技术栈。
5. 个人职责。
6. 核心功能。
7. 技术难点。
8. 优化成果。
9. 补充说明。

### 不负责

1. 不上传或解析 PDF / Word 文件。
2. 不存储文件元数据。
3. 不做 AI 简历优化。
4. 不直接生成项目深挖问题。

---

## 5.6 codecoachai-interview

### 职责

1. 创建面试。
2. 生成面试大纲和面试阶段。
3. 管理面试会话状态。
4. 管理面试阶段。
5. 获取当前问题。
6. 提交用户回答。
7. 调用 ai-service 评分与追问。
8. 控制每个主问题追问次数。
9. 结束面试。
10. 生成面试报告。
11. 查询面试历史。
12. 查询面试详情。

### 核心实体

```text
InterviewSession
InterviewStage
InterviewMessage
InterviewReport
```

### 主要调用关系

1. 调用 question-service 获取题目、问题组和参考答案。
2. 调用 resume-service 获取简历和项目经历。
3. 调用 ai-service 生成问题、评分、追问和报告。

### 面试状态

```text
NOT_STARTED       未开始
IN_PROGRESS       进行中
WAITING_ANSWER    等待用户回答
AI_EVALUATING     AI 评估中
REPORT_GENERATING 报告生成中
COMPLETED         已完成
CANCELED          已取消
FAILED            失败
```

### 面试模式

```text
TECHNICAL_BASIC    八股文技术面试
PROJECT_DEEP_DIVE 简历项目深挖面试
COMPREHENSIVE     综合模拟面试
```

### 不负责

1. 不直接调用大模型 API。
2. 不维护题库基础数据。
3. 不维护简历基础数据。
4. 不维护 Prompt 模板。

---

## 5.7 codecoachai-ai

### 职责

1. 统一 AIClient。
2. 模型供应商适配。
3. Prompt 模板渲染。
4. 八股文问题生成。
5. 项目深挖问题生成。
6. 回答评分。
7. 动态追问生成。
8. 面试报告生成。
9. AI 调用日志记录。
10. Prompt 模板管理相关接口。

### 核心实体

```text
PromptTemplate
AiCallLog
```

V1 可暂不落地 `AiModelConfig`，模型配置可以先通过配置文件或系统配置简化处理。

### AI 场景枚举

```text
INTERVIEW_QUESTION_GENERATE
INTERVIEW_ANSWER_EVALUATE
INTERVIEW_FOLLOW_UP_GENERATE
INTERVIEW_REPORT_GENERATE
PROJECT_DEEP_DIVE_QUESTION
```

### 不负责

1. 不控制面试状态机。
2. 不决定面试阶段是否推进。
3. 不直接读取 question-service 或 resume-service 数据库。
4. 不维护用户资料。

AI 服务只提供 AI 能力，业务编排由 interview-service 控制。

---

## 5.8 codecoachai-system

### 职责

1. 系统配置管理。
2. 管理端首页简化统计。
3. 管理端菜单查询，V1 可选。
4. 系统级参数说明。

### 不负责

1. 不负责用户注册登录。
2. 不负责 `sys_user`、`sys_role`、`sys_user_role`。
3. 不负责角色管理和用户角色关系维护。
4. 不负责复杂 RBAC、菜单权限、按钮权限、数据权限。
5. 不负责 Prompt 模板和 AI 调用日志。
6. 不负责题库、简历、面试业务数据。
7. 不负责完整操作日志、登录日志、站内通知、复杂数据看板。

边界说明：

1. user-service 管理 `sys_user`、`sys_role`、`sys_user_role`。
2. `/admin/roles` 归属 user-service。
3. system-service 不直接查询或维护 user-service 的用户角色表。
4. 如果 system-service 需要角色相关统计，只能通过 Spring Cloud OpenFeign 调 user-service 内部接口，不能直接访问 user-service 数据表。

### V1 简化原则

1. 保持系统服务轻量。
2. 只保留系统配置和管理端首页简化统计。
3. 菜单查询为 V1 可选能力，前端也可以先使用静态路由。
4. 不做完整操作日志。
5. 不做登录日志中心。
6. 不做站内通知。
7. 不做复杂数据看板。

### 核心实体

```text
SystemConfig
SysMenu，可选
SysPermission，可选
```

V1 如果开发压力较大，菜单查询和可选 `SysPermission` 预留可进一步简化，优先保证系统配置、管理端首页概览和基础 `ADMIN` 角色访问控制。角色数据和 `/admin/roles` 由 user-service 负责。

---

## 6. V1 暂不拆分的服务说明

V1 采用微服务骨架，但不一次性拆出完整目标架构。

| 后续服务 | V1 处理方式 | 原因 |
|---|---|---|
| practice-service | 刷题记录、错题、收藏、掌握状态先放 question-service | V1 刷题能力较轻，避免服务过多 |
| file-service | V1 不做文件上传，暂不拆 | 简历只支持手动录入 |
| search-service | V1 使用 MySQL 条件查询 | 暂不引入 Elasticsearch |
| task-service | V1 同步处理 AI 任务 | 暂不引入消息队列和异步任务中心 |
| notification-service | V1 不做站内通知 | 不属于核心闭环 |

V1 不拆分这些服务，是为了控制开发复杂度，优先保证 AI 面试核心闭环可稳定演示。

---

## 7. 服务之间的调用关系

### 7.1 总调用关系

```text
CodeCoachAI-vue
  ↓
codecoachai-gateway
  ↓
auth-service
user-service
question-service
resume-service
interview-service
ai-service
system-service
```

前端不直接访问具体业务服务，所有 HTTP 请求统一经过 Gateway。

`/inner/**` 是服务内部接口，不属于前端访问入口。Gateway 不对外暴露 `/inner/**`。

### 7.2 核心业务调用关系

```text
interview-service
  ├── 通过 /inner/questions/** 调用 question-service 获取题目、问题组、参考答案
  ├── 通过 /inner/resumes/** 调用 resume-service 获取简历和项目经历
  ├── 通过 /inner/ai/** 调用 ai-service 生成问题、评分、追问、报告
  └── 写入 interview_session / interview_stage / interview_message / interview_report
```

### 7.3 认证调用关系

```text
gateway
  └── 校验 Token，透传用户上下文

auth-service
  ├── 调用 user-service 查询用户账号和角色
  └── 写入或校验 Redis 登录态

user-service
  └── 维护 sys_user / sys_role / sys_user_role
```

### 7.4 题库和面试调用关系

```text
interview-service
  ↓
question-service
  ↓
返回候选题目、group_id、参考答案、知识点
```

question-service 只负责提供题目数据和抽题能力，不负责 AI 化表达。

interview-service 拿到题目后，再调用 ai-service 将题目转为自然面试问题。

### 7.5 简历和面试调用关系

```text
interview-service
  ↓
resume-service
  ↓
返回简历基础信息、项目经历、技能栈、职责、难点
```

resume-service 只负责简历数据管理，不直接生成项目深挖问题。

项目深挖问题由 interview-service 编排上下文后调用 ai-service 生成。

### 7.6 AI 调用关系

```text
interview-service
  ↓
ai-service
  ├── 渲染 Prompt
  ├── 调用大模型 API
  ├── 解析 AI 响应
  ├── 记录 ai_call_log
  └── 返回结构化结果
```

业务服务不得绕过 ai-service 直接调用模型 API。

AI 能力接口只通过 `/inner/ai/**` 提供给 interview-service 内部调用，前端不直接访问 AI 能力接口。

### 7.7 `/inner/**` 内部调用边界

1. `/inner/**` 只用于服务内部 Spring Cloud OpenFeign 调用。
2. 前端不允许直接访问 `/inner/**`。
3. Gateway 不对外暴露 `/inner/**`。
4. AI 能力接口 `/inner/ai/**` 只允许 interview-service 调用。
5. 内部接口仍需要服务间鉴权或内部调用标识，不能只依赖“前端不调用”作为安全措施。
6. 涉及用户数据的内部接口必须透传可信 `userId` 或用户上下文，并进行数据归属校验。

---

## 8. Gateway、Auth、User、Question、Resume、Interview、AI、System 的关系

### 8.1 Gateway 与各服务

Gateway 是所有前端请求的唯一入口。

它负责：

1. `/auth/**` 转发到 auth-service。
2. `/users/**` 转发到 user-service。
3. `/admin/users/**` 和 `/admin/roles` 转发到 user-service。
4. `/questions/**` 和 `/admin/questions/**` 转发到 question-service。
5. `/admin/question-categories/**`、`/admin/question-tags/**`、`/admin/question-groups/**` 转发到 question-service。
6. `/resumes/**` 转发到 resume-service。
7. `/interviews/**` 转发到 interview-service。
8. `/admin/ai/**` 转发到 ai-service。
9. `/admin/system/**` 和 `/admin/configs/**` 转发到 system-service。

说明：

1. V1 不提供前端直连的 `/ai/**` 用户端 AI 能力接口。
2. `/inner/**` 不对外暴露，Gateway 路由配置应排除 `/inner/**`。
3. AI 能力接口 `/inner/ai/**` 只允许 interview-service 通过 Spring Cloud OpenFeign 内部调用。

### 8.2 Auth 与 User

auth-service 负责认证流程，user-service 负责用户资料和角色数据。

登录时：

```text
auth-service
  ↓
user-service
  ↓
查询用户账号、密码、状态、角色
```

认证成功后 auth-service 生成 Token，并返回给前端。

### 8.3 Question 与 Interview

question-service 是八股文技术面试的数据来源。

interview-service 根据面试阶段、难度、经验年限、已使用 group_id 调用 question-service 抽题。

question-service 返回题目后，interview-service 再调用 ai-service 生成自然面试问题。

### 8.4 Resume 与 Interview

resume-service 是项目深挖面试的数据来源。

用户创建项目深挖或综合模拟面试时，interview-service 调用 resume-service 获取简历和项目经历。

### 8.5 Interview 与 AI

interview-service 是 AI 面试流程控制中心。

ai-service 是 AI 能力提供方。

边界如下：

| 能力 | 归属 |
|---|---|
| 创建面试会话 | interview-service |
| 生成面试阶段 | interview-service |
| 判断当前阶段 | interview-service |
| 控制追问次数 | interview-service |
| 判断是否进入下一题 | interview-service |
| 渲染 Prompt | ai-service |
| 调用大模型 | ai-service |
| 回答评分 | ai-service |
| 生成追问 | ai-service |
| 生成报告内容 | ai-service |
| 保存面试记录 | interview-service |
| 保存 AI 调用日志 | ai-service |

### 8.6 System 与管理端

system-service 提供系统配置和管理端首页简化统计。

题库、Prompt、AI 日志等具体业务后台能力仍由对应业务服务提供。

例如：

1. 题目管理由 question-service 提供。
2. Prompt 模板管理由 ai-service 提供。
3. AI 调用日志查询由 ai-service 提供。
4. 系统配置由 system-service 提供。
5. 角色列表 `/admin/roles` 由 user-service 提供。

system-service 不直接维护 `sys_role`、`sys_user_role`，也不直接查询 user-service 数据库表。

---

## 9. common 公共模块设计

## 9.1 common-core

包含基础公共能力：

1. 统一返回对象 `Result`。
2. 分页对象 `PageResult`。
3. 统一错误码 `ErrorCode`。
4. 业务异常 `BusinessException`。
5. 常量类。
6. 基础枚举。
7. 基础实体 `BaseEntity`。

## 9.2 common-web

包含 Web 层公共能力：

1. 全局异常处理。
2. 参数校验异常处理。
3. 请求日志基础处理。
4. Web 工具类。
5. Controller 基础能力。

## 9.3 common-security

包含认证和用户上下文公共能力：

1. 登录用户上下文 `LoginUser`。
2. 用户 ID 获取工具。
3. 权限注解封装。
4. Token 校验工具。
5. 网关鉴权工具。
6. 用户信息透传。

## 9.4 common-mybatis

包含 MyBatis-Plus 公共配置：

1. MyBatis-Plus 配置。
2. 分页插件。
3. 自动填充 `created_at`、`updated_at`。
4. 逻辑删除配置。
5. 通用字段处理。

## 9.5 common-redis

包含 Redis 公共能力：

1. Redis Key 常量。
2. Redis 工具类。
3. 缓存工具。
4. 限流工具。
5. 防重复提交工具。

V1 Redis 使用范围控制在：

1. 登录 Token。
2. 用户权限缓存。
3. 分类标签缓存。
4. AI 调用限流。
5. 面试临时上下文缓存，可选。
6. 提交回答防重复，可选。

## 9.6 common-feign

包含服务间调用公共能力：

1. Spring Cloud OpenFeign 配置。
2. Spring Cloud OpenFeign 拦截器。
3. 用户上下文透传。
4. 服务调用异常处理。
5. Spring Cloud OpenFeign 统一返回处理。

## 9.7 common-ai

包含 AI 场景公共对象：

1. AI 请求对象。
2. AI 响应对象。
3. AI 场景枚举。
4. Prompt 渲染工具基础接口。
5. AI 异常定义。
6. AI 结构化输出工具。

注意：common-ai 只放通用对象和工具，不直接实现具体模型调用。具体 AIClient 放在 codecoachai-ai 服务中。

## 9.8 common-log

V1 简化实现：

1. TraceId 生成。
2. 请求日志基础记录。
3. 后续操作日志 AOP 预留。

V1 不实现完整操作日志审计。

---

## 10. 服务内部包结构规范

每个业务服务内部统一采用以下分层：

```text
com.codecoachai.{service}
├── controller
├── service
├── service.impl
├── mapper
├── domain
│   ├── entity
│   ├── dto
│   ├── vo
│   └── enums
├── convert
├── feign
├── config
└── mq
```

V1 暂不使用消息队列，`mq` 包可以预留，不实现具体逻辑。

### 10.1 controller 层

职责：

1. 接收 HTTP 请求。
2. 参数校验。
3. 调用 service。
4. 返回统一响应对象。

禁止：

1. 编写复杂业务逻辑。
2. 直接调用 mapper。
3. 直接调用 AI SDK。

### 10.2 service 层

职责：

1. 定义业务能力。
2. 方法命名体现业务动作。
3. 避免只暴露 save、update、delete 这类纯 CRUD 方法。

示例：

```text
createInterview()
startInterview()
submitAnswer()
finishInterview()
generateReport()
```

### 10.3 service.impl 层

职责：

1. 实现业务流程。
2. 编排 mapper、Spring Cloud OpenFeign Client 和 AI 服务调用。
3. 控制事务。
4. 处理状态流转。
5. 处理业务异常。

### 10.4 mapper 层

职责：

1. MyBatis-Plus 数据访问。
2. 单表 CRUD。
3. 简单关联查询。

禁止：

1. 编写复杂业务判断。
2. 调用其他服务。

### 10.5 domain.entity

职责：

1. 与数据库表结构对应。
2. 仅用于持久化。
3. 不直接返回给前端。

### 10.6 domain.dto

职责：

1. 接收前端请求参数。
2. 包含新增、修改、查询、提交等请求对象。

命名示例：

```text
QuestionCreateDTO
QuestionUpdateDTO
QuestionQueryDTO
InterviewCreateDTO
SubmitAnswerDTO
ResumeCreateDTO
```

### 10.7 domain.vo

职责：

1. 返回前端展示数据。
2. 屏蔽敏感字段。
3. 聚合展示字段。

命名示例：

```text
QuestionVO
QuestionDetailVO
InterviewSessionVO
InterviewReportVO
ResumeDetailVO
```

### 10.8 domain.enums

职责：

1. 管理业务状态。
2. 避免魔法值。

示例：

```text
InterviewStatusEnum
QuestionTypeEnum
QuestionDifficultyEnum
AiSceneEnum
ResumeStatusEnum
```

### 10.9 convert 层

职责：

1. DTO 转 Entity。
2. Entity 转 VO。
3. List 转换。
4. 聚合对象转换。

V1 可以手写 Convert 类，后续可替换为 MapStruct。

### 10.10 Spring Cloud OpenFeign Client 层

职责：

1. 当前服务通过 Spring Cloud OpenFeign 调用其他微服务。
2. 用户上下文透传。
3. 服务间接口隔离。

示例：

```text
QuestionFeignClient
ResumeFeignClient
AiFeignClient
UserFeignClient
```

### 10.11 config 层

职责：

1. 当前服务配置类。
2. MyBatis、Redis、Spring Cloud OpenFeign、Swagger、AI 等配置。

---

## 11. V1 数据归属原则

### 11.1 总原则

1. 每个服务只直接读写自己领域内的数据表。
2. 跨服务获取数据必须通过 Spring Cloud OpenFeign 调用服务接口。
3. 不允许一个服务直接操作另一个服务的数据表。
4. V1 不设计复杂跨库分布式事务。
5. 面试主流程数据归属 interview-service。
6. AI 调用日志归属 ai-service。
7. 用户基础账号和角色归属 user-service。
8. 登录态和 Token 归属 auth-service 与 common-security 协作处理。

### 11.2 数据表归属

| 数据域 | 表 | 归属服务 |
|---|---|---|
| 用户权限 | `sys_user`、`sys_role`、`sys_user_role` | user-service |
| 题库 | `question_category`、`question_tag`、`question_group`、`question`、`question_tag_relation` | question-service |
| 刷题 | `user_question_record`、`wrong_question`、`favorite_question`、`user_question_mastery` | question-service |
| 简历 | `resume`、`resume_project` | resume-service |
| 面试 | `interview_session`、`interview_stage`、`interview_message`、`interview_report` | interview-service |
| AI | `prompt_template`、`ai_call_log` | ai-service |
| 系统 | `system_config`、可选 `sys_menu`、可选 `sys_permission` | system-service |

### 11.3 面试流程数据写入原则

1. 创建面试时，由 interview-service 写入 `interview_session` 和 `interview_stage`。
2. AI 提问后，由 interview-service 写入或更新 `interview_message`。
3. 用户提交回答后，由 interview-service 保存用户回答。
4. AI 评分和点评由 ai-service 返回结果，interview-service 保存到面试消息表。
5. AI 调用过程和异常由 ai-service 写入 `ai_call_log`。
6. 面试结束后，由 interview-service 汇总问答和评分，调用 ai-service 生成报告内容，再保存 `interview_report`。

### 11.4 跨服务事务原则

V1 避免跨服务强事务。

示例：

1. 面试记录保存成功，但 AI 评分失败时，interview-service 应将本轮消息标记为失败或允许重试。
2. AI 调用失败必须由 ai-service 记录日志。
3. 面试报告生成失败时，V1 推荐面试主状态仍保持 `COMPLETED`，表示问答流程已结束；报告状态记录为 `FAILED`，并允许后续重试。
4. 不引入 Seata。
5. 不引入消息队列补偿机制。

---

## 12. V1 不做的能力边界

V1 明确不做以下能力：

| 能力 | V1 处理方式 |
|---|---|
| 简历 PDF / Word 上传解析 | 不做，简历手动录入 |
| MinIO 文件存储 | 不做 |
| RabbitMQ / RocketMQ 异步任务 | 不做，同步处理 |
| Elasticsearch 全文搜索 | 不做，使用 MySQL 条件查询 |
| AI 题目批量生成与审核 | 不做 |
| 学习计划自动生成 | 不做 |
| AI 简历优化 | 不做 |
| Prompt 版本管理 | 不做，仅维护简化版 Prompt 模板 |
| SSE 流式输出 | 不做，普通 HTTP 同步返回 |
| Embedding 语义去重 | 不做 |
| AI 自动判重 | 不做 |
| 题目关系网络 | 不做 |
| 完整数据看板 | 不做，只做简化统计 |
| 完整操作日志审计 | 不做，common-log 仅预留 |
| 登录日志中心 | 不做 |
| 站内通知 | 不做 |
| Docker Compose 一键部署 | V1 可后置 |
| 分布式事务 | 不做 |
| 支付 / 会员 / 企业端 / 招聘投递 | 不做 |
| 语音 / 视频面试 | 不做 |

V1 的核心边界是：

```text
微服务骨架
  ↓
用户认证
  ↓
题库与问题组
  ↓
简历手动录入
  ↓
AI 模拟面试
  ↓
动态追问
  ↓
面试记录
  ↓
面试报告
```

---

## 13. 后续 V2 / V3 扩展预留说明

V1 不实现 V2 / V3 功能，但架构上保留可扩展空间。

### 13.1 V2 预留方向

V2 可在 V1 基础上扩展：

1. 拆出 practice-service，承接刷题、错题、收藏、掌握状态。
2. 新增 file-service，支持简历文件上传。
3. 增强 resume-service，支持简历解析状态、AI 结构化解析、简历优化记录。
4. 增强 question-service，支持 AI 题目生成审核、文本归一化、疑似重复题提示。
5. 增强 ai-service，支持简历解析、简历优化、题目生成、学习计划、题库去重判断和 Prompt 版本管理。
6. 增强 system-service，支持系统配置分类、配置说明和更完善的管理端首页统计。
7. 增加 SSE 流式输出能力。

V1 预留点：

1. question 表保留 `normalized_title` 字段。
2. common-ai 保留结构化输出工具。
3. ai-service 独立封装模型调用。
4. service 包结构保留 `mq` 包。
5. Prompt 模板不写死在业务代码中。

### 13.2 V3 预留方向

V3 可在 V2 基础上完善工程化能力：

1. 新增 search-service，接入 Elasticsearch。
2. 新增 task-service，处理异步任务、重试、死信。
3. 文件服务升级 MinIO。
4. 消息中间件处理简历解析、报告生成、题目生成、ES 同步。
5. Redis 增强缓存、幂等、分布式锁。
6. 完整操作日志、登录日志、通知中心。
7. Docker Compose 一键部署。
8. Embedding 语义去重和题目关系网络增强。

V1 预留点：

1. common-redis 预留限流和防重复提交工具。
2. common-log 预留 TraceId 和日志上下文。
3. 服务间调用统一通过 Spring Cloud OpenFeign，便于后续拆分更多服务。
4. 数据按服务归属划分，便于后续拆库或服务独立演进。
5. interview-service 将报告生成封装为独立业务动作，后续可改造为异步任务。

---

## 14. V1 架构验收标准

V1 架构完成后，应满足：

1. 各服务能注册到 Nacos。
2. 前端请求统一经过 Gateway。
3. Gateway 能正确路由到各业务服务。
4. 登录认证链路可跑通。
5. 服务间通过 Spring Cloud OpenFeign 调用。
6. MyBatis-Plus 能正常操作 MySQL。
7. Redis 能完成登录态或基础缓存。
8. AI 调用统一经过 codecoachai-ai。
9. AI 调用日志能在后台查询。
10. Prompt 模板能在后台维护。
11. 核心接口有 Knife4j 文档。
12. 每个服务包结构符合分层规范。
13. interview-service 能编排 question-service、resume-service 和 ai-service，完成 AI 面试主流程。
14. V1 不引入超出边界的 V2 / V3 能力。

---

## 15. 结论

CodeCoachAI V1 的微服务架构设计原则是：

> 微服务骨架先定下来，业务功能收敛到 AI 面试核心闭环。

V1 采用 Gateway + Auth + User + Question + Resume + Interview + AI + System 的轻量微服务拆分。

其中：

1. Gateway 负责统一入口。
2. Auth 负责认证。
3. User 负责用户资料和角色。
4. Question 负责题库、刷题和问题组。
5. Resume 负责简历和项目经历。
6. Interview 负责面试流程、状态机和报告。
7. AI 负责大模型调用、Prompt 和 AI 日志。
8. System 负责系统配置、管理端首页简化统计和可选菜单查询。
9. Common 负责统一返回、异常、安全、MyBatis、Redis、Spring Cloud OpenFeign、AI 通用对象和日志基础能力。

该架构能支撑 V1 快速落地，同时为后续 V2 / V3 的文件服务、搜索服务、任务服务、异步化、语义去重和工程化增强保留扩展空间。
