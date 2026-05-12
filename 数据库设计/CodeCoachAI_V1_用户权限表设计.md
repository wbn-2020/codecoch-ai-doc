# CodeCoachAI V1 用户权限表设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 用户权限表设计 |
| 项目阶段 | V1 |
| 最高依据 | PRD/CodeCoachAI_PRD_V1_微服务分层架构版.md |
| 前置文档 | 数据库设计/CodeCoachAI_V1_数据库设计总览.md |
| 归属服务 | codecoachai-user |
| 相关服务 | codecoachai-auth、codecoachai-gateway、codecoachai-system |

本文档只设计 V1 用户权限相关数据表。

V1 只保留两类角色：

```text
USER  普通用户
ADMIN 管理员
```

V1 不设计完整 RBAC 权限体系，不展开角色菜单关系、角色接口权限关系、数据权限等后续能力。

---

## 2. 设计目标

V1 用户权限表需要支撑：

1. 普通用户注册。
2. 普通用户登录。
3. 管理员登录。
4. 用户基础资料查询和修改。
5. 用户启用 / 禁用。
6. 用户角色判断。
7. 管理端接口判断管理员身份。
8. auth-service 通过 Feign 查询用户认证信息。

V1 权限控制目标：

1. 普通用户只能访问自己的简历、答题记录、面试记录和面试报告。
2. 管理员可以访问后台管理功能。
3. Prompt 模板和 AI 调用日志仅管理员可访问。
4. Gateway 进行 Token 基础校验。
5. 服务内部进行角色和数据归属校验。

---

## 3. 表清单

| 表名 | 中文名 | 归属服务 | V1 是否必需 |
|---|---|---|---:|
| `sys_user` | 用户表 | user-service | 是 |
| `sys_role` | 角色表 | user-service | 是 |
| `sys_user_role` | 用户角色关联表 | user-service | 是 |

V1 暂不设计：

| 表名 | 不做原因 |
|---|---|
| `sys_role_menu` | V1 可先按角色编码控制后台访问 |
| `sys_role_permission` | V1 不做细粒度接口权限 |
| `sys_dept` | V1 不做组织架构 |
| `sys_post` | V1 不做岗位管理 |
| `login_log` | V1 不做登录日志中心 |
| `operation_log` | V1 不做完整操作日志审计 |

---

## 4. 表关系

```text
sys_user
  └── sys_user_role
        └── sys_role
```

关系说明：

1. 一个用户可以绑定多个角色。
2. V1 实际上普通用户一般只绑定 `USER`。
3. 管理员账号绑定 `ADMIN`。
4. 是否允许一个用户同时拥有 `USER` 和 `ADMIN` 可由业务决定，V1 建议管理员账号独立初始化。

---

## 5. sys_user 用户表

## 5.1 表说明

`sys_user` 用于保存用户账号、密码、昵称、头像、邮箱、手机号、状态等基础信息。

该表由 user-service 直接管理。

auth-service 登录时通过 Feign 调用 user-service 查询该表信息，不直接访问该表。

## 5.2 字段设计

| 字段名 | 类型 | 是否必填 | 默认值 | 说明 |
|---|---|---:|---|---|
| `id` | `bigint` | 是 | 无 | 主键 ID |
| `username` | `varchar(64)` | 是 | 无 | 登录账号，唯一 |
| `password` | `varchar(255)` | 是 | 无 | 加密后的密码 |
| `nickname` | `varchar(64)` | 否 | 空 | 用户昵称 |
| `avatar` | `varchar(500)` | 否 | 空 | 头像 URL，V1 可先为空 |
| `email` | `varchar(128)` | 否 | 空 | 邮箱 |
| `phone` | `varchar(32)` | 否 | 空 | 手机号，V1 可选 |
| `status` | `tinyint` | 是 | `1` | 用户状态：1 正常，0 禁用 |
| `last_login_time` | `datetime` | 否 | 空 | 最近登录时间，V1 可选 |
| `created_at` | `datetime` | 是 | 当前时间 | 创建时间 |
| `updated_at` | `datetime` | 是 | 当前时间 | 更新时间 |
| `deleted` | `tinyint` | 是 | `0` | 逻辑删除：0 未删除，1 已删除 |

## 5.3 字段说明

### username

登录账号，V1 建议唯一。

约束：

```text
uk_sys_user_username(username)
```

### password

只保存加密后的密码。

要求：

1. 不允许明文存储。
2. 不允许返回前端。
3. 不允许写入日志。
4. 可使用 BCrypt、Sa-Token 推荐加密方式或其他安全哈希方案。

### status

用户状态：

| 值 | 含义 |
|---:|---|
| `1` | 正常 |
| `0` | 禁用 |

登录时 auth-service 需要校验用户状态。禁用用户不允许登录。

## 5.4 建表 SQL 草案

```sql
CREATE TABLE sys_user (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  username VARCHAR(64) NOT NULL COMMENT '登录账号',
  password VARCHAR(255) NOT NULL COMMENT '加密密码',
  nickname VARCHAR(64) DEFAULT NULL COMMENT '用户昵称',
  avatar VARCHAR(500) DEFAULT NULL COMMENT '头像URL',
  email VARCHAR(128) DEFAULT NULL COMMENT '邮箱',
  phone VARCHAR(32) DEFAULT NULL COMMENT '手机号',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1正常，0禁用',
  last_login_time DATETIME DEFAULT NULL COMMENT '最近登录时间',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  UNIQUE KEY uk_sys_user_username (username),
  KEY idx_sys_user_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';
```

---

## 6. sys_role 角色表

## 6.1 表说明

`sys_role` 用于保存系统角色。

V1 只需要：

```text
USER
ADMIN
```

## 6.2 字段设计

| 字段名 | 类型 | 是否必填 | 默认值 | 说明 |
|---|---|---:|---|---|
| `id` | `bigint` | 是 | 无 | 主键 ID |
| `role_code` | `varchar(64)` | 是 | 无 | 角色编码，唯一 |
| `role_name` | `varchar(64)` | 是 | 无 | 角色名称 |
| `description` | `varchar(255)` | 否 | 空 | 角色说明 |
| `status` | `tinyint` | 是 | `1` | 状态：1 启用，0 禁用 |
| `created_at` | `datetime` | 是 | 当前时间 | 创建时间 |
| `updated_at` | `datetime` | 是 | 当前时间 | 更新时间 |
| `deleted` | `tinyint` | 是 | `0` | 逻辑删除 |

## 6.3 角色编码

| role_code | role_name | 说明 |
|---|---|---|
| `USER` | 普通用户 | 使用用户端功能 |
| `ADMIN` | 管理员 | 访问管理端功能 |

## 6.4 建表 SQL 草案

```sql
CREATE TABLE sys_role (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  role_code VARCHAR(64) NOT NULL COMMENT '角色编码',
  role_name VARCHAR(64) NOT NULL COMMENT '角色名称',
  description VARCHAR(255) DEFAULT NULL COMMENT '角色说明',
  status TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1启用，0禁用',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  UNIQUE KEY uk_sys_role_code (role_code),
  KEY idx_sys_role_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='角色表';
```

---

## 7. sys_user_role 用户角色关联表

## 7.1 表说明

`sys_user_role` 用于保存用户和角色的关联关系。

V1 注册普通用户时，默认插入用户与 `USER` 角色的关联。

管理员账号可以通过初始化 SQL 预置。

## 7.2 字段设计

| 字段名 | 类型 | 是否必填 | 默认值 | 说明 |
|---|---|---:|---|---|
| `id` | `bigint` | 是 | 无 | 主键 ID |
| `user_id` | `bigint` | 是 | 无 | 用户 ID |
| `role_id` | `bigint` | 是 | 无 | 角色 ID |
| `created_at` | `datetime` | 是 | 当前时间 | 创建时间 |
| `updated_at` | `datetime` | 是 | 当前时间 | 更新时间 |
| `deleted` | `tinyint` | 是 | `0` | 逻辑删除 |

## 7.3 约束说明

建议增加唯一索引：

```text
uk_sys_user_role_user_role(user_id, role_id)
```

避免重复绑定同一个角色。

## 7.4 建表 SQL 草案

```sql
CREATE TABLE sys_user_role (
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键ID',
  user_id BIGINT NOT NULL COMMENT '用户ID',
  role_id BIGINT NOT NULL COMMENT '角色ID',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  deleted TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0未删除，1已删除',
  PRIMARY KEY (id),
  UNIQUE KEY uk_sys_user_role_user_role (user_id, role_id),
  KEY idx_sys_user_role_user_id (user_id),
  KEY idx_sys_user_role_role_id (role_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户角色关联表';
```

---

## 8. 初始化数据

## 8.1 角色初始化

```sql
INSERT INTO sys_role (id, role_code, role_name, description, status)
VALUES
  (1, 'USER', '普通用户', '用户端普通用户', 1),
  (2, 'ADMIN', '管理员', '管理端管理员', 1);
```

## 8.2 管理员账号初始化

管理员账号建议通过初始化 SQL 创建。

密码必须使用加密后的密文，以下 SQL 中的密码仅作为占位示例，实际初始化时需要替换为加密值。

```sql
INSERT INTO sys_user (id, username, password, nickname, email, status)
VALUES
  (1, 'admin', '{encrypted_password}', '系统管理员', 'admin@codecoachai.local', 1);

INSERT INTO sys_user_role (user_id, role_id)
VALUES
  (1, 2);
```

## 8.3 普通用户注册默认角色

普通用户注册流程中：

```text
1. 创建 sys_user
2. 查询 role_code = 'USER' 的角色
3. 创建 sys_user_role
```

---

## 9. 服务接口使用说明

## 9.1 auth-service 使用方式

auth-service 不直接访问用户表，只通过 user-service 内部接口获取认证信息。

建议内部接口：

```text
GET  /inner/users/by-username
POST /inner/users
GET  /inner/users/{id}
GET  /inner/users/{id}/roles
```

## 9.2 user-service 对外接口

用户端接口：

```text
GET /users/profile
PUT /users/profile
GET /users/overview
```

管理端接口：

```text
GET /admin/users
PUT /admin/users/{id}/status
```

## 9.3 Gateway 鉴权依赖

Gateway 需要依赖 Token 校验结果和用户上下文。

V1 推荐：

1. Gateway 进行 Token 基础校验。
2. 下游服务通过 common-security 获取当前用户 ID 和角色。
3. 管理端接口在服务内部校验 `ADMIN` 角色。

---

## 10. 权限控制规则

## 10.1 普通用户

普通用户允许：

1. 查看和修改自己的用户资料。
2. 管理自己的简历。
3. 查看自己的答题记录、错题、收藏。
4. 创建和查看自己的面试。
5. 查看自己的面试报告。

普通用户禁止：

1. 访问管理端接口。
2. 查看其他用户的简历。
3. 查看其他用户的面试记录。
4. 查看 AI 调用日志。
5. 修改 Prompt 模板。

## 10.2 管理员

管理员允许：

1. 登录管理端。
2. 维护题库、分类、标签、问题组。
3. 维护 Prompt 模板。
4. 查看 AI 调用日志。
5. 查看用户列表。
6. 启用 / 禁用用户，V1 可选。
7. 修改系统配置，V1 简化版。

管理员边界：

1. 不应随意修改普通用户的个人面试记录。
2. 不应查看用户敏感信息明文。
3. 不允许看到用户密码、Token、API Key。

---

## 11. 索引设计

| 表名 | 索引名 | 字段 | 类型 | 说明 |
|---|---|---|---|---|
| `sys_user` | `uk_sys_user_username` | `username` | 唯一索引 | 登录账号唯一 |
| `sys_user` | `idx_sys_user_status` | `status` | 普通索引 | 管理端按状态筛选 |
| `sys_role` | `uk_sys_role_code` | `role_code` | 唯一索引 | 角色编码唯一 |
| `sys_role` | `idx_sys_role_status` | `status` | 普通索引 | 查询启用角色 |
| `sys_user_role` | `uk_sys_user_role_user_role` | `user_id, role_id` | 唯一索引 | 防止重复授权 |
| `sys_user_role` | `idx_sys_user_role_user_id` | `user_id` | 普通索引 | 查询用户角色 |
| `sys_user_role` | `idx_sys_user_role_role_id` | `role_id` | 普通索引 | 查询角色用户 |

---

## 12. 安全要求

1. 密码必须加密存储。
2. 密码字段不得返回前端。
3. Token 不得写入业务日志。
4. 用户只能访问自己的数据。
5. 管理端接口必须校验管理员角色。
6. 禁用用户不允许登录。
7. 删除用户建议使用逻辑删除。
8. 用户 ID 从登录上下文获取，不信任前端传入的 userId。

---

## 13. V1 不做的用户权限能力

| 能力 | V1 不做原因 |
|---|---|
| 完整菜单权限 | V1 先用角色控制管理端访问 |
| 角色菜单关联 | V1 后台页面数量有限 |
| 角色接口权限关联 | V1 不做细粒度接口权限 |
| 数据权限 | V1 只做用户数据归属校验 |
| 部门组织架构 | 与 V1 核心闭环无关 |
| 多管理员权限分级 | V1 管理员角色即可 |
| 登录日志中心 | V1 不做完整日志审计 |
| 操作日志中心 | V1 不做完整操作日志 |
| 手机验证码登录 | V1 用户名密码登录即可 |
| 第三方登录 | 与 V1 核心闭环无关 |

---

## 14. 后续实现注意事项

1. `sys_user.password` 只能在认证内部 DTO 中使用，不能出现在 VO 中。
2. 用户注册时需要在同一个本地事务中创建用户和用户角色关系。
3. 禁用用户时，如果使用 Redis 保存登录态，建议让已登录 Token 失效。
4. 管理端用户列表不返回密码字段。
5. `username` 建议不可修改，避免登录身份混乱。
6. 用户删除优先逻辑删除，保留历史面试记录中的 `user_id` 关联。
7. 普通用户查询个人数据时，后端必须使用登录上下文中的 userId 过滤。

---

## 15. 结论

CodeCoachAI V1 用户权限数据设计保持轻量：

```text
sys_user
  ↓
sys_user_role
  ↓
sys_role
```

V1 只需要支撑普通用户和管理员两类角色，优先保证注册登录、用户资料、管理端角色校验和用户数据隔离。

完整菜单权限、接口权限、操作日志、登录日志等能力放到后续版本，不进入 V1 数据库设计范围。
