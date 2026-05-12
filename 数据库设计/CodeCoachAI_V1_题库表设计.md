# CodeCoachAI V1 题库表设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 题库表设计 |
| 项目阶段 | V1 |
| 最高依据 | PRD/CodeCoachAI_PRD_V1_微服务分层架构版.md |
| 前置文档 | 数据库设计/CodeCoachAI_V1_数据库设计总览.md |
| 归属服务 | codecoachai-question |
| 相关服务 | codecoachai-interview、codecoachai-ai |

本文档只设计 V1 题库基础数据表。

V1 题库服务同时承载基础刷题数据，但刷题记录、错题、收藏、掌握状态将在单独文档中设计。

---

## 2. 设计目标

V1 题库表需要支撑：

1. 管理员维护题目分类。
2. 管理员维护题目标签。
3. 管理员维护问题组。
4. 管理员维护题目。
5. 一个题目绑定一个分类。
6. 一个题目绑定一个问题组。
7. 一个题目绑定多个标签。
8. 用户按分类、标签、难度、经验年限查询题目。
9. 面试服务按分类、难度、经验年限抽题。
10. 面试抽题时按 `group_id` 避免同一场面试重复命中同一考察意图。

V1 重点不是做复杂题库治理，而是通过 `question_group` 解决“问法不同但考察意图相同”的基础去重问题。

---

## 3. 表清单

| 表名 | 中文名 | 归属服务 | V1 是否必需 |
|---|---|---|---:|
| `question_category` | 题目分类表 | question-service | 是 |
| `question_tag` | 题目标签表 | question-service | 是 |
| `question_group` | 问题组表 | question-service | 是 |
| `question` | 题目表 | question-service | 是 |
| `question_tag_relation` | 题目标签关联表 | question-service | 是 |

V1 暂不设计：

| 表名 | 不做原因 |
|---|---|
| `question_relation` | V1 不做题目关系网络 |
| `question_review` | V1 不做 AI 生成题审核 |
| `question_duplicate_review` | V1 不做疑似重复题审核 |
| `question_embedding` | V1 不做 Embedding 语义去重 |
| `knowledge_point` | V1 可使用文本字段保存知识点，不单独建表 |
| `question_knowledge_point` | V1 不拆知识点关系表 |

---

## 4. 表关系

```text
question_category
  └── question
        ├── question_group
        └── question_tag_relation
              └── question_tag
```

关系说明：

1. 一个分类下有多道题。
2. 一个问题组下有多道同一考察意图的题。
3. 一个题目必须归属一个问题组。
4. 一个题目可以绑定多个标签。
5. 一个标签可以绑定多道题。

---

## 5. question_category 题目分类表

## 5.1 表说明

`question_category` 用于保存 Java 面试题分类。

V1 分类建议包括：

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

V1 分类可先支持一级分类，也可以预留 `parent_id` 支持简单树形结构。

## 5.2 字段设计

| 字段名 | 类型 | 是否必填 | 默认值 | 说明 |
|---|---|---:|---|---|
| `id` | `bigint` | 是 | 无 | 主键 ID |
| `parent_id` | `bigint` | 是 | `0` | 父分类 ID，0 表示一级分类 |
| `category_name` | `varchar(64)` | 是 | 无 | 分类名称 |
| `category_code` | `varchar(64)` | 是 | 无 | 分类编码 |
| `sort_order` | `int` | 是 | `0` | 排序值 |
| `status` | `tinyint` | 是 | `1` | 状态：1 启用，0 禁用 |
| `created_at` | `datetime` | 是 | 当前时间 | 创建时间 |
| `updated_at` | `datetime` | 是 | 当前时间 | 更新时间 |
| `deleted` | `tinyint` | 是 | `0` | 逻辑删除 |

## 5.3 建表 SQL 草案

```sql
CREATE TABLE question_category (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  parent_id BIGINT NOT NULL DEFAULT 0 COMMENT '父分类ID，0表示一级分类',
  category_name VARCHAR(64) NOT NULL COMMENT '分类名称',
  category_code VARCHAR(64) NOT NULL COMMENT '分类编码',
  sort_order INT NOT NULL DEFAULT 0 COMMENT '排序值',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1启用，0禁用',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  UNIQUE KEY uk_question_category_code (category_code),
  KEY idx_question_category_parent_id (parent_id),
  KEY idx_question_category_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='题目分类表';
```

---

## 6. question_tag 题目标签表

## 6.1 表说明

`question_tag` 用于保存题目标签。

标签用于更细粒度标记题目知识点或面试关注点，例如：

```text
HashMap
线程池
CAS
JVM 调优
索引优化
缓存穿透
分布式锁
```

## 6.2 字段设计

| 字段名 | 类型 | 是否必填 | 默认值 | 说明 |
|---|---|---:|---|---|
| `id` | `bigint` | 是 | 无 | 主键 ID |
| `tag_name` | `varchar(64)` | 是 | 无 | 标签名称 |
| `tag_code` | `varchar(64)` | 否 | 空 | 标签编码，V1 可选 |
| `status` | `tinyint` | 是 | `1` | 状态：1 启用，0 禁用 |
| `created_at` | `datetime` | 是 | 当前时间 | 创建时间 |
| `updated_at` | `datetime` | 是 | 当前时间 | 更新时间 |
| `deleted` | `tinyint` | 是 | `0` | 逻辑删除 |

## 6.3 建表 SQL 草案

```sql
CREATE TABLE question_tag (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  tag_name VARCHAR(64) NOT NULL COMMENT '标签名称',
  tag_code VARCHAR(64) DEFAULT NULL COMMENT '标签编码',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1启用，0禁用',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  UNIQUE KEY uk_question_tag_name (tag_name),
  KEY idx_question_tag_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='题目标签表';
```

---

## 7. question_group 问题组表

## 7.1 表说明

`question_group` 是 V1 题库设计的关键表，用于归并同一考察意图下的不同问法。

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

1. 管理员新增题目时必须选择已有问题组或新建问题组。
2. 同一场练习或面试中避免重复抽取相同 `group_id` 的题目。
3. V1 不做自动语义去重。
4. V1 不做 AI 自动判重。

## 7.2 字段设计

| 字段名 | 类型 | 是否必填 | 默认值 | 说明 |
|---|---|---:|---|---|
| `id` | `bigint` | 是 | 无 | 主键 ID |
| `canonical_title` | `varchar(255)` | 是 | 无 | 标准问题标题 |
| `canonical_answer` | `text` | 否 | 空 | 标准答案 |
| `main_category_id` | `bigint` | 是 | 无 | 主分类 ID |
| `main_knowledge_point` | `varchar(255)` | 否 | 空 | 主知识点 |
| `difficulty` | `varchar(32)` | 是 | `MEDIUM` | 难度 |
| `experience_level` | `varchar(32)` | 否 | 空 | 适合经验年限 |
| `description` | `varchar(1000)` | 否 | 空 | 考察意图说明 |
| `status` | `tinyint` | 是 | `1` | 状态：1 启用，0 禁用 |
| `created_at` | `datetime` | 是 | 当前时间 | 创建时间 |
| `updated_at` | `datetime` | 是 | 当前时间 | 更新时间 |
| `deleted` | `tinyint` | 是 | `0` | 逻辑删除 |

## 7.3 字段说明

### canonical_title

标准问题标题，用于描述该问题组代表的核心问题。

示例：

```text
请介绍 HashMap 的底层结构与核心原理
```

### main_knowledge_point

主知识点，用于面试报告和薄弱点分析。

示例：

```text
HashMap
线程池
MySQL 索引
Redis 缓存一致性
```

### difficulty

建议使用统一枚举：

```text
EASY
MEDIUM
HARD
```

## 7.4 建表 SQL 草案

```sql
CREATE TABLE question_group (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  canonical_title VARCHAR(255) NOT NULL COMMENT '标准问题标题',
  canonical_answer TEXT DEFAULT NULL COMMENT '标准答案',
  main_category_id BIGINT NOT NULL COMMENT '主分类ID',
  main_knowledge_point VARCHAR(255) DEFAULT NULL COMMENT '主知识点',
  difficulty VARCHAR(32) NOT NULL DEFAULT 'MEDIUM' COMMENT '难度',
  experience_level VARCHAR(32) DEFAULT NULL COMMENT '适合经验年限',
  description VARCHAR(1000) DEFAULT NULL COMMENT '考察意图说明',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1启用，0禁用',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  KEY idx_question_group_category_id (main_category_id),
  KEY idx_question_group_status (status),
  KEY idx_question_group_difficulty (difficulty)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='问题组表';
```

---

## 8. question 题目表

## 8.1 表说明

`question` 用于保存具体题目。

题目既服务于普通刷题，也服务于 AI 模拟面试抽题。

每道题必须绑定：

1. 分类。
2. 问题组。

每道题可以绑定多个标签。

## 8.2 字段设计

| 字段名 | 类型 | 是否必填 | 默认值 | 说明 |
|---|---|---:|---|---|
| `id` | `bigint` | 是 | 无 | 主键 ID |
| `group_id` | `bigint` | 是 | 无 | 问题组 ID |
| `category_id` | `bigint` | 是 | 无 | 分类 ID |
| `title` | `varchar(255)` | 是 | 无 | 题目标题 |
| `content` | `text` | 否 | 空 | 题目内容 |
| `reference_answer` | `text` | 否 | 空 | 参考答案 |
| `analysis` | `text` | 否 | 空 | 答案解析 |
| `difficulty` | `varchar(32)` | 是 | `MEDIUM` | 难度 |
| `question_type` | `varchar(32)` | 是 | `SHORT_ANSWER` | 题型 |
| `experience_level` | `varchar(32)` | 否 | 空 | 适合经验年限 |
| `is_high_frequency` | `tinyint` | 是 | `0` | 是否高频：1 是，0 否 |
| `status` | `tinyint` | 是 | `1` | 状态：1 启用，0 禁用 |
| `normalized_title` | `varchar(255)` | 否 | 空 | 归一化标题，为后续去重预留 |
| `created_at` | `datetime` | 是 | 当前时间 | 创建时间 |
| `updated_at` | `datetime` | 是 | 当前时间 | 更新时间 |
| `deleted` | `tinyint` | 是 | `0` | 逻辑删除 |

## 8.3 题型枚举

V1 建议支持：

```text
SHORT_ANSWER 简答题
SCENARIO     场景题
CODE         代码题
CHOICE       选择题
JUDGMENT     判断题
```

V1 重点开发简答题和场景题。

## 8.4 难度枚举

建议统一使用：

```text
EASY
MEDIUM
HARD
```

如果后续希望表达更细，可以扩展为 1-5 星，但 V1 先保持简单。

## 8.5 normalized_title

`normalized_title` 是 V1 的预留字段。

V1 可以在新增题目时简单保存规范化标题，也可以暂时为空。

V1 不基于该字段实现复杂去重。

后续可用于：

1. 文本归一化去重。
2. 疑似重复题提示。
3. AI 判重前置过滤。

## 8.6 建表 SQL 草案

```sql
CREATE TABLE question (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  group_id BIGINT NOT NULL COMMENT '问题组ID',
  category_id BIGINT NOT NULL COMMENT '分类ID',
  title VARCHAR(255) NOT NULL COMMENT '题目标题',
  content TEXT DEFAULT NULL COMMENT '题目内容',
  reference_answer TEXT DEFAULT NULL COMMENT '参考答案',
  analysis TEXT DEFAULT NULL COMMENT '答案解析',
  difficulty VARCHAR(32) NOT NULL DEFAULT 'MEDIUM' COMMENT '难度',
  question_type VARCHAR(32) NOT NULL DEFAULT 'SHORT_ANSWER' COMMENT '题型',
  experience_level VARCHAR(32) DEFAULT NULL COMMENT '适合经验年限',
  is_high_frequency TINYINT NOT NULL DEFAULT 0 COMMENT '是否高频：1是，0否',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1启用，0禁用',
  normalized_title VARCHAR(255) DEFAULT NULL COMMENT '归一化标题',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  KEY idx_question_group_id (group_id),
  KEY idx_question_category_id (category_id),
  KEY idx_question_difficulty (difficulty),
  KEY idx_question_type (question_type),
  KEY idx_question_status (status),
  KEY idx_question_experience_level (experience_level),
  KEY idx_question_high_frequency (is_high_frequency)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='题目表';
```

---

## 9. question_tag_relation 题目标签关联表

## 9.1 表说明

`question_tag_relation` 用于保存题目和标签的多对多关系。

## 9.2 字段设计

| 字段名 | 类型 | 是否必填 | 默认值 | 说明 |
|---|---|---:|---|---|
| `id` | `bigint` | 是 | 无 | 主键 ID |
| `question_id` | `bigint` | 是 | 无 | 题目 ID |
| `tag_id` | `bigint` | 是 | 无 | 标签 ID |
| `created_at` | `datetime` | 是 | 当前时间 | 创建时间 |
| `updated_at` | `datetime` | 是 | 当前时间 | 更新时间 |
| `deleted` | `tinyint` | 是 | `0` | 逻辑删除 |

## 9.3 约束说明

建议增加唯一索引：

```text
uk_question_tag_relation(question_id, tag_id)
```

避免同一题目重复绑定同一标签。

## 9.4 建表 SQL 草案

```sql
CREATE TABLE question_tag_relation (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  question_id BIGINT NOT NULL COMMENT '题目ID',
  tag_id BIGINT NOT NULL COMMENT '标签ID',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  UNIQUE KEY uk_question_tag_relation (question_id, tag_id),
  KEY idx_question_tag_relation_question_id (question_id),
  KEY idx_question_tag_relation_tag_id (tag_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='题目标签关联表';
```

---

## 10. 题目录入规则

## 10.1 新增题目流程

```text
管理员进入题目管理页
  ↓
填写题目标题、内容、答案、解析、难度、题型、经验年限
  ↓
选择分类
  ↓
选择标签
  ↓
选择已有问题组或新建问题组
  ↓
保存 question
  ↓
保存 question_tag_relation
```

## 10.2 必填规则

新增题目必须填写：

1. `title`
2. `category_id`
3. `group_id`
4. `difficulty`
5. `question_type`
6. `status`

建议填写：

1. `reference_answer`
2. `analysis`
3. `experience_level`
4. 标签

## 10.3 问题组规则

1. 如果已有同类问题组，题目必须绑定已有问题组。
2. 如果没有同类问题组，管理员新建问题组。
3. 一个题目只能绑定一个问题组。
4. 一个问题组可以关联多个具体问法。

---

## 11. 面试抽题规则

## 11.1 抽题输入条件

interview-service 调用 question-service 抽题时，建议传入：

```text
categoryCode
difficulty
experienceLevel
excludeGroupIds
questionCount
interviewMode
```

## 11.2 抽题过滤规则

question-service 抽题时需要：

1. 只抽取 `status = 1` 的题目。
2. 排除 `deleted = 1` 的题目。
3. 按分类过滤。
4. 按难度过滤。
5. 按经验年限过滤，可选。
6. 排除当前面试已使用的 `group_id`。
7. 优先高频题，可选。

## 11.3 抽题返回字段

返回给 interview-service 的数据建议包括：

```text
questionId
groupId
categoryId
categoryName
title
content
referenceAnswer
analysis
difficulty
questionType
experienceLevel
tagNames
mainKnowledgePoint
```

---

## 12. 索引设计

| 表名 | 索引名 | 字段 | 类型 | 说明 |
|---|---|---|---|---|
| `question_category` | `uk_question_category_code` | `category_code` | 唯一索引 | 分类编码唯一 |
| `question_category` | `idx_question_category_parent_id` | `parent_id` | 普通索引 | 查询子分类 |
| `question_category` | `idx_question_category_status` | `status` | 普通索引 | 查询启用分类 |
| `question_tag` | `uk_question_tag_name` | `tag_name` | 唯一索引 | 标签名称唯一 |
| `question_tag` | `idx_question_tag_status` | `status` | 普通索引 | 查询启用标签 |
| `question_group` | `idx_question_group_category_id` | `main_category_id` | 普通索引 | 按主分类查询 |
| `question_group` | `idx_question_group_status` | `status` | 普通索引 | 查询启用问题组 |
| `question_group` | `idx_question_group_difficulty` | `difficulty` | 普通索引 | 按难度查询 |
| `question` | `idx_question_group_id` | `group_id` | 普通索引 | 按问题组查询 |
| `question` | `idx_question_category_id` | `category_id` | 普通索引 | 按分类查询 |
| `question` | `idx_question_difficulty` | `difficulty` | 普通索引 | 按难度查询 |
| `question` | `idx_question_type` | `question_type` | 普通索引 | 按题型查询 |
| `question` | `idx_question_status` | `status` | 普通索引 | 查询启用题目 |
| `question` | `idx_question_experience_level` | `experience_level` | 普通索引 | 按经验年限查询 |
| `question_tag_relation` | `uk_question_tag_relation` | `question_id, tag_id` | 唯一索引 | 防止重复绑定 |
| `question_tag_relation` | `idx_question_tag_relation_question_id` | `question_id` | 普通索引 | 查询题目标签 |
| `question_tag_relation` | `idx_question_tag_relation_tag_id` | `tag_id` | 普通索引 | 查询标签下题目 |

---

## 13. 初始化分类建议

V1 可初始化以下分类：

```sql
INSERT INTO question_category (id, parent_id, category_name, category_code, sort_order, status)
VALUES
  (1, 0, 'Java 基础', 'JAVA_BASIC', 1, 1),
  (2, 0, '集合框架', 'COLLECTION', 2, 1),
  (3, 0, '多线程与并发', 'CONCURRENCY', 3, 1),
  (4, 0, 'JVM', 'JVM', 4, 1),
  (5, 0, 'Spring', 'SPRING', 5, 1),
  (6, 0, 'Spring Boot', 'SPRING_BOOT', 6, 1),
  (7, 0, 'MyBatis', 'MYBATIS', 7, 1),
  (8, 0, 'MySQL', 'MYSQL', 8, 1),
  (9, 0, 'Redis', 'REDIS', 9, 1),
  (10, 0, 'RabbitMQ', 'RABBITMQ', 10, 1),
  (11, 0, '微服务', 'MICROSERVICE', 11, 1),
  (12, 0, '分布式', 'DISTRIBUTED', 12, 1),
  (13, 0, '设计模式', 'DESIGN_PATTERN', 13, 1),
  (14, 0, '项目场景题', 'PROJECT_SCENARIO', 14, 1);
```

---

## 14. V1 不做的题库能力

| 能力 | V1 不做原因 |
|---|---|
| AI 题目生成 | 放到后续版本 |
| AI 生成题审核 | V1 题目由管理员手动维护 |
| Embedding 语义去重 | V1 只做手动问题组 |
| AI 自动判重 | V1 不调用 AI 做题目判重 |
| 题目关系网络 | V1 不做 SAME_INTENT、FOLLOW_UP、RELATED 等关系表 |
| 批量导入导出 | V1 可选，不作为核心闭环 |
| Elasticsearch 搜索 | V1 使用 MySQL 条件查询 |
| 知识点独立建模 | V1 使用文本字段表示知识点 |

---

## 15. 后续实现注意事项

1. 删除分类前需要检查分类下是否存在未删除题目。
2. 删除标签时需要同步处理题目标签关联，V1 可逻辑删除标签并过滤已删除标签。
3. 删除问题组前需要检查是否存在绑定题目。
4. 新增题目必须绑定问题组。
5. 面试抽题必须排除已使用的 `group_id`。
6. 题目详情返回前端时可以聚合分类名称、标签名称、问题组标题。
7. Entity 不直接返回前端，应转换为 VO。
8. 管理端可以看到参考答案和解析，用户端题目详情是否展示答案由刷题流程决定。

---

## 16. 结论

CodeCoachAI V1 题库数据设计围绕五张表展开：

```text
question_category
question_tag
question_group
question
question_tag_relation
```

其中 `question_group` 是 V1 的关键设计，用于手动归并同一考察意图的问题，并在刷题和 AI 面试抽题时避免重复命中同类问题。

V1 题库设计保持轻量，不提前引入 AI 生成题、语义去重、题目关系网络和 Elasticsearch 搜索，优先保证题库管理、基础刷题和 AI 面试抽题稳定可用。
