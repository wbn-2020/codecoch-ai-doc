# CodeCoachAI V1 认证用户接口设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 认证用户接口设计 |
| 项目阶段 | V1 |
| 最高依据 | 接口设计/CodeCoachAI_V1_接口设计总览.md |
| 参考文档 | PRD/CodeCoachAI_PRD_V1_微服务分层架构版.md、数据库设计/CodeCoachAI_V1_数据库设计总览.md、数据库设计/CodeCoachAI_V1_用户权限表设计.md |
| 涉及服务 | codecoachai-auth、codecoachai-user |
| 相关服务 | codecoachai-gateway、codecoachai-question、codecoachai-resume、codecoachai-interview |

本文档用于详细设计 CodeCoachAI V1 阶段认证与用户相关接口，包括：

1. auth-service 对外认证接口。
2. user-service 用户端接口。
3. user-service 管理端用户接口。
4. user-service 提供给 auth-service 的内部 Feign 接口。

本文档只设计 V1 接口，不设计复杂 RBAC、菜单权限、OAuth2、第三方登录、短信登录、邮箱验证码登录等后续能力。

V1 推荐使用 Sa-Token 完成登录认证，也可兼容 JWT 思路；本文档以 Sa-Token 风格为主。

---

## 2. 模块职责

## 2.1 auth-service 职责

auth-service 负责认证入口和登录态处理。

职责包括：

1. 用户注册入口。
2. 用户登录。
3. 用户退出。
4. 当前登录用户查询。
5. Token 刷新，V1 可选。
6. 登录态和 Token 处理。
7. 通过 user-service 的 `/inner/**` 接口完成用户查询、用户创建和角色查询。

auth-service 不负责：

1. 不直接操作 `sys_user`、`sys_role`、`sys_user_role`。
2. 不维护用户资料。
3. 不管理角色关系。
4. 不设计复杂菜单权限。

## 2.2 user-service 职责

user-service 负责用户数据归属和用户资料管理。

职责包括：

1. 用户基础信息管理。
2. 用户角色关系管理。
3. 当前用户资料查询和修改。
4. 修改密码。
5. 用户学习概览。
6. 管理端用户列表。
7. 管理端用户启用 / 禁用。
8. 管理端角色列表查询。
9. 给 auth-service 提供内部 Feign 用户查询能力。

user-service 管理以下数据表：

```text
sys_user
sys_role
sys_user_role
```

user-service 不负责：

1. 不生成 Token。
2. 不维护登录态。
3. 不管理题库、简历、面试、AI 日志数据。

---

## 3. 统一约定

## 3.1 统一响应结构

所有接口返回统一响应对象。

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
| `code` | `int` | 业务状态码，0 表示成功 |
| `message` | `string` | 响应提示 |
| `data` | `object` | 响应数据 |
| `traceId` | `string` | 链路追踪 ID，V1 可选 |

## 3.2 分页结构

分页请求参数：

| 字段名 | 类型 | 是否必填 | 默认值 | 说明 |
|---|---|---:|---|---|
| `pageNo` | `int` | 否 | `1` | 当前页码 |
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

## 3.3 Token 请求头格式

登录成功后，前端在请求头中携带 Token。

```text
Authorization: Bearer {token}
```

V1 统一使用 `Authorization: Bearer {token}`。

使用 Sa-Token 时，建议配置 Sa-Token 读取 `Authorization` 请求头并兼容 `Bearer ` 前缀，避免前端使用 `satoken` 请求头造成联调不一致。

## 3.4 当前登录用户上下文

后端从登录态中获取当前用户上下文，不信任前端传入的 userId。

当前用户上下文至少包含：

| 字段 | 说明 |
|---|---|
| `userId` | 当前登录用户 ID |
| `username` | 登录账号 |
| `roles` | 角色编码列表，例如 `USER`、`ADMIN` |

## 3.5 普通用户权限

普通用户只能：

1. 查询和修改自己的用户资料。
2. 修改自己的密码。
3. 查看自己的学习概览。
4. 访问用户端功能。

普通用户不能：

1. 访问 `/admin/**` 接口。
2. 修改自己的角色和状态。
3. 查看其他用户资料。

## 3.6 管理员权限

管理端接口必须满足：

1. 已登录。
2. 拥有 `ADMIN` 角色。

V1 管理员可以：

1. 查询用户列表。
2. 启用 / 禁用用户。

V1 不设计多管理员分级、菜单权限、按钮权限和数据权限。

## 3.7 数据归属校验

用户端接口必须使用当前登录用户 ID 作为数据归属条件。

示例：

1. `GET /users/profile` 只能查询当前登录用户。
2. `PUT /users/profile` 只能修改当前登录用户。
3. `PUT /users/password` 只能修改当前登录用户密码。

## 3.8 密码加密存储原则

密码安全要求：

1. 密码必须加密存储。
2. 不允许明文存储密码。
3. 不允许在日志中打印密码。
4. 不允许在响应中返回密码哈希。
5. 推荐使用 BCrypt 或 Sa-Token 生态中合适的密码哈希方案。

密码加密责任：

1. 注册时 auth-service 校验明文密码和确认密码，并使用统一 `PasswordEncoder` 生成 `passwordHash`，再调用 user-service 的 `POST /inner/users` 创建用户。
2. 修改密码时 user-service 校验 `oldPassword`，并生成新的 `passwordHash`。
3. `PasswordEncoder` 建议放在 common-security 或 common-core，供 auth-service 和 user-service 复用。

## 3.9 敏感字段不返回原则

以下字段不得返回前端：

1. `password`。
2. `passwordHash`。
3. Token 原始存储信息。
4. 后端内部权限细节。
5. API Key。

## 3.10 枚举说明

### UserStatus

| 枚举 | 值 | 说明 |
|---|---:|---|
| `ENABLED` | `1` | 启用 |
| `DISABLED` | `0` | 禁用 |

### RoleCode

| 枚举 | 说明 |
|---|---|
| `USER` | 普通用户 |
| `ADMIN` | 管理员 |

## 3.11 参数校验规则

| 字段 | 校验规则 |
|---|---|
| `username` | 长度 4-32，只允许字母、数字、下划线 |
| `password` | 长度 6-32 |
| `newPassword` | 长度 6-32 |
| `confirmPassword` | 长度 6-32 |
| `nickname` | 最大 50 |
| `email` | 符合邮箱格式，最大 100 |
| `avatarUrl` | 最大 255 |
| `pageNo` | 最小 1 |
| `pageSize` | 最大 100 |
| `status` | 只能是 0 或 1 |

---

## 4. auth-service 接口设计

## 4.1 POST /auth/register

### 接口说明

用户注册接口，用于创建普通用户账号。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/auth/register` |
| Method | `POST` |
| 是否需要登录 | 否 |
| 是否管理员权限 | 否 |
| Content-Type | `application/json` |

### 请求参数

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `username` | `string` | 是 | 登录账号，唯一 |
| `password` | `string` | 是 | 密码 |
| `confirmPassword` | `string` | 是 | 确认密码 |
| `nickname` | `string` | 否 | 用户昵称 |
| `email` | `string` | 否 | 邮箱 |

### 请求示例

```json
{
  "username": "java_user",
  "password": "Password123",
  "confirmPassword": "Password123",
  "nickname": "Java 求职者",
  "email": "user@example.com"
}
```

### 响应参数

| 字段名 | 类型 | 说明 |
|---|---|---|
| `userId` | `long` | 用户 ID |
| `username` | `string` | 登录账号 |
| `nickname` | `string` | 用户昵称 |

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "userId": 10001,
    "username": "java_user",
    "nickname": "Java 求职者"
  }
}
```

### 业务规则

1. 用户名必须唯一。
2. 密码和确认密码必须一致。
3. auth-service 使用统一 `PasswordEncoder` 生成 `passwordHash`。
4. auth-service 通过 user-service 的 `POST /inner/users` 创建用户。
5. 注册成功后默认分配 `USER` 角色。
6. 注册成功后用户默认启用。

### 错误码

| code | message | 场景 |
|---:|---|---|
| `40001` | 参数校验失败 | 用户名或密码为空 |
| `42001` | 用户名已存在 | username 重复 |
| `42006` | 两次密码不一致 | password 与 confirmPassword 不一致 |

---

## 4.2 POST /auth/login

### 接口说明

用户登录接口，普通用户和管理员共用。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/auth/login` |
| Method | `POST` |
| 是否需要登录 | 否 |
| 是否管理员权限 | 否 |
| Content-Type | `application/json` |

### 请求参数

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `username` | `string` | 是 | 登录账号 |
| `password` | `string` | 是 | 密码 |

### 请求示例

```json
{
  "username": "java_user",
  "password": "Password123"
}
```

### 响应参数

| 字段名 | 类型 | 说明 |
|---|---|---|
| `token` | `string` | 登录 Token |
| `tokenName` | `string` | Token 请求头名称 |
| `expireTime` | `string` | 过期时间，格式 `yyyy-MM-dd HH:mm:ss` |
| `userInfo` | `CurrentUserVO` | 当前用户基础信息 |
| `roles` | `array<string>` | 角色编码列表 |

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "token": "token-value",
    "tokenName": "Authorization",
    "expireTime": "2026-05-13 18:00:00",
    "userInfo": {
      "id": 10001,
      "username": "java_user",
      "nickname": "Java 求职者",
      "avatarUrl": null,
      "email": "user@example.com",
      "roles": ["USER"]
    },
    "roles": ["USER"]
  }
}
```

### 业务规则

1. auth-service 调用 user-service 的 `GET /inner/users/by-username` 查询用户认证信息。
2. 校验用户是否存在。
3. 校验密码是否正确。
4. 校验用户状态是否启用。
5. 登录成功后生成 Token。
6. 登录成功后返回用户基础信息和角色标识。
7. 登录失败不返回密码、密码哈希和具体敏感细节。
8. 内部错误码可以区分用户不存在和密码错误，但前端展示文案建议统一为“用户名或密码错误”。

### 错误码

| code | message | 场景 |
|---:|---|---|
| `40001` | 参数校验失败 | 用户名或密码为空 |
| `42002` | 用户不存在 | username 不存在 |
| `42003` | 密码错误 | 密码校验失败 |
| `42004` | 账号已禁用 | 用户 status 为禁用 |

---

## 4.3 POST /auth/logout

### 接口说明

当前用户退出登录。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/auth/logout` |
| Method | `POST` |
| 是否需要登录 | 是 |
| 是否管理员权限 | 否 |

### 请求参数

无。

### 响应参数

可为空。

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": null
}
```

### 业务规则

1. 清理当前登录态。
2. Sa-Token 实现时调用当前会话 logout 能力。
3. 退出后当前 Token 不应继续访问受保护接口。

### 错误码

| code | message | 场景 |
|---:|---|---|
| `41000` | 未登录 | 未携带有效 Token |
| `41001` | Token 无效或已过期 | Token 不可用 |

---

## 4.4 GET /auth/current-user

### 接口说明

获取当前登录用户信息。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/auth/current-user` |
| Method | `GET` |
| 是否需要登录 | 是 |
| 是否管理员权限 | 否 |

### 请求参数

无。

### 响应参数

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 用户 ID |
| `username` | `string` | 登录账号 |
| `nickname` | `string` | 昵称 |
| `avatarUrl` | `string` | 头像 URL |
| `email` | `string` | 邮箱 |
| `roles` | `array<string>` | 角色编码列表 |

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "id": 10001,
    "username": "java_user",
    "nickname": "Java 求职者",
    "avatarUrl": null,
    "email": "user@example.com",
    "roles": ["USER"]
  }
}
```

### 业务规则

1. 根据当前 Token 获取登录用户 ID。
2. 可从登录上下文读取基础信息，也可调用 user-service 内部接口刷新基础信息。
3. 不返回 `password`、`passwordHash` 等敏感字段。

### 错误码

| code | message | 场景 |
|---:|---|---|
| `41000` | 未登录 | 未登录访问 |
| `41001` | Token 无效或已过期 | Token 不可用 |
| `42002` | 用户不存在 | 用户已被删除或不存在 |

---

## 4.5 POST /auth/refresh-token

### 接口说明

Token 刷新接口，V1 暂不实现。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/auth/refresh-token` |
| Method | `POST` |
| 是否需要登录 | 是 |
| 是否管理员权限 | 否 |

### V1 实现说明

1. `POST /auth/refresh-token` 不纳入 V1 前后端联调清单。
2. 如使用 Sa-Token，可通过会话有效期和自动续期机制处理。
3. V1 前端不依赖该接口完成核心流程。
4. 后续如需要显式刷新 Token，再补充接口设计。

### 业务规则

V1 暂不提供实际业务能力。

### 错误码

| code | message | 场景 |
|---:|---|---|
| `40000` | 接口暂未实现 | V1 调用该接口 |

---

## 5. user-service 用户端接口设计

## 5.1 GET /users/profile

### 接口说明

查询当前登录用户资料。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/users/profile` |
| Method | `GET` |
| 是否需要登录 | 是 |
| 是否管理员权限 | 否 |

### 请求参数

无。

### 响应参数

返回 `UserProfileVO`。

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 用户 ID |
| `username` | `string` | 登录账号 |
| `nickname` | `string` | 昵称 |
| `avatarUrl` | `string` | 头像 URL |
| `email` | `string` | 邮箱 |
| `status` | `int` | 用户状态 |
| `roles` | `array<string>` | 角色编码列表 |
| `createdAt` | `string` | 创建时间 |

### 业务规则

1. 使用当前登录用户 ID 查询。
2. 不允许通过请求参数传入 userId 查询其他用户。
3. 不返回密码哈希。

### 错误码

| code | message | 场景 |
|---:|---|---|
| `41000` | 未登录 | 未登录访问 |
| `42002` | 用户不存在 | 当前用户不存在 |

---

## 5.2 PUT /users/profile

### 接口说明

修改当前登录用户资料。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/users/profile` |
| Method | `PUT` |
| 是否需要登录 | 是 |
| 是否管理员权限 | 否 |
| Content-Type | `application/json` |

### 请求参数

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `nickname` | `string` | 否 | 昵称 |
| `avatarUrl` | `string` | 否 | 头像 URL |
| `email` | `string` | 否 | 邮箱 |

### 业务规则

1. 只能修改当前登录用户资料。
2. 不允许修改 `username`。
3. 不允许修改 `role`。
4. 不允许修改 `status`。
5. 如果邮箱需要唯一性，V1 可根据实现决定是否校验。

### 响应参数

返回更新后的 `UserProfileVO`。

### 错误码

| code | message | 场景 |
|---:|---|---|
| `41000` | 未登录 | 未登录访问 |
| `40001` | 参数校验失败 | 邮箱格式错误等 |
| `42002` | 用户不存在 | 当前用户不存在 |

---

## 5.3 PUT /users/password

### 接口说明

修改当前登录用户密码。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/users/password` |
| Method | `PUT` |
| 是否需要登录 | 是 |
| 是否管理员权限 | 否 |
| Content-Type | `application/json` |

### 请求参数

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `oldPassword` | `string` | 是 | 原密码 |
| `newPassword` | `string` | 是 | 新密码 |
| `confirmPassword` | `string` | 是 | 确认新密码 |

### 业务规则

1. 校验旧密码是否正确。
2. 新密码和确认密码必须一致。
3. user-service 使用统一 `PasswordEncoder` 生成新的 `passwordHash` 并保存。
4. 修改密码后，V1 建议让当前用户重新登录。
5. 如果采用 Sa-Token，可选择修改密码后注销当前会话。

### 响应参数

可为空。

### 错误码

| code | message | 场景 |
|---:|---|---|
| `41000` | 未登录 | 未登录访问 |
| `42007` | 原密码错误 | oldPassword 校验失败 |
| `42006` | 两次密码不一致 | newPassword 与 confirmPassword 不一致 |
| `42002` | 用户不存在 | 当前用户不存在 |

---

## 5.4 GET /users/overview

### 接口说明

查询当前用户学习概览。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/users/overview` |
| Method | `GET` |
| 是否需要登录 | 是 |
| 是否管理员权限 | 否 |

### 响应参数

返回 `UserOverviewVO`。

| 字段名 | 类型 | 说明 |
|---|---|---|
| `resumeCount` | `int` | 简历数量 |
| `interviewCount` | `int` | 面试总数 |
| `completedInterviewCount` | `int` | 已完成面试数 |
| `questionAnsweredCount` | `int` | 已答题数 |
| `wrongQuestionCount` | `int` | 错题数 |
| `favoriteQuestionCount` | `int` | 收藏题数 |

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "resumeCount": 1,
    "interviewCount": 3,
    "completedInterviewCount": 2,
    "questionAnsweredCount": 24,
    "wrongQuestionCount": 5,
    "favoriteQuestionCount": 8
  }
}
```

### 业务规则

1. 只统计当前登录用户的数据。
2. 该接口保留为首页工作台聚合接口，接口结构保持不变。
3. V1 有两种实现方式：

方案 A：简化实现。

1. user-service 只返回 user-service 可直接获得的基础用户数据。
2. `resumeCount`、`interviewCount`、`completedInterviewCount`、`questionAnsweredCount`、`wrongQuestionCount`、`favoriteQuestionCount` 等跨服务统计字段可以先返回 `0` 或暂时不展示。
3. 该方案不阻塞 V1 核心面试闭环。

方案 B：聚合实现。

1. user-service 通过 Spring Cloud OpenFeign 调用 question-service、resume-service、interview-service 的内部统计接口聚合数据。
2. question-service 提供当前用户刷题统计、错题统计、收藏统计。
3. resume-service 提供当前用户简历数量和默认简历信息。
4. interview-service 提供当前用户面试总数、已完成面试数、平均分、最近一次面试等统计。
5. 需要在 `CodeCoachAI_V1_OpenFeign内部接口总表.md` 中补充对应内部接口。

推荐：文档采用方案 B 作为完整设计，但 V1 第一阶段可先按方案 A 简化返回。

### 错误码

| code | message | 场景 |
|---:|---|---|
| `41000` | 未登录 | 未登录访问 |
| `42002` | 用户不存在 | 当前用户不存在 |

---

## 6. user-service 管理端接口设计

## 6.1 GET /admin/users

### 接口说明

管理端用户分页列表。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/admin/users` |
| Method | `GET` |
| 是否需要登录 | 是 |
| 是否管理员权限 | 是 |

### 查询参数

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 用户名、昵称、邮箱关键字 |
| `status` | `int` | 否 | 用户状态：1 正常，0 禁用 |
| `roleCode` | `string` | 否 | 角色编码：USER / ADMIN |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

### 响应参数

分页返回 `AdminUserPageVO`。

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 用户 ID |
| `username` | `string` | 登录账号 |
| `nickname` | `string` | 昵称 |
| `avatarUrl` | `string` | 头像 URL |
| `email` | `string` | 邮箱 |
| `status` | `int` | 用户状态 |
| `statusName` | `string` | 状态名称 |
| `roles` | `array<string>` | 角色编码 |
| `createdAt` | `string` | 创建时间 |

### 业务规则

1. 仅管理员可访问。
2. 不返回密码。
3. 支持按关键字、状态、角色筛选。
4. V1 用户管理只做基础列表，不做复杂数据看板。

### 错误码

| code | message | 场景 |
|---:|---|---|
| `41000` | 未登录 | 未登录访问 |
| `41003` | 无访问权限 | 非管理员访问 |

---

## 6.2 PUT /admin/users/{id}/status

### 接口说明

管理员启用或禁用用户。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/admin/users/{id}/status` |
| Method | `PUT` |
| 是否需要登录 | 是 |
| 是否管理员权限 | 是 |
| Content-Type | `application/json` |

### 路径参数

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 用户 ID |

### 请求参数

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `status` | `int` | 是 | 用户状态：1 启用，0 禁用 |

### 业务规则

1. 仅管理员可访问。
2. 管理员不能禁用自己。
3. 禁用后用户不能登录。
4. 已登录用户是否立即失效，V1 可选处理。
5. 如果启用已禁用用户，用户可重新登录。

### 响应参数

可为空。

### 错误码

| code | message | 场景 |
|---:|---|---|
| `41000` | 未登录 | 未登录访问 |
| `41003` | 无访问权限 | 非管理员访问 |
| `42002` | 用户不存在 | 用户 ID 不存在 |
| `42008` | 不能禁用自己 | 管理员操作自己的状态为禁用 |

---

## 6.3 GET /admin/roles

### 接口说明

管理端查询角色列表。

V1 只需要 `USER`、`ADMIN` 两类基础角色，不设计复杂 RBAC、菜单权限、按钮权限和数据权限。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/admin/roles` |
| Method | `GET` |
| 是否需要登录 | 是 |
| 是否管理员权限 | 是 |

### 查询参数

无。V1 不设计复杂角色筛选。

### 响应参数

返回 `array<AdminRoleVO>`。

| 字段名 | 类型 | 说明 |
|---|---|---|
| `roleId` | `long` | 角色 ID |
| `roleCode` | `string` | 角色编码：`USER` / `ADMIN` |
| `roleName` | `string` | 角色名称 |
| `status` | `int` | 角色状态：`1` 启用，`0` 禁用 |

### 业务规则

1. 仅管理员可访问。
2. 角色数据归 user-service 管理。
3. 只返回未删除角色。
4. V1 只需要 `USER`、`ADMIN` 两类角色。
5. 不设计角色新增、修改、删除接口。
6. 不设计角色菜单授权、按钮权限、数据权限。

### 错误码

| code | message | 场景 |
|---:|---|---|
| `41000` | 未登录 | 未登录访问 |
| `41003` | 无访问权限 | 非管理员访问 |

---

## 7. user-service 内部 Feign 接口设计

## 7.1 内部接口安全约束

`/inner/**` 接口只允许服务内部 Feign 调用。

约束：

1. `/inner/**` 不经过 Gateway 对外暴露。
2. 前端不能访问 `/inner/**`。
3. 需要内部调用鉴权或内网隔离。
4. 如果内部调用经过 Gateway，需要内部调用标识或服务间鉴权。
5. 内部接口仍然不能随意绕过业务规则。
6. 内部接口返回敏感字段时，仅限必要调用方使用。

## 7.2 GET /inner/users/by-username

### 接口说明

auth-service 登录时调用，用于查询用户认证信息。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/inner/users/by-username` |
| Method | `GET` |
| 调用方 | auth-service |
| 是否前端可访问 | 否 |

### 请求参数

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `username` | `string` | 是 | 登录账号 |

### 响应参数

返回 `InnerUserAuthVO`。

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 用户 ID |
| `username` | `string` | 登录账号 |
| `passwordHash` | `string` | 密码哈希，仅内部使用 |
| `nickname` | `string` | 昵称 |
| `avatarUrl` | `string` | 头像 URL |
| `email` | `string` | 邮箱 |
| `status` | `int` | 用户状态 |
| `roles` | `array<string>` | 角色编码列表 |

### 业务规则

1. 仅 auth-service 登录流程调用。
2. 返回 `passwordHash` 仅用于密码校验。
3. 不允许前端访问。
4. 调用方不得记录 `passwordHash` 到日志。

---

## 7.3 POST /inner/users

### 接口说明

auth-service 注册时调用，用于创建普通用户。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/inner/users` |
| Method | `POST` |
| 调用方 | auth-service |
| 是否前端可访问 | 否 |
| Content-Type | `application/json` |

### 请求参数

请求体为 `InnerCreateUserRequest`。

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `username` | `string` | 是 | 登录账号 |
| `passwordHash` | `string` | 是 | 已加密密码 |
| `nickname` | `string` | 否 | 昵称 |
| `email` | `string` | 否 | 邮箱 |

### 响应参数

| 字段名 | 类型 | 说明 |
|---|---|---|
| `userId` | `long` | 用户 ID |
| `username` | `string` | 登录账号 |
| `nickname` | `string` | 昵称 |

### 业务规则

1. 默认分配 `USER` 角色。
2. 默认启用状态。
3. 用户名必须唯一。
4. 创建用户和绑定角色应在 user-service 本地事务中完成。

---

## 7.4 GET /inner/users/{id}/roles

### 接口说明

auth-service 查询用户角色。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/inner/users/{id}/roles` |
| Method | `GET` |
| 调用方 | auth-service |
| 是否前端可访问 | 否 |

### 路径参数

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 用户 ID |

### 响应参数

返回 `InnerUserRoleVO`。

| 字段名 | 类型 | 说明 |
|---|---|---|
| `userId` | `long` | 用户 ID |
| `roles` | `array<string>` | 角色编码列表 |

### 业务规则

1. 仅内部调用。
2. 只返回启用状态的角色。
3. 用户不存在时返回用户不存在错误。

---

## 7.5 GET /inner/users/{id}

### 接口说明

内部查询用户基础信息，V1 可选。

### 请求信息

| 项目 | 内容 |
|---|---|
| URL | `/inner/users/{id}` |
| Method | `GET` |
| 调用方 | auth-service 或其他内部服务 |
| 是否前端可访问 | 否 |

### 响应参数

| 字段名 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 用户 ID |
| `username` | `string` | 登录账号 |
| `nickname` | `string` | 昵称 |
| `avatarUrl` | `string` | 头像 URL |
| `email` | `string` | 邮箱 |
| `status` | `int` | 用户状态 |
| `roles` | `array<string>` | 角色编码列表 |

### 业务规则

1. 不返回 `passwordHash`。
2. 仅内部调用。
3. 仍需遵守服务间调用安全约束。

---

## 8. DTO / VO 设计

## 8.1 RegisterRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `username` | `string` | 是 | 登录账号 |
| `password` | `string` | 是 | 密码 |
| `confirmPassword` | `string` | 是 | 确认密码 |
| `nickname` | `string` | 否 | 昵称 |
| `email` | `string` | 否 | 邮箱 |

## 8.2 LoginRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `username` | `string` | 是 | 登录账号 |
| `password` | `string` | 是 | 密码 |

## 8.3 LoginResponse

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `token` | `string` | 是 | 登录 Token |
| `tokenName` | `string` | 是 | Token 请求头名称 |
| `expireTime` | `string` | 否 | 过期时间 |
| `userInfo` | `CurrentUserVO` | 是 | 当前用户信息 |
| `roles` | `array<string>` | 是 | 角色编码列表 |

## 8.4 CurrentUserVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 用户 ID |
| `username` | `string` | 是 | 登录账号 |
| `nickname` | `string` | 否 | 昵称 |
| `avatarUrl` | `string` | 否 | 头像 URL |
| `email` | `string` | 否 | 邮箱 |
| `roles` | `array<string>` | 是 | 角色编码列表 |

## 8.5 UserProfileVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 用户 ID |
| `username` | `string` | 是 | 登录账号 |
| `nickname` | `string` | 否 | 昵称 |
| `avatarUrl` | `string` | 否 | 头像 URL |
| `email` | `string` | 否 | 邮箱 |
| `status` | `int` | 是 | 用户状态 |
| `roles` | `array<string>` | 是 | 角色编码列表 |
| `createdAt` | `string` | 是 | 创建时间 |

## 8.6 UpdateUserProfileRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `nickname` | `string` | 否 | 昵称 |
| `avatarUrl` | `string` | 否 | 头像 URL |
| `email` | `string` | 否 | 邮箱 |

## 8.7 UpdatePasswordRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `oldPassword` | `string` | 是 | 原密码 |
| `newPassword` | `string` | 是 | 新密码 |
| `confirmPassword` | `string` | 是 | 确认密码 |

## 8.8 UserOverviewVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `resumeCount` | `int` | 是 | 简历数量 |
| `interviewCount` | `int` | 是 | 面试总数 |
| `completedInterviewCount` | `int` | 是 | 已完成面试数 |
| `questionAnsweredCount` | `int` | 是 | 已答题数 |
| `wrongQuestionCount` | `int` | 是 | 错题数 |
| `favoriteQuestionCount` | `int` | 是 | 收藏题数 |

## 8.9 AdminUserQueryRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 用户名、昵称、邮箱关键字 |
| `status` | `int` | 否 | 用户状态 |
| `roleCode` | `string` | 否 | 角色编码 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

## 8.10 AdminUserPageVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 用户 ID |
| `username` | `string` | 是 | 登录账号 |
| `nickname` | `string` | 否 | 昵称 |
| `avatarUrl` | `string` | 否 | 头像 URL |
| `email` | `string` | 否 | 邮箱 |
| `status` | `int` | 是 | 用户状态 |
| `statusName` | `string` | 是 | 状态名称 |
| `roles` | `array<string>` | 是 | 角色编码列表 |
| `createdAt` | `string` | 是 | 创建时间 |

## 8.11 UpdateUserStatusRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `status` | `int` | 是 | 用户状态：1 启用，0 禁用 |

## 8.12 AdminRoleVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `roleId` | `long` | 是 | 角色 ID |
| `roleCode` | `string` | 是 | 角色编码：`USER` / `ADMIN` |
| `roleName` | `string` | 是 | 角色名称 |
| `status` | `int` | 是 | 角色状态：1 启用，0 禁用 |

## 8.13 InnerUserAuthVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 用户 ID |
| `username` | `string` | 是 | 登录账号 |
| `passwordHash` | `string` | 是 | 密码哈希，仅内部使用 |
| `nickname` | `string` | 否 | 昵称 |
| `avatarUrl` | `string` | 否 | 头像 URL |
| `email` | `string` | 否 | 邮箱 |
| `status` | `int` | 是 | 用户状态 |
| `roles` | `array<string>` | 是 | 角色编码列表 |

## 8.14 InnerCreateUserRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `username` | `string` | 是 | 登录账号 |
| `passwordHash` | `string` | 是 | 已加密密码 |
| `nickname` | `string` | 否 | 昵称 |
| `email` | `string` | 否 | 邮箱 |

## 8.15 InnerUserRoleVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `userId` | `long` | 是 | 用户 ID |
| `roles` | `array<string>` | 是 | 角色编码列表 |

---

## 9. 错误码设计

## 9.1 认证权限错误码

| code | message | 说明 |
|---:|---|---|
| `41000` | 未登录 | 未携带 Token 或未登录 |
| `41001` | Token 无效或已过期 | Token 不存在、无效或过期 |
| `41003` | 无访问权限 | 非管理员访问管理端接口 |

## 9.2 用户模块错误码

| code | message | 说明 |
|---:|---|---|
| `42001` | 用户名已存在 | 注册时 username 重复 |
| `42002` | 用户不存在 | 用户 ID 或 username 不存在 |
| `42003` | 密码错误 | 登录密码错误 |
| `42004` | 账号已禁用 | 用户 status 为禁用 |
| `42006` | 两次密码不一致 | 确认密码不一致 |
| `42007` | 原密码错误 | 修改密码时旧密码错误 |
| `42008` | 不能禁用自己 | 管理员不能禁用自己的账号 |

---

## 10. 数据库表关系

本模块涉及以下表：

```text
sys_user
sys_role
sys_user_role
```

表关系：

```text
sys_user
  └── sys_user_role
        └── sys_role
```

数据归属说明：

1. `sys_user`、`sys_role`、`sys_user_role` 归 user-service 管理。
2. auth-service 不直接管理这些表。
3. auth-service 通过 user-service 的 `/inner/**` 接口完成登录查询、注册创建用户、角色查询。
4. system-service V1 不直接维护角色数据。

---

## 11. 前端页面对应关系

| 页面 | 使用接口 |
|---|---|
| 登录页 | `POST /auth/login` |
| 注册页 | `POST /auth/register` |
| 个人中心页 | `GET /users/profile`、`PUT /users/profile`、`PUT /users/password` |
| 首页工作台 | `GET /users/overview` |
| 管理端用户管理页 | `GET /admin/users`、`PUT /admin/users/{id}/status`、`GET /admin/roles` |

---

## 12. 注意事项

1. 不返回密码哈希。
2. 管理端接口必须校验 `ADMIN` 角色。
3. 普通用户只能操作自己的资料。
4. 注册默认分配 `USER` 角色。
5. 禁用用户不能登录。
6. `/inner/**` 内部接口不对前端暴露。
7. Gateway 不对外暴露 `/inner/**`。
8. auth-service 不直接操作用户权限表。
9. user-service 是 `sys_user`、`sys_role`、`sys_user_role` 的数据归属服务。
10. 修改密码后 V1 建议让用户重新登录。
11. 管理端用户列表不返回密码、Token 等敏感信息。
