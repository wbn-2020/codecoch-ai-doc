# CodeCoachAI V1 系统配置表设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 系统配置表设计 |
| 项目阶段 | V1 |
| 归属服务 | codecoachai-system |

V1 系统服务保持轻量，重点支持基础系统配置。菜单和权限表为可选表，如果 V1 只按 `ADMIN` 角色控制后台访问，可以暂不落地。

---

## 2. 表清单

| 表名 | 中文名 | V1 是否必需 |
|---|---|---:|
| `system_config` | 系统配置表 | 是，简化版 |
| `sys_menu` | 菜单表 | 可选 |
| `sys_permission` | 权限标识表 | 可选 |

---

## 3. system_config 系统配置表

### SQL 草案

```sql
CREATE TABLE system_config (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  config_key VARCHAR(128) NOT NULL COMMENT '配置Key',
  config_value VARCHAR(1000) DEFAULT NULL COMMENT '配置值',
  config_name VARCHAR(128) DEFAULT NULL COMMENT '配置名称',
  description VARCHAR(500) DEFAULT NULL COMMENT '配置说明',
  config_type VARCHAR(32) NOT NULL DEFAULT 'STRING' COMMENT '配置类型：STRING/NUMBER/BOOLEAN/JSON',
  editable TINYINT NOT NULL DEFAULT 1 COMMENT '是否可编辑：1是，0否',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1启用，0禁用',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  UNIQUE KEY uk_system_config_key (config_key),
  KEY idx_system_config_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='系统配置表';
```

V1 建议配置项：

```text
interview.max_follow_up_count       每个主问题最大追问次数，默认 2
interview.max_question_count        每场面试最大问题数
ai.timeout_seconds                  AI 调用超时时间
ai.daily_limit_per_user             单用户每日 AI 调用上限
```

---

## 4. sys_menu 菜单表，可选

如果 V1 管理端需要动态菜单，可创建该表。

```sql
CREATE TABLE sys_menu (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  parent_id BIGINT NOT NULL DEFAULT 0 COMMENT '父菜单ID',
  menu_name VARCHAR(64) NOT NULL COMMENT '菜单名称',
  menu_path VARCHAR(255) DEFAULT NULL COMMENT '前端路由路径',
  component VARCHAR(255) DEFAULT NULL COMMENT '前端组件路径',
  icon VARCHAR(64) DEFAULT NULL COMMENT '图标',
  sort_order INT NOT NULL DEFAULT 0 COMMENT '排序',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1启用，0禁用',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  KEY idx_sys_menu_parent_id (parent_id),
  KEY idx_sys_menu_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='菜单表';
```

---

## 5. sys_permission 权限标识表，可选

如果 V1 需要按钮或接口级权限标识，可创建该表。默认建议暂不实现。

```sql
CREATE TABLE sys_permission (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  permission_code VARCHAR(128) NOT NULL COMMENT '权限编码',
  permission_name VARCHAR(128) NOT NULL COMMENT '权限名称',
  description VARCHAR(500) DEFAULT NULL COMMENT '权限说明',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1启用，0禁用',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  UNIQUE KEY uk_sys_permission_code (permission_code),
  KEY idx_sys_permission_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='权限标识表';
```

---

## 6. V1 边界

1. V1 不做完整 RBAC。
2. V1 不做角色菜单关联表。
3. V1 不做角色权限关联表。
4. V1 不做操作日志表。
5. V1 不做登录日志表。
6. 管理端访问优先使用 `ADMIN` 角色控制。

