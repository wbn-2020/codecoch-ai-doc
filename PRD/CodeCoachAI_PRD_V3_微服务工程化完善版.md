# CodeCoachAI Java 面试训练与简历优化平台 PRD - V3 微服务工程化完善版

| 项目 | 内容 |
|---|---|
| 后端项目名 | CodeCoachAI-java |
| 前端项目名 | CodeCoachAI-vue |
| 版本 | V3 |
| 版本定位 | 微服务工程化 + 中间件能力完善 + 可部署演示 |
| 依赖版本 | V1 微服务核心闭环版、V2 AI 能力增强版 |
| 核心目标 | 完善 Spring Cloud Alibaba 微服务体系、缓存、搜索、异步任务、文件存储、部署和运维能力 |

---

## 1. V3 版本结论

V3 的目标不是新增大量业务功能，而是把项目做成一个更完整、更能体现 3 年 Java 后端能力的工程化项目。

V1 已完成微服务骨架和 AI 面试闭环。V2 已增强 AI 简历解析、简历优化、行业场景、学习计划、AI 题目生成和题库去重。V3 重点补齐：

1. Redis 缓存、限流、幂等和分布式锁。
2. 消息中间件异步任务。
3. MinIO 文件存储。
4. Elasticsearch 全文搜索。
5. Docker Compose 一键部署。
6. 操作日志、登录日志、任务中心。
7. 通知中心和数据看板。
8. AI 调用治理增强。
9. 服务观测、异常重试和失败兜底。
10. 题库语义去重增强。

V3 的开发原则是：

> 不为了堆技术而堆技术，每个中间件都必须绑定具体业务场景。

---

## 2. V3 版本目标

V3 要完成以下工程化闭环：

```text
用户上传简历文件
  ↓
MinIO 保存文件
  ↓
消息队列异步触发解析任务
  ↓
AI 服务解析简历
  ↓
任务状态更新
  ↓
通知用户完成
```

```text
用户结束面试
  ↓
interview-service 创建报告任务
  ↓
消息队列异步投递
  ↓
ai-service 生成报告
  ↓
report 状态更新
  ↓
通知用户查看报告
```

```text
管理员新增题目
  ↓
question-service 保存题目
  ↓
消息队列同步 ES 索引
  ↓
用户通过 Elasticsearch 搜索题库
```

V3 要让项目具备可演示、可部署、可排错、可扩展的工程能力。

---

## 3. V3 新增和增强范围

### 3.1 工程化能力

| 模块 | 功能范围 | 优先级 |
|---|---|---|
| Redis 缓存体系 | Token、权限、题目详情、分类标签、面试上下文、AI 限流 | P0 |
| Redis 幂等控制 | 创建面试、提交回答、生成报告、上传简历防重复 | P0 |
| 消息中间件 | 简历解析、报告生成、题目生成、学习计划、ES 同步 | P0 |
| 异步任务中心 | 任务状态、失败原因、重试次数、手动重试 | P0 |
| MinIO 文件存储 | 简历文件、头像、导入导出文件、报告导出 | P0 |
| Elasticsearch 搜索 | 题库、简历、面试记录、报告全文搜索 | P0 |
| Docker Compose | MySQL、Redis、Nacos、MQ、MinIO、ES、后端、前端 | P0 |
| 操作日志 | 后台关键操作审计 | P1 |
| 登录日志 | 登录成功、失败、IP、设备、时间 | P1 |
| 通知中心 | 报告完成、解析完成、任务失败、系统公告 | P1 |
| 数据看板 | 用户数、题目数、面试数、AI 调用、任务成功率 | P1 |
| AI 调用治理 | 限流、超时、失败重试、备用模型预留 | P1 |
| 题库语义去重增强 | Embedding 相似题检索、AI 考察意图判断 | P2 |

### 3.2 用户端增强

| 模块 | 功能范围 |
|---|---|
| 通知中心 | 查看报告完成、简历解析完成、学习任务提醒 |
| 搜索体验 | 题库全文搜索、面试记录搜索、报告搜索 |
| 文件管理 | 查看已上传简历文件、重新解析、删除文件 |
| 报告导出 | 面试报告导出 Markdown / PDF，可选 |
| 看板增强 | 学习趋势、面试分数趋势、薄弱点雷达图 |

### 3.3 管理端增强

| 模块 | 功能范围 |
|---|---|
| 异步任务管理 | 查询任务、查看失败原因、手动重试 |
| 死信任务管理 | 多次失败任务查看和处理 |
| 文件管理 | 文件元数据、业务关联、下载、删除 |
| ES 索引管理 | 索引同步、重建索引、同步状态 |
| 操作日志 | 关键操作查询和筛选 |
| 登录日志 | 登录成功失败记录 |
| 数据看板 | 系统运行和业务数据概览 |
| 系统配置 | AI 限流、文件大小、任务重试、报告生成方式 |
| AI 模型配置 | 默认模型、备用模型、超时参数、温度参数 |

---

## 4. V3 微服务完整结构

```text
CodeCoachAI-java
  ├── codecoach-common
  │   ├── common-core
  │   ├── common-security
  │   ├── common-mybatis
  │   ├── common-redis
  │   ├── common-log
  │   ├── common-feign
  │   └── common-ai
  ├── codecoach-gateway
  ├── codecoach-auth
  ├── codecoach-user
  ├── codecoach-question
  ├── codecoach-practice
  ├── codecoach-resume
  ├── codecoach-interview
  ├── codecoach-ai
  ├── codecoach-file
  ├── codecoach-search
  ├── codecoach-task
  └── codecoach-system
```

---

## 5. V3 服务职责

| 服务 | 职责 |
|---|---|
| codecoach-gateway | 统一入口、路由、鉴权、跨域、网关限流、请求日志 |
| codecoach-auth | 登录、注册、Token、权限缓存、登录日志 |
| codecoach-user | 用户信息、个人中心、用户看板 |
| codecoach-question | 题库、分类、标签、问题组、题目关系、题库去重 |
| codecoach-practice | 答题记录、错题、收藏、掌握状态、每日推荐 |
| codecoach-resume | 简历、项目经历、解析状态、简历优化记录 |
| codecoach-interview | 面试会话、面试大纲、状态机、消息、报告 |
| codecoach-ai | AIClient、Prompt、AI 日志、模型配置、AI 任务执行 |
| codecoach-file | MinIO 文件上传下载、文件元数据、访问鉴权 |
| codecoach-search | Elasticsearch 索引、全文搜索、高亮、索引同步 |
| codecoach-task | 异步任务、消息消费、重试、死信、定时任务 |
| codecoach-system | 角色权限、菜单、系统配置、通知、日志、数据看板 |

---

## 6. V3 技术栈

### 6.1 后端

| 类型 | 技术 |
|---|---|
| 语言 | Java 17 |
| 基础框架 | Spring Boot 3 |
| 微服务框架 | Spring Cloud Alibaba |
| 注册配置 | Nacos |
| 网关 | Spring Cloud Gateway |
| 服务调用 | OpenFeign |
| 限流熔断 | Sentinel |
| ORM | MyBatis-Plus |
| 数据库 | MySQL 8 |
| 缓存 | Redis |
| 搜索 | Elasticsearch |
| 消息中间件 | RocketMQ 或 RabbitMQ |
| 文件存储 | MinIO |
| 定时任务 | Spring Task，后续可扩展 XXL-JOB |
| 接口文档 | Knife4j |
| 部署 | Docker Compose |
| 日志 | Logback / SLF4J + TraceId |

### 6.2 消息中间件选型

如果希望贴合 Spring Cloud Alibaba 体系，优先选择 RocketMQ。

如果开发者更熟悉 RabbitMQ，也可以选择 RabbitMQ。PRD 中统一称为“消息中间件”，技术设计阶段需要最终确认一种，避免同时引入两个 MQ。

建议个人项目优先选择一个：

- 熟悉 RabbitMQ：选 RabbitMQ，讲清楚队列、重试、死信。
- 希望贴合 Alibaba 技术栈：选 RocketMQ，讲清楚 Topic、Tag、消费组。

---

## 7. Redis 设计

### 7.1 使用场景

| 场景 | 说明 |
|---|---|
| 登录 Token | 保存用户登录态 |
| 验证码 | 注册登录验证码 |
| 用户权限缓存 | 管理端权限和菜单 |
| 分类标签缓存 | 高频读取，低频变更 |
| 题目详情缓存 | 热门题目详情 |
| 热门题目缓存 | 高频题榜单 |
| 面试上下文 | 当前阶段、当前问题、上下文摘要 |
| AI 调用限流 | 每分钟、每日调用次数 |
| 防重复提交 | 提交回答、生成报告、上传简历 |
| 分布式锁 | 报告生成、任务重试、索引同步 |
| 任务状态缓存 | 异步任务进度临时缓存 |

### 7.2 Redis Key 规范

```text
login:token:{token}
captcha:{uuid}
user:permission:{userId}
question:detail:{questionId}
question:category:list
question:tag:list
question:hot:list
practice:progress:{userId}
interview:context:{sessionId}
ai:limit:daily:{userId}:{date}
ai:limit:minute:{userId}
idempotent:submit-answer:{requestId}
idempotent:generate-report:{sessionId}
lock:report:{sessionId}
lock:resume-parse:{resumeId}
task:progress:{taskId}
```

### 7.3 缓存更新策略

| 数据 | 策略 |
|---|---|
| 分类标签 | 后台修改后删除缓存 |
| 题目详情 | 修改题目后删除缓存 |
| 用户权限 | 分配角色后删除缓存 |
| 面试上下文 | 面试结束后删除 |
| AI 限流 | 使用 Redis 计数器 + 过期时间 |
| 幂等 Token | 使用 SETNX + 过期时间 |

---

## 8. 消息中间件设计

### 8.1 使用场景

| 任务 | 生产者 | 消费者 |
|---|---|---|
| 简历解析 | resume-service | ai-service / task-service |
| 面试报告生成 | interview-service | ai-service |
| AI 题目批量生成 | question-service | ai-service |
| 学习计划生成 | practice-service / interview-service | ai-service |
| ES 题库索引同步 | question-service | search-service |
| ES 简历索引同步 | resume-service | search-service |
| 通知发送 | task-service | system-service |
| 操作日志异步入库 | gateway / services | system-service |
| 文件清理 | task-service | file-service |

### 8.2 消息设计原则

1. 消息必须包含业务 ID。
2. 消息必须包含消息唯一 ID。
3. 消费端必须幂等。
4. 任务状态必须落库。
5. 失败原因必须记录。
6. 支持最大重试次数。
7. 多次失败进入失败任务表或死信队列。
8. 消费成功后更新任务状态。
9. 消费失败不能丢失业务上下文。
10. 不依赖 MQ 做唯一状态来源。

### 8.3 示例消息结构

```json
{
  "messageId": "uuid",
  "businessId": "interview_session_id",
  "businessType": "INTERVIEW_REPORT_GENERATE",
  "userId": 10001,
  "retryCount": 0,
  "createdAt": "2026-05-12 10:00:00"
}
```

---

## 9. 异步任务中心

### 9.1 任务类型

- 简历解析任务。
- 面试报告生成任务。
- AI 题目生成任务。
- 学习计划生成任务。
- ES 索引同步任务。
- AI 调用重试任务。
- 通知发送任务。
- 文件清理任务。
- 日志归档任务。

### 9.2 任务状态

- 待执行。
- 执行中。
- 执行成功。
- 执行失败。
- 等待重试。
- 已取消。
- 已进入死信队列。

### 9.3 任务字段

- 任务 ID。
- 任务类型。
- 业务 ID。
- 用户 ID。
- 任务参数 JSON。
- 当前状态。
- 执行次数。
- 最大重试次数。
- 失败原因。
- 开始时间。
- 结束时间。
- 创建时间。
- 更新时间。

---

## 10. MinIO 文件存储设计

### 10.1 支持文件类型

- 简历 PDF。
- 简历 Word。
- Markdown。
- TXT。
- 用户头像。
- 题库导入 Excel。
- 面试报告导出文件。

### 10.2 文件元数据

`file_info` 表记录：

- 文件 ID。
- 原始文件名。
- 存储文件名。
- 文件类型。
- 文件大小。
- Bucket。
- Object Key。
- 上传用户。
- 业务类型。
- 关联业务 ID。
- 上传时间。
- 删除标识。

### 10.3 文件安全

1. 上传文件限制大小。
2. 上传文件限制类型。
3. 下载文件校验用户权限。
4. 管理员可查看文件元数据。
5. 删除业务数据时同步删除或标记文件。
6. 临时文件定时清理。
7. 文件 URL 不长期暴露，建议使用临时访问链接。

---

## 11. Elasticsearch 搜索设计

### 11.1 使用场景

- 题库全文搜索。
- 简历关键词搜索。
- 面试记录搜索。
- 面试报告搜索。
- AI 调用日志搜索，可选。
- 搜索高亮。
- 多条件组合过滤。

### 11.2 题库索引字段

```text
question_id
title
content
reference_answer
analysis
category_name
tags
difficulty
question_type
experience_level
group_id
knowledge_points
status
updated_at
```

### 11.3 搜索能力

用户端：

- 搜索题目标题。
- 搜索知识点。
- 搜索参考答案。
- 按分类、标签、难度过滤。
- 高亮命中内容。

管理端：

- 搜索题库。
- 搜索简历。
- 搜索面试记录。
- 搜索报告。
- 重建索引。
- 查看同步状态。

### 11.4 索引同步

方式：

1. 题目新增、修改、删除后发送 MQ 消息。
2. search-service 消费消息并更新 ES。
3. 消费失败记录任务状态。
4. 管理端支持手动重建索引。

---

## 12. 数据看板

### 12.1 用户端看板

展示：

- 累计刷题数。
- 累计错题数。
- 收藏题数。
- 模拟面试次数。
- 平均面试得分。
- 最近一次面试得分。
- 薄弱知识点。
- 掌握知识点。
- 学习计划进度。
- 连续学习天数。
- 最近学习趋势。

### 12.2 管理端看板

展示：

- 用户总数。
- 题目总数。
- 问题组总数。
- 面试总次数。
- AI 调用次数。
- AI 调用成功率。
- AI 调用失败率。
- 异步任务成功率。
- ES 搜索次数。
- 热门题目。
- 热门分类。
- 高频薄弱知识点。
- 用户活跃趋势。

### 12.3 图表类型

- 折线图。
- 柱状图。
- 饼图。
- 雷达图。
- 排行榜。
- 统计卡片。

---

## 13. 操作日志与登录日志

### 13.1 操作日志

记录场景：

- 新增题目。
- 修改题目。
- 删除题目。
- 审核 AI 题目。
- 修改 Prompt。
- 修改系统配置。
- 删除用户数据。
- 重新执行异步任务。
- 重建 ES 索引。

字段：

- 操作用户。
- 操作模块。
- 操作类型。
- 请求路径。
- 请求方法。
- 请求参数。
- 响应状态。
- IP 地址。
- 浏览器。
- 操作时间。
- 耗时。
- 失败原因。

### 13.2 登录日志

字段：

- 登录账号。
- 登录 IP。
- 登录地点，可选。
- 登录设备。
- 浏览器。
- 登录状态。
- 失败原因。
- 登录时间。

---

## 14. 通知中心

### 14.1 通知场景

- 简历解析完成。
- 简历解析失败。
- 面试报告生成完成。
- 面试报告生成失败。
- 学习计划生成完成。
- AI 生成题目审核通过。
- 今日学习任务提醒。
- 系统公告。
- 异步任务失败提醒。

### 14.2 通知字段

- 通知 ID。
- 用户 ID。
- 通知类型。
- 标题。
- 内容。
- 是否已读。
- 业务 ID。
- 创建时间。

---

## 15. AI 调用治理增强

### 15.1 AI 日志增强字段

- 用户 ID。
- 调用场景。
- 模型供应商。
- 模型名称。
- Prompt 模板 ID。
- Prompt 模板版本。
- Prompt 内容。
- 请求参数。
- 响应内容。
- 输入 token。
- 输出 token。
- 总 token。
- 调用耗时。
- 调用状态。
- 失败原因。
- 异常堆栈。
- 业务 ID。
- 重试次数。
- 创建时间。

### 15.2 AI 限流策略

- 单用户每日 AI 调用次数限制。
- 单用户每分钟调用次数限制。
- 单 IP 调用次数限制。
- 指定场景限流。
- 管理员豁免。
- 失败调用是否计入次数可配置。

### 15.3 AI 失败处理

- 超时失败记录日志。
- 格式异常记录原始响应。
- 支持手动重试。
- 支持异步重试任务。
- 多次失败后标记处理失败。
- 关键场景返回兜底提示。

---

## 16. 题库语义去重增强

### 16.1 增强目标

V2 已完成文本归一化和疑似重复提示。V3 增加 Embedding 相似题检索和 AI 考察意图判断。

### 16.2 流程

```text
管理员新增题目
  ↓
生成 normalized_title
  ↓
生成 embedding 向量
  ↓
检索相似题 TopK
  ↓
AI 判断考察意图
  ↓
输出 SAME_INTENT / FOLLOW_UP / RELATED / ADVANCED
  ↓
管理员审核处理
```

### 16.3 注意事项

1. 系统不自动删除题目。
2. 系统只给出建议。
3. 管理员最终确认合并关系。
4. Embedding 结果作为辅助，不能作为唯一判断依据。
5. 同义题归入 question_group。
6. 追问题和相关题进入 question_relation。

---

## 17. Docker Compose 部署设计

### 17.1 需要编排的组件

- Nacos。
- MySQL。
- Redis。
- RabbitMQ 或 RocketMQ。
- MinIO。
- Elasticsearch。
- Kibana，可选。
- CodeCoachAI-java 后端服务。
- CodeCoachAI-vue 前端服务。

### 17.2 目录建议

```text
deploy
  ├── docker-compose.yml
  ├── mysql
  │   └── init.sql
  ├── nacos
  ├── redis
  ├── minio
  ├── elasticsearch
  ├── mq
  ├── backend
  └── frontend
```

### 17.3 部署目标

1. 本地一键启动基础环境。
2. 能导入初始化 SQL。
3. 前端能访问网关。
4. 后端服务能注册到 Nacos。
5. Redis、MQ、MinIO、ES 能被服务正常连接。
6. README 中提供启动步骤和演示账号。

---

## 18. V3 新增数据表

### 系统与任务

- async_task
- message_dead_letter
- notification
- system_config
- operation_log
- login_log

### 文件

- file_info

### 搜索

- search_index_record
- es_sync_task

### AI 增强

- ai_retry_record
- ai_model_config

### 题库增强

- question_duplicate_review
- question_embedding
- question_relation

---

## 19. V3 新增接口

### 文件接口

- POST /files/upload
- GET /files/{id}
- GET /files/{id}/download-url
- DELETE /files/{id}

### 搜索接口

- GET /search/questions
- GET /search/resumes
- GET /search/interviews
- GET /search/reports
- POST /admin/search/rebuild-index

### 任务接口

- GET /admin/tasks
- GET /admin/tasks/{id}
- POST /admin/tasks/{id}/retry
- POST /admin/tasks/{id}/cancel

### 通知接口

- GET /notifications
- POST /notifications/{id}/read
- POST /notifications/read-all

### 日志接口

- GET /admin/operation-logs
- GET /admin/login-logs

### 系统配置接口

- GET /admin/system-configs
- PUT /admin/system-configs/{key}

---

## 20. V3 非功能需求

1. 消息消费必须保证幂等。
2. 异步任务必须有状态记录。
3. AI 任务失败必须可追踪。
4. 文件下载必须鉴权。
5. ES 索引同步失败必须可重试。
6. Redis Key 必须有统一命名规范。
7. 热点缓存必须有过期时间。
8. 网关层需要基础限流。
9. 服务间调用异常需要统一处理。
10. 后端日志需要 TraceId。
11. Docker Compose 需要提供完整启动文档。
12. 管理端必须能查询操作日志和任务状态。

---

## 21. V3 验收标准

V3 完成后，应能演示：

1. 所有核心服务注册到 Nacos。
2. 前端请求统一经过 Gateway。
3. Redis 缓存分类、标签、题目详情和权限信息。
4. Redis 能限制用户 AI 调用频率。
5. 用户上传简历文件后，MinIO 保存文件。
6. 简历解析任务通过消息中间件异步处理。
7. 面试报告通过异步任务生成。
8. AI 题目生成可以异步执行。
9. Elasticsearch 可以搜索题库。
10. 题目变更后可以同步 ES 索引。
11. 管理端可以查看异步任务和失败原因。
12. 管理端可以手动重试失败任务。
13. 管理端可以查看操作日志和登录日志。
14. 用户可以收到报告完成通知。
15. Docker Compose 可以启动基础依赖环境。
16. README 中有完整部署说明和演示流程。

---

## 22. V3 开发顺序

1. 完善 Redis Key 规范和缓存工具。
2. 增加幂等工具和分布式锁工具。
3. 接入消息中间件。
4. 设计异步任务表和任务状态机。
5. 将简历解析改造成异步任务。
6. 将面试报告生成改造成异步任务。
7. 接入 MinIO 并迁移文件上传能力。
8. 接入 Elasticsearch。
9. 完成题库索引同步。
10. 完成管理端任务中心。
11. 完成操作日志和登录日志。
12. 完成通知中心。
13. 完成数据看板。
14. 完成 Docker Compose 部署。
15. 完成题库语义去重增强。
16. 编写工程化总结和面试讲解稿。

---

## 23. V3 简历表达建议

可以在简历中补充：

> 在 V3 版本中完善项目微服务工程化能力，基于 Redis 实现热点题目缓存、AI 调用限流、面试提交幂等和报告生成分布式锁；基于消息中间件将简历解析、面试报告生成、AI 题目生成等耗时任务异步化，并设计任务状态机、失败重试和死信任务处理；使用 MinIO 存储简历文件，使用 Elasticsearch 实现题库全文搜索，通过 Docker Compose 编排 MySQL、Redis、Nacos、MQ、MinIO、ES 等依赖，提升项目可部署和可演示能力。
