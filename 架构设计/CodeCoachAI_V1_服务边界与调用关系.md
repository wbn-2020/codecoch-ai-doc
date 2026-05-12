# CodeCoachAI V1 服务边界与调用关系

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 服务边界与调用关系 |
| 项目阶段 | V1 |
| 最高依据 | PRD/CodeCoachAI_PRD_V1_微服务分层架构版.md |
| 前置文档 | 架构设计/CodeCoachAI_V1_微服务架构设计.md |
| 文档用途 | 明确 V1 服务职责、数据归属、Feign 内部调用和核心业务调用链路 |

本文档只针对 CodeCoachAI V1。

V1 不提前设计和实现 V2 / V3 功能，不拆出 practice-service、file-service、search-service、task-service、notification-service。

---

## 2. V1 服务清单

V1 后端服务清单如下：

| 服务 | 类型 | 说明 |
|---|---|---|
| codecoachai-gateway | 网关服务 | 统一入口、路由、跨域、Token 基础校验 |
| codecoachai-auth | 认证服务 | 注册、登录、退出、Token 生成与校验 |
| codecoachai-user | 用户服务 | 用户资料、角色关系、用户基础信息 |
| codecoachai-question | 题库服务 | 题库、分类、标签、问题组、刷题、错题、收藏 |
| codecoachai-resume | 简历服务 | 简历手动录入、项目经历维护 |
| codecoachai-interview | 面试服务 | 面试会话、阶段、消息、状态机、报告 |
| codecoachai-ai | AI 服务 | AIClient、Prompt、AI 评分、追问、报告、调用日志 |
| codecoachai-system | 系统服务 | 系统配置、管理端基础能力 |
| codecoachai-common | 公共模块 | 统一返回、异常、安全、MyBatis、Redis、Feign、AI 通用对象 |

---

## 3. 服务职责边界

## 3.1 codecoachai-gateway

### 职责

1. 接收前端所有 HTTP 请求。
2. 按路由规则转发到后端服务。
3. 处理跨域。
4. 做 Token 基础校验。
5. 将用户上下文透传给下游服务。
6. 返回统一网关异常。

### 边界

1. 不处理业务逻辑。
2. 不直接访问数据库。
3. 不调用大模型 API。
4. 不保存业务数据。

---

## 3.2 codecoachai-auth

### 职责

1. 用户注册。
2. 用户登录。
3. 用户退出。
4. Token 生成。
5. Token 校验。
6. 查询当前登录用户。
7. 管理员登录。

### 边界

1. 不维护用户资料详情。
2. 不维护角色菜单完整权限模型。
3. 不处理题库、简历、面试业务。
4. 不管理 AI Prompt 和 AI 调用日志。

---

## 3.3 codecoachai-user

### 职责

1. 用户账号基础信息。
2. 用户资料维护。
3. 用户角色关系维护。
4. 用户状态管理。
5. 当前用户资料查询。
6. 用户基础学习概览。
7. 向 auth-service 提供用户查询接口。

### 边界

1. 不生成和校验 Token。
2. 不保存登录态。
3. 不保存简历、题库、面试记录。
4. 不保存 AI 调用日志。

---

## 3.4 codecoachai-question

### 职责

1. 题目管理。
2. 分类管理。
3. 标签管理。
4. 问题组管理。
5. 用户基础刷题。
6. 用户答题记录。
7. 错题本。
8. 收藏题目。
9. 题目掌握状态。
10. 面试抽题。
11. 基于 group_id 避免同场面试抽到同一考察意图的问题。

### 边界

1. 不生成 AI 面试问题。
2. 不评价用户面试回答。
3. 不管理面试阶段和面试状态。
4. 不保存简历数据。
5. 不做 Embedding 语义去重。
6. 不做 AI 自动判重。

---

## 3.5 codecoachai-resume

### 职责

1. 简历手动录入。
2. 简历列表、详情、编辑、删除。
3. 设置默认简历。
4. 项目经历新增、编辑、删除、查询。
5. 向 interview-service 提供简历和项目经历查询接口。

### 边界

1. 不上传 PDF / Word 简历文件。
2. 不解析简历文件。
3. 不做 AI 简历优化。
4. 不生成项目深挖问题。
5. 不保存面试记录。

---

## 3.6 codecoachai-interview

### 职责

1. 创建面试会话。
2. 生成面试阶段。
3. 管理面试状态机。
4. 获取当前问题。
5. 保存 AI 问题和用户回答。
6. 调用 ai-service 进行评分、点评、追问和报告生成。
7. 控制每个主问题最多追问 2 次。
8. 推进下一题或下一阶段。
9. 结束面试。
10. 保存面试报告。
11. 查询面试历史、详情和报告。

### 边界

1. 不直接调用大模型 API。
2. 不直接读取题库数据库。
3. 不直接读取简历数据库。
4. 不维护 Prompt 模板。
5. 不保存 AI 调用日志。

---

## 3.7 codecoachai-ai

### 职责

1. 统一 AIClient。
2. 模型供应商适配。
3. Prompt 模板渲染。
4. 八股文问题生成。
5. 项目深挖问题生成。
6. 用户回答评分。
7. AI 点评生成。
8. 动态追问生成。
9. 面试报告内容生成。
10. Prompt 模板管理。
11. AI 调用日志记录和查询。

### 边界

1. 不控制面试阶段。
2. 不决定面试是否结束。
3. 不直接读写 interview 表。
4. 不直接读写 question 表。
5. 不直接读写 resume 表。
6. 不维护用户资料。

---

## 3.8 codecoachai-system

### 职责

1. 系统配置。
2. 数据字典，可选。
3. 角色、菜单、权限轻量管理。
4. 管理端首页简化统计。

### 边界

1. 不承接题目管理，题目归 question-service。
2. 不承接 Prompt 管理，Prompt 归 ai-service。
3. 不承接 AI 调用日志，AI 日志归 ai-service。
4. 不做完整操作日志。
5. 不做站内通知。
6. 不做复杂数据看板。

---

## 4. 数据归属

## 4.1 数据归属原则

1. 谁负责业务，谁管理数据。
2. 一个服务只能直接读写自己负责的数据表。
3. 跨服务数据访问通过 Feign 接口完成。
4. V1 不允许服务直接跨库或跨模块操作其他服务的数据表。
5. V1 不做分布式事务，跨服务流程通过状态字段和失败提示兜底。

## 4.2 数据表归属表

| 数据域 | 表名 | 归属服务 | 说明 |
|---|---|---|---|
| 用户权限 | `sys_user` | user-service | 用户账号、密码、昵称、邮箱、状态 |
| 用户权限 | `sys_role` | user-service | 普通用户、管理员角色 |
| 用户权限 | `sys_user_role` | user-service | 用户角色关系 |
| 题库 | `question_category` | question-service | 题目分类 |
| 题库 | `question_tag` | question-service | 题目标签 |
| 题库 | `question_group` | question-service | 同一考察意图的问题组 |
| 题库 | `question` | question-service | 题目主体、答案、解析、难度 |
| 题库 | `question_tag_relation` | question-service | 题目和标签关系 |
| 刷题 | `user_question_record` | question-service | 用户答题记录 |
| 刷题 | `wrong_question` | question-service | 错题记录 |
| 刷题 | `favorite_question` | question-service | 收藏题目 |
| 刷题 | `user_question_mastery` | question-service | 掌握状态 |
| 简历 | `resume` | resume-service | 简历主体 |
| 简历 | `resume_project` | resume-service | 项目经历 |
| 面试 | `interview_session` | interview-service | 面试会话 |
| 面试 | `interview_stage` | interview-service | 面试阶段 |
| 面试 | `interview_message` | interview-service | 面试问答消息 |
| 面试 | `interview_report` | interview-service | 面试报告 |
| AI | `prompt_template` | ai-service | Prompt 模板 |
| AI | `ai_call_log` | ai-service | AI 调用日志 |
| 系统 | `system_config` | system-service | 系统配置 |
| 系统 | `sys_menu` | system-service | V1 可选，后台菜单 |
| 系统 | `sys_permission` | system-service | V1 可选，权限标识 |

## 4.3 典型数据访问规则

| 场景 | 正确方式 | 不允许方式 |
|---|---|---|
| auth 查询用户密码 | auth-service Feign 调 user-service | auth-service 直接查 `sys_user` |
| interview 获取题目 | interview-service Feign 调 question-service | interview-service 直接查 `question` |
| interview 获取简历 | interview-service Feign 调 resume-service | interview-service 直接查 `resume` |
| interview 调用 AI | interview-service Feign 调 ai-service | interview-service 直接调用模型 SDK |
| 管理 AI 日志 | ai-service 查询 `ai_call_log` | system-service 直接查 `ai_call_log` |
| 管理题库 | question-service 查询题库表 | system-service 直接管理题库表 |

---

## 5. Feign 内部调用设计

## 5.1 Feign 调用原则

1. 只为服务间必要调用设计 Feign 接口。
2. Feign 接口返回 DTO，不直接暴露 Entity。
3. Feign 接口需要透传用户上下文。
4. Feign 调用异常需要统一转换为业务异常。
5. Feign 接口优先服务核心闭环，不提前设计 V2 / V3 能力。

## 5.2 V1 Feign 调用清单

| 调用方 | 被调用方 | Feign Client | 主要用途 |
|---|---|---|---|
| auth-service | user-service | `UserFeignClient` | 注册前检查用户、登录时查询用户、获取用户角色 |
| interview-service | question-service | `QuestionFeignClient` | 按条件抽题、查询题目详情、获取参考答案、校验 group_id |
| interview-service | resume-service | `ResumeFeignClient` | 查询简历详情、查询项目经历、校验简历归属 |
| interview-service | ai-service | `AiFeignClient` | 生成面试问题、评分点评、生成追问、生成报告 |
| gateway | auth-service，可选 | `AuthFeignClient` 或 Token 工具 | Token 校验，具体可由 common-security 实现 |
| system-service | user-service，可选 | `UserFeignClient` | 管理端统计用户数量或用户概览 |

## 5.3 auth-service 调 user-service

### 用途

1. 用户注册前检查账号是否存在。
2. 注册成功时创建用户。
3. 登录时查询用户账号、密码、状态。
4. 登录后查询用户角色。

### 内部接口建议

```text
GET  /inner/users/by-username
POST /inner/users
GET  /inner/users/{id}
GET  /inner/users/{id}/roles
```

说明：`/inner/**` 表示服务内部调用接口，不直接暴露给前端。

## 5.4 interview-service 调 question-service

### 用途

1. 按面试阶段、分类、难度、经验年限抽题。
2. 排除本场面试已使用的 group_id。
3. 获取题目参考答案和解析。
4. 查询推荐练习题，用于报告中的推荐题。

### 内部接口建议

```text
POST /inner/questions/pick-for-interview
GET  /inner/questions/{id}
POST /inner/questions/recommend-for-report
```

### 抽题入参重点

```text
categoryCode
difficulty
experienceLevel
excludeGroupIds
questionCount
interviewMode
```

### 抽题出参重点

```text
questionId
groupId
title
content
referenceAnswer
analysis
difficulty
categoryName
tagNames
knowledgePoints
```

## 5.5 interview-service 调 resume-service

### 用途

1. 创建面试时校验简历是否属于当前用户。
2. 项目深挖面试获取简历详情。
3. 综合模拟面试获取项目经历和技术栈。

### 内部接口建议

```text
GET /inner/resumes/{id}
GET /inner/resumes/{id}/projects
GET /inner/users/{userId}/default-resume
```

### 返回数据重点

```text
resumeId
userId
resumeName
targetPosition
skills
workSummary
education
projects
```

## 5.6 interview-service 调 ai-service

### 用途

1. 生成八股文自然面试问题。
2. 生成项目深挖问题。
3. 对用户回答评分和点评。
4. 判断是否建议追问。
5. 生成追问问题。
6. 生成面试报告内容。

### 内部接口建议

```text
POST /inner/ai/interview/question
POST /inner/ai/interview/evaluate
POST /inner/ai/interview/follow-up
POST /inner/ai/interview/report
```

### AI 评分返回重点

```text
score
comment
missingPoints
suggestFollowUp
followUpReason
followUpDirection
knowledgePoints
```

## 5.7 system-service 可选调 user-service

### 用途

1. 管理端首页简化统计用户数量。
2. 系统配置页展示基础用户统计。

### 内部接口建议

```text
GET /inner/users/statistics
```

V1 如果管理端首页统计先做静态或简化展示，可以不实现该 Feign 调用。

---

## 6. V1 先不拆服务的功能

| 功能 | V1 所属服务 | 后续可能拆分 | V1 不拆原因 |
|---|---|---|---|
| 刷题记录 | question-service | practice-service | V1 刷题能力较轻 |
| 错题本 | question-service | practice-service | 与题目强相关 |
| 收藏题目 | question-service | practice-service | 与题目强相关 |
| 掌握状态 | question-service | practice-service | 与题目和练习记录强相关 |
| 文件上传 | 不做 | file-service | V1 简历手动录入 |
| 简历文件解析 | 不做 | file-service / resume-service | V1 不做 PDF / Word 解析 |
| 全文搜索 | question-service MySQL 查询 | search-service | V1 不引入 Elasticsearch |
| AI 异步任务 | ai-service 同步处理 | task-service | V1 不引入 MQ |
| 面试报告异步生成 | interview-service 同步调用 ai-service | task-service | V1 优先跑通闭环 |
| 站内通知 | 不做 | notification-service | 非 V1 核心闭环 |
| 操作日志 | common-log 预留 | system-service / task-service | V1 不做完整审计 |

---

## 7. 服务调用关系表

| 业务场景 | 入口服务 | 内部调用 | 数据写入服务 | 说明 |
|---|---|---|---|---|
| 用户注册 | auth-service | user-service | user-service、auth-service/Redis | auth 负责认证流程，user 负责用户数据 |
| 用户登录 | auth-service | user-service | auth-service/Redis | auth 查询用户后生成 Token |
| 用户资料修改 | user-service | 无 | user-service | 用户资料归 user |
| 题库管理 | question-service | 无 | question-service | 管理员维护题库数据 |
| 基础刷题 | question-service | 无 | question-service | 答题记录、错题、收藏归 question |
| 简历管理 | resume-service | 无 | resume-service | 简历和项目经历归 resume |
| 创建面试 | interview-service | resume-service 可选 | interview-service | 校验简历并创建会话和阶段 |
| 获取面试问题 | interview-service | question-service、resume-service、ai-service | interview-service、ai-service | interview 编排上下文，ai 生成内容并记日志 |
| 提交回答 | interview-service | ai-service | interview-service、ai-service | ai 评分点评，interview 保存结果 |
| 动态追问 | interview-service | ai-service | interview-service、ai-service | interview 控制追问次数 |
| 面试报告 | interview-service | ai-service、question-service 可选 | interview-service、ai-service | ai 生成报告内容，interview 保存报告 |
| Prompt 管理 | ai-service | 无 | ai-service | Prompt 归 ai |
| AI 日志查询 | ai-service | 无 | ai-service | AI 调用日志归 ai |
| 系统配置 | system-service | 无 | system-service | 业务参数归 system |

---

## 8. 核心业务流程调用链路

## 8.1 登录注册流程

### 注册流程

```text
前端注册页
  ↓
gateway
  ↓
auth-service: register
  ↓
user-service: 检查用户名/邮箱是否存在
  ↓
user-service: 创建 sys_user、绑定默认普通用户角色
  ↓
auth-service: 返回注册结果
  ↓
前端提示注册成功
```

### 注册职责划分

| 步骤 | 负责服务 | 说明 |
|---|---|---|
| 参数校验 | auth-service | 校验用户名、密码、邮箱等 |
| 检查账号是否存在 | user-service | 查询 `sys_user` |
| 创建用户 | user-service | 写入用户和默认角色 |
| 返回结果 | auth-service | 统一注册响应 |

### 登录流程

```text
前端登录页
  ↓
gateway
  ↓
auth-service: login
  ↓
user-service: 根据账号查询用户、角色、状态
  ↓
auth-service: 校验密码
  ↓
auth-service: 生成 Token
  ↓
Redis: 保存登录态或权限缓存
  ↓
auth-service: 返回 LoginVO
  ↓
前端保存 Token
```

### 登录职责划分

| 步骤 | 负责服务 | 说明 |
|---|---|---|
| 登录参数校验 | auth-service | 校验账号密码 |
| 查询用户 | user-service | 返回用户账号、密码、状态 |
| 密码校验 | auth-service | 校验加密密码 |
| Token 生成 | auth-service | Sa-Token 或 JWT |
| 登录态缓存 | auth-service / Redis | 保存 Token 或会话 |
| 角色返回 | user-service | 返回角色编码 |

---

## 8.2 题库管理流程

### 新增题目流程

```text
管理员前端
  ↓
gateway
  ↓
question-service: 新增题目
  ↓
question-service: 校验分类是否存在
  ↓
question-service: 校验标签是否存在
  ↓
question-service: 校验或创建问题组
  ↓
question-service: 保存 question
  ↓
question-service: 保存 question_tag_relation
  ↓
返回新增成功
```

### 题库管理职责划分

| 功能 | 负责服务 | 数据表 |
|---|---|---|
| 分类 CRUD | question-service | `question_category` |
| 标签 CRUD | question-service | `question_tag` |
| 问题组 CRUD | question-service | `question_group` |
| 题目 CRUD | question-service | `question` |
| 题目标签关系 | question-service | `question_tag_relation` |

### V1 题库规则

1. 管理员新增题目时必须选择已有问题组或新建问题组。
2. 同一问题组表示同一核心考察意图。
3. V1 不做 AI 自动判重。
4. V1 不做 Embedding 语义相似度检测。
5. V1 可保留 `normalized_title` 字段，但不实现复杂去重。

---

## 8.3 简历管理流程

### 新增简历流程

```text
用户前端
  ↓
gateway
  ↓
resume-service: 新增简历
  ↓
resume-service: 校验当前用户
  ↓
resume-service: 保存 resume
  ↓
resume-service: 保存 resume_project 列表
  ↓
返回简历 ID
```

### 设置默认简历流程

```text
用户前端
  ↓
gateway
  ↓
resume-service: 设置默认简历
  ↓
resume-service: 校验简历归属当前用户
  ↓
resume-service: 取消该用户其他默认简历
  ↓
resume-service: 设置当前简历为默认
  ↓
返回设置成功
```

### 简历管理职责划分

| 功能 | 负责服务 | 数据表 |
|---|---|---|
| 简历列表 | resume-service | `resume` |
| 简历详情 | resume-service | `resume`、`resume_project` |
| 新增简历 | resume-service | `resume` |
| 编辑简历 | resume-service | `resume` |
| 删除简历 | resume-service | `resume` |
| 设置默认简历 | resume-service | `resume` |
| 项目经历维护 | resume-service | `resume_project` |

### V1 简历边界

1. 只支持手动录入。
2. 不支持简历文件上传。
3. 不支持 AI 简历解析。
4. 不支持 AI 简历优化。

---

## 8.4 创建面试流程

### 创建面试调用链路

```text
用户前端创建面试页
  ↓
gateway
  ↓
interview-service: createInterview
  ↓
resume-service: 如果选择简历，校验简历归属并获取简历摘要
  ↓
interview-service: 创建 interview_session
  ↓
interview-service: 根据面试模式生成 interview_stage
  ↓
interview-service: 返回 sessionId
```

### 创建面试职责划分

| 步骤 | 负责服务 | 说明 |
|---|---|---|
| 校验面试配置 | interview-service | 模式、岗位、经验、难度、风格 |
| 校验简历归属 | resume-service | 当前用户只能使用自己的简历 |
| 创建面试会话 | interview-service | 写入 `interview_session` |
| 生成面试阶段 | interview-service | 写入 `interview_stage` |
| 返回会话 ID | interview-service | 前端进入面试房间 |

### 面试阶段生成规则

八股文技术面试阶段建议：

```text
Java 基础
集合框架
并发
JVM
Spring
MySQL
Redis
```

项目深挖面试阶段建议：

```text
项目背景
个人职责
技术选型
核心流程
数据库与缓存
难点与优化
扩展性问题
```

综合模拟面试阶段建议：

```text
开场
Java 基础
数据库
缓存与中间件
框架基础
项目深挖
场景设计
总结报告
```

---

## 8.5 提交回答与 AI 追问流程

### 获取当前问题流程

```text
用户进入面试房间
  ↓
gateway
  ↓
interview-service: getCurrentQuestion
  ↓
interview-service: 获取当前 session 和 stage
  ↓
question-service: 八股文阶段按条件抽题，并排除已使用 group_id
  ↓
resume-service: 项目深挖阶段获取简历和项目经历，可选
  ↓
ai-service: 根据题目/简历/阶段生成自然面试问题
  ↓
ai-service: 记录 ai_call_log
  ↓
interview-service: 保存 AI 问题到 interview_message
  ↓
返回当前问题
```

### 提交回答与评分流程

```text
用户提交回答
  ↓
gateway
  ↓
interview-service: submitAnswer
  ↓
interview-service: 校验 session 状态
  ↓
interview-service: 保存用户回答
  ↓
ai-service: 基于问题、参考答案、用户回答进行评分和点评
  ↓
ai-service: 返回 score、comment、是否建议追问、追问方向
  ↓
ai-service: 记录 ai_call_log
  ↓
interview-service: 保存评分和点评
  ↓
interview-service: 判断是否追问
```

### 动态追问流程

```text
interview-service: 判断 AI 是否建议追问
  ↓
interview-service: 查询当前主问题已追问次数
  ↓
如果未超过 2 次
  ↓
ai-service: 生成追问问题
  ↓
ai-service: 记录 ai_call_log
  ↓
interview-service: 保存追问消息，parent_message_id 指向主问题
  ↓
返回追问问题

如果已超过 2 次
  ↓
interview-service: 进入下一题或下一阶段
  ↓
返回下一题或阶段完成状态
```

### 提交回答职责划分

| 能力 | 负责服务 |
|---|---|
| 校验面试状态 | interview-service |
| 保存用户回答 | interview-service |
| AI 评分点评 | ai-service |
| 判断是否追问 | ai-service 给建议，interview-service 最终决定 |
| 控制最大追问次数 | interview-service |
| 生成追问内容 | ai-service |
| 保存追问关系 | interview-service |
| 记录 AI 调用日志 | ai-service |

### 追问控制规则

1. AI 可以建议追问，但不能直接控制流程。
2. 后端状态机由 interview-service 控制。
3. 每个主问题最多追问 2 次。
4. 超过追问次数后进入下一题或下一阶段。
5. 每轮问答必须及时保存，避免刷新页面导致数据丢失。

---

## 8.6 面试报告生成流程

### 报告生成调用链路

```text
用户主动结束面试或系统达到结束条件
  ↓
gateway
  ↓
interview-service: finishInterview
  ↓
interview-service: 更新 session 状态为 REPORT_GENERATING
  ↓
interview-service: 汇总 interview_stage 和 interview_message
  ↓
question-service: 获取推荐练习题，可选
  ↓
ai-service: 生成结构化面试报告内容
  ↓
ai-service: 记录 ai_call_log
  ↓
interview-service: 保存 interview_report
  ↓
interview-service: 更新 session 状态为 COMPLETED
  ↓
返回报告 ID 或报告详情
```

### 报告生成职责划分

| 步骤 | 负责服务 | 说明 |
|---|---|---|
| 结束面试 | interview-service | 校验会话归属和状态 |
| 汇总问答记录 | interview-service | 查询 `interview_message` |
| 统计阶段得分 | interview-service | 查询 `interview_stage` |
| 推荐练习题 | question-service | V1 可选 |
| 生成报告文本 | ai-service | 调用报告生成 Prompt |
| 保存 AI 日志 | ai-service | 写入 `ai_call_log` |
| 保存报告 | interview-service | 写入 `interview_report` |
| 更新会话状态 | interview-service | `COMPLETED` 或 `FAILED` |

### 报告内容

V1 报告包含：

1. 总分。
2. 各阶段得分。
3. 知识点掌握情况。
4. 回答亮点。
5. 主要问题。
6. 项目表达问题。
7. 薄弱知识点。
8. 推荐复习方向。
9. 推荐练习题。
10. 完整问答明细。

### V1 报告边界

1. V1 同步生成报告。
2. 不使用消息队列。
3. 不使用异步任务中心。
4. 不使用 SSE 流式输出。
5. 报告生成失败时记录失败状态并提示用户。

---

## 9. 内部接口与外部接口边界

## 9.1 外部接口

外部接口指前端通过 Gateway 访问的接口。

示例：

```text
POST /auth/register
POST /auth/login
GET  /questions
GET  /questions/{id}
POST /resumes
POST /interviews
POST /interviews/{id}/answer
GET  /interviews/{id}/report
GET  /admin/questions
POST /admin/questions
GET  /admin/ai/logs
```

外部接口需要：

1. 走 Gateway。
2. 校验 Token。
3. 管理端接口校验管理员角色。
4. 返回统一响应结构。

## 9.2 内部接口

内部接口指服务间 Feign 调用接口。

建议统一使用 `/inner/**` 前缀。

示例：

```text
GET  /inner/users/by-username
POST /inner/questions/pick-for-interview
GET  /inner/resumes/{id}
POST /inner/ai/interview/evaluate
POST /inner/ai/interview/report
```

内部接口要求：

1. 不直接暴露给前端。
2. 通过网关路由或服务间直连访问时需要有内部调用保护策略。
3. DTO 尽量稳定，避免直接返回数据库 Entity。
4. 必须透传用户上下文或明确传入 userId。

---

## 10. V1 结论

V1 服务边界的核心原则是：

> interview-service 控制面试流程，ai-service 提供 AI 能力，question-service 和 resume-service 提供上下文数据。

各服务核心关系如下：

```text
auth-service 依赖 user-service 完成登录注册

interview-service 依赖：
  question-service 获取八股文题目
  resume-service 获取简历项目上下文
  ai-service 生成问题、评分、追问和报告

ai-service 独立负责：
  Prompt 模板
  模型调用
  AI 调用日志

question-service 独立负责：
  题库
  问题组
  刷题
  错题
  收藏

resume-service 独立负责：
  手动简历
  项目经历
```

V1 暂不拆出 practice-service、file-service、search-service、task-service 和 notification-service，确保第一阶段聚焦 AI 模拟面试核心闭环。
