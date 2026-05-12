# CodeCoachAI V1 接口设计总览

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 接口设计总览 |
| 项目阶段 | V1 |
| 最高依据 | PRD/CodeCoachAI_PRD_V1_微服务分层架构版.md |
| 参考文档 | 数据库设计/CodeCoachAI_V1_数据库设计总览.md 及 V1 各表设计文档 |
| 文档用途 | 统一 V1 接口风格、响应结构、认证方式、模块划分和接口边界 |

本文档只设计 CodeCoachAI V1 接口。

V1 不设计文件上传解析、MinIO、消息队列异步任务、Elasticsearch、学习计划、AI 题目批量生成、语音视频面试、支付会员、企业端等后续接口。

---

## 2. 接口设计原则

## 2.1 RESTful 风格约定

V1 接口采用偏 RESTful 的资源风格。

基本约定：

| 操作 | HTTP 方法 | 示例 |
|---|---|---|
| 查询列表 | `GET` | `GET /questions` |
| 查询详情 | `GET` | `GET /questions/{id}` |
| 新增资源 | `POST` | `POST /resumes` |
| 修改资源 | `PUT` | `PUT /resumes/{id}` |
| 删除资源 | `DELETE` | `DELETE /resumes/{id}` |
| 业务动作 | `POST` / `PUT` | `POST /interviews/{id}/start` |

URL 命名规则：

1. 用户端接口使用业务资源名，例如 `/questions`、`/resumes`、`/interviews`。
2. 管理端接口统一使用 `/admin` 前缀。
3. 服务内部 Feign 接口统一使用 `/inner` 前缀，前端不允许直接访问。
4. URL 使用小写字母和中划线，路径参数使用 `{id}`。
5. 接口版本暂不放入 URL，V1 项目整体即为 V1。

示例：

```text
GET    /questions
GET    /questions/{id}
POST   /admin/questions
PUT    /admin/questions/{id}
DELETE /admin/questions/{id}
POST   /interviews/{id}/answer
```

## 2.2 统一响应结构

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

成功响应：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "id": 10001
  }
}
```

失败响应：

```json
{
  "code": 40001,
  "message": "参数校验失败",
  "data": null
}
```

## 2.3 分页请求规范

分页查询统一使用以下参数：

| 参数 | 类型 | 是否必填 | 默认值 | 说明 |
|---|---|---:|---|---|
| `pageNo` | `int` | 否 | `1` | 当前页码 |
| `pageSize` | `int` | 否 | `10` | 每页数量 |

示例：

```text
GET /questions?pageNo=1&pageSize=10&categoryId=1
```

建议限制：

1. `pageNo` 最小为 1。
2. `pageSize` 默认 10。
3. `pageSize` 最大不超过 100。

## 2.4 分页响应规范

分页响应统一结构：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "records": [],
    "total": 0,
    "pageNo": 1,
    "pageSize": 10,
    "pages": 0
  }
}
```

字段说明：

| 字段 | 类型 | 说明 |
|---|---|---|
| `records` | `array` | 当前页数据 |
| `total` | `long` | 总记录数 |
| `pageNo` | `int` | 当前页码 |
| `pageSize` | `int` | 每页数量 |
| `pages` | `long` | 总页数 |

## 2.5 Token 认证方式

V1 推荐使用 Sa-Token，也可使用 JWT。

前端登录成功后保存 Token，并在后续请求头中携带：

```text
Authorization: Bearer {token}
```

认证规则：

1. 登录、注册接口不需要 Token。
2. 用户端业务接口需要登录。
3. 管理端接口需要登录且具备 `ADMIN` 角色。
4. Gateway 做 Token 基础校验。
5. 下游服务通过 common-security 获取当前登录用户。
6. 普通用户数据接口必须校验数据归属。

## 2.6 管理端权限要求

管理端接口统一使用 `/admin` 前缀。

要求：

1. 必须登录。
2. 必须具备 `ADMIN` 角色。
3. 不能返回用户密码、Token、API Key 等敏感信息。
4. Prompt 模板和 AI 调用日志仅管理员可访问。
5. 用户管理 V1 只做简化能力。

## 2.7 错误码规范

错误码建议按模块分段：

| 范围 | 模块 | 示例 |
|---|---|---|
| `0` | 成功 | `0 success` |
| `40000-40999` | 通用错误 | 参数错误、请求非法 |
| `41000-41999` | 认证权限 | 未登录、无权限、Token 失效 |
| `42000-42999` | 用户模块 | 用户不存在、账号禁用 |
| `43000-43999` | 题库模块 | 题目不存在、问题组不存在 |
| `44000-44999` | 简历模块 | 简历不存在、无权访问 |
| `45000-45999` | 面试模块 | 面试不存在、状态不允许 |
| `46000-46999` | AI 模块 | AI 调用失败、Prompt 不存在 |
| `47000-47999` | 系统模块 | 配置不存在 |

通用错误码建议：

| code | message |
|---:|---|
| `0` | success |
| `40000` | 请求参数错误 |
| `40001` | 参数校验失败 |
| `41000` | 未登录 |
| `41001` | Token 无效或已过期 |
| `41003` | 无访问权限 |
| `50000` | 系统内部错误 |

## 2.8 日期时间格式规范

接口中的日期时间统一使用：

```text
yyyy-MM-dd HH:mm:ss
```

示例：

```json
{
  "createdAt": "2026-05-12 18:00:00"
}
```

约定：

1. 后端统一按服务器时区处理。
2. 前端展示时可按用户本地时区格式化。
3. 日期字段命名使用驼峰，例如 `createdAt`、`updatedAt`、`startTime`。
4. 数据库字段使用下划线，例如 `created_at`、`updated_at`、`start_time`。

## 2.9 `/inner/**` 内部接口安全约束

`/inner/**` 接口只用于服务内部 Feign 调用。

约束：

1. Gateway 不对外暴露 `/inner/**`。
2. 前端不允许直接访问 `/inner/**`。
3. `/inner/**` 仅允许服务内部 Feign 调用。
4. 如果 `/inner/**` 调用经过 Gateway，需要增加内部调用标识或服务间鉴权。
5. 内部调用仍需透传用户上下文或明确传入 userId，不能绕过数据归属校验。

---

## 3. V1 接口模块划分

V1 接口按服务划分：

| 服务 | 模块 | URL 前缀 |
|---|---|---|
| auth-service | 认证登录接口 | `/auth` |
| user-service | 用户信息接口 | `/users`、`/admin/users`、`/admin/roles` |
| question-service | 题库、分类、标签、问题组、刷题接口 | `/questions`、`/admin/questions` |
| resume-service | 简历和项目经历接口 | `/resumes` |
| interview-service | 面试会话、阶段、消息、报告接口 | `/interviews` |
| ai-service | AI 内部能力、Prompt、AI 日志接口 | `/inner/ai`、`/admin/ai` |
| system-service | 系统配置、后台首页统计接口 | `/admin/system`、`/admin/configs` |

---

## 4. auth-service 认证登录接口

## 4.1 模块职责

auth-service 负责：

1. 用户注册。
2. 用户登录。
3. 用户退出。
4. 当前登录用户查询。
5. Token 刷新，V1 可选。

## 4.2 URL 前缀

```text
/auth
```

## 4.3 主要接口列表

| 接口名称 | 方法 | URL | 需要登录 | 管理员权限 | 说明 |
|---|---|---|---:|---:|---|
| 用户注册 | `POST` | `/auth/register` | 否 | 否 | 注册普通用户 |
| 用户登录 | `POST` | `/auth/login` | 否 | 否 | 普通用户或管理员登录 |
| 用户退出 | `POST` | `/auth/logout` | 是 | 否 | 当前用户退出登录 |
| 当前用户 | `GET` | `/auth/current-user` | 是 | 否 | 获取当前登录用户 |
| 刷新 Token | `POST` | `/auth/refresh-token` | 是 | 否 | V1 可选 |

## 4.4 关联数据库表

| 表名 | 归属服务 | 说明 |
|---|---|---|
| `sys_user` | user-service | auth 通过 Feign 查询 |
| `sys_role` | user-service | auth 通过 Feign 查询角色 |
| `sys_user_role` | user-service | auth 通过 Feign 查询用户角色 |

## 4.5 前端对应页面

| 页面 | 说明 |
|---|---|
| 登录页 | 用户登录、管理员登录 |
| 注册页 | 普通用户注册 |
| 个人中心页 | 获取当前用户信息 |

---

## 5. user-service 用户信息接口

## 5.1 模块职责

user-service 负责：

1. 用户资料查询。
2. 用户资料修改。
3. 用户基础学习概览。
4. 管理端用户列表。
5. 用户启用 / 禁用，V1 可选。
6. 管理端角色列表查询，V1 只返回基础角色。

## 5.2 URL 前缀

```text
/users
/admin/users
/admin/roles
```

## 5.3 主要接口列表

| 接口名称 | 方法 | URL | 需要登录 | 管理员权限 | 说明 |
|---|---|---|---:|---:|---|
| 查询个人资料 | `GET` | `/users/profile` | 是 | 否 | 当前用户资料 |
| 修改个人资料 | `PUT` | `/users/profile` | 是 | 否 | 修改昵称、头像、邮箱 |
| 修改密码 | `PUT` | `/users/password` | 是 | 否 | 修改当前用户密码 |
| 用户学习概览 | `GET` | `/users/overview` | 是 | 否 | 简化统计 |
| 管理端用户列表 | `GET` | `/admin/users` | 是 | 是 | 用户分页列表 |
| 修改用户状态 | `PUT` | `/admin/users/{id}/status` | 是 | 是 | 启用 / 禁用用户 |
| 管理端角色列表 | `GET` | `/admin/roles` | 是 | 是 | 查询 USER、ADMIN 等基础角色 |

`GET /admin/roles` 说明：

1. 归属服务为 user-service。
2. 用于管理端查询角色列表。
3. V1 只返回 `USER`、`ADMIN` 等基础角色。
4. 可用于用户管理、权限展示或前端静态权限判断。
5. 不设计复杂菜单权限、按钮权限和数据权限。

## 5.4 关联数据库表

| 表名 | 说明 |
|---|---|
| `sys_user` | 用户资料 |
| `sys_role` | 角色 |
| `sys_user_role` | 用户角色关系 |

## 5.5 前端对应页面

| 页面 | 说明 |
|---|---|
| 个人中心页 | 查看和修改个人资料 |
| 首页工作台 | 展示用户学习概览 |
| 管理端用户管理页 | 管理用户列表和状态 |

---

## 6. question-service 题库、分类、标签、问题组、刷题接口

## 6.1 模块职责

question-service 负责：

1. 用户端题库查询。
2. 题目详情。
3. 用户答题记录。
4. 掌握状态。
5. 错题本。
6. 收藏题目。
7. 管理端题目、分类、标签、问题组维护。
8. 为 interview-service 提供面试抽题内部接口。

## 6.2 URL 前缀

```text
/questions
/admin/questions
/admin/question-categories
/admin/question-tags
/admin/question-groups
```

## 6.3 用户端主要接口列表

| 接口名称 | 方法 | URL | 需要登录 | 管理员权限 | 说明 |
|---|---|---|---:|---:|---|
| 题目列表 | `GET` | `/questions` | 是 | 否 | 按分类、标签、难度查询 |
| 题目详情 | `GET` | `/questions/{id}` | 是 | 否 | 查看题目详情 |
| 提交答案 | `POST` | `/questions/{id}/answers` | 是 | 否 | 保存用户答案 |
| 收藏题目 | `POST` | `/questions/{id}/favorite` | 是 | 否 | 收藏 |
| 取消收藏 | `DELETE` | `/questions/{id}/favorite` | 是 | 否 | 取消收藏 |
| 收藏列表 | `GET` | `/questions/favorites` | 是 | 否 | 当前用户收藏题 |
| 错题列表 | `GET` | `/questions/wrong-records` | 是 | 否 | 当前用户错题 |
| 修改掌握状态 | `PUT` | `/questions/{id}/mastery` | 是 | 否 | MASTERED / VAGUE / UNKNOWN |

## 6.4 管理端主要接口列表

| 接口名称 | 方法 | URL | 需要登录 | 管理员权限 | 说明 |
|---|---|---|---:|---:|---|
| 管理端题目列表 | `GET` | `/admin/questions` | 是 | 是 | 题目分页查询 |
| 新增题目 | `POST` | `/admin/questions` | 是 | 是 | 创建题目 |
| 修改题目 | `PUT` | `/admin/questions/{id}` | 是 | 是 | 编辑题目 |
| 启用 / 禁用题目 | `PUT` | `/admin/questions/{id}/status` | 是 | 是 | 修改题目状态 |
| 删除题目 | `DELETE` | `/admin/questions/{id}` | 是 | 是 | 逻辑删除 |
| 分类列表 | `GET` | `/admin/question-categories` | 是 | 是 | 分类查询 |
| 新增分类 | `POST` | `/admin/question-categories` | 是 | 是 | 创建分类 |
| 修改分类 | `PUT` | `/admin/question-categories/{id}` | 是 | 是 | 编辑分类 |
| 启用 / 禁用分类 | `PUT` | `/admin/question-categories/{id}/status` | 是 | 是 | 修改分类状态 |
| 删除分类 | `DELETE` | `/admin/question-categories/{id}` | 是 | 是 | 删除分类 |
| 标签列表 | `GET` | `/admin/question-tags` | 是 | 是 | 标签查询 |
| 新增标签 | `POST` | `/admin/question-tags` | 是 | 是 | 创建标签 |
| 修改标签 | `PUT` | `/admin/question-tags/{id}` | 是 | 是 | 编辑标签 |
| 启用 / 禁用标签 | `PUT` | `/admin/question-tags/{id}/status` | 是 | 是 | 修改标签状态 |
| 删除标签 | `DELETE` | `/admin/question-tags/{id}` | 是 | 是 | 删除标签 |
| 问题组列表 | `GET` | `/admin/question-groups` | 是 | 是 | 问题组查询 |
| 新增问题组 | `POST` | `/admin/question-groups` | 是 | 是 | 创建问题组 |
| 修改问题组 | `PUT` | `/admin/question-groups/{id}` | 是 | 是 | 编辑问题组 |
| 启用 / 禁用问题组 | `PUT` | `/admin/question-groups/{id}/status` | 是 | 是 | 修改问题组状态 |
| 删除问题组 | `DELETE` | `/admin/question-groups/{id}` | 是 | 是 | 删除问题组 |

## 6.5 内部 Feign 接口

| 接口名称 | 方法 | URL | 调用方 | 说明 |
|---|---|---|---|---|
| 面试抽题 | `POST` | `/inner/questions/pick-for-interview` | interview-service | 按条件抽题，排除 groupId |
| 内部题目详情 | `GET` | `/inner/questions/{id}` | interview-service | 获取题目、答案、解析 |
| 报告推荐题 | `POST` | `/inner/questions/recommend-for-report` | interview-service | V1 可选 |

## 6.6 关联数据库表

| 表名 | 说明 |
|---|---|
| `question_category` | 分类 |
| `question_tag` | 标签 |
| `question_group` | 问题组 |
| `question` | 题目 |
| `question_tag_relation` | 题目标签关系 |
| `user_question_record` | 答题记录 |
| `wrong_question` | 错题 |
| `favorite_question` | 收藏 |
| `user_question_mastery` | 掌握状态 |

## 6.7 前端对应页面

| 页面 | 说明 |
|---|---|
| 题库列表页 | 查询题目 |
| 题目详情页 | 查看题目、答案、解析、提交答案 |
| 错题本页 | 查看错题 |
| 收藏题目页 | 查看收藏 |
| 管理端题目管理页 | 题目 CRUD |
| 管理端分类管理页 | 分类 CRUD |
| 管理端标签管理页 | 标签 CRUD |
| 管理端问题组管理页 | 问题组 CRUD |

---

## 7. resume-service 简历和项目经历接口

## 7.1 模块职责

resume-service 负责：

1. 简历手动录入。
2. 简历列表、详情、编辑、删除。
3. 默认简历设置。
4. 项目经历维护。
5. 为 interview-service 提供简历查询内部接口。

## 7.2 URL 前缀

```text
/resumes
```

## 7.3 主要接口列表

| 接口名称 | 方法 | URL | 需要登录 | 管理员权限 | 说明 |
|---|---|---|---:|---:|---|
| 简历列表 | `GET` | `/resumes` | 是 | 否 | 当前用户简历 |
| 新增简历 | `POST` | `/resumes` | 是 | 否 | 手动创建简历 |
| 简历详情 | `GET` | `/resumes/{id}` | 是 | 否 | 简历和项目经历详情 |
| 修改简历 | `PUT` | `/resumes/{id}` | 是 | 否 | 编辑简历 |
| 删除简历 | `DELETE` | `/resumes/{id}` | 是 | 否 | 删除简历 |
| 设置默认简历 | `PUT` | `/resumes/{id}/default` | 是 | 否 | 设置默认 |
| 新增项目经历 | `POST` | `/resumes/{resumeId}/projects` | 是 | 否 | 新增项目 |
| 修改项目经历 | `PUT` | `/resumes/projects/{projectId}` | 是 | 否 | 编辑项目 |
| 删除项目经历 | `DELETE` | `/resumes/projects/{projectId}` | 是 | 否 | 删除项目 |

## 7.4 内部 Feign 接口

| 接口名称 | 方法 | URL | 调用方 | 说明 |
|---|---|---|---|---|
| 内部简历详情 | `GET` | `/inner/resumes/{id}` | interview-service | 校验归属并返回简历 |
| 内部项目经历 | `GET` | `/inner/resumes/{id}/projects` | interview-service | 返回项目经历 |
| 默认简历 | `GET` | `/inner/users/{userId}/default-resume` | interview-service | V1 可选 |

## 7.5 关联数据库表

| 表名 | 说明 |
|---|---|
| `resume` | 简历 |
| `resume_project` | 项目经历 |

## 7.6 前端对应页面

| 页面 | 说明 |
|---|---|
| 简历列表页 | 查看简历 |
| 简历编辑页 | 新增和编辑简历 |
| 创建面试页 | 选择简历 |

---

## 8. interview-service 面试接口

## 8.1 模块职责

interview-service 负责：

1. 创建面试。
2. 生成面试阶段。
3. 开始面试。
4. 获取当前问题。
5. 提交回答。
6. 控制动态追问。
7. 结束面试。
8. 保存和查询面试记录。
9. 保存和查询面试报告。

## 8.2 URL 前缀

```text
/interviews
```

## 8.3 主要接口列表

| 接口名称 | 方法 | URL | 需要登录 | 管理员权限 | 说明 |
|---|---|---|---:|---:|---|
| 创建面试 | `POST` | `/interviews` | 是 | 否 | 创建会话和阶段 |
| 开始面试 | `POST` | `/interviews/{id}/start` | 是 | 否 | 开始面试 |
| 当前问题 | `GET` | `/interviews/{id}/current` | 是 | 否 | 获取当前问题 |
| 提交回答 | `POST` | `/interviews/{id}/answer` | 是 | 否 | 提交用户回答，返回 AI 评分、点评和下一步动作 |
| 结束面试 | `POST` | `/interviews/{id}/finish` | 是 | 否 | V1 可同步生成报告 |
| 重试生成报告 | `POST` | `/interviews/{id}/report/retry` | 是 | 否 | 报告生成失败后重试生成报告 |
| 面试历史 | `GET` | `/interviews` | 是 | 否 | 当前用户历史面试 |
| 面试详情 | `GET` | `/interviews/{id}` | 是 | 否 | 面试阶段和问答详情 |
| 面试报告 | `GET` | `/interviews/{id}/report` | 是 | 否 | 查看报告 |

`POST /interviews/{id}/answer` 响应需要包含：

1. AI 评分。
2. AI 点评。
3. 下一步动作。
4. 下一题或追问题内容。
5. 当前面试阶段和会话状态。

下一步动作枚举：

```text
FOLLOW_UP      继续追问
NEXT_QUESTION  进入下一题
NEXT_STAGE     进入下一阶段
FINISH         结束面试
```

`POST /interviews/{id}/finish` 说明：

1. V1 可同步调用 ai-service 生成面试报告。
2. 报告生成成功后，面试状态更新为 `COMPLETED`，`reportStatus` 更新为 `GENERATED`。
3. AI 生成失败时，需要记录失败状态和失败原因。
4. 失败后允许通过 `POST /interviews/{id}/report/retry` 重试生成报告。

`POST /interviews/{id}/report/retry` 说明：

1. 报告生成失败后重试生成报告。
2. 仅当前用户自己的面试可操作。
3. 仅 `reportStatus = FAILED` 或 `NOT_GENERATED` 时允许调用。
4. 不生成多份有效报告。
5. 成功后 `reportStatus` 更新为 `GENERATED`。
6. 失败后 `reportStatus` 保持 `FAILED`，并记录失败原因。

## 8.4 关联数据库表

| 表名 | 说明 |
|---|---|
| `interview_session` | 面试会话 |
| `interview_stage` | 面试阶段 |
| `interview_message` | 面试消息 |
| `interview_report` | 面试报告 |

## 8.5 内部依赖

| 被调用服务 | 用途 |
|---|---|
| question-service | 抽题、获取参考答案、推荐题 |
| resume-service | 获取简历和项目经历 |
| ai-service | 生成问题、评分、追问、报告 |

## 8.6 前端对应页面

| 页面 | 说明 |
|---|---|
| 创建面试页 | 创建面试 |
| AI 面试房间页 | 当前问题、提交回答、追问 |
| 面试历史页 | 历史列表 |
| 面试详情页 | 问答明细 |
| 面试报告页 | 查看报告 |

---

## 9. ai-service AI、Prompt、AI 日志接口

## 9.1 模块职责

ai-service 负责：

1. AI 面试问题生成。
2. AI 回答评分。
3. AI 动态追问生成。
4. AI 面试报告生成。
5. Prompt 模板管理。
6. AI 调用日志记录和查询。

## 9.2 URL 前缀

```text
/inner/ai
/admin/ai
```

## 9.3 AI 能力接口

AI 能力接口 V1 只通过 `/inner/ai/**` 提供给 interview-service 内部调用，前端不直接访问 `/ai/interview/**`。

| 接口名称 | 方法 | URL | 调用方 | 说明 |
|---|---|---|---|---|
| 生成面试问题 | `POST` | `/inner/ai/interview/question` | interview-service | 基于题库、阶段和面试配置生成问题 |
| 回答评分 | `POST` | `/inner/ai/interview/evaluate` | interview-service | 基于问题、参考答案、用户回答生成评分和点评 |
| 生成追问 | `POST` | `/inner/ai/interview/follow-up` | interview-service | 根据评分结果和追问方向生成追问 |
| 生成报告 | `POST` | `/inner/ai/interview/report` | interview-service | 根据面试问答和阶段得分生成报告内容 |

## 9.4 管理端 Prompt 接口

| 接口名称 | 方法 | URL | 需要登录 | 管理员权限 | 说明 |
|---|---|---|---:|---:|---|
| Prompt 列表 | `GET` | `/admin/ai/prompts` | 是 | 是 | 分页查询 |
| Prompt 详情 | `GET` | `/admin/ai/prompts/{id}` | 是 | 是 | 查看模板 |
| 新增 Prompt | `POST` | `/admin/ai/prompts` | 是 | 是 | 新增模板 |
| 修改 Prompt | `PUT` | `/admin/ai/prompts/{id}` | 是 | 是 | 编辑模板 |
| 启用禁用 Prompt | `PUT` | `/admin/ai/prompts/{id}/status` | 是 | 是 | 修改状态 |

## 9.5 管理端 AI 日志接口

| 接口名称 | 方法 | URL | 需要登录 | 管理员权限 | 说明 |
|---|---|---|---:|---:|---|
| AI 日志列表 | `GET` | `/admin/ai/logs` | 是 | 是 | 分页筛选 |
| AI 日志详情 | `GET` | `/admin/ai/logs/{id}` | 是 | 是 | 查看请求响应 |

## 9.6 关联数据库表

| 表名 | 说明 |
|---|---|
| `prompt_template` | Prompt 模板 |
| `ai_call_log` | AI 调用日志 |

## 9.7 前端对应页面

| 页面 | 说明 |
|---|---|
| AI 面试房间页 | 间接使用 AI 提问、评分、追问 |
| 面试报告页 | 间接使用报告生成 |
| Prompt 模板管理页 | 管理模板 |
| AI 调用日志页 | 查看 AI 调用日志 |

---

## 10. system-service 系统配置和后台管理接口

## 10.1 模块职责

system-service 负责：

1. 系统配置管理。
2. 管理端首页简化统计。
3. 菜单能力，V1 可选或后续增强。

V1 系统服务保持轻量，不做完整操作日志、通知中心、复杂数据看板。

角色权限归属说明：

1. V1 推荐由 user-service 管理 `sys_user`、`sys_role`、`sys_user_role`。
2. system-service V1 主要管理 `system_config` 和管理端首页统计。
3. `sys_menu`、`sys_permission` 属于 V1 可选表，菜单和复杂权限可以后续增强。
4. `/admin/roles` 明确归属 user-service，system-service 不直接维护 `sys_role` 和 `sys_user_role`。

## 10.2 URL 前缀

```text
/admin/system
/admin/configs
```

## 10.3 主要接口列表

| 接口名称 | 方法 | URL | 需要登录 | 管理员权限 | 说明 |
|---|---|---|---:|---:|---|
| 管理端首页统计 | `GET` | `/admin/system/overview` | 是 | 是 | 简化统计 |
| 菜单列表 | `GET` | `/admin/menus` | 是 | 是 | V1 可选 |
| 系统配置列表 | `GET` | `/admin/configs` | 是 | 是 | 配置列表 |
| 新增系统配置 | `POST` | `/admin/configs` | 是 | 是 | 新增非核心配置 |
| 系统配置详情 | `GET` | `/admin/configs/{key}` | 是 | 是 | 查询单个系统配置详情 |
| 修改系统配置 | `PUT` | `/admin/configs/{key}` | 是 | 是 | 修改配置 |
| 启用 / 禁用配置 | `PUT` | `/admin/configs/{key}/status` | 是 | 是 | 修改配置状态 |
| 删除系统配置 | `DELETE` | `/admin/configs/{key}` | 是 | 是 | 删除非核心配置 |

系统配置补充说明：

1. `GET /admin/configs/{key}` 用于管理端配置编辑页回显，仅管理员可访问，对敏感配置值需要脱敏或不返回 `configValue`。
2. `PUT /admin/configs/{key}/status` 用于启用或禁用系统配置，仅管理员可访问。
3. `editable = 0` 的核心配置不建议允许禁用。
4. 禁用后业务读取时应忽略该配置或使用默认配置。
5. `POST /admin/configs` 和 `DELETE /admin/configs/{key}` 只建议用于非核心配置，V1 不做配置发布、灰度和历史版本。

## 10.4 关联数据库表

| 表名 | 说明 |
|---|---|
| `system_config` | 系统配置 |
| `sys_menu` | 菜单，V1 可选 |
| `sys_permission` | 权限标识，V1 可选 |
| `sys_role` | 角色数据由 user-service 管理，system-service 不直接维护 |

## 10.5 前端对应页面

| 页面 | 说明 |
|---|---|
| 管理端首页 | 简化统计 |
| 系统配置页 | 修改基础配置 |
| 管理端布局菜单 | V1 可选动态菜单 |

---

## 11. V1 暂不设计的接口

V1 明确不设计以下接口：

| 接口类型 | V1 不做原因 |
|---|---|
| 文件上传解析接口 | V1 简历只手动录入 |
| MinIO 文件接口 | V1 不引入文件服务 |
| RabbitMQ / RocketMQ 异步任务接口 | V1 同步处理 AI 任务 |
| Elasticsearch 搜索接口 | V1 使用 MySQL 条件查询 |
| 学习计划接口 | V2 再做 |
| AI 题目批量生成接口 | V2 再做 |
| AI 生成题审核接口 | V2 再做 |
| 简历 AI 优化接口 | V2 再做 |
| 简历 PDF / Word 解析接口 | V2 再做 |
| SSE 流式输出接口 | V2 再做 |
| 语音 / 视频面试接口 | 不做 |
| 支付、会员接口 | 不做 |
| 企业端、HR 端、招聘投递接口 | 不做 |
| 完整操作日志接口 | V3 再做 |
| 站内通知接口 | V3 再做 |

---

## 12. 内部 Feign 接口总览

内部 Feign 接口统一使用 `/inner/**` 前缀。Gateway 不对外暴露这些接口；如果内部调用链路经过 Gateway，需要内部调用标识或服务间鉴权。

| 调用方 | 被调用方 | 接口 | 用途 |
|---|---|---|---|
| auth-service | user-service | `/inner/users/by-username` | 登录查询用户 |
| auth-service | user-service | `/inner/users` | 注册创建用户 |
| auth-service | user-service | `/inner/users/{id}/roles` | 查询角色 |
| interview-service | question-service | `/inner/questions/pick-for-interview` | 面试抽题 |
| interview-service | question-service | `/inner/questions/{id}` | 题目详情 |
| interview-service | resume-service | `/inner/resumes/{id}` | 简历详情 |
| interview-service | resume-service | `/inner/resumes/{id}/projects` | 项目经历 |
| interview-service | ai-service | `/inner/ai/interview/question` | 生成问题 |
| interview-service | ai-service | `/inner/ai/interview/evaluate` | 回答评分 |
| interview-service | ai-service | `/inner/ai/interview/follow-up` | 生成追问 |
| interview-service | ai-service | `/inner/ai/interview/report` | 生成报告 |

---

## 13. 结论

CodeCoachAI V1 接口设计围绕核心闭环展开：

```text
认证登录
  ↓
题库管理和刷题
  ↓
简历和项目经历
  ↓
AI 模拟面试
  ↓
动态追问
  ↓
面试报告
  ↓
Prompt 模板和 AI 日志管理
```

V1 接口设计重点是：

1. 统一响应结构。
2. 统一分页规范。
3. 统一 Token 认证。
4. 管理端接口使用 `/admin` 前缀并要求 `ADMIN` 角色。
5. 服务间调用使用 `/inner` 前缀，且不对前端暴露。
6. AI 能力统一归属 ai-service，并仅通过 `/inner/ai/**` 供 interview-service 调用。
7. 面试流程编排统一归属 interview-service。
8. 不提前设计 V2 / V3 接口。
