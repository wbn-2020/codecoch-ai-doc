# CodeCoachAI Java 面试训练与简历优化平台 PRD - V2 AI 能力增强版

| 项目 | 内容 |
|---|---|
| 后端项目名 | CodeCoachAI-java |
| 前端项目名 | CodeCoachAI-vue |
| 版本 | V2 |
| 版本定位 | AI 能力增强 + 产品体验完善 |
| 依赖版本 | V1 微服务核心闭环版 |
| 核心目标 | 在 V1 AI 面试闭环基础上，增强简历解析、简历优化、行业场景、学习计划、AI 题目生成和题库去重能力 |

---

## 1. V2 版本结论

V2 不再解决“系统能不能跑通”的问题，而是解决“系统是否像一个完整的 AI 面试训练产品”的问题。

V1 已完成微服务骨架、题库、简历手动录入、AI 面试、动态追问、面试报告和 AI 日志。V2 在此基础上重点增强：

1. 简历文件上传与 AI 结构化解析。
2. AI 简历优化。
3. 行业场景面试。
4. 学习计划。
5. AI 题目生成与审核。
6. 题库去重初版。
7. Prompt 模板版本管理。
8. SSE 流式输出。
9. 更完整的用户端和管理端体验。

V2 仍然不追求复杂中间件全面落地。Elasticsearch、完整消息队列任务中心、复杂数据看板、Docker Compose 全量部署放到 V3。

---

## 2. V2 版本目标

V2 的主线是：

```text
用户上传或录入简历
  ↓
AI 解析简历并识别项目风险
  ↓
用户创建行业化模拟面试
  ↓
AI 结合题库、简历和行业模板进行深挖
  ↓
面试报告生成后识别薄弱点
  ↓
系统生成学习计划
  ↓
用户根据计划刷题、复盘、再次面试
```

V2 要让项目形成更完整的求职训练闭环：

> 刷题 → 面试 → 报告 → 简历优化 → 学习计划 → 再次训练。

---

## 3. V2 新增和增强范围

### 3.1 用户端新增功能

| 模块 | 功能范围 | 优先级 |
|---|---|---|
| 简历文件上传 | 支持 PDF、Word、Markdown、TXT 上传 | P0 |
| 简历解析状态 | 待解析、解析中、解析成功、解析失败、待确认 | P0 |
| AI 简历结构化 | 将简历文本转为基本信息、技能栈、工作经历、项目经历 | P0 |
| AI 简历优化 | 简历评分、项目表达建议、风险提示、改写建议 | P0 |
| 行业场景面试 | 电商、金融支付、在线教育、SaaS、内容社区、物流/ERP | P0 |
| 学习计划 | 7 天、14 天、30 天计划、每日任务、进度打卡 | P1 |
| 简答题 AI 点评 | 用户刷题后获取回答点评和建议得分 | P1 |
| AI 流式输出 | 面试提问、点评、简历优化、报告生成支持 SSE | P1 |
| 用户首页增强 | 最近面试、薄弱点、学习计划、今日推荐 | P1 |

### 3.2 管理端新增功能

| 模块 | 功能范围 | 优先级 |
|---|---|---|
| AI 题目生成 | 按技术点、难度、经验年限生成题目 | P0 |
| AI 生成题审核 | 通过、驳回、编辑后通过、合并问题组 | P0 |
| 行业模板管理 | 行业场景、业务问题、技术关注点 | P0 |
| Prompt 版本管理 | 模板版本、历史记录、回滚、测试 | P1 |
| AI 调用日志增强 | Token、耗时、场景、业务 ID、失败重试入口 | P1 |
| 题库去重初版 | 文本归一化、疑似重复提示、人工确认 | P1 |
| 题目关系管理初版 | SAME_INTENT、FOLLOW_UP、RELATED、ADVANCED | P2 |
| 文件记录管理 | 查看上传文件元数据和解析状态 | P2 |

### 3.3 AI 能力增强

| 能力 | 说明 | 优先级 |
|---|---|---|
| 简历解析 Prompt | 从 PDF / Word / Markdown / TXT 文本中提取结构化 JSON | P0 |
| 简历优化 Prompt | 分析简历问题、项目风险、岗位匹配度 | P0 |
| 行业场景 Prompt | 根据行业模板生成业务场景题 | P0 |
| 学习计划 Prompt | 根据面试报告、错题和目标岗位生成学习计划 | P1 |
| AI 题目生成 Prompt | 生成题目、答案、解析、追问题、标签建议 | P0 |
| 题库去重 Prompt | 判断同义题、追问题、相关题 | P1 |
| 流式输出 | AI 输出内容通过 SSE 分段返回 | P1 |

---

## 4. V2 不做什么

V2 暂不做以下能力：

1. 不做完整 Elasticsearch 搜索。
2. 不做完整异步任务中心。
3. 不做死信队列和复杂失败补偿。
4. 不做完整 Docker Compose 多组件部署。
5. 不做多租户。
6. 不做支付、会员、商业化。
7. 不做语音和视频面试。
8. 不做复杂用户画像算法。
9. 不做大规模数据统计看板。
10. 不做 Seata 分布式事务。

---

## 5. V2 微服务变化

V2 在 V1 服务基础上新增或增强以下服务。

### 5.1 服务结构

```text
CodeCoachAI-java
  ├── codecoach-common
  ├── codecoach-gateway
  ├── codecoach-auth
  ├── codecoach-user
  ├── codecoach-question
  ├── codecoach-practice
  ├── codecoach-resume
  ├── codecoach-interview
  ├── codecoach-ai
  ├── codecoach-file
  └── codecoach-system
```

### 5.2 新增服务

| 服务 | 说明 |
|---|---|
| codecoach-practice | 从 question-service 中拆出刷题、错题、收藏、掌握状态 |
| codecoach-file | 处理简历文件上传、文件元数据、文件下载鉴权 |

### 5.3 增强服务

| 服务 | 增强内容 |
|---|---|
| codecoach-question | 增加 AI 题目生成审核、文本归一化、疑似重复题 |
| codecoach-resume | 增加文件解析、AI 结构化结果、简历优化记录 |
| codecoach-interview | 增加行业场景面试、正式/练习模式、报告增强 |
| codecoach-ai | 增加简历解析、简历优化、题目生成、学习计划、题库去重判断 |
| codecoach-system | 增加行业模板、Prompt 版本管理、系统配置增强 |

---

## 6. V2 技术栈变化

### 6.1 新增技术

| 技术 | 用途 |
|---|---|
| SSE / EventSource | AI 内容流式输出 |
| 文件解析库 | PDF、Word、Markdown、TXT 文本提取 |
| Redis 增强 | AI 调用限流、SSE 会话上下文、热点题目缓存 |
| 本地文件存储或轻量对象存储 | V2 支持简历上传，V3 再统一升级 MinIO |
| Spring Task | 轻量处理简历解析重试、日志清理等任务 |

### 6.2 暂不强制引入

| 技术 | 原因 |
|---|---|
| Elasticsearch | V2 题库搜索仍可用 MySQL 条件查询，ES 放 V3 |
| RabbitMQ / RocketMQ | V2 可先同步或 Spring Task，复杂异步放 V3 |
| MinIO | V2 可先本地存储，V3 统一文件服务升级 |
| XXL-JOB | V2 任务规模不大，Spring Task 足够 |
| Seata | 当前业务不需要强一致分布式事务 |

---

## 7. V2 功能详细设计

## 7.1 简历文件上传与解析

### 功能说明

V2 支持用户上传简历文件，系统解析文本后调用 AI 生成结构化简历。

### 支持文件类型

- PDF。
- Word。
- Markdown。
- TXT。

### 简历解析流程

```text
用户上传简历文件
  ↓
file-service 保存文件
  ↓
resume-service 创建解析记录
  ↓
提取文件文本
  ↓
ai-service 调用简历解析 Prompt
  ↓
生成结构化 JSON
  ↓
resume-service 保存解析结果
  ↓
用户确认或编辑结果
```

### 解析状态

- 待解析。
- 解析中。
- 解析成功。
- 解析失败。
- 待人工确认。

### 解析结果

- 基本信息。
- 求职岗位。
- 技能栈。
- 工作经历。
- 项目经历。
- 教育经历。
- 项目技术栈。
- 个人职责。
- 项目难点。
- 项目成果。

---

## 7.2 AI 简历优化模块

### 功能说明

AI 简历优化用于帮助用户基于真实经历优化表达，不生成虚假项目。

### 输入

- 简历结构化内容。
- 目标岗位。
- 经验年限。
- 行业方向。
- 用户选择的项目经历。

### 输出

- 简历整体评分。
- 技术栈匹配度。
- 项目描述完整度。
- 个人职责清晰度。
- 技术难点表达。
- 量化结果表达。
- 面试可追问风险。
- 过度包装风险。
- 项目描述改写建议。
- 可能被追问的问题。

### 安全边界

系统需要明确提示：

1. 只优化真实经历表达。
2. 不帮助伪造项目经验。
3. 不鼓励夸大职责。
4. 不生成虚假公司经历。
5. 不生成虚假项目数据。

---

## 7.3 行业场景面试模块

### 功能说明

行业场景面试让 AI 面试更接近真实业务面试。用户选择行业后，AI 在项目深挖和场景设计中结合行业问题。

### 内置行业

| 行业 | 关注点 |
|---|---|
| 电商 | 商品、订单、库存、购物车、优惠券、支付、秒杀、超卖、分布式事务 |
| 金融支付 | 金额精度、幂等、对账、事务一致性、支付回调、风控、数据安全 |
| 在线教育 | 课程、学习进度、题库、考试、视频播放、防刷、订单支付 |
| SaaS | 多租户、数据隔离、权限模型、套餐限制、配置化 |
| 内容社区 | 内容审核、点赞收藏、评论、推荐、反刷 |
| 物流 / ERP | 单据流转、库存、调度、状态机、权限 |
| 通用后台 | RBAC、菜单权限、数据权限、操作日志、审批流 |

### 示例

用户选择电商行业，AI 可以追问：

- 库存扣减如何避免超卖？
- 支付回调重复通知怎么保证幂等？
- 订单状态流转如何设计？
- 优惠券并发领取如何控制？
- 秒杀场景下 Redis 和数据库如何配合？

---

## 7.4 AI 题目生成与审核

### 管理员生成题目

管理员输入：

- 技术点。
- 题目类型。
- 难度。
- 经验年限。
- 题目数量。
- 是否生成参考答案。
- 是否生成追问题。
- 是否生成标签建议。
- 是否生成分类建议。

AI 输出：

- 题目标题。
- 题目内容。
- 参考答案。
- 答案解析。
- 追问题。
- 标签建议。
- 分类建议。
- 难度建议。
- 所属问题组建议。

### 审核流程

```text
AI 生成题目
  ↓
进入待审核状态
  ↓
管理员查看题目、答案、解析和追问
  ↓
管理员选择通过、驳回、编辑后通过
  ↓
通过后进入正式题库
```

### 审核操作

- 通过。
- 驳回。
- 编辑后通过。
- 合并到已有问题组。
- 标记为追问题。
- 标记为重复题。
- 标记为相关题。

---

## 7.5 题库去重初版

### 功能说明

V2 在 V1 手动问题组基础上，增加文本归一化和疑似重复提示。

### 去重流程

```text
管理员新增题目
  ↓
系统生成 normalized_title
  ↓
系统按分类、关键词、归一化标题查找相似题
  ↓
展示疑似重复题和所属问题组
  ↓
AI 可选判断考察意图
  ↓
管理员选择合并、新建、标记追问或忽略
```

### 关系类型

- SAME_INTENT：同义题。
- FOLLOW_UP：追问题。
- RELATED：相关题。
- ADVANCED：进阶题。
- PREREQUISITE：前置题。
- COMPARE：对比题。

### V2 不做

- 不做 Embedding 向量检索。
- 不做自动删除。
- 不做全自动合并。
- 不做复杂知识图谱。

---

## 7.6 学习计划模块

### 功能说明

学习计划用于把刷题、错题、模拟面试报告串成闭环。

### 生成依据

- 目标岗位。
- 经验年限。
- 面试报告。
- 错题记录。
- 薄弱知识点。
- 用户可用学习时间。
- 准备周期。

### 计划类型

- 7 天冲刺计划。
- 14 天提升计划。
- 30 天系统复习计划。
- 自定义计划。

### 每日任务

- 复习知识点。
- 完成指定题目。
- 重刷错题。
- 进行专项模拟面试。
- 优化简历项目描述。
- 查看面试报告建议。
- 完成打卡。

---

## 7.7 SSE 流式输出模块

### 功能说明

AI 生成内容耗时较长，V2 支持 SSE 流式输出，改善用户等待体验。

### 使用场景

- AI 面试官提问。
- AI 回答点评。
- 面试报告生成。
- 简历优化建议生成。
- 学习计划生成。
- AI 题目生成。

### 事件类型

- start：开始生成。
- chunk：内容片段。
- progress：进度。
- done：生成完成。
- error：生成失败。

### 后端要求

- 支持连接超时。
- 支持用户主动断开。
- 支持异常关闭。
- 支持生成内容落库。
- 支持接口限流。

---

## 7.8 Prompt 模板版本管理

### 功能说明

V2 对 Prompt 模板进行版本化管理，便于调试 AI 效果和回滚。

### 功能点

- 新增模板。
- 编辑模板。
- 启用 / 禁用模板。
- 保存历史版本。
- 版本说明。
- 模板测试。
- 回滚历史版本。
- 查看模板调用记录。

### 模板类型

- 八股文提问 Prompt。
- 项目深挖提问 Prompt。
- 行业场景提问 Prompt。
- 用户回答评分 Prompt。
- 动态追问 Prompt。
- 面试大纲生成 Prompt。
- 面试报告生成 Prompt。
- 简历解析 Prompt。
- 简历优化 Prompt。
- 题目生成 Prompt。
- 学习计划生成 Prompt。
- 题库去重判断 Prompt。

---

## 8. V2 页面清单

### 用户端新增页面

1. 简历上传页。
2. 简历解析结果页。
3. 简历优化页。
4. 行业场景面试创建页。
5. 学习计划页。
6. 每日任务页。
7. AI 流式输出展示组件。
8. 首页看板增强版。

### 管理端新增页面

1. AI 题目生成页。
2. AI 生成题审核页。
3. 疑似重复题审核页。
4. 题目关系管理页。
5. 行业模板管理页。
6. Prompt 模板版本页。
7. 文件记录管理页。
8. AI 调用日志详情页。

---

## 9. V2 新增数据表

### 文件与简历

- file_info
- resume_analysis_record
- resume_skill
- resume_work_experience
- resume_optimize_record

### 题库增强

- question_review
- question_duplicate_review
- question_relation
- knowledge_point
- question_knowledge_point

### 学习计划

- study_plan
- study_task
- study_checkin

### Prompt 与 AI 增强

- prompt_template_version
- ai_model_config
- ai_task_record

### 行业场景

- industry_template
- industry_scene
- industry_question_template

---

## 10. V2 新增接口

### 简历解析接口

- POST /resumes/upload
- GET /resumes/{id}/parse-status
- POST /resumes/{id}/reparse
- GET /resumes/{id}/analysis-result
- POST /resumes/{id}/confirm-analysis

### 简历优化接口

- POST /resumes/{id}/optimize
- GET /resumes/{id}/optimize-records
- GET /resumes/optimize-records/{recordId}

### AI 题目生成接口

- POST /admin/ai/questions/generate
- GET /admin/question-reviews
- GET /admin/question-reviews/{id}
- POST /admin/question-reviews/{id}/approve
- POST /admin/question-reviews/{id}/reject

### 题库去重接口

- POST /admin/questions/check-duplicate
- GET /admin/question-duplicate-reviews
- POST /admin/question-duplicate-reviews/{id}/merge
- POST /admin/question-duplicate-reviews/{id}/ignore

### 学习计划接口

- POST /study-plans/generate
- GET /study-plans
- GET /study-plans/{id}
- GET /study-plans/{id}/tasks
- POST /study-tasks/{id}/complete
- POST /study-tasks/{id}/skip

### SSE 接口

- GET /ai/sse/interview-question
- GET /ai/sse/interview-comment
- GET /ai/sse/report
- GET /ai/sse/resume-optimize
- GET /ai/sse/study-plan

---

## 11. V2 非功能需求

1. AI 流式接口需要支持超时控制。
2. 简历解析失败要记录失败原因。
3. AI 返回 JSON 需要做格式校验。
4. AI 生成题不能直接进入正式题库。
5. 简历优化必须提示“基于真实经历优化表达”。
6. 文件上传需要校验类型和大小。
7. Prompt 修改需要保留历史版本。
8. AI 调用日志需要记录业务 ID 和模板版本。
9. 行业模板需要支持后台维护。
10. 学习计划生成失败应允许重新生成。

---

## 12. V2 验收标准

V2 完成后，应能演示：

1. 用户上传简历文件。
2. 系统解析简历并生成结构化结果。
3. 用户确认解析结果。
4. AI 生成简历优化建议。
5. 用户选择行业方向创建面试。
6. AI 能结合行业模板进行场景追问。
7. 面试报告生成后，可以生成学习计划。
8. 管理员可以使用 AI 生成题目。
9. AI 生成题目进入审核流程。
10. 管理员新增题目时，系统可以提示疑似重复题。
11. AI 输出可以通过 SSE 流式展示。
12. Prompt 模板支持版本记录和回滚。

---

## 13. V2 开发顺序

1. 拆出 codecoach-practice 服务。
2. 增加 codecoach-file 服务。
3. 完成简历上传和文本解析。
4. 完成 AI 简历结构化解析。
5. 完成 AI 简历优化。
6. 完成行业模板和行业场景面试。
7. 完成 AI 题目生成与审核。
8. 完成题库去重初版。
9. 完成学习计划。
10. 完成 SSE 流式输出。
11. 完成 Prompt 模板版本管理。
12. 增强用户首页和面试报告展示。

---

## 14. V2 简历表达建议

可以在简历中补充：

> 在 V2 版本中扩展 AI 简历解析、简历优化、行业场景面试、学习计划和 AI 题目生成审核能力。针对 AI 生成内容不可控的问题，设计审核流程和 Prompt 版本管理；针对题库重复问题，引入 normalized_title 和疑似重复题审核机制，将同一考察意图的问题归并到问题组，提高题库质量。
