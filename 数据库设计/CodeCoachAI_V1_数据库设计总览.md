# CodeCoachAI V1 数据库设计总览

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 数据库设计总览 |
| 项目阶段 | V1 |
| 最高依据 | PRD/CodeCoachAI_PRD_V1_微服务分层架构版.md |
| 前置文档 | 架构设计/CodeCoachAI_V1_微服务架构设计.md、架构设计/CodeCoachAI_V1_服务边界与调用关系.md |
| 文档用途 | 明确 V1 数据库设计原则、数据域划分、表归属、命名规范和核心表清单 |

本文档只描述 CodeCoachAI V1 阶段的数据库设计总览。

V1 不设计 V2 / V3 的文件存储、搜索、异步任务、通知、完整操作日志、登录日志、Embedding、题目关系网络等数据表。

---

## 2. V1 数据库设计目标

CodeCoachAI V1 数据库设计目标是支撑以下核心闭环：

```text
用户注册登录
  ↓
管理员维护题库、分类、标签、问题组
  ↓
用户刷题、收藏、错题和掌握状态沉淀
  ↓
用户手动录入简历和项目经历
  ↓
用户创建 AI 模拟面试
  ↓
系统保存面试阶段、问答消息、评分点评和追问关系
  ↓
面试结束后生成结构化面试报告
  ↓
管理员维护 Prompt 模板并查看 AI 调用日志
```

V1 数据库设计需要满足：

1. 支撑微服务数据归属。
2. 支撑用户数据隔离。
3. 支撑题库问题组去重。
4. 支撑面试状态机。
5. 支撑每轮面试问答及时落库。
6. 支撑 AI 调用日志排查。
7. 支撑 Prompt 模板后台维护。
8. 保留必要扩展字段，但不提前实现 V2 / V3 功能。

---

## 3. 数据库总体策略

## 3.1 V1 存储选型

| 类型 | 技术 | 用途 |
|---|---|---|
| 主数据库 | MySQL 8 | 业务数据持久化 |
| 缓存 | Redis | 登录态、权限缓存、分类标签缓存、AI 限流、可选面试上下文 |

V1 不使用：

1. Elasticsearch。
2. MinIO。
3. RabbitMQ / RocketMQ。
4. 异步任务表。
5. 向量数据库。

## 3.2 逻辑库策略

V1 作为个人作品集项目，可以使用同一个 MySQL 实例。

推荐两种方式：

### 方案一：单逻辑库，按表名前缀或数据域区分

```text
codecoachai_v1
```

优点：

1. 本地开发简单。
2. 初始化 SQL 管理方便。
3. 适合 V1 快速落地。

### 方案二：同实例多逻辑库，按服务拆分

```text
codecoachai_user
codecoachai_question
codecoachai_resume
codecoachai_interview
codecoachai_ai
codecoachai_system
```

优点：

1. 更接近微服务数据隔离。
2. 后续拆库更自然。

V1 建议优先采用方案一，文档和代码中仍按服务明确数据归属，避免跨服务直接访问其他服务数据表。

---

## 4. 数据归属原则

## 4.1 总原则

1. 每个服务只直接读写自己领域内的数据表。
2. 跨服务数据访问通过 Feign 接口完成。
3. 不允许一个服务直接操作另一个服务的数据表。
4. V1 不设计跨服务强一致分布式事务。
5. 面试服务负责面试主流程数据。
6. AI 服务负责 Prompt 模板和 AI 调用日志。
7. 题库服务负责题库、刷题、错题、收藏和问题组。
8. 简历服务负责简历和项目经历。
9. 用户服务负责用户账号、资料和角色关系。

## 4.2 数据归属总表

| 数据域 | 表名 | 归属服务 | V1 是否必需 |
|---|---|---|---:|
| 用户权限 | `sys_user` | user-service | 是 |
| 用户权限 | `sys_role` | user-service | 是 |
| 用户权限 | `sys_user_role` | user-service | 是 |
| 题库 | `question_category` | question-service | 是 |
| 题库 | `question_tag` | question-service | 是 |
| 题库 | `question_group` | question-service | 是 |
| 题库 | `question` | question-service | 是 |
| 题库 | `question_tag_relation` | question-service | 是 |
| 刷题 | `user_question_record` | question-service | 是 |
| 刷题 | `wrong_question` | question-service | 是，简化版 |
| 刷题 | `favorite_question` | question-service | 是，简化版 |
| 刷题 | `user_question_mastery` | question-service | 是 |
| 简历 | `resume` | resume-service | 是 |
| 简历 | `resume_project` | resume-service | 是 |
| 面试 | `interview_session` | interview-service | 是 |
| 面试 | `interview_stage` | interview-service | 是 |
| 面试 | `interview_message` | interview-service | 是 |
| 面试 | `interview_report` | interview-service | 是 |
| AI | `prompt_template` | ai-service | 是 |
| AI | `ai_call_log` | ai-service | 是 |
| 系统 | `system_config` | system-service | 是，简化版 |
| 系统权限 | `sys_menu` | system-service | 可选 |
| 系统权限 | `sys_permission` | system-service | 可选 |

---

## 5. V1 数据域划分

## 5.1 用户权限数据域

归属服务：`codecoachai-user`

核心表：

```text
sys_user
sys_role
sys_user_role
```

说明：

1. V1 角色只需要普通用户和管理员。
2. 用户注册后默认绑定普通用户角色。
3. 管理员账号可通过初始化 SQL 创建。
4. V1 可先不实现完整角色菜单权限模型。

## 5.2 题库数据域

归属服务：`codecoachai-question`

核心表：

```text
question_category
question_tag
question_group
question
question_tag_relation
```

说明：

1. `question_group` 是 V1 必须设计的核心表。
2. 同一考察意图的不同问法归入同一个问题组。
3. 面试抽题时按 `group_id` 去重。
4. V1 问题组由管理员手动维护。
5. V1 不做 Embedding 语义去重。

## 5.3 刷题数据域

归属服务：`codecoachai-question`

核心表：

```text
user_question_record
wrong_question
favorite_question
user_question_mastery
```

说明：

1. V1 不单独拆 `practice-service`。
2. 刷题记录、错题、收藏、掌握状态先放在 question-service。
3. 错题和收藏做简化版，支撑用户列表查询和题目详情跳转即可。

## 5.4 简历数据域

归属服务：`codecoachai-resume`

核心表：

```text
resume
resume_project
```

说明：

1. V1 只支持手动录入简历。
2. 简历用于项目深挖和综合模拟面试。
3. 项目经历是 AI 项目深挖的核心上下文。
4. V1 不设计文件上传、解析状态、优化记录等表。

## 5.5 面试数据域

归属服务：`codecoachai-interview`

核心表：

```text
interview_session
interview_stage
interview_message
interview_report
```

说明：

1. `interview_session` 保存面试配置和会话状态。
2. `interview_stage` 保存面试阶段、大纲和阶段得分。
3. `interview_message` 保存 AI 问题、用户回答、评分、点评、追问关系。
4. `interview_report` 保存面试结束后的结构化报告。
5. 面试流程状态由 interview-service 控制。

## 5.6 AI 数据域

归属服务：`codecoachai-ai`

核心表：

```text
prompt_template
ai_call_log
```

说明：

1. Prompt 模板不写死在业务代码中。
2. V1 需要支持八股文提问、项目深挖、回答评分、动态追问、报告生成等模板。
3. 每次 AI 调用必须记录日志。
4. V1 可暂不设计 `ai_model_config` 表，模型配置可先通过配置文件或系统配置简化处理。

## 5.7 系统数据域

归属服务：`codecoachai-system`

核心表：

```text
system_config
```

可选表：

```text
sys_menu
sys_permission
```

说明：

1. V1 系统服务保持轻量。
2. `system_config` 用于保存每题最大追问次数、每场面试最大问题数、AI 超时时间等基础配置。
3. 菜单权限如果开发压力较大，可以先不完整落地。

---

## 6. 通用字段规范

## 6.1 推荐通用字段

业务表建议统一包含以下字段：

| 字段名 | 类型建议 | 说明 |
|---|---|---|
| `id` | `bigint` | 主键 ID |
| `created_at` | `datetime` | 创建时间 |
| `updated_at` | `datetime` | 更新时间 |
| `deleted` | `tinyint` | 逻辑删除标识，0 未删除，1 已删除 |

根据业务需要可增加：

| 字段名 | 类型建议 | 说明 |
|---|---|---|
| `created_by` | `bigint` | 创建人 |
| `updated_by` | `bigint` | 更新人 |
| `status` | `tinyint` / `varchar` | 业务状态 |
| `remark` | `varchar(500)` | 备注 |

## 6.2 字段命名规范

1. 表名使用小写下划线。
2. 字段名使用小写下划线。
3. 主键统一使用 `id`。
4. 外键字段使用 `{业务名}_id`，例如 `user_id`、`resume_id`、`session_id`。
5. 时间字段统一使用 `_at` 后缀，例如 `created_at`、`updated_at`、`start_time` 可保留业务语义。
6. JSON 字段使用 `_json` 后缀，例如 `module_scores_json`、`recommended_questions_json`。
7. 布尔字段可使用 `is_` 前缀，例如 `is_default`、`is_follow_up`、`is_high_frequency`。

## 6.3 主键规范

V1 推荐使用 `bigint` 主键。

可选方案：

1. MySQL 自增 ID。
2. 雪花 ID。

V1 如果希望开发简单，可以使用 MySQL 自增 ID；如果后续希望更贴近分布式表达，可以使用雪花 ID。

无论选择哪种方式，所有表的主键字段统一为：

```text
id
```

## 6.4 逻辑删除规范

建议业务表统一使用：

```text
deleted tinyint default 0
```

含义：

```text
0 未删除
1 已删除
```

说明：

1. 管理端删除题目、分类、标签、简历、Prompt 模板时优先逻辑删除。
2. 用户面试记录和 AI 调用日志不建议物理删除。
3. 关联表可根据业务选择物理删除或逻辑删除，V1 优先简单可控。

## 6.5 状态字段规范

状态字段可根据业务使用 `tinyint` 或 `varchar`。

建议：

1. 简单启用禁用状态使用 `tinyint`。
2. 业务流程状态使用 `varchar`，便于可读性和扩展。

示例：

```text
status tinyint
interview_status varchar(32)
report_status varchar(32)
```

---

## 7. 核心枚举建议

## 7.1 用户状态

```text
NORMAL   正常
DISABLED 禁用
```

## 7.2 用户角色

```text
USER  普通用户
ADMIN 管理员
```

## 7.3 题目状态

```text
ENABLED  启用
DISABLED 禁用
```

## 7.4 题目难度

```text
EASY    简单
MEDIUM  中等
HARD    困难
```

或使用：

```text
1 入门
2 初级
3 中级
4 高级
5 进阶
```

V1 后续接口和数据库设计需要统一选择一种。

## 7.5 题目类型

```text
SHORT_ANSWER 简答题
SCENARIO     场景题
CODE         代码题
CHOICE       选择题
JUDGMENT     判断题
```

V1 重点支持简答题和场景题，选择题、判断题可保留类型。

## 7.6 掌握状态

```text
MASTERED 已掌握
VAGUE    模糊
UNKNOWN  不会
```

## 7.7 面试模式

```text
TECHNICAL_BASIC    八股文技术面试
PROJECT_DEEP_DIVE 简历项目深挖面试
COMPREHENSIVE     综合模拟面试
```

## 7.8 面试状态

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

## 7.9 AI 场景

```text
INTERVIEW_QUESTION_GENERATE
INTERVIEW_ANSWER_EVALUATE
INTERVIEW_FOLLOW_UP_GENERATE
INTERVIEW_REPORT_GENERATE
PROJECT_DEEP_DIVE_QUESTION
```

---

## 8. 索引设计原则

## 8.1 通用索引原则

1. 外键字段建议建立普通索引。
2. 高频查询条件建立组合索引。
3. 唯一业务约束建立唯一索引。
4. 避免过度建立索引。
5. 列表查询常用的 `user_id`、`status`、`created_at` 可作为组合索引候选。

## 8.2 各数据域索引建议

### 用户权限

| 表 | 建议索引 |
|---|---|
| `sys_user` | `uk_username`、`uk_email` 可选、`idx_status` |
| `sys_role` | `uk_role_code` |
| `sys_user_role` | `idx_user_id`、`idx_role_id`、`uk_user_role` |

### 题库

| 表 | 建议索引 |
|---|---|
| `question_category` | `idx_parent_id`、`idx_status` |
| `question_tag` | `uk_tag_name` |
| `question_group` | `idx_main_category_id`、`idx_status` |
| `question` | `idx_group_id`、`idx_category_id`、`idx_difficulty`、`idx_status`、`idx_experience_level` |
| `question_tag_relation` | `idx_question_id`、`idx_tag_id`、`uk_question_tag` |

### 刷题

| 表 | 建议索引 |
|---|---|
| `user_question_record` | `idx_user_id`、`idx_question_id`、`idx_user_question` |
| `wrong_question` | `idx_user_id`、`idx_question_id`、`uk_user_question` |
| `favorite_question` | `idx_user_id`、`idx_question_id`、`uk_user_question` |
| `user_question_mastery` | `idx_user_id`、`idx_question_id`、`uk_user_question` |

### 简历

| 表 | 建议索引 |
|---|---|
| `resume` | `idx_user_id`、`idx_user_default`、`idx_status` |
| `resume_project` | `idx_resume_id` |

### 面试

| 表 | 建议索引 |
|---|---|
| `interview_session` | `idx_user_id`、`idx_status`、`idx_user_created_at` |
| `interview_stage` | `idx_session_id`、`idx_session_order` |
| `interview_message` | `idx_session_id`、`idx_stage_id`、`idx_parent_message_id`、`idx_question_id`、`idx_group_id` |
| `interview_report` | `idx_session_id`、`idx_user_id`、`idx_status` |

### AI

| 表 | 建议索引 |
|---|---|
| `prompt_template` | `uk_template_code`、`idx_scene`、`idx_status` |
| `ai_call_log` | `idx_user_id`、`idx_scene`、`idx_status`、`idx_created_at` |

---

## 9. 表关系总览

## 9.1 用户权限关系

```text
sys_user
  └── sys_user_role
        └── sys_role
```

## 9.2 题库关系

```text
question_category
  └── question
        ├── question_group
        └── question_tag_relation
              └── question_tag
```

## 9.3 刷题关系

```text
sys_user
  └── user_question_record
        └── question

sys_user
  └── wrong_question
        └── question

sys_user
  └── favorite_question
        └── question

sys_user
  └── user_question_mastery
        └── question
```

说明：在微服务实现中，question-service 不直接依赖 user-service 的表结构，只保存 `user_id` 作为业务关联。

## 9.4 简历关系

```text
sys_user
  └── resume
        └── resume_project
```

说明：resume-service 只保存 `user_id`，不直接查询 user-service 数据表。

## 9.5 面试关系

```text
sys_user
  └── interview_session
        ├── interview_stage
        ├── interview_message
        └── interview_report
```

`interview_message` 支持追问关系：

```text
interview_message
  └── parent_message_id -> interview_message.id
```

说明：

1. 主问题 `is_follow_up = 0`，`parent_message_id` 为空。
2. 追问 `is_follow_up = 1`，`parent_message_id` 指向主问题或上一轮问题。
3. V1 建议所有追问都归属同一个主问题，便于统计追问次数。

## 9.6 AI 关系

```text
prompt_template
  └── ai_call_log.prompt_template_id
```

说明：

1. 每次 AI 调用尽量记录使用的 Prompt 模板 ID。
2. 如果调用时未使用模板，也需要记录调用场景和请求内容。

---

## 10. V1 不设计的数据表

以下表属于 V2 / V3 或完整目标版本，V1 不设计：

| 表或数据域 | V1 不做原因 |
|---|---|
| `file_info` | V1 不做文件上传 |
| `resume_analysis_record` | V1 不做简历解析 |
| `resume_optimize_record` | V1 不做 AI 简历优化 |
| `question_review` | V1 不做 AI 题目生成审核 |
| `question_duplicate_review` | V1 不做疑似重复题审核 |
| `question_relation` | V1 不做题目关系网络 |
| `question_embedding` | V1 不做 Embedding |
| `knowledge_point` | V1 可用文本字段表示知识点，不单独建表 |
| `study_plan` | V1 不做学习计划 |
| `study_task` | V1 不做学习任务 |
| `study_checkin` | V1 不做打卡 |
| `async_task` | V1 不做异步任务中心 |
| `message_dead_letter` | V1 不做消息队列和死信 |
| `notification` | V1 不做站内通知 |
| `operation_log` | V1 不做完整操作日志 |
| `login_log` | V1 不做登录日志中心 |
| `ai_model_config` | V1 可先用配置文件或 system_config 简化 |
| `prompt_template_version` | V1 不做 Prompt 版本管理 |
| `search_index_record` | V1 不做 Elasticsearch |

---

## 11. V1 表清单汇总

V1 必需表：

```text
sys_user
sys_role
sys_user_role

question_category
question_tag
question_group
question
question_tag_relation

user_question_record
wrong_question
favorite_question
user_question_mastery

resume
resume_project

interview_session
interview_stage
interview_message
interview_report

prompt_template
ai_call_log

system_config
```

V1 可选表：

```text
sys_menu
sys_permission
```

如果 V1 后台权限先简化为管理员角色判断，可以暂不创建 `sys_menu` 和 `sys_permission`。

---

## 12. 后续详细数据库设计文档拆分

本总览文档之后，建议继续拆分生成以下详细设计文档：

1. `数据库设计/CodeCoachAI_V1_用户权限表设计.md`
2. `数据库设计/CodeCoachAI_V1_题库表设计.md`
3. `数据库设计/CodeCoachAI_V1_刷题表设计.md`
4. `数据库设计/CodeCoachAI_V1_简历表设计.md`
5. `数据库设计/CodeCoachAI_V1_面试表设计.md`
6. `数据库设计/CodeCoachAI_V1_AI表设计.md`
7. `数据库设计/CodeCoachAI_V1_系统配置表设计.md`
8. `数据库设计/CodeCoachAI_V1_初始化数据设计.md`

其中最优先的是：

1. 用户权限表设计。
2. 题库表设计。
3. 简历表设计。
4. 面试表设计。
5. AI 表设计。

---

## 13. 结论

CodeCoachAI V1 数据库设计围绕核心闭环展开：

```text
用户
  ↓
题库与问题组
  ↓
简历与项目经历
  ↓
面试会话与阶段
  ↓
问答消息与追问关系
  ↓
面试报告
  ↓
Prompt 模板与 AI 调用日志
```

V1 数据库不追求完整商业系统的数据模型，而是优先保证：

1. AI 模拟面试主流程稳定。
2. 每轮问答和 AI 调用可追踪。
3. 用户数据按 user_id 隔离。
4. 服务之间数据归属清晰。
5. 后续 V2 / V3 扩展时不需要大面积推翻核心表结构。
