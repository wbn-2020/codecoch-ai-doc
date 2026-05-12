# CodeCoachAI V1 简历表设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 简历表设计 |
| 项目阶段 | V1 |
| 归属服务 | codecoachai-resume |
| 相关服务 | codecoachai-interview |

V1 只支持简历手动录入，不做 PDF / Word 上传解析，不设计文件表、解析记录表和简历优化表。

---

## 2. 表清单

| 表名 | 中文名 | V1 用途 |
|---|---|---|
| `resume` | 简历表 | 保存用户简历主体信息 |
| `resume_project` | 简历项目经历表 | 保存简历中的项目经历 |

---

## 3. resume 简历表

### 字段设计

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `bigint` | 主键 ID |
| `user_id` | `bigint` | 用户 ID |
| `resume_name` | `varchar(128)` | 简历名称 |
| `target_position` | `varchar(128)` | 求职方向 |
| `skills` | `text` | 技能栈 |
| `work_summary` | `text` | 工作经历摘要 |
| `education` | `text` | 教育经历 |
| `is_default` | `tinyint` | 是否默认简历 |
| `status` | `tinyint` | 状态：1 启用，0 禁用 |
| `created_at` | `datetime` | 创建时间 |
| `updated_at` | `datetime` | 更新时间 |
| `deleted` | `tinyint` | 逻辑删除 |

### SQL 草案

```sql
CREATE TABLE resume (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  user_id BIGINT NOT NULL COMMENT '用户ID',
  resume_name VARCHAR(128) NOT NULL COMMENT '简历名称',
  target_position VARCHAR(128) DEFAULT NULL COMMENT '求职方向',
  skills TEXT DEFAULT NULL COMMENT '技能栈',
  work_summary TEXT DEFAULT NULL COMMENT '工作经历摘要',
  education TEXT DEFAULT NULL COMMENT '教育经历',
  is_default TINYINT NOT NULL DEFAULT 0 COMMENT '是否默认简历：1是，0否',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1启用，0禁用',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  KEY idx_resume_user_id (user_id),
  KEY idx_resume_user_default (user_id, is_default),
  KEY idx_resume_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='简历表';
```

---

## 4. resume_project 简历项目经历表

### 字段设计

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `bigint` | 主键 ID |
| `resume_id` | `bigint` | 简历 ID |
| `project_name` | `varchar(128)` | 项目名称 |
| `project_time` | `varchar(128)` | 项目时间 |
| `project_background` | `text` | 项目背景 |
| `tech_stack` | `text` | 技术栈 |
| `responsibility` | `text` | 个人职责 |
| `core_features` | `text` | 核心功能 |
| `technical_challenges` | `text` | 技术难点 |
| `optimization_result` | `text` | 优化成果 |
| `extra_info` | `text` | 补充说明 |
| `sort_order` | `int` | 排序 |
| `created_at` | `datetime` | 创建时间 |
| `updated_at` | `datetime` | 更新时间 |
| `deleted` | `tinyint` | 逻辑删除 |

### SQL 草案

```sql
CREATE TABLE resume_project (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  resume_id BIGINT NOT NULL COMMENT '简历ID',
  project_name VARCHAR(128) NOT NULL COMMENT '项目名称',
  project_time VARCHAR(128) DEFAULT NULL COMMENT '项目时间',
  project_background TEXT DEFAULT NULL COMMENT '项目背景',
  tech_stack TEXT DEFAULT NULL COMMENT '技术栈',
  responsibility TEXT DEFAULT NULL COMMENT '个人职责',
  core_features TEXT DEFAULT NULL COMMENT '核心功能',
  technical_challenges TEXT DEFAULT NULL COMMENT '技术难点',
  optimization_result TEXT DEFAULT NULL COMMENT '优化成果',
  extra_info TEXT DEFAULT NULL COMMENT '补充说明',
  sort_order INT NOT NULL DEFAULT 0 COMMENT '排序',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  KEY idx_resume_project_resume_id (resume_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='简历项目经历表';
```

---

## 5. 业务规则

1. 用户只能查询和修改自己的简历。
2. 一个用户可以创建多份简历。
3. 一个用户最多只能有一份默认简历。
4. 创建项目深挖或综合模拟面试时，interview-service 通过 Feign 查询简历详情和项目经历。
5. V1 不保存简历文件，不保存解析状态。

