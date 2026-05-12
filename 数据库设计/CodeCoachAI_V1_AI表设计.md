# CodeCoachAI V1 AI 表设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 AI 表设计 |
| 项目阶段 | V1 |
| 归属服务 | codecoachai-ai |
| 相关服务 | codecoachai-interview |

V1 AI 服务负责 Prompt 模板、AI 调用日志和大模型调用封装。

---

## 2. 表清单

| 表名 | 中文名 | V1 用途 |
|---|---|---|
| `prompt_template` | Prompt 模板表 | 后台维护不同 AI 场景提示词 |
| `ai_call_log` | AI 调用日志表 | 记录 AI 请求、响应、耗时和失败原因 |

V1 不设计 `prompt_template_version`、`ai_model_config`、`ai_task_record`。

---

## 3. prompt_template Prompt 模板表

### SQL 草案

```sql
CREATE TABLE prompt_template (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  template_name VARCHAR(128) NOT NULL COMMENT '模板名称',
  template_code VARCHAR(128) NOT NULL COMMENT '模板编码',
  scene VARCHAR(64) NOT NULL COMMENT 'AI场景',
  template_content LONGTEXT NOT NULL COMMENT '模板内容',
  variable_description TEXT DEFAULT NULL COMMENT '变量说明',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1启用，0禁用',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  UNIQUE KEY uk_prompt_template_code (template_code),
  KEY idx_prompt_template_scene (scene),
  KEY idx_prompt_template_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='Prompt模板表';
```

V1 必需模板场景：

```text
INTERVIEW_QUESTION_GENERATE
PROJECT_DEEP_DIVE_QUESTION
INTERVIEW_ANSWER_EVALUATE
INTERVIEW_FOLLOW_UP_GENERATE
INTERVIEW_REPORT_GENERATE
```

---

## 4. ai_call_log AI 调用日志表

### SQL 草案

```sql
CREATE TABLE ai_call_log (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  user_id BIGINT DEFAULT NULL COMMENT '用户ID',
  scene VARCHAR(64) NOT NULL COMMENT '调用场景',
  model_name VARCHAR(128) DEFAULT NULL COMMENT '模型名称',
  prompt_template_id BIGINT DEFAULT NULL COMMENT 'Prompt模板ID',
  business_id VARCHAR(128) DEFAULT NULL COMMENT '业务ID，如面试会话ID',
  request_content LONGTEXT DEFAULT NULL COMMENT '请求内容',
  response_content LONGTEXT DEFAULT NULL COMMENT '响应内容',
  cost_time_ms BIGINT DEFAULT NULL COMMENT '调用耗时毫秒',
  status VARCHAR(32) NOT NULL COMMENT '调用状态：SUCCESS/FAILED',
  error_message TEXT DEFAULT NULL COMMENT '失败原因',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  KEY idx_ai_call_log_user_id (user_id),
  KEY idx_ai_call_log_scene (scene),
  KEY idx_ai_call_log_status (status),
  KEY idx_ai_call_log_business_id (business_id),
  KEY idx_ai_call_log_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='AI调用日志表';
```

---

## 5. 业务规则

1. 所有 AI 调用必须经过 ai-service。
2. 每次 AI 调用都要写入 `ai_call_log`，成功失败都记录。
3. Prompt 模板仅管理员可维护。
4. AI 调用日志仅管理员可查询。
5. V1 不记录 token 消耗和模型版本历史，后续再增强。
6. 日志中不得记录 API Key、Token、用户密码。

