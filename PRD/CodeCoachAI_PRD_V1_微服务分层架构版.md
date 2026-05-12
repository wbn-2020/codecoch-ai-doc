# 在·CodeCoachAI V1 PRD - 微服务骨架与分层架构版

## 1. 文档信息

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 PRD - 微服务骨架与分层架构版 |
| 后端项目名 | CodeCoachAI-java |
| 前端项目名 | CodeCoachAI-vue |
| 项目定位 | 面向 Java 求职者的个人作品集项目 |
| V1 目标 | 跑通 AI 模拟面试核心闭环，并在 V1 阶段确定 Spring Cloud Alibaba 微服务骨架 |
| 目标用户 | 准备 Java 后端面试的求职者，重点面向 1-3 年经验 Java 开发者 |
| 管理角色 | 管理员维护题库、问题组、Prompt 模板、AI 调用日志和基础系统配置 |
| 技术路线 | Spring Cloud Alibaba + Vue3 + MySQL + Redis + AI 大模型 API |
| 文档用途 | 指导 V1 数据库设计、接口设计、后端服务拆分、前端页面开发和后续 AI 编程助手开发 |

---

## 2. V1 版本定位

V1 版本不是完整商业系统，也不包装成真实企业项目。  
V1 的核心定位是：

> 个人作品集项目：基于 Spring Cloud Alibaba 微服务架构的 AI Java 面试训练平台，重点展示用户认证、题库管理、简历录入、AI 模拟面试、动态追问、面试记录、面试报告和基础 AI 工程化能力。

V1 要解决的问题：

1. 普通刷题网站只能看题和答案，缺少真实面试追问。
2. 用户不知道自己的简历项目会被如何深挖。
3. AI 问答容易变成普通聊天，需要后端控制面试流程。
4. 题库存在不同问法但同一考察意图的问题，需要 question_group 做基础去重。
5. AI 调用需要统一封装、Prompt 模板和调用日志，不能在业务代码中直接调用模型 API。

---

## 3. V1 核心目标

V1 的目标是完成以下闭环：

```text
用户注册登录
  ↓
管理员维护题库、分类、标签、问题组
  ↓
用户维护简历和项目经历
  ↓
用户创建 AI 模拟面试
  ↓
系统生成面试会话和面试阶段
  ↓
AI 根据题库、简历、面试配置提出问题
  ↓
用户回答
  ↓
AI 评分、点评、判断是否追问
  ↓
系统控制追问次数和阶段流转
  ↓
面试结束
  ↓
生成结构化面试报告
  ↓
用户查看历史面试和报告
```

V1 的技术目标：

1. 后端项目采用 Maven 多模块结构。
2. V1 阶段确定 Spring Cloud Alibaba 微服务骨架。
3. 使用 Nacos 做服务注册与配置管理。
4. 使用 Spring Cloud Gateway 作为统一入口。
5. 使用 OpenFeign 完成服务间调用。
6. 使用 Sa-Token 或 JWT 完成登录认证。
7. 使用 MyBatis-Plus 操作 MySQL。
8. 使用 Redis 做登录态、权限缓存、AI 调用限流和面试上下文缓存。
9. 使用统一 AI 服务封装大模型调用。
10. 每个微服务内部采用 controller / service / service.impl / mapper / domain 分层。

---

## 4. V1 做什么、不做什么

### 4.1 V1 必须做

| 模块 | 是否 V1 必做 | 说明 |
|---|---:|---|
| 微服务基础骨架 | 是 | Gateway、Nacos、OpenFeign、common 公共模块 |
| 用户注册登录 | 是 | 普通用户和管理员登录 |
| 用户信息管理 | 是 | 当前用户信息、基础资料 |
| 管理员后台 | 是 | 题库、分类、标签、问题组、Prompt、AI 日志 |
| 题库管理 | 是 | 题目 CRUD、分类、标签、难度、题型 |
| question_group 问题组 | 是 | V1 手动分组，避免同一场面试重复同类问题 |
| 基础刷题 | 是 | 题目列表、详情、提交答案、掌握状态 |
| 错题与收藏 | 是，简化版 | 收藏题、不会题、错题列表 |
| 简历手动录入 | 是 | 不做文件上传解析 |
| 项目经历维护 | 是 | 项目深挖基础数据 |
| AI 模拟面试 | 是 | 八股文、项目深挖、综合模拟 |
| 面试大纲 | 是 | V1 规则生成，少量 AI 辅助 |
| 动态追问 | 是 | 每个主问题最多追问 2 次 |
| 面试记录 | 是 | 保存问答、评分、点评、追问关系 |
| 面试报告 | 是 | 总分、模块分、薄弱点、复习建议 |
| Prompt 模板 | 是，简化版 | 支持后台维护和变量替换 |
| AI 调用日志 | 是，简化版 | 记录场景、请求、响应、耗时、状态 |

### 4.2 V1 暂不做

| 功能 | 放到后续版本原因 |
|---|---|
| 简历 PDF / Word 上传解析 | V2 / V3 再引入文件服务和解析任务 |
| MinIO 文件存储 | V1 先手动录入简历 |
| RabbitMQ / RocketMQ 异步任务 | V1 先同步生成报告，减少复杂度 |
| Elasticsearch 全文搜索 | V1 用 MySQL 条件查询 |
| AI 题目批量生成与审核 | V2 再做 |
| 学习计划自动生成 | V2 再做 |
| Embedding 语义去重 | V3 再做 |
| 数据看板复杂统计 | V2 / V3 增强 |
| 操作日志完整审计 | V3 完善 |
| 站内通知 | V3 完善 |
| 微服务分布式事务 | V1 避免跨服务强事务 |
| 语音 / 视频面试 | 不做 |
| 支付 / 会员 / 企业端 / 招聘投递 | 不做 |

---

## 5. V1 微服务架构

### 5.1 V1 服务清单

V1 采用微服务骨架，但服务数量控制在可开发范围内。

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

### 5.2 V1 暂不单独拆出的服务

| 后续服务 | V1 处理方式 |
|---|---|
| practice-service | V1 刷题记录、错题、收藏先放在 question-service |
| file-service | V1 不做文件上传，暂不拆 |
| search-service | V1 用 MySQL 查询，暂不拆 |
| task-service | V1 同步处理 AI 任务，暂不拆 |
| notification-service | V1 不做站内通知，暂不拆 |

### 5.3 服务调用关系

```text
CodeCoachAI-vue
  ↓
codecoachai-gateway
  ↓
auth / user / question / resume / interview / ai / system
  ↓
MySQL / Redis / 大模型 API
```

V1 中最核心的调用关系：

```text
interview-service
  ├── 调用 question-service 获取题目、问题组、参考答案
  ├── 调用 resume-service 获取简历和项目经历
  ├── 调用 ai-service 生成问题、评分、追问、报告
  └── 写入 interview_session / interview_stage / interview_message / interview_report
```

---

## 6. V1 技术栈

### 6.1 后端技术栈

| 类型 | 技术 |
|---|---|
| 开发语言 | Java 17 |
| 基础框架 | Spring Boot 3 |
| 微服务框架 | Spring Cloud Alibaba |
| 注册与配置 | Nacos |
| 网关 | Spring Cloud Gateway |
| 服务调用 | OpenFeign |
| 限流熔断 | Sentinel，V1 可先接入基础依赖和简单规则 |
| ORM | MyBatis-Plus |
| 数据库 | MySQL 8 |
| 缓存 | Redis |
| 登录认证 | Sa-Token，或 JWT |
| 接口文档 | Knife4j / Swagger |
| 参数校验 | Jakarta Validation |
| 工具库 | Lombok、Hutool |
| AI 接入 | 大模型 Chat API |
| 部署 | V1 本地启动，Docker Compose 放后续版本 |

### 6.2 前端技术栈

| 类型 | 技术 |
|---|---|
| 项目名 | CodeCoachAI-vue |
| 框架 | Vue 3 |
| 构建工具 | Vite |
| 语言 | TypeScript |
| UI 组件 | Element Plus |
| 路由 | Vue Router |
| 状态管理 | Pinia |
| HTTP 请求 | Axios |
| 图表 | ECharts |
| Markdown | Markdown 渲染组件 |

---

## 7. 服务内部统一分层规范

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

V1 阶段如果某服务没有 mq 消息处理，可以保留包结构但不实现。

### 7.1 controller 层

职责：

1. 接收 HTTP 请求。
2. 参数校验。
3. 调用 service。
4. 返回统一响应对象。

禁止：

1. 写复杂业务逻辑。
2. 直接调用 mapper。
3. 直接调用 AI SDK。

### 7.2 service 层

职责：

1. 定义业务能力。
2. 命名应体现业务动作。
3. 不要只保留 save、update、delete 这类 CRUD 方法。

示例：

```text
createInterview()
startInterview()
submitAnswer()
finishInterview()
generateReport()
```

### 7.3 service.impl 层

职责：

1. 实现业务流程。
2. 编排 mapper、FeignClient、AI 服务调用。
3. 控制事务。
4. 处理状态流转。
5. 处理业务异常。

### 7.4 mapper 层

职责：

1. MyBatis-Plus 数据访问。
2. 单表 CRUD。
3. 简单关联查询。
4. 不写复杂业务判断。

### 7.5 domain.entity

职责：

1. 与数据库表结构对应。
2. 仅用于持久化。
3. 不直接返回前端。

### 7.6 domain.dto

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

### 7.7 domain.vo

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

### 7.8 domain.enums

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

### 7.9 convert 层

职责：

1. DTO 转 Entity。
2. Entity 转 VO。
3. List 转换。
4. 聚合对象转换。

V1 可以手写 Convert 类，后续可替换为 MapStruct。

### 7.10 feign 层

职责：

1. 当前服务调用其他微服务。
2. 用户上下文透传。
3. 服务间接口隔离。

示例：

```text
QuestionFeignClient
ResumeFeignClient
AiFeignClient
UserFeignClient
```

### 7.11 config 层

职责：

1. 当前服务配置类。
2. MyBatis、Redis、Feign、Swagger、AI 配置。

### 7.12 mq 层

V1 暂不强制使用消息队列，但预留：

```text
mq
├── producer
└── listener
```

---

## 8. V1 公共模块设计

### 8.1 common-core

包含：

1. 统一返回对象 Result。
2. 分页对象 PageResult。
3. 统一错误码 ErrorCode。
4. 业务异常 BusinessException。
5. 常量类。
6. 基础枚举。
7. 基础实体 BaseEntity。

### 8.2 common-web

包含：

1. 全局异常处理。
2. 参数校验异常处理。
3. 请求日志基础处理。
4. Web 工具类。
5. Controller 基础能力。

### 8.3 common-security

包含：

1. 登录用户上下文 LoginUser。
2. 用户 ID 获取工具。
3. 权限注解封装。
4. Token 校验工具。
5. 网关鉴权工具。
6. 用户信息透传。

### 8.4 common-mybatis

包含：

1. MyBatis-Plus 配置。
2. 分页插件。
3. 自动填充 created_at、updated_at。
4. 逻辑删除配置。
5. 通用字段处理。

### 8.5 common-redis

包含：

1. Redis Key 常量。
2. Redis 工具类。
3. 缓存工具。
4. 限流工具。
5. 防重复提交工具。

### 8.6 common-feign

包含：

1. Feign 配置。
2. Feign 拦截器。
3. 用户上下文透传。
4. 服务调用异常处理。
5. Feign 统一返回处理。

### 8.7 common-ai

包含：

1. AI 请求对象。
2. AI 响应对象。
3. AI 场景枚举。
4. Prompt 渲染工具。
5. AI 异常定义。
6. AI 结构化输出工具。

### 8.8 common-log

V1 简化实现：

1. TraceId 生成。
2. 请求日志基础记录。
3. 后续操作日志 AOP 预留。

---

## 9. V1 服务职责与包结构

## 9.1 codecoachai-gateway

### 职责

1. 统一请求入口。
2. 路由转发。
3. 跨域处理。
4. Token 基础校验。
5. 用户信息透传。
6. 基础限流预留。
7. 统一网关异常响应。

### 包结构

```text
com.codecoachai.gateway
├── config
├── filter
├── handler
└── util
```

---

## 9.2 codecoachai-auth

### 职责

1. 用户注册。
2. 用户登录。
3. 用户退出。
4. Token 生成。
5. Token 校验。
6. 当前登录用户信息。
7. 登录失败处理。
8. 管理员登录。

### 包结构

```text
com.codecoachai.auth
├── controller
│   └── AuthController
├── service
│   └── AuthService
├── service.impl
│   └── AuthServiceImpl
├── domain
│   ├── dto
│   │   ├── LoginDTO
│   │   └── RegisterDTO
│   ├── vo
│   │   ├── LoginVO
│   │   └── CurrentUserVO
│   └── enums
├── feign
│   └── UserFeignClient
└── config
```

---

## 9.3 codecoachai-user

### 职责

1. 用户信息管理。
2. 查询当前用户资料。
3. 修改昵称、头像、邮箱。
4. 用户角色关系。
5. 用户基础学习概览。
6. 向 auth-service 提供用户查询接口。

### 核心实体

```text
SysUser
SysRole
SysUserRole
```

### 包结构

```text
com.codecoachai.user
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
└── config
```

---

## 9.4 codecoachai-question

### 职责

1. 题目管理。
2. 分类管理。
3. 标签管理。
4. 问题组管理。
5. 基础刷题。
6. 答题记录。
7. 错题本。
8. 收藏题。
9. 掌握状态。
10. 面试抽题接口。
11. V1 手动问题组去重。

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

### V1 重点要求

1. 管理员新增题目时必须选择已有问题组或新建问题组。
2. 同一场练习或面试中避免重复抽取相同 group_id 的题目。
3. V1 不做 Embedding 语义去重。
4. V1 不做 AI 自动判重。
5. V1 可以保留 normalized_title 字段，为后续去重做准备。

### 包结构

```text
com.codecoachai.question
├── controller
│   ├── QuestionController
│   ├── QuestionCategoryController
│   ├── QuestionTagController
│   ├── QuestionGroupController
│   └── PracticeController
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
└── config
```

---

## 9.5 codecoachai-resume

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

### V1 简历字段

简历：

1. 简历名称。
2. 求职方向。
3. 技能栈。
4. 工作经历摘要。
5. 教育经历。
6. 默认简历标识。
7. 状态。

项目经历：

1. 项目名称。
2. 项目时间。
3. 项目背景。
4. 技术栈。
5. 个人职责。
6. 核心功能。
7. 技术难点。
8. 优化成果。
9. 补充说明。

### 包结构

```text
com.codecoachai.resume
├── controller
│   ├── ResumeController
│   └── ResumeProjectController
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
└── config
```

---

## 9.6 codecoachai-interview

### 职责

1. 创建面试。
2. 生成面试大纲。
3. 管理面试会话状态。
4. 管理面试阶段。
5. 获取当前问题。
6. 提交用户回答。
7. 调用 ai-service 评分与追问。
8. 控制追问次数。
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

### 包结构

```text
com.codecoachai.interview
├── controller
│   ├── InterviewController
│   └── InterviewReportController
├── service
│   ├── InterviewService
│   ├── InterviewStageService
│   ├── InterviewMessageService
│   └── InterviewReportService
├── service.impl
├── mapper
├── domain
│   ├── entity
│   ├── dto
│   ├── vo
│   └── enums
├── convert
├── feign
│   ├── QuestionFeignClient
│   ├── ResumeFeignClient
│   └── AiFeignClient
└── config
```

---

## 9.7 codecoachai-ai

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
AiModelConfig
```

V1 可以先只落地：

```text
PromptTemplate
AiCallLog
```

### AI 场景枚举

```text
INTERVIEW_QUESTION_GENERATE
INTERVIEW_ANSWER_EVALUATE
INTERVIEW_FOLLOW_UP_GENERATE
INTERVIEW_REPORT_GENERATE
PROJECT_DEEP_DIVE_QUESTION
```

### 包结构

```text
com.codecoachai.ai
├── controller
│   ├── PromptTemplateController
│   └── AiCallLogController
├── service
│   ├── AiService
│   ├── PromptTemplateService
│   └── AiCallLogService
├── service.impl
├── mapper
├── domain
│   ├── entity
│   ├── dto
│   ├── vo
│   └── enums
├── client
│   └── AiClient
├── prompt
│   └── PromptRenderer
├── convert
└── config
```

---

## 9.8 codecoachai-system

### 职责

1. 管理员后台基础能力。
2. 菜单管理。
3. 角色管理。
4. 权限管理。
5. 系统配置。
6. 数据字典。
7. 简化版管理端首页统计。

V1 中系统服务保持轻量，不做完整操作日志和通知中心。

### 核心实体

```text
SysMenu
SysPermission
SystemConfig
```

### 包结构

```text
com.codecoachai.system
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
└── config
```

---

## 10. V1 功能需求详细设计

## 10.1 用户与认证模块

### 普通用户功能

1. 注册账号。
2. 登录系统。
3. 退出登录。
4. 获取当前用户信息。
5. 修改个人资料。
6. 修改密码。
7. 查看基础学习概览。

### 管理员功能

1. 登录后台。
2. 查看用户列表。
3. 启用 / 禁用用户，V1 可选。
4. 分配用户角色，V1 可简化。
5. 访问后台题库、Prompt、AI 日志功能。

### 权限要求

1. 普通用户只能访问自己的简历、答题记录、面试记录和面试报告。
2. 管理员可以访问后台管理功能。
3. Prompt 模板和 AI 调用日志仅管理员可访问。
4. 网关进行 Token 基础校验。
5. 服务内部进行角色权限校验。

---

## 10.2 题库与问题组模块

### 题目管理

管理员可以：

1. 新增题目。
2. 编辑题目。
3. 删除题目。
4. 查询题目。
5. 启用 / 禁用题目。
6. 维护参考答案和解析。
7. 绑定分类、标签、问题组。
8. 设置难度、题型、经验年限、高频标识。

### 分类管理

V1 分类建议：

1. Java 基础。
2. 集合框架。
3. 多线程与并发。
4. JVM。
5. Spring。
6. Spring Boot。
7. MyBatis。
8. MySQL。
9. Redis。
10. RabbitMQ。
11. 微服务。
12. 分布式。
13. 设计模式。
14. 项目场景题。

### 问题组管理

V1 引入 question_group，用于归并同一考察意图的问题。

示例：

```text
问题组：HashMap 核心原理
标准问题：请介绍 HashMap 的底层结构与核心原理
关联题目：
- 说一下 HashMap
- 介绍一下 HashMap
- HashMap 的底层结构是什么
```

V1 规则：

1. 管理员手动选择问题组。
2. 如果无合适问题组，则新建问题组。
3. 刷题和面试抽题时，按 group_id 去重。
4. V1 不做自动语义去重。

---

## 10.3 刷题练习、错题与收藏模块

### 刷题功能

用户可以：

1. 按分类查询题目。
2. 按标签查询题目。
3. 按难度查询题目。
4. 查看题目详情。
5. 提交自己的答案。
6. 查看参考答案。
7. 查看答案解析。
8. 标记掌握状态。

### 错题功能

错题来源：

1. 用户手动标记不会。
2. 用户掌握状态选择“不会”或“模糊”。
3. 后续可由 AI 点评低分自动加入。

V1 功能：

1. 错题列表。
2. 从错题进入题目详情。
3. 标记已掌握。
4. 移出错题本。

### 收藏功能

1. 收藏题目。
2. 取消收藏。
3. 收藏列表。
4. 从收藏进入题目详情。

---

## 10.4 简历管理模块

### 简历功能

用户可以：

1. 新增简历。
2. 编辑简历。
3. 删除简历。
4. 查看简历详情。
5. 设置默认简历。

### 项目经历功能

用户可以：

1. 新增项目经历。
2. 编辑项目经历。
3. 删除项目经历。
4. 查看项目经历详情。

### V1 不做

1. PDF 上传。
2. Word 上传。
3. AI 简历解析。
4. MinIO 文件存储。

---

## 10.5 AI 模拟面试模块

### 面试模式

V1 支持三种：

1. 八股文技术面试。
2. 简历项目深挖面试。
3. 综合模拟面试。

### 创建面试配置

用户选择：

1. 面试模式。
2. 目标岗位。
3. 经验年限。
4. 行业方向。
5. 难度等级。
6. 面试官风格。
7. 是否基于简历。
8. 使用哪份简历。

### 面试阶段

综合模拟面试建议阶段：

1. 开场。
2. Java 基础。
3. 数据库。
4. 缓存与中间件。
5. 框架基础。
6. 项目深挖。
7. 场景设计。
8. 总结报告。

八股文面试阶段：

1. Java 基础。
2. 集合框架。
3. 并发。
4. JVM。
5. Spring。
6. MySQL。
7. Redis。

项目深挖面试阶段：

1. 项目背景。
2. 个人职责。
3. 技术选型。
4. 核心流程。
5. 数据库与缓存。
6. 难点与优化。
7. 扩展性问题。

### 面试流程

```text
用户创建面试
  ↓
interview-service 创建 interview_session
  ↓
interview-service 生成 interview_stage
  ↓
用户开始面试
  ↓
interview-service 获取当前阶段
  ↓
interview-service 调 question-service 或 resume-service 获取上下文
  ↓
interview-service 调 ai-service 生成问题
  ↓
用户提交回答
  ↓
interview-service 保存用户回答
  ↓
interview-service 调 ai-service 评分和点评
  ↓
interview-service 判断是否追问
  ↓
追问或进入下一题 / 下一阶段
  ↓
面试结束后生成报告
```

### 动态追问规则

1. 每个主问题最多追问 2 次。
2. 回答错误：追问基础概念。
3. 回答太浅：追问底层原理。
4. 回答模板化：要求结合项目。
5. 回答提到项目技术：追问项目落地。
6. 回答较好：提高难度或进入场景题。
7. 超过追问次数：进入下一题。

---

## 10.6 面试报告模块

### 报告内容

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

### 报告生成方式

V1 采用同步生成：

```text
面试结束
  ↓
汇总 interview_message
  ↓
统计阶段得分
  ↓
调用 ai-service 生成总结
  ↓
保存 interview_report
  ↓
用户查看报告
```

V1 不做异步生成和 SSE 流式展示。

---

## 10.7 Prompt 模板与 AI 调用日志

### Prompt 模板

V1 需要支持以下模板：

1. 八股文提问 Prompt。
2. 项目深挖提问 Prompt。
3. 回答评分 Prompt。
4. 动态追问 Prompt。
5. 面试报告生成 Prompt。

模板变量：

1. 用户目标岗位。
2. 经验年限。
3. 行业方向。
4. 当前阶段。
5. 当前问题。
6. 用户回答。
7. 题库参考答案。
8. 简历项目内容。
9. 历史问答摘要。

### AI 调用日志

记录字段：

1. 用户 ID。
2. 调用场景。
3. 模型名称。
4. Prompt 模板 ID。
5. 请求内容。
6. 响应内容。
7. 调用耗时。
8. 调用状态。
9. 失败原因。
10. 创建时间。

---

## 11. V1 页面清单

### 11.1 用户端页面

1. 登录页。
2. 注册页。
3. 首页工作台，简化版。
4. 题库列表页。
5. 题目详情页。
6. 错题本页。
7. 收藏题目页。
8. 简历列表页。
9. 简历编辑页。
10. 创建面试页。
11. AI 面试房间页。
12. 面试历史页。
13. 面试详情页。
14. 面试报告页。
15. 个人中心页。

### 11.2 管理端页面

1. 管理端首页，简化版。
2. 用户管理页，简化版。
3. 题目管理页。
4. 分类管理页。
5. 标签管理页。
6. 问题组管理页。
7. Prompt 模板管理页。
8. AI 调用日志页。
9. 系统配置页，简化版。

---

## 12. V1 核心接口草案

> 注意：本章节为早期接口草案，实际开发以 `CodeCoachAI_V1_接口设计总览.md` 及各模块接口设计文档为准。若本章节与接口设计文档存在不一致，以接口设计文档为最高依据。

## 12.1 auth-service

```text
POST /auth/register
POST /auth/login
POST /auth/logout
GET  /auth/current-user
POST /auth/refresh-token
```

## 12.2 user-service

```text
GET  /users/profile
PUT  /users/profile
GET  /users/overview
GET  /admin/users
PUT  /admin/users/{id}/status
GET  /admin/roles
```

说明：

1. `GET /admin/roles` 归属 user-service。
2. user-service 管理 `sys_user`、`sys_role`、`sys_user_role`。
3. V1 只需要 `USER`、`ADMIN` 两类基础角色。
4. 不设计复杂 RBAC、菜单权限、按钮权限和数据权限。
5. system-service 不直接维护角色和用户角色关系；如需角色相关统计，只能通过 Spring Cloud OpenFeign 调用 user-service 内部接口，不能直接查询 user-service 数据库表。

## 12.3 question-service

```text
GET  /questions
GET  /questions/{id}
POST /questions/{id}/answers
POST /questions/{id}/favorite
DELETE /questions/{id}/favorite
GET  /questions/favorites
GET  /questions/wrong-records
PUT  /questions/{id}/mastery

GET  /admin/questions
POST /admin/questions
PUT  /admin/questions/{id}
PUT  /admin/questions/{id}/status
DELETE /admin/questions/{id}

GET  /admin/question-categories
POST /admin/question-categories
PUT  /admin/question-categories/{id}
PUT  /admin/question-categories/{id}/status
DELETE /admin/question-categories/{id}

GET  /admin/question-tags
POST /admin/question-tags
PUT  /admin/question-tags/{id}
PUT  /admin/question-tags/{id}/status
DELETE /admin/question-tags/{id}

GET  /admin/question-groups
POST /admin/question-groups
PUT  /admin/question-groups/{id}
PUT  /admin/question-groups/{id}/status
DELETE /admin/question-groups/{id}
```

说明：

1. `POST /questions/{id}/answers` 用于用户提交指定题目的答案。
2. V1 刷题提交答案不调用 AI 评分。
3. 刷题答案只保存用户答题记录、错题、掌握状态等数据。
4. `GET /questions/wrong-records` 用于查询当前用户错题列表，只返回当前用户自己的错题。

## 12.4 resume-service

```text
GET  /resumes
POST /resumes
GET  /resumes/{id}
PUT  /resumes/{id}
DELETE /resumes/{id}
PUT  /resumes/{id}/default

POST /resumes/{resumeId}/projects
PUT  /resumes/projects/{projectId}
DELETE /resumes/projects/{projectId}
```

## 12.5 interview-service

```text
POST /interviews
POST /interviews/{id}/start
GET  /interviews/{id}/current
POST /interviews/{id}/answer
POST /interviews/{id}/finish
POST /interviews/{id}/report/retry
GET  /interviews
GET  /interviews/{id}
GET  /interviews/{id}/report
```

说明：

1. `POST /interviews/{id}/answer` 需要返回 AI 评分、AI 点评、`nextAction`、下一题或追问题。
2. `nextAction` 枚举包括 `FOLLOW_UP`、`NEXT_QUESTION`、`NEXT_STAGE`、`FINISH`。
3. `POST /interviews/{id}/answer` 不直接生成面试报告。
4. 当 `nextAction = FINISH` 时，前端继续调用 `POST /interviews/{id}/finish`。
5. `POST /interviews/{id}/finish` 用于结束面试，V1 可同步生成报告。
6. 报告成功时 `reportStatus = GENERATED`。
7. 报告失败时 `reportStatus = FAILED`，并记录失败原因；面试主状态仍可保持 `COMPLETED`，表示问答流程已结束。
8. `POST /interviews/{id}/report/retry` 用于报告生成失败或未生成时重试。
9. 报告重试只允许当前用户操作自己的面试报告，仅 `reportStatus = FAILED` 或 `NOT_GENERATED` 时允许调用，不重复生成多份有效报告。

## 12.6 ai-service

```text
POST /inner/ai/interview/question
POST /inner/ai/interview/evaluate
POST /inner/ai/interview/follow-up
POST /inner/ai/interview/report

GET  /admin/ai/prompts
GET  /admin/ai/prompts/{id}
POST /admin/ai/prompts
PUT  /admin/ai/prompts/{id}
PUT  /admin/ai/prompts/{id}/status
GET  /admin/ai/logs
GET  /admin/ai/logs/{id}
```

说明：

1. `/inner/ai/**` 能力接口只允许 interview-service 内部通过 Spring Cloud OpenFeign 调用。
2. 前端不能直接访问 `/inner/ai/**`。
3. Gateway 不对外暴露 `/inner/**`。
4. AI 能力接口不作为用户端 API。
5. 前端只访问 interview-service 的 `/interviews/**` 接口，由 interview-service 编排 AI 提问、评分、追问和报告生成。

## 12.7 system-service

```text
GET  /admin/menus
GET  /admin/configs
POST /admin/configs
GET  /admin/configs/{key}
PUT  /admin/configs/{key}
PUT  /admin/configs/{key}/status
DELETE /admin/configs/{key}
```

说明：`/admin/roles` 归属 user-service；system-service 不直接维护 `sys_role`、`sys_user_role`。

## 12.8 `/inner/**` 内部接口安全边界

1. `/inner/**` 只用于服务内部 Spring Cloud OpenFeign 调用。
2. 前端不允许直接访问 `/inner/**`。
3. Gateway 不对外暴露 `/inner/**`。
4. AI 能力接口 `/inner/ai/**` 只允许 interview-service 调用。
5. 内部接口仍需要服务间鉴权或内部调用标识，不能只依赖“前端不调用”作为安全措施。
6. 涉及用户数据的内部接口必须透传可信 `userId` 或用户上下文，并进行数据归属校验。
7. auth-service 不直接操作 `sys_user`、`sys_role`、`sys_user_role`，只能通过 user-service 内部接口完成认证相关查询。
8. interview-service 不直接访问 question-service、resume-service、ai-service 的数据库表，只通过 `/inner/**` 内部接口调用。

---

## 13. V1 数据表草案

## 13.1 用户权限相关

```text
sys_user
sys_role
sys_user_role
sys_menu
sys_permission
```

V1 可先简化为：

```text
sys_user
sys_role
sys_user_role
```

菜单权限可后续补全。

## 13.2 题库相关

```text
question_category
question_tag
question_group
question
question_tag_relation
user_question_record
wrong_question
favorite_question
user_question_mastery
```

## 13.3 简历相关

```text
resume
resume_project
```

## 13.4 面试相关

```text
interview_session
interview_stage
interview_message
interview_report
```

## 13.5 AI 相关

```text
prompt_template
ai_call_log
```

## 13.6 系统相关

```text
system_config
```

V1 暂不做完整操作日志、通知、异步任务表。

---

## 14. V1 关键数据模型说明

### 14.1 question_group

用于解决同一考察意图下不同问法的问题归并。

核心字段：

```text
id
canonical_title
canonical_answer
main_category_id
main_knowledge_point
difficulty
experience_level
description
status
created_at
updated_at
```

### 14.2 question

核心字段：

```text
id
group_id
category_id
title
content
reference_answer
analysis
difficulty
question_type
experience_level
is_high_frequency
status
normalized_title
created_at
updated_at
```

### 14.3 interview_session

核心字段：

```text
id
user_id
interview_mode
target_position
experience_level
industry_direction
difficulty
interviewer_style
resume_id
status
total_score
start_time
end_time
created_at
updated_at
```

### 14.4 interview_stage

核心字段：

```text
id
session_id
stage_name
stage_order
expected_question_count
actual_question_count
focus_points
status
stage_score
created_at
updated_at
```

### 14.5 interview_message

核心字段：

```text
id
session_id
stage_id
question_id
group_id
role
question_content
user_answer
ai_comment
score
is_follow_up
parent_message_id
follow_up_reason
knowledge_points
created_at
```

### 14.6 interview_report

核心字段：

```text
id
session_id
user_id
total_score
module_scores_json
strength_summary
problem_summary
weak_knowledge_points
project_expression_problems
review_suggestions
recommended_questions_json
report_content
status
created_at
updated_at
```

---

## 15. V1 核心业务流程

## 15.1 创建 AI 面试流程

```text
用户进入创建面试页
  ↓
选择面试模式、目标岗位、经验年限、行业方向、难度、面试官风格
  ↓
如果基于简历，选择一份简历
  ↓
interview-service 校验参数
  ↓
创建 interview_session
  ↓
根据面试模式创建 interview_stage
  ↓
返回 sessionId
```

## 15.2 八股文面试流程

```text
interview-service 获取当前阶段
  ↓
调用 question-service 按分类、难度、经验年限、group_id 去重抽题
  ↓
调用 ai-service 将题目转成自然面试问题
  ↓
返回给前端
  ↓
用户回答
  ↓
调用 ai-service 基于参考答案评分
  ↓
保存 interview_message
  ↓
判断追问或进入下一题
```

## 15.3 项目深挖流程

```text
用户选择简历
  ↓
interview-service 调用 resume-service 获取项目经历
  ↓
提取项目技术栈和职责信息
  ↓
调用 ai-service 生成项目深挖问题
  ↓
用户回答
  ↓
AI 根据项目背景、技术选型、职责、难点评分
  ↓
决定是否继续追问
```

## 15.4 动态追问流程

```text
用户提交回答
  ↓
ai-service 返回评分、点评、是否建议追问、追问方向
  ↓
interview-service 判断当前主问题追问次数
  ↓
如果未超限，调用 ai-service 生成追问
  ↓
如果已超限，进入下一题或下一阶段
```

## 15.5 面试报告流程

```text
用户主动结束或达到题目上限
  ↓
interview-service 汇总阶段、问答、评分
  ↓
调用 ai-service 生成报告
  ↓
保存 interview_report
  ↓
更新 interview_session 状态为 COMPLETED
  ↓
用户查看报告
```

---

## 16. V1 非功能需求

### 16.1 性能要求

1. 普通查询接口响应时间建议控制在 500ms 内。
2. 管理端普通列表查询建议控制在 1s 内。
3. AI 调用类接口允许较长耗时，建议超时时间 30-60s。
4. 面试过程中每轮问答必须及时落库，避免刷新页面导致数据丢失。

### 16.2 安全要求

1. 密码必须加密存储。
2. Token 不允许明文写入日志。
3. 普通用户只能访问自己的简历、答题记录、面试记录和报告。
4. 管理端接口必须校验管理员角色。
5. AI Prompt 中避免发送无关敏感信息。
6. API Key 不允许返回前端。

### 16.3 稳定性要求

1. AI 调用失败必须记录 ai_call_log。
2. AI 调用失败时，前端返回明确错误提示。
3. AI 返回格式不符合预期时，需要兜底解析或失败提示。
4. 面试会话需要明确状态。
5. 提交回答接口需要防重复提交。

### 16.4 可维护性要求

1. Prompt 模板不写死在业务代码中。
2. AI 调用统一经过 ai-service。
3. 服务间调用统一通过 Feign。
4. 公共异常统一处理。
5. 统一返回结构。
6. 每个服务内部遵守统一包结构。

---

## 17. V1 Redis 使用场景

V1 使用 Redis，但控制范围：

1. 登录 Token。
2. 用户权限缓存。
3. 题目分类标签缓存。
4. AI 调用限流。
5. 面试临时上下文缓存，可选。
6. 提交回答防重复，可选。

Redis Key 示例：

```text
login:token:{token}
user:permission:{userId}
question:category:list
question:tag:list
ai:limit:daily:{userId}:{date}
interview:context:{sessionId}
idempotent:submit-answer:{requestId}
```

---

## 18. V1 验收标准

### 18.1 用户主流程验收

V1 完成后，应能稳定演示：

1. 用户注册并登录。
2. 管理员登录后台。
3. 管理员维护题目分类、标签、题目和问题组。
4. 用户浏览题库并查看题目详情。
5. 用户提交答案并记录掌握状态。
6. 用户收藏题目或加入错题。
7. 用户创建并编辑一份简历。
8. 用户创建综合模拟面试。
9. 系统生成面试阶段。
10. AI 根据题库和简历提问。
11. 用户回答后，AI 能点评和追问。
12. 系统限制每个主问题追问次数。
13. 面试结束后生成结构化报告。
14. 用户能查看历史面试和报告。

### 18.2 技术架构验收

V1 技术侧应满足：

1. 各服务能注册到 Nacos。
2. 前端请求统一经过 Gateway。
3. Gateway 能路由到各业务服务。
4. 服务间通过 OpenFeign 调用。
5. MyBatis-Plus 能正常操作 MySQL。
6. Redis 能完成登录态或基础缓存。
7. AI 调用统一经过 codecoachai-ai。
8. AI 调用日志能在后台查询。
9. Prompt 模板能在后台维护。
10. 核心接口有 Knife4j 文档。
11. 每个服务包结构符合分层规范。

### 18.3 AI 能力验收

AI 面试应满足：

1. 每次只问一个问题。
2. 能基于题库生成八股文问题。
3. 能基于简历项目生成项目深挖问题。
4. 能根据用户回答评分。
5. 能根据回答质量生成追问。
6. 能控制追问次数。
7. 能生成结构化面试报告。
8. 不直接变成普通聊天机器人。

---

## 19. V1 开发优先级

### P0：必须完成

1. 微服务项目骨架。
2. Gateway。
3. Nacos 注册。
4. 用户注册登录。
5. 题库、分类、标签、问题组。
6. 简历手动录入。
7. 创建面试。
8. 面试阶段生成。
9. AI 提问、评分、追问。
10. 面试记录。
11. 面试报告。
12. AI 调用日志。

### P1：V1 中后期完成

1. 基础刷题。
2. 错题本。
3. 收藏题目。
4. Prompt 模板后台管理。
5. 管理端基础页面。
6. 首页工作台简化版。
7. Redis 基础缓存和 AI 限流。

### P2：V1 可选，优先放后续

1. 首页图表美化。
2. 管理端数据看板。
3. Sentinel 复杂限流规则。
4. 操作日志完整实现。
5. Docker Compose。
6. SSE 流式输出。

---

## 20. V1 之后的版本衔接

### V2 重点

1. AI 简历优化。
2. 学习计划。
3. AI 题目生成与审核。
4. 行业场景面试增强。
5. 题库规则去重。
6. SSE 流式输出。
7. 简历文件上传解析初版。

### V3 重点

1. RabbitMQ / RocketMQ 异步任务。
2. MinIO 文件存储。
3. Elasticsearch 搜索。
4. Docker Compose 部署。
5. 完整操作日志和登录日志。
6. 任务中心。
7. 数据看板。
8. Embedding 语义去重。
9. 题目关系网络增强。

---

## 21. V1 结论

CodeCoachAI V1 的建设原则是：

> 微服务架构先定下来，业务功能先收敛到核心闭环。

V1 不追求功能大而全，而是优先完成：

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

代码组织上，V1 统一采用：

```text
controller
service
service.impl
mapper
domain.entity
domain.dto
domain.vo
domain.enums
convert
feign
config
```

这样既能保证 V1 快速落地，也能避免后续从单体重构到微服务时大面积返工。
