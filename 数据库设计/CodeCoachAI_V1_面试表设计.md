# CodeCoachAI V1 面试表设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 面试表设计 |
| 项目阶段 | V1 |
| 归属服务 | codecoachai-interview |
| 相关服务 | codecoachai-question、codecoachai-resume、codecoachai-ai |

面试数据由 interview-service 管理。AI 调用日志不放在面试服务，归属 ai-service。

---

## 2. 表清单

| 表名 | 中文名 | V1 用途 |
|---|---|---|
| `interview_session` | 面试会话表 | 保存面试配置、状态和总分 |
| `interview_stage` | 面试阶段表 | 保存面试大纲阶段 |
| `interview_message` | 面试消息表 | 保存 AI 问题、用户回答、点评、追问关系 |
| `interview_report` | 面试报告表 | 保存结构化面试报告 |

---

## 3. interview_session 面试会话表

### SQL 草案

```sql
CREATE TABLE interview_session (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  user_id BIGINT NOT NULL COMMENT '用户ID',
  interview_mode VARCHAR(32) NOT NULL COMMENT '面试模式',
  target_position VARCHAR(128) DEFAULT NULL COMMENT '目标岗位',
  experience_level VARCHAR(32) DEFAULT NULL COMMENT '经验年限',
  industry_direction VARCHAR(128) DEFAULT NULL COMMENT '行业方向',
  difficulty VARCHAR(32) NOT NULL DEFAULT 'MEDIUM' COMMENT '难度',
  interviewer_style VARCHAR(64) DEFAULT NULL COMMENT '面试官风格',
  resume_id BIGINT DEFAULT NULL COMMENT '简历ID',
  status VARCHAR(32) NOT NULL DEFAULT 'NOT_STARTED' COMMENT '面试状态',
  total_score DECIMAL(5,2) DEFAULT NULL COMMENT '总分',
  start_time DATETIME DEFAULT NULL COMMENT '开始时间',
  end_time DATETIME DEFAULT NULL COMMENT '结束时间',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  KEY idx_interview_session_user_id (user_id),
  KEY idx_interview_session_status (status),
  KEY idx_interview_session_user_created_at (user_id, created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='面试会话表';
```

---

## 4. interview_stage 面试阶段表

### SQL 草案

```sql
CREATE TABLE interview_stage (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  session_id BIGINT NOT NULL COMMENT '面试会话ID',
  stage_name VARCHAR(128) NOT NULL COMMENT '阶段名称',
  stage_order INT NOT NULL COMMENT '阶段顺序',
  expected_question_count INT NOT NULL DEFAULT 0 COMMENT '预计问题数',
  actual_question_count INT NOT NULL DEFAULT 0 COMMENT '实际问题数',
  focus_points VARCHAR(1000) DEFAULT NULL COMMENT '考察重点',
  status VARCHAR(32) NOT NULL DEFAULT 'NOT_STARTED' COMMENT '阶段状态',
  stage_score DECIMAL(5,2) DEFAULT NULL COMMENT '阶段得分',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  KEY idx_interview_stage_session_id (session_id),
  KEY idx_interview_stage_session_order (session_id, stage_order)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='面试阶段表';
```

---

## 5. interview_message 面试消息表

### SQL 草案

```sql
CREATE TABLE interview_message (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  session_id BIGINT NOT NULL COMMENT '面试会话ID',
  stage_id BIGINT NOT NULL COMMENT '面试阶段ID',
  question_id BIGINT DEFAULT NULL COMMENT '题目ID，项目深挖问题可为空',
  group_id BIGINT DEFAULT NULL COMMENT '问题组ID',
  role VARCHAR(32) NOT NULL COMMENT '角色：AI/USER/SYSTEM',
  question_content TEXT DEFAULT NULL COMMENT 'AI问题内容',
  user_answer TEXT DEFAULT NULL COMMENT '用户回答',
  ai_comment TEXT DEFAULT NULL COMMENT 'AI点评',
  score DECIMAL(5,2) DEFAULT NULL COMMENT '本轮得分',
  is_follow_up TINYINT NOT NULL DEFAULT 0 COMMENT '是否追问：1是，0否',
  parent_message_id BIGINT DEFAULT NULL COMMENT '父消息ID',
  follow_up_reason VARCHAR(500) DEFAULT NULL COMMENT '追问原因',
  knowledge_points VARCHAR(500) DEFAULT NULL COMMENT '知识点',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  KEY idx_interview_message_session_id (session_id),
  KEY idx_interview_message_stage_id (stage_id),
  KEY idx_interview_message_question_id (question_id),
  KEY idx_interview_message_group_id (group_id),
  KEY idx_interview_message_parent_id (parent_message_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='面试消息表';
```

---

## 6. interview_report 面试报告表

### SQL 草案

```sql
CREATE TABLE interview_report (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  session_id BIGINT NOT NULL COMMENT '面试会话ID',
  user_id BIGINT NOT NULL COMMENT '用户ID',
  total_score DECIMAL(5,2) DEFAULT NULL COMMENT '总分',
  module_scores_json TEXT DEFAULT NULL COMMENT '模块评分JSON',
  strength_summary TEXT DEFAULT NULL COMMENT '优势总结',
  problem_summary TEXT DEFAULT NULL COMMENT '问题总结',
  weak_knowledge_points TEXT DEFAULT NULL COMMENT '薄弱知识点',
  project_expression_problems TEXT DEFAULT NULL COMMENT '项目表达问题',
  review_suggestions TEXT DEFAULT NULL COMMENT '复习建议',
  recommended_questions_json TEXT DEFAULT NULL COMMENT '推荐题目JSON',
  report_content LONGTEXT DEFAULT NULL COMMENT '报告完整内容',
  status VARCHAR(32) NOT NULL DEFAULT 'COMPLETED' COMMENT '报告状态',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  UNIQUE KEY uk_interview_report_session_id (session_id),
  KEY idx_interview_report_user_id (user_id),
  KEY idx_interview_report_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='面试报告表';
```

---

## 7. 核心枚举

面试模式：

```text
TECHNICAL_BASIC
PROJECT_DEEP_DIVE
COMPREHENSIVE
```

面试状态：

```text
NOT_STARTED
IN_PROGRESS
WAITING_ANSWER
AI_EVALUATING
REPORT_GENERATING
COMPLETED
CANCELED
FAILED
```

---

## 8. 业务规则

1. 创建面试时写入 `interview_session` 和 `interview_stage`。
2. 每轮 AI 问题和用户回答必须及时保存到 `interview_message`。
3. 每个主问题最多追问 2 次，由 interview-service 统计 `parent_message_id` 控制。
4. 面试结束后同步调用 ai-service 生成报告，再写入 `interview_report`。
5. 用户只能查看自己的面试记录和报告。

