# CodeCoachAI V1 刷题表设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 刷题表设计 |
| 项目阶段 | V1 |
| 归属服务 | codecoachai-question |
| 前置文档 | 数据库设计/CodeCoachAI_V1_数据库设计总览.md、数据库设计/CodeCoachAI_V1_题库表设计.md |

V1 不单独拆 `practice-service`，刷题记录、错题、收藏、掌握状态统一归属 `codecoachai-question`。

---

## 2. 表清单

| 表名 | 中文名 | V1 用途 |
|---|---|---|
| `user_question_record` | 用户答题记录表 | 保存用户每次提交答案 |
| `wrong_question` | 错题表 | 保存用户不会、模糊或答错的题 |
| `favorite_question` | 收藏题表 | 保存用户收藏题目 |
| `user_question_mastery` | 用户题目掌握状态表 | 保存用户对题目的当前掌握状态 |

---

## 3. user_question_record 用户答题记录表

### 字段设计

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `bigint` | 主键 ID |
| `user_id` | `bigint` | 用户 ID |
| `question_id` | `bigint` | 题目 ID |
| `user_answer` | `text` | 用户答案 |
| `mastery_status` | `varchar(32)` | 掌握状态：MASTERED / VAGUE / UNKNOWN |
| `viewed_answer` | `tinyint` | 是否查看参考答案：1 是，0 否 |
| `answer_time` | `datetime` | 答题时间 |
| `created_at` | `datetime` | 创建时间 |
| `updated_at` | `datetime` | 更新时间 |
| `deleted` | `tinyint` | 逻辑删除 |

### SQL 草案

```sql
CREATE TABLE user_question_record (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  user_id BIGINT NOT NULL COMMENT '用户ID',
  question_id BIGINT NOT NULL COMMENT '题目ID',
  user_answer TEXT DEFAULT NULL COMMENT '用户答案',
  mastery_status VARCHAR(32) DEFAULT NULL COMMENT '掌握状态：MASTERED/VAGUE/UNKNOWN',
  viewed_answer TINYINT NOT NULL DEFAULT 0 COMMENT '是否查看参考答案：1是，0否',
  answer_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '答题时间',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  KEY idx_user_question_record_user_id (user_id),
  KEY idx_user_question_record_question_id (question_id),
  KEY idx_user_question_record_user_question (user_id, question_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户答题记录表';
```

---

## 4. wrong_question 错题表

### 字段设计

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `bigint` | 主键 ID |
| `user_id` | `bigint` | 用户 ID |
| `question_id` | `bigint` | 题目 ID |
| `source_type` | `varchar(32)` | 来源：MANUAL / MASTERY / INTERVIEW |
| `wrong_reason` | `varchar(500)` | 错题原因 |
| `status` | `tinyint` | 状态：1 未掌握，0 已移出 |
| `created_at` | `datetime` | 创建时间 |
| `updated_at` | `datetime` | 更新时间 |
| `deleted` | `tinyint` | 逻辑删除 |

### SQL 草案

```sql
CREATE TABLE wrong_question (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  user_id BIGINT NOT NULL COMMENT '用户ID',
  question_id BIGINT NOT NULL COMMENT '题目ID',
  source_type VARCHAR(32) NOT NULL DEFAULT 'MANUAL' COMMENT '来源：MANUAL/MASTERY/INTERVIEW',
  wrong_reason VARCHAR(500) DEFAULT NULL COMMENT '错题原因',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1未掌握，0已移出',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  UNIQUE KEY uk_wrong_question_user_question (user_id, question_id),
  KEY idx_wrong_question_user_id (user_id),
  KEY idx_wrong_question_question_id (question_id),
  KEY idx_wrong_question_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='错题表';
```

---

## 5. favorite_question 收藏题表

### 字段设计

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `bigint` | 主键 ID |
| `user_id` | `bigint` | 用户 ID |
| `question_id` | `bigint` | 题目 ID |
| `created_at` | `datetime` | 创建时间 |
| `updated_at` | `datetime` | 更新时间 |
| `deleted` | `tinyint` | 逻辑删除 |

### SQL 草案

```sql
CREATE TABLE favorite_question (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  user_id BIGINT NOT NULL COMMENT '用户ID',
  question_id BIGINT NOT NULL COMMENT '题目ID',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  UNIQUE KEY uk_favorite_question_user_question (user_id, question_id),
  KEY idx_favorite_question_user_id (user_id),
  KEY idx_favorite_question_question_id (question_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='收藏题表';
```

---

## 6. user_question_mastery 用户题目掌握状态表

### 字段设计

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `bigint` | 主键 ID |
| `user_id` | `bigint` | 用户 ID |
| `question_id` | `bigint` | 题目 ID |
| `mastery_status` | `varchar(32)` | MASTERED / VAGUE / UNKNOWN |
| `last_answer_time` | `datetime` | 最近答题时间 |
| `created_at` | `datetime` | 创建时间 |
| `updated_at` | `datetime` | 更新时间 |
| `deleted` | `tinyint` | 逻辑删除 |

### SQL 草案

```sql
CREATE TABLE user_question_mastery (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  user_id BIGINT NOT NULL COMMENT '用户ID',
  question_id BIGINT NOT NULL COMMENT '题目ID',
  mastery_status VARCHAR(32) NOT NULL DEFAULT 'UNKNOWN' COMMENT '掌握状态：MASTERED/VAGUE/UNKNOWN',
  last_answer_time DATETIME DEFAULT NULL COMMENT '最近答题时间',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  UNIQUE KEY uk_user_question_mastery_user_question (user_id, question_id),
  KEY idx_user_question_mastery_user_id (user_id),
  KEY idx_user_question_mastery_question_id (question_id),
  KEY idx_user_question_mastery_status (mastery_status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户题目掌握状态表';
```

---

## 7. 业务规则

1. 用户提交答案后写入 `user_question_record`。
2. 用户标记不会或模糊时，更新 `user_question_mastery`。
3. 掌握状态为 `UNKNOWN` 或 `VAGUE` 时，可同步加入 `wrong_question`。
4. 用户收藏题目时写入 `favorite_question`，取消收藏可逻辑删除或物理删除。
5. V1 不做 AI 刷题点评，AI 面试评分归面试流程。

