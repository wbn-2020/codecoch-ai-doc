# CodeCoachAI V1 系统管理接口设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 系统管理接口设计 |
| 项目阶段 | V1 |
| 最高依据 | 接口设计/CodeCoachAI_V1_接口设计总览.md |
| 参考文档 | PRD/CodeCoachAI_PRD_V1_微服务分层架构版.md、数据库设计/CodeCoachAI_V1_数据库设计总览.md、数据库设计/CodeCoachAI_V1_系统配置表设计.md、数据库设计/CodeCoachAI_V1_用户权限表设计.md |
| 文档用途 | 详细设计 system-service 在 V1 阶段需要提供的系统配置接口、管理端首页统计接口，以及 V1 可选的菜单接口；角色查询仅说明归属边界 |

本文档只设计 CodeCoachAI V1 系统管理相关接口。V1 system-service 保持轻量，不设计完整操作日志、站内通知、复杂数据看板、复杂 RBAC 权限管理等后续能力。

涉及服务：

| 服务 | 说明 |
|---|---|
| `codecoachai-system` | 系统配置和管理端首页简化统计的数据归属服务 |
| `codecoachai-user` | 可选，用于角色数据来源说明和首页用户统计 |
| `codecoachai-question` | 可选，用于首页题库统计 |
| `codecoachai-resume` | 可选，用于首页简历统计 |
| `codecoachai-interview` | 可选，用于首页面试统计 |
| `codecoachai-ai` | 可选，用于首页 AI 调用、Prompt 统计 |
| `codecoachai-gateway` | 对外统一入口，不对外暴露 `/inner/**` |
| `codecoachai-auth` / `codecoachai-user` | 提供认证与管理员权限 |

字段说明：

1. 本文档字段优先贴合 `数据库设计/CodeCoachAI_V1_系统配置表设计.md`。
2. `system_config` 当前字段包含 `config_key`、`config_value`、`config_name`、`description`、`config_type`、`editable`、`status`、`created_at`、`updated_at`、`deleted`。
3. 接口层使用驼峰字段，例如 `configKey` 对应数据库字段 `config_key`，`configType` 对应数据库字段 `config_type`。
4. `sys_menu`、`sys_permission` 为 V1 可选表；如果 V1 管理端采用静态路由和 `ADMIN` 角色控制，可以暂不实现菜单权限接口。

---

## 2. 模块职责

system-service 负责：

1. 系统配置查询。
2. 系统配置新增，V1 仅建议用于非核心配置。
2. 系统配置修改。
3. 系统配置启用 / 禁用。
4. 系统配置删除，V1 仅建议用于非核心配置。
5. 管理端首页简化统计。
6. 菜单查询，V1 可选。
7. 系统级参数说明。

system-service 不负责：

| 能力 | V1 处理方式 |
|---|---|
| 用户注册登录 | 归属 auth-service |
| 用户角色关系维护 | 归属 user-service |
| 角色列表查询核心能力 | 推荐由 user-service 直接提供 `/admin/roles` |
| 题库管理 | 归属 question-service |
| 简历管理 | 归属 resume-service |
| 面试流程 | 归属 interview-service |
| AI Prompt 模板管理 | 归属 ai-service |
| AI 调用日志管理 | 归属 ai-service |
| 完整操作日志 | V1 不设计 |
| 复杂数据看板 | V1 不设计 |
| 站内通知 | V1 不设计 |
| 复杂 RBAC 权限管理 | V1 不设计，管理端优先按 `ADMIN` 角色控制 |

服务边界说明：

1. system-service 是 `system_config` 的数据归属服务。
2. user-service 是 `sys_user`、`sys_role`、`sys_user_role` 的数据归属服务。
3. system-service V1 不直接维护 `sys_role` 和用户角色关系。
4. 角色列表接口推荐由 user-service 提供，system-service 不直接管理角色数据。
5. system-service 如需在首页统计中展示角色数据，只能通过 Spring Cloud OpenFeign 调用 user-service 的内部统计接口，不能直接查询 user-service 的数据库表。
6. 首页统计可以通过 Spring Cloud OpenFeign 调用其他服务的内部统计接口，也可以在 V1 初始阶段先返回简化统计。

---

## 3. 统一约定

### 3.1 统一响应结构

所有接口使用接口总览中定义的统一响应结构：

```json
{
  "code": 0,
  "message": "success",
  "data": {},
  "traceId": "optional-trace-id"
}
```

字段说明：

| 字段 | 类型 | 说明 |
|---|---|---|
| `code` | `int` | 业务状态码，`0` 表示成功 |
| `message` | `string` | 响应提示 |
| `data` | `object` | 响应数据 |
| `traceId` | `string` | 链路追踪 ID，V1 可选 |

### 3.2 分页结构

分页请求参数：

| 字段名 | 类型 | 是否必填 | 默认值 | 说明 |
|---|---|---:|---|---|
| `pageNo` | `int` | 否 | `1` | 当前页码，最小为 1 |
| `pageSize` | `int` | 否 | `10` | 每页数量，最大不超过 100 |

分页响应结构：

```json
{
  "records": [],
  "total": 0,
  "pageNo": 1,
  "pageSize": 10,
  "pages": 0
}
```

### 3.3 Token 请求头格式

管理端接口统一使用：

```text
Authorization: Bearer {token}
```

说明：

1. `/admin/system/**`、`/admin/configs/**`、`/admin/menus` 都需要登录。
2. 管理端接口必须具备 `ADMIN` 角色。
3. 使用 Sa-Token 时，建议统一读取 `Authorization` 请求头并兼容 `Bearer ` 前缀。

### 3.4 管理端权限要求

管理端接口统一要求：

1. 必须登录。
2. 必须具备 `ADMIN` 角色。
3. 不返回用户密码、Token、API Key 等敏感信息。
4. 不允许普通用户访问 `/admin/**`。

### 3.5 system_config 配置 key 规则

`configKey` 对应数据库字段 `config_key`。

建议规则：

1. 使用小写字母、数字、点号、下划线、中划线。
2. 按业务域分组，例如 `interview.max_follow_up_count`、`ai.timeout_seconds`。
3. 不建议使用中文、空格和特殊符号。
4. `configKey` 创建后不建议修改。

V1 建议预置配置项：

| configKey | configName | configType | 说明 |
|---|---|---|---|
| `interview.max_follow_up_count` | 每题最大追问次数 | `NUMBER` | 默认可为 2 |
| `interview.max_question_count` | 每场面试最大问题数 | `NUMBER` | 控制单场面试题量 |
| `ai.timeout_seconds` | AI 调用超时时间 | `NUMBER` | 控制同步 AI 请求超时 |
| `ai.daily_limit_per_user` | 单用户每日 AI 调用上限 | `NUMBER` | V1 可选限流参数 |

### 3.6 配置值 value 类型规则

`configType` 对应数据库字段 `config_type`。

| configType | configValue 规则 |
|---|---|
| `STRING` | 普通字符串 |
| `NUMBER` | 必须能转换为数字 |
| `BOOLEAN` | 必须为 `true` 或 `false` |
| `JSON` | 必须是合法 JSON 字符串 |

### 3.7 敏感配置脱敏规则

1. `system_config` 可存储系统基础配置，但不建议存储明文 API Key。
2. 如果确实需要管理 AI 模型供应商配置，V1 建议只保存非敏感配置，例如 `modelName`、`baseUrl` 配置标识等。
3. API Key 建议通过环境变量、Nacos 配置中心或服务器安全配置管理。
4. API Key 不允许通过普通接口返回。
5. 对疑似敏感配置，列表和详情接口应返回脱敏值或不返回 `configValue`。

### 3.8 配置修改审慎原则

1. `editable=0` 的配置不允许通过接口修改。
2. V1 不设计配置发布、灰度、历史版本。
3. 修改后是否立即生效取决于业务读取方式，V1 可先约定“下次读取生效”，部分配置需要重启时应在 `description` 中说明。
4. 核心必需配置不建议禁用，可以通过 `editable=0` 或内置配置约束控制。

### 3.9 Spring Cloud OpenFeign 内部统计调用说明

管理端首页统计可以由 system-service 通过 Spring Cloud OpenFeign 调用其他服务的内部统计接口。

约束：

1. 内部统计接口只允许 system-service 调用。
2. Gateway 不对外暴露 `/inner/**`。
3. 前端不能直接访问内部统计接口。
4. V1 初始阶段如果内部统计接口尚未补齐，`GET /admin/system/overview` 可以返回部分统计或简化统计，接口结构保持稳定。

---

## 4. 枚举设计

### 4.1 ConfigValueType

对应数据库字段 `system_config.config_type`。

| 值 | 说明 |
|---|---|
| `STRING` | 字符串 |
| `NUMBER` | 数字 |
| `BOOLEAN` | 布尔值 |
| `JSON` | JSON 配置 |

### 4.2 ConfigStatus

对应数据库字段 `system_config.status`。

| 值 | 枚举名 | 说明 |
|---|---|---|
| `1` | `ENABLED` | 启用 |
| `0` | `DISABLED` | 禁用 |

### 4.3 ConfigEditable

对应数据库字段 `system_config.editable`。

| 值 | 说明 |
|---|---|
| `1` | 可编辑 |
| `0` | 不可编辑 |

### 4.4 MenuStatus，V1 可选

对应可选表 `sys_menu.status`。

| 值 | 枚举名 | 说明 |
|---|---|---|
| `1` | `ENABLED` | 启用 |
| `0` | `DISABLED` | 禁用 |

---

## 5. 管理端首页统计接口设计

### 5.1 GET /admin/system/overview

管理端首页统计。

| 项目 | 内容 |
|---|---|
| URL | `/admin/system/overview` |
| Method | `GET` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 归属服务 | `codecoachai-system` |

接口用途：

1. 用于管理端首页展示项目运行概览。
2. 用于让管理员快速了解用户、题库、简历、面试、AI 调用等 V1 核心数据规模。
3. 该接口不影响用户端 AI 模拟面试核心闭环。

查询参数：

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `dateRange` | `string` | 否 | V1 可不实现复杂时间筛选；如实现，可使用 `TODAY`、`LAST_7_DAYS` 等简单枚举 |

响应字段：

| 字段名 | 类型 | 说明 |
|---|---|---|
| `userCount` | `long` | 用户总数 |
| `questionCount` | `long` | 题目总数 |
| `resumeCount` | `long` | 简历总数 |
| `interviewCount` | `long` | 面试总数 |
| `completedInterviewCount` | `long` | 已完成面试数 |
| `aiCallCount` | `long` | AI 调用总数 |
| `aiCallFailedCount` | `long` | AI 调用失败总数 |
| `promptCount` | `long` | Prompt 模板总数 |
| `todayInterviewCount` | `long` | 今日新增面试数 |
| `todayAiCallCount` | `long` | 今日 AI 调用数 |

响应示例：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "userCount": 128,
    "questionCount": 360,
    "resumeCount": 92,
    "interviewCount": 210,
    "completedInterviewCount": 176,
    "aiCallCount": 1850,
    "aiCallFailedCount": 23,
    "promptCount": 8,
    "todayInterviewCount": 12,
    "todayAiCallCount": 96
  }
}
```

业务规则：

1. 仅管理员可访问。
2. V1 只做简化统计。
3. 可以通过 Spring Cloud OpenFeign 调用 user-service、question-service、resume-service、interview-service、ai-service 的内部统计接口。
4. 如果内部统计接口尚未设计，V1 初始实现允许先返回 system-service 可获得或 mock 的简化统计，但接口结构保持稳定。
5. 不做复杂趋势图、漏斗图、同比环比统计。
6. 不做大屏看板。
7. 如果为降低 V1 复杂度，可以在 Spring Cloud OpenFeign 内部接口设计总表中统一补充统计接口，或后续实现阶段逐步补齐。

---

## 6. 系统配置接口设计

### 6.1 GET /admin/configs

系统配置列表。

| 项目 | 内容 |
|---|---|
| URL | `/admin/configs` |
| Method | `GET` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 归属服务 | `codecoachai-system` |

查询参数：

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 按配置名称、配置 Key、说明模糊查询 |
| `configKey` | `string` | 否 | 配置 Key，精确或前缀查询由实现决定 |
| `status` | `int` | 否 | 状态：`1` 启用，`0` 禁用 |
| `pageNo` | `int` | 否 | 当前页码，默认 1 |
| `pageSize` | `int` | 否 | 每页数量，默认 10，最大 100 |

响应字段：

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 配置 ID |
| `configKey` | `string` | 配置 Key，对应 `config_key` |
| `configName` | `string` | 配置名称，对应 `config_name` |
| `configValue` | `string` | 配置值，对应 `config_value`；敏感配置返回脱敏值或空 |
| `configType` | `string` | 配置类型，对应 `config_type` |
| `editable` | `int` | 是否可编辑：`1` 是，`0` 否 |
| `status` | `int` | 状态：`1` 启用，`0` 禁用 |
| `description` | `string` | 配置说明 |
| `createdAt` | `string` | 创建时间 |
| `updatedAt` | `string` | 更新时间 |

业务规则：

1. 仅管理员可访问。
2. 支持按配置 Key、名称、状态查询。
3. 敏感配置不返回明文值。
4. 不可编辑配置可以展示，但不允许修改。
5. 默认按 `updatedAt` 倒序。
6. 不返回已逻辑删除配置。
7. V1 可设计新增和删除系统配置接口，但仅建议用于非核心、可编辑配置；核心配置建议通过初始化数据预置。

### 6.2 POST /admin/configs

新增系统配置。

| 项目 | 内容 |
|---|---|
| URL | `/admin/configs` |
| Method | `POST` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 归属服务 | `codecoachai-system` |

请求参数：

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `configKey` | `string` | 是 | 配置 Key，对应 `config_key` |
| `configName` | `string` | 否 | 配置名称，对应 `config_name` |
| `configValue` | `string` | 否 | 配置值，对应 `config_value` |
| `configType` | `string` | 是 | 配置类型：`STRING` / `NUMBER` / `BOOLEAN` / `JSON` |
| `editable` | `int` | 否 | 是否可编辑，默认 `1` |
| `status` | `int` | 否 | 状态，默认 `1` |
| `description` | `string` | 否 | 配置说明 |

响应字段：

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 配置 ID |
| `configKey` | `string` | 配置 Key |
| `createdAt` | `string` | 创建时间 |

业务规则：

1. 仅管理员可操作。
2. `configKey` 必须唯一。
3. `configValue` 必须符合 `configType`。
4. V1 推荐该接口只用于新增非核心配置。
5. 核心配置、敏感配置建议通过初始化数据、Nacos 配置中心或环境变量管理。
6. 不设计配置发布、灰度、历史版本。

### 6.3 GET /admin/configs/{key}

系统配置详情。

> 说明：该接口用于系统配置编辑页回显，已补充到接口总览。

| 项目 | 内容 |
|---|---|
| URL | `/admin/configs/{key}` |
| Method | `GET` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 归属服务 | `codecoachai-system` |

路径参数：

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `key` | `string` | 是 | 配置 Key，对应 `system_config.config_key` |

响应字段：

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 配置 ID |
| `configKey` | `string` | 配置 Key |
| `configName` | `string` | 配置名称 |
| `configValue` | `string` | 配置值；敏感配置返回脱敏值或空 |
| `configType` | `string` | 配置类型：`STRING` / `NUMBER` / `BOOLEAN` / `JSON` |
| `editable` | `int` | 是否可编辑 |
| `status` | `int` | 状态 |
| `description` | `string` | 配置说明 |
| `createdAt` | `string` | 创建时间 |
| `updatedAt` | `string` | 更新时间 |

业务规则：

1. 仅管理员可访问。
2. 配置不存在返回错误。
3. 不返回已逻辑删除配置。
4. 敏感配置返回脱敏值或不返回 `configValue`。
5. 用于系统配置编辑页回显。

### 6.4 PUT /admin/configs/{key}

修改系统配置。

| 项目 | 内容 |
|---|---|
| URL | `/admin/configs/{key}` |
| Method | `PUT` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 归属服务 | `codecoachai-system` |

路径参数：

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `key` | `string` | 是 | 配置 Key，对应 `system_config.config_key` |

请求参数：

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `configValue` | `string` | 是 | 新配置值，必须符合 `configType` |
| `description` | `string` | 否 | 配置说明，可用于补充备注 |

响应字段：

| 字段名 | 类型 | 说明 |
|---|---|---|
| `configKey` | `string` | 配置 Key |
| `configValue` | `string` | 修改后的配置值；敏感配置返回脱敏值或空 |
| `updatedAt` | `string` | 更新时间 |

业务规则：

1. 仅管理员可操作。
2. 配置必须存在。
3. 配置必须 `editable=1`。
4. V1 推荐禁用配置不允许修改，避免管理员误以为修改后已生效；如需调整，应先启用再修改。
5. `configValue` 必须符合 `configType`。
6. 敏感配置不建议通过该接口修改明文值。
7. 修改后是否立即生效取决于配置类型，V1 可先要求下次读取生效；如果需要重启，应在 `description` 中明确。
8. 不设计配置发布、灰度、历史版本。

### 6.5 PUT /admin/configs/{key}/status

启用 / 禁用系统配置。

> 说明：该接口用于管理配置启用状态，已补充到接口总览。

| 项目 | 内容 |
|---|---|
| URL | `/admin/configs/{key}/status` |
| Method | `PUT` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 归属服务 | `codecoachai-system` |

路径参数：

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `key` | `string` | 是 | 配置 Key |

请求参数：

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `status` | `int` | 是 | `1` 启用，`0` 禁用 |

响应字段：

| 字段名 | 类型 | 说明 |
|---|---|---|
| `configKey` | `string` | 配置 Key |
| `status` | `int` | 修改后的状态 |
| `updatedAt` | `string` | 更新时间 |

业务规则：

1. 仅管理员可操作。
2. 配置必须存在。
3. `status` 只能是 `0` 或 `1`。
4. 禁用后业务读取配置时应忽略该配置或使用默认值。
5. 核心必需配置不建议禁用，可以通过 `editable=0` 或内置配置约束控制。
6. 不返回已逻辑删除配置。

### 6.6 DELETE /admin/configs/{key}

删除系统配置。

| 项目 | 内容 |
|---|---|
| URL | `/admin/configs/{key}` |
| Method | `DELETE` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 归属服务 | `codecoachai-system` |

路径参数：

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `key` | `string` | 是 | 配置 Key |

响应字段：

可为空。

业务规则：

1. 仅管理员可操作。
2. 配置必须存在。
3. V1 推荐使用逻辑删除。
4. `editable=0` 的核心配置不允许删除。
5. 敏感配置、内置配置不建议通过普通接口删除。
6. 删除后业务读取配置时应使用默认值或返回配置不存在。

---

## 7. 菜单接口设计，V1 可选

数据库设计中已提供可选表 `sys_menu`。如果 V1 管理端需要动态菜单，可以实现本节接口；如果前端先使用本地静态路由，则 V1 可暂不实现该接口。

### 7.1 GET /admin/menus

管理端菜单列表。

| 项目 | 内容 |
|---|---|
| URL | `/admin/menus` |
| Method | `GET` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 归属服务 | `codecoachai-system`，V1 可选 |

响应字段：

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 菜单 ID |
| `parentId` | `long` | 父菜单 ID，对应 `parent_id` |
| `menuName` | `string` | 菜单名称，对应 `menu_name` |
| `path` | `string` | 前端路由路径，对应数据库 `menu_path`，接口层可简化为 `path` |
| `component` | `string` | 前端组件路径 |
| `icon` | `string` | 图标 |
| `sort` | `int` | 排序，对应数据库 `sort_order` |
| `status` | `int` | 状态：`1` 启用，`0` 禁用 |
| `children` | `array` | 子菜单列表 |

业务规则：

1. 仅管理员可访问。
2. V1 可返回静态菜单或数据库菜单。
3. 不做复杂按钮权限。
4. 不做动态路由权限细粒度控制。
5. 前端也可以先使用本地静态路由，接口可选。
6. 只返回未删除菜单。
7. 如果 V1 不创建 `sys_menu` 表，则该接口暂不实现。

---

## 8. 角色接口归属说明

V1 角色数据归属 user-service。system-service 不直接维护 `sys_role`，也不维护用户角色关系。

### 8.1 GET /admin/roles

角色列表接口推荐由 user-service 提供。

| 项目 | 内容 |
|---|---|
| URL | `/admin/roles` |
| Method | `GET` |
| 需要登录 | 是 |
| 管理员权限 | 是 |
| 推荐归属服务 | `codecoachai-user` |
| system-service 处理方式 | 不作为 system-service 核心接口；如需角色统计或基础角色信息，应通过 Spring Cloud OpenFeign 调用 user-service 内部接口 |

响应字段：

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 角色 ID |
| `roleCode` | `string` | 角色编码，例如 `USER`、`ADMIN` |
| `roleName` | `string` | 角色名称 |
| `description` | `string` | 角色说明 |
| `status` | `int` | 状态：`1` 启用，`0` 禁用 |

业务规则：

1. 仅管理员可访问。
2. V1 角色数据归 user-service 管理。
3. system-service V1 不直接维护 `sys_role`、`sys_user_role`。
4. system-service 不直接查询 user-service 的数据库表。
5. 不设计角色新增、修改、删除接口。
6. 不设计角色授权菜单接口。
7. 接口总览中 `/admin/roles` 的实际归属应为 user-service。

---

## 9. 内部统计接口建议，V1 可选

本章用于说明管理端首页统计可能依赖的内部接口。以下接口为“建议接口”，不强制全部实现。

统一说明：

1. 这些接口只允许 system-service 通过 Spring Cloud OpenFeign 内部调用。
2. 不对前端开放。
3. Gateway 不对外暴露。
4. V1 初始阶段如果不实现这些接口，`GET /admin/system/overview` 可以先降级返回部分统计。
5. 后续 Spring Cloud OpenFeign 内部接口总表中可再统一整理。

### 9.1 GET /inner/users/statistics

调用方：system-service。

建议返回字段：

| 字段名 | 类型 | 说明 |
|---|---|---|
| `userCount` | `long` | 用户总数 |
| `enabledUserCount` | `long` | 启用用户数 |
| `disabledUserCount` | `long` | 禁用用户数 |
| `adminCount` | `long` | 管理员数量，V1 可选 |
| `todayRegisterCount` | `long` | 今日注册用户数，V1 可选 |

### 9.2 GET /inner/questions/statistics

调用方：system-service。

建议返回字段：

| 字段名 | 类型 | 说明 |
|---|---|---|
| `questionCount` | `long` | 题目总数 |
| `enabledQuestionCount` | `long` | 启用题目数 |
| `categoryCount` | `long` | 分类数量 |
| `tagCount` | `long` | 标签数量 |
| `questionGroupCount` | `long` | 问题组数量 |

### 9.3 GET /inner/resumes/statistics

调用方：system-service。

建议返回字段：

| 字段名 | 类型 | 说明 |
|---|---|---|
| `resumeCount` | `long` | 简历总数 |
| `projectCount` | `long` | 项目经历总数 |
| `todayResumeCount` | `long` | 今日新增简历数，V1 可选 |

### 9.4 GET /inner/interviews/statistics

调用方：system-service。

建议返回字段：

| 字段名 | 类型 | 说明 |
|---|---|---|
| `interviewCount` | `long` | 面试总数 |
| `completedInterviewCount` | `long` | 已完成面试数 |
| `inProgressInterviewCount` | `long` | 进行中面试数 |
| `todayInterviewCount` | `long` | 今日新增面试数 |
| `reportGeneratedCount` | `long` | 已生成报告数 |
| `reportFailedCount` | `long` | 报告生成失败数 |

### 9.5 GET /inner/ai/statistics

调用方：system-service。

建议返回字段：

| 字段名 | 类型 | 说明 |
|---|---|---|
| `aiCallCount` | `long` | AI 调用总数 |
| `aiCallFailedCount` | `long` | AI 调用失败总数 |
| `todayAiCallCount` | `long` | 今日 AI 调用数 |
| `promptCount` | `long` | Prompt 模板总数 |
| `enabledPromptCount` | `long` | 启用 Prompt 模板数 |

---

## 10. DTO / VO 设计

本节只描述主要请求 DTO 和响应 VO 字段，不编写 Java 代码。

### 10.1 AdminSystemOverviewVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userCount` | `long` | 是 | 用户总数 |
| `questionCount` | `long` | 是 | 题目总数 |
| `resumeCount` | `long` | 是 | 简历总数 |
| `interviewCount` | `long` | 是 | 面试总数 |
| `completedInterviewCount` | `long` | 是 | 已完成面试数 |
| `aiCallCount` | `long` | 是 | AI 调用总数 |
| `aiCallFailedCount` | `long` | 是 | AI 调用失败总数 |
| `promptCount` | `long` | 是 | Prompt 模板总数 |
| `todayInterviewCount` | `long` | 是 | 今日新增面试数 |
| `todayAiCallCount` | `long` | 是 | 今日 AI 调用数 |

### 10.2 SystemConfigQueryRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 关键字，匹配配置 Key、名称、说明 |
| `configKey` | `string` | 否 | 配置 Key |
| `status` | `int` | 否 | 状态：`1` 启用，`0` 禁用 |
| `pageNo` | `int` | 否 | 当前页码，默认 1 |
| `pageSize` | `int` | 否 | 每页数量，默认 10，最大 100 |

### 10.3 SystemConfigPageVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 配置 ID |
| `configKey` | `string` | 是 | 配置 Key |
| `configName` | `string` | 否 | 配置名称 |
| `configValue` | `string` | 否 | 配置值，敏感配置返回脱敏值或空 |
| `configType` | `string` | 是 | 配置类型：`STRING` / `NUMBER` / `BOOLEAN` / `JSON` |
| `editable` | `int` | 是 | 是否可编辑：`1` 是，`0` 否 |
| `status` | `int` | 是 | 状态：`1` 启用，`0` 禁用 |
| `description` | `string` | 否 | 配置说明 |
| `createdAt` | `string` | 是 | 创建时间 |
| `updatedAt` | `string` | 是 | 更新时间 |

### 10.4 SystemConfigDetailVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 配置 ID |
| `configKey` | `string` | 是 | 配置 Key |
| `configName` | `string` | 否 | 配置名称 |
| `configValue` | `string` | 否 | 配置值，敏感配置返回脱敏值或空 |
| `configType` | `string` | 是 | 配置类型 |
| `editable` | `int` | 是 | 是否可编辑 |
| `status` | `int` | 是 | 状态 |
| `description` | `string` | 否 | 配置说明 |
| `createdAt` | `string` | 是 | 创建时间 |
| `updatedAt` | `string` | 是 | 更新时间 |

### 10.5 SaveSystemConfigRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `configKey` | `string` | 是 | 配置 Key |
| `configName` | `string` | 否 | 配置名称 |
| `configValue` | `string` | 否 | 配置值 |
| `configType` | `string` | 是 | 配置类型：`STRING` / `NUMBER` / `BOOLEAN` / `JSON` |
| `editable` | `int` | 否 | 是否可编辑，默认 `1` |
| `status` | `int` | 否 | 状态，默认 `1` |
| `description` | `string` | 否 | 配置说明 |

### 10.6 UpdateSystemConfigRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `configValue` | `string` | 是 | 新配置值 |
| `description` | `string` | 否 | 配置说明 |

### 10.7 UpdateConfigStatusRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `status` | `int` | 是 | `1` 启用，`0` 禁用 |

### 10.8 AdminMenuVO，V1 可选

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 菜单 ID |
| `parentId` | `long` | 是 | 父菜单 ID |
| `menuName` | `string` | 是 | 菜单名称 |
| `path` | `string` | 否 | 前端路由路径，对应 `menu_path` |
| `component` | `string` | 否 | 前端组件路径 |
| `icon` | `string` | 否 | 图标 |
| `sort` | `int` | 是 | 排序，对应 `sort_order` |
| `status` | `int` | 是 | 状态 |
| `children` | `array<AdminMenuVO>` | 否 | 子菜单 |

### 10.9 AdminRoleVO，V1 可选

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 角色 ID |
| `roleCode` | `string` | 是 | 角色编码 |
| `roleName` | `string` | 是 | 角色名称 |
| `description` | `string` | 否 | 角色说明 |
| `status` | `int` | 是 | 状态 |

### 10.10 InnerUserStatisticsVO，V1 可选

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userCount` | `long` | 是 | 用户总数 |
| `enabledUserCount` | `long` | 否 | 启用用户数 |
| `disabledUserCount` | `long` | 否 | 禁用用户数 |
| `adminCount` | `long` | 否 | 管理员数量 |
| `todayRegisterCount` | `long` | 否 | 今日注册用户数 |

### 10.11 InnerQuestionStatisticsVO，V1 可选

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `questionCount` | `long` | 是 | 题目总数 |
| `enabledQuestionCount` | `long` | 否 | 启用题目数 |
| `categoryCount` | `long` | 否 | 分类数量 |
| `tagCount` | `long` | 否 | 标签数量 |
| `questionGroupCount` | `long` | 否 | 问题组数量 |

### 10.12 InnerResumeStatisticsVO，V1 可选

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `resumeCount` | `long` | 是 | 简历总数 |
| `projectCount` | `long` | 否 | 项目经历总数 |
| `todayResumeCount` | `long` | 否 | 今日新增简历数 |

### 10.13 InnerInterviewStatisticsVO，V1 可选

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `interviewCount` | `long` | 是 | 面试总数 |
| `completedInterviewCount` | `long` | 否 | 已完成面试数 |
| `inProgressInterviewCount` | `long` | 否 | 进行中面试数 |
| `todayInterviewCount` | `long` | 否 | 今日新增面试数 |
| `reportGeneratedCount` | `long` | 否 | 已生成报告数 |
| `reportFailedCount` | `long` | 否 | 报告生成失败数 |

### 10.14 InnerAiStatisticsVO，V1 可选

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `aiCallCount` | `long` | 是 | AI 调用总数 |
| `aiCallFailedCount` | `long` | 否 | AI 调用失败总数 |
| `todayAiCallCount` | `long` | 否 | 今日 AI 调用数 |
| `promptCount` | `long` | 否 | Prompt 模板总数 |
| `enabledPromptCount` | `long` | 否 | 启用 Prompt 模板数 |

---

## 11. 参数校验规则

| 参数 | 校验规则 |
|---|---|
| `pageNo` | 最小 1 |
| `pageSize` | 最大 100 |
| `configKey` | 不能为空；建议只允许小写字母、数字、点号、下划线、中划线；最大 128 |
| `configName` | 最大 128，对应数据库 `config_name` |
| `configValue` | 长度按数据库字段决定，当前 `system_config.config_value` 最大 1000 |
| `configType` | 必须是 `STRING` / `NUMBER` / `BOOLEAN` / `JSON` |
| `JSON` 类型配置 | `configValue` 必须是合法 JSON |
| `BOOLEAN` 类型配置 | `configValue` 必须是 `true` / `false` |
| `NUMBER` 类型配置 | `configValue` 必须能转为数字 |
| `status` | 只能是 `0` / `1` |
| `editable` | 只能是 `0` / `1` |
| `keyword` | 最大 100 |
| `description` | 最大 500，对应数据库 `description` |

补充规则：

1. 修改配置时必须先根据 `configKey` 查询配置记录。
2. 配置不存在、已逻辑删除、不可编辑时，不允许修改。
3. 敏感配置不建议通过普通接口修改明文值。
4. 禁用配置 V1 推荐不允许修改。

---

## 12. 错误码设计

系统模块错误码范围：`47000-47999`。

| code | message | 说明 |
|---:|---|---|
| `47000` | 配置不存在 | 根据 `configKey` 查询不到配置 |
| `47001` | 配置不可编辑 | `editable=0` |
| `47002` | 配置值类型错误 | `configValue` 不符合 `configType` |
| `47003` | 配置状态非法 | `status` 不是 `0` 或 `1` |
| `47004` | 配置 Key 非法 | `configKey` 格式不合法 |
| `47005` | 敏感配置不允许通过接口修改 | API Key 等敏感配置禁止普通接口修改 |
| `47006` | 菜单不存在 | V1 可选 |
| `47007` | 角色不存在 | V1 可选，角色归 user-service 管理 |
| `47008` | 首页统计获取失败 | 调用内部统计接口失败或统计异常 |
| `47009` | 配置已禁用 | 禁用配置不允许修改或读取时返回 |
| `47010` | 核心配置不允许禁用 | 内置关键配置禁止禁用 |

复用通用错误码：

| code | message | 说明 |
|---:|---|---|
| `40001` | 参数校验失败 | 参数格式或必填校验失败 |
| `41000` | 未登录 | 未携带有效登录态 |
| `41003` | 无访问权限 | 非管理员访问管理端接口 |
| `50000` | 系统内部错误 | 未预期系统异常 |

---

## 13. 数据库表关系

本模块涉及：

| 表名 | 归属服务 | V1 说明 |
|---|---|---|
| `system_config` | system-service | 系统配置表，V1 必需 |
| `sys_menu` | system-service | V1 可选，动态菜单基础表 |
| `sys_permission` | system-service | V1 可选，权限标识基础表 |
| `sys_role` | user-service | 角色数据由 user-service 管理，system-service 不直接维护 |

说明：

1. system-service 是 `system_config` 的数据归属服务。
2. user-service 是 `sys_user`、`sys_role`、`sys_user_role` 的数据归属服务。
3. system-service V1 不直接维护角色关系。
4. `sys_menu` 和 `sys_permission` 如果存在，V1 可暂不做完整权限体系，只作为后续扩展基础。
5. AI Prompt 模板由 ai-service 管理，不归 system-service 管理。
6. AI 调用日志由 ai-service 管理，不归 system-service 管理。
7. V1 不设计 `sys_role_menu`、`sys_role_permission`，不做复杂 RBAC。

表关系说明：

```text
system-service
  └── system_config
  └── sys_menu，可选
  └── sys_permission，可选

user-service
  └── sys_user
        └── sys_user_role
              └── sys_role
```

---

## 14. 前端页面对应关系

| 页面 | 相关接口 | 说明 |
|---|---|---|
| 管理端首页 | `GET /admin/system/overview` | 展示简化统计 |
| 系统配置页 | `GET /admin/configs`、`POST /admin/configs`、`GET /admin/configs/{key}`、`PUT /admin/configs/{key}`、`PUT /admin/configs/{key}/status`、`DELETE /admin/configs/{key}` | 查看、编辑、启用禁用、维护非核心配置 |
| 管理端布局菜单 | `GET /admin/menus` | V1 可选，前端也可先使用静态路由 |
| 角色选择下拉框 | `GET /admin/roles` | 推荐由 user-service 提供，不归 system-service 直接维护 |

说明：

1. V1 管理端首页只做简化统计。
2. V1 系统配置页只做基础配置查看和修改。
3. V1 不做复杂权限管理页面。
4. V1 不做操作日志页面。
5. V1 不做通知中心页面。
6. 系统配置接口已包含列表、新增、详情、修改、状态修改和删除；新增和删除仅建议用于非核心配置。

---

## 15. 注意事项

1. system-service V1 保持轻量。
2. system-service 主要负责 `system_config` 和管理端首页统计。
3. user-service 负责 `sys_user`、`sys_role`、`sys_user_role`。
4. Prompt 模板和 AI 日志归 ai-service。
5. 管理端接口必须具备 `ADMIN` 角色。
6. 敏感配置不能明文返回。
7. API Key 不建议通过系统配置接口管理明文。
8. 管理端首页统计可以逐步补齐，不影响核心闭环。
9. V1 不做复杂 RBAC。
10. V1 不做操作日志。
11. V1 不做通知中心。
12. 服务间调用统一使用 Spring Cloud OpenFeign。
13. `/inner/**` 不对前端暴露。
14. 角色列表接口推荐由 user-service 提供，system-service 不直接管理角色数据。
