# CodeCoachAI V1 简历接口设计

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 简历接口设计 |
| 项目阶段 | V1 |
| 最高依据 | 接口设计/CodeCoachAI_V1_接口设计总览.md |
| 参考文档 | PRD/CodeCoachAI_PRD_V1_微服务分层架构版.md、数据库设计/CodeCoachAI_V1_数据库设计总览.md、数据库设计/CodeCoachAI_V1_简历表设计.md |
| 文档用途 | 详细设计 resume-service 在 V1 阶段的简历、项目经历、默认简历和内部 Feign 查询接口 |

本文档只设计 CodeCoachAI V1 简历相关接口，不设计 V2 / V3 功能。

涉及服务：

| 服务 | 说明 |
|---|---|
| `codecoachai-resume` | 简历和项目经历的数据归属服务 |
| `codecoachai-interview` | 通过 `/inner/resumes/**` 内部接口查询简历和项目经历 |
| `codecoachai-gateway` | 对外统一入口，不对外暴露 `/inner/**` |
| `codecoachai-auth` / `codecoachai-user` | 提供认证、登录态和当前用户上下文 |

字段说明：

1. 本文档优先贴合 `数据库设计/CodeCoachAI_V1_简历表设计.md`。
2. 数据库当前未设计 `real_name`、`phone`、`email`、`years_of_experience`、`self_evaluation`、`start_date`、`end_date`、`role` 等字段，因此 V1 简历接口不强行新增这些字段。
3. 个人姓名、手机号、邮箱等基础资料如需展示，V1 建议从 user-service 用户资料接口获取。
4. 经验年限属于创建面试配置中的字段，V1 不存入 `resume` 表。

---

## 2. 模块职责

resume-service 负责：

1. 简历手动创建。
2. 简历列表查询。
3. 简历详情查询。
4. 简历编辑。
5. 简历删除。
6. 默认简历设置。
7. 项目经历新增。
8. 项目经历编辑。
9. 项目经历删除。
10. 为 interview-service 提供简历和项目经历内部查询接口。

resume-service 不负责：

| 能力 | V1 处理方式 |
|---|---|
| 文件上传 | V1 不设计文件上传接口 |
| PDF / Word 解析 | V1 只支持手动录入 |
| AI 简历优化 | V1 不设计 AI 简历优化接口 |
| 面试流程编排 | 归属 interview-service |
| AI 提问和评分 | 归属 ai-service，由 interview-service 编排调用 |
| 简历模板导出 | V1 不设计导出能力 |
| 管理端简历管理 | V1 暂不设计管理端简历接口 |

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

### 3.2 分页结构

分页请求统一使用：

| 参数 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `pageNo` | `int` | `1` | 当前页码，最小 1 |
| `pageSize` | `int` | `10` | 每页数量，最大 100 |

分页响应统一使用：

| 字段 | 类型 | 说明 |
|---|---|---|
| `records` | `array` | 当前页数据 |
| `total` | `long` | 总记录数 |
| `pageNo` | `int` | 当前页码 |
| `pageSize` | `int` | 每页数量 |
| `pages` | `long` | 总页数 |

### 3.3 Token 请求头格式

用户端简历接口统一使用：

```text
Authorization: Bearer {token}
```

### 3.4 当前登录用户上下文

resume-service 从登录态或 Gateway 透传信息中获取当前用户上下文。

当前用户上下文至少包含：

| 字段 | 说明 |
|---|---|
| `userId` | 当前登录用户 ID |
| `username` | 登录账号 |
| `roles` | 角色编码列表 |

用户端所有 `/resumes/**` 接口都必须基于当前登录用户 `userId` 进行数据归属校验，不能信任前端传入的 `userId`。

### 3.5 数据归属校验

| 数据 | 归属规则 |
|---|---|
| `resume` | 只能由当前登录用户查询、创建、修改、删除 |
| `resume_project` | 只能在当前登录用户自己的简历下新增、修改、删除 |
| `/inner/resumes/**` | 必须传入或透传 `userId` 校验简历归属 |

前端不能通过传 `userId` 操作其他用户简历；用户端接口请求体中也不设计 `userId` 字段。

### 3.6 逻辑删除规则

V1 简历和项目经历使用 `deleted` 字段进行逻辑删除：

| 值 | 说明 |
|---|---|
| `0` | 未删除 |
| `1` | 已删除 |

规则：

1. 删除简历使用逻辑删除。
2. 删除项目经历使用逻辑删除。
3. 删除简历时，该简历下项目经历也应逻辑删除或在展示查询时不再返回。
4. 已删除简历不可查询、不可编辑、不可设置默认。

### 3.7 默认简历规则

默认简历通过 `resume.is_default` 维护：

| 值 | 说明 |
|---|---|
| `1` | 默认简历 |
| `0` | 非默认简历 |

规则：

1. 每个用户最多只能有一份默认简历。
2. 创建第一份简历时自动设为默认简历。
3. 用户已有默认简历时，新建简历默认 `isDefault = 0`。
4. 设置新默认简历时，需要取消该用户其他未删除简历的默认状态。
5. 删除默认简历后，V1 推荐自动选择最近更新的一份未删除简历作为新的默认简历；如果没有其他简历，则用户没有默认简历。

### 3.8 `/inner/**` 内部接口安全约束

`/inner/**` 接口只允许服务内部 Feign 调用：

1. Gateway 不对外暴露 `/inner/**`。
2. 前端不能直接访问 `/inner/resumes/**`。
3. interview-service 只能通过 `/inner/resumes/**` 查询简历和项目经历。
4. 如果内部调用经过 Gateway，需要内部调用标识或服务间鉴权。
5. 内部接口仍需校验 `userId` 和简历归属，不能绕过业务规则。

---

## 4. 枚举设计

### 4.1 ResumeStatus

| 值 | 枚举名 | 说明 |
|---|---|---|
| `1` | `ENABLED` | 正常 |
| `0` | `DISABLED` | 停用，V1 可选 |

### 4.2 IsDefault

| 值 | 说明 |
|---|---|
| `1` | 默认简历 |
| `0` | 非默认简历 |

### 4.3 ProjectType

`resume_project` 表当前未设计项目类型字段，V1 暂不设计项目类型枚举。

如果后续需要区分企业项目、个人项目、开源项目等类型，应先补充数据库字段，再扩展接口枚举。

---

## 5. 用户端简历接口设计

### 5.1 GET /resumes

接口说明：简历列表。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/resumes` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `resume`、`resume_project` |

查询参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 简历名称、求职方向关键字 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 简历 ID |
| `resumeName` | `string` | 简历名称，对应 `resume.resume_name` |
| `targetPosition` | `string` | 求职方向，对应 `resume.target_position` |
| `skills` | `string` | 技能栈摘要 |
| `isDefault` | `int` | 是否默认简历 |
| `status` | `int` | 简历状态 |
| `projectCount` | `int` | 项目经历数量 |
| `updatedAt` | `string` | 更新时间 |
| `createdAt` | `string` | 创建时间 |

不返回字段说明：

| 字段 | V1 处理方式 |
|---|---|
| `realName` | 当前简历表未设计，若需展示可从用户资料获取 |
| `yearsOfExperience` | 当前简历表未设计，创建面试时在 interview-service 配置中填写 |

业务规则：

1. 只返回当前登录用户的简历。
2. 不返回已删除简历。
3. 默认按 `updatedAt` 倒序。
4. V1 支持 MySQL 条件查询。
5. V1 不支持全文检索。

### 5.2 POST /resumes

接口说明：新增简历。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/resumes` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `resume` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `resumeName` | `string` | 是 | 简历名称 |
| `targetPosition` | `string` | 否 | 求职方向 |
| `skills` | `string` | 是 | 技能栈 |
| `workSummary` | `string` | 否 | 工作经历摘要 |
| `education` | `string` | 否 | 教育经历 |

当前不接收字段：

| 字段 | V1 处理方式 |
|---|---|
| `userId` | 从登录态获取，不允许前端传入 |
| `realName` | 简历表未设计 |
| `phone` | 简历表未设计，可由 user-service 用户资料维护 |
| `email` | 简历表未设计，可由 user-service 用户资料维护 |
| `yearsOfExperience` | 简历表未设计 |
| `selfEvaluation` | 简历表未设计，可先并入 `workSummary` |
| `isDefault` | 不允许新增时直接指定，由默认简历规则处理 |

响应参数：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 简历 ID |
| `resumeName` | `string` | 简历名称 |
| `isDefault` | `int` | 是否默认简历 |
| `createdAt` | `string` | 创建时间 |

业务规则：

1. 创建简历必须绑定当前登录用户。
2. 如果当前用户没有任何未删除简历，则第一份简历自动设为默认简历。
3. 如果当前用户已有默认简历，新建简历默认 `isDefault = 0`。
4. 不允许前端传 `userId`。
5. V1 不做简历文件解析。

### 5.3 GET /resumes/{id}

接口说明：简历详情。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/resumes/{id}` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `resume`、`resume_project` |

路径参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 简历 ID |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 简历 ID |
| `resumeName` | `string` | 简历名称 |
| `targetPosition` | `string` | 求职方向 |
| `skills` | `string` | 技能栈 |
| `workSummary` | `string` | 工作经历摘要 |
| `education` | `string` | 教育经历 |
| `isDefault` | `int` | 是否默认简历 |
| `status` | `int` | 简历状态 |
| `projects` | `array<ResumeProjectVO>` | 项目经历列表 |
| `createdAt` | `string` | 创建时间 |
| `updatedAt` | `string` | 更新时间 |

`projects` 字段包含：

| 字段 | 类型 | 说明 |
|---|---|---|
| `projectId` | `long` | 项目经历 ID |
| `projectName` | `string` | 项目名称 |
| `projectTime` | `string` | 项目时间 |
| `projectBackground` | `string` | 项目背景 |
| `techStack` | `string` | 技术栈 |
| `responsibility` | `string` | 个人职责 |
| `coreFeatures` | `string` | 核心功能 |
| `technicalChallenges` | `string` | 技术难点 |
| `optimizationResult` | `string` | 优化成果 |
| `extraInfo` | `string` | 补充说明 |
| `sort` | `int` | 排序值，对应 `sort_order` |

字段兼容说明：

| 建议字段 | V1 当前字段 |
|---|---|
| `realName` | 简历表未设计，当前不返回 |
| `phone` | 简历表未设计，当前不返回 |
| `email` | 简历表未设计，当前不返回 |
| `yearsOfExperience` | 简历表未设计，当前不返回 |
| `selfEvaluation` | 简历表未设计，当前不返回 |
| `role` | 项目表未设计，可在 `responsibility` 中描述 |
| `startDate` / `endDate` | 项目表未设计，使用 `projectTime` 文本字段 |
| `description` | 对应 `projectBackground` |
| `highlight` | 可由 `technicalChallenges`、`optimizationResult`、`extraInfo` 共同表达 |

业务规则：

1. 只能查询当前用户自己的简历。
2. 不返回已删除简历。
3. 同时返回未删除项目经历列表。
4. 项目经历按 `sort_order` 升序、`updatedAt` 倒序作为兜底排序。

### 5.4 PUT /resumes/{id}

接口说明：修改简历。

| 项目 | 内容 |
|---|---|
| 请求方式 | `PUT` |
| URL | `/resumes/{id}` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `resume` |

请求参数同 `POST /resumes`。

业务规则：

1. 只能修改当前用户自己的简历。
2. 不允许修改 `userId`。
3. 不允许直接通过该接口修改 `isDefault`。
4. 默认简历需要通过 `PUT /resumes/{id}/default` 修改。
5. 不允许修改已删除简历。

### 5.5 DELETE /resumes/{id}

接口说明：删除简历。

| 项目 | 内容 |
|---|---|
| 请求方式 | `DELETE` |
| URL | `/resumes/{id}` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `resume`、`resume_project` |

业务规则：

1. 只能删除当前用户自己的简历。
2. 使用逻辑删除。
3. 删除简历后，该简历下项目经历也应逻辑删除或不再展示。
4. 如果删除的是默认简历，V1 推荐自动选择该用户最近更新的一份未删除简历作为新的默认简历。
5. 如果没有其他简历，则用户没有默认简历。
6. 已被历史面试使用的简历仍可删除，但历史面试应依赖 interview-service 保存的简历快照，不应强依赖当前简历表实时数据。

### 5.6 PUT /resumes/{id}/default

接口说明：设置默认简历。

| 项目 | 内容 |
|---|---|
| 请求方式 | `PUT` |
| URL | `/resumes/{id}/default` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `resume` |

响应参数：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 简历 ID |
| `isDefault` | `int` | 是否默认简历 |
| `updatedAt` | `string` | 更新时间 |

业务规则：

1. 只能设置当前用户自己的简历。
2. 目标简历必须存在且未删除。
3. 每个用户最多只能有一份默认简历。
4. 设置新默认简历时，需要取消该用户其他简历的默认状态。
5. 建议在 resume-service 本地事务中完成。

---

## 6. 用户端项目经历接口设计

### 6.1 POST /resumes/{resumeId}/projects

接口说明：新增项目经历。

| 项目 | 内容 |
|---|---|
| 请求方式 | `POST` |
| URL | `/resumes/{resumeId}/projects` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `resume`、`resume_project` |

路径参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `resumeId` | `long` | 是 | 简历 ID |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `projectName` | `string` | 是 | 项目名称 |
| `projectTime` | `string` | 否 | 项目时间，例如 `2024.01-2024.06` |
| `projectBackground` | `string` | 否 | 项目背景 |
| `techStack` | `string` | 否 | 技术栈 |
| `responsibility` | `string` | 否 | 个人职责 |
| `coreFeatures` | `string` | 否 | 核心功能 |
| `technicalChallenges` | `string` | 否 | 技术难点 |
| `optimizationResult` | `string` | 否 | 优化成果 |
| `extraInfo` | `string` | 否 | 补充说明 |
| `sort` | `int` | 否 | 排序值，对应 `sort_order` |

字段兼容说明：

| 建议字段 | V1 当前处理 |
|---|---|
| `role` | 项目表未设计，可写入 `responsibility` |
| `startDate` / `endDate` | 项目表未设计，使用 `projectTime` 文本字段 |
| `description` | 使用 `projectBackground` |
| `highlight` | 使用 `technicalChallenges`、`optimizationResult`、`extraInfo` 表达 |

响应参数：

| 字段 | 类型 | 说明 |
|---|---|---|
| `projectId` | `long` | 项目经历 ID |
| `resumeId` | `long` | 简历 ID |
| `createdAt` | `string` | 创建时间 |

业务规则：

1. 只能给当前用户自己的简历新增项目经历。
2. 简历必须存在且未删除。
3. `projectName` 不能为空。
4. V1 不限制项目数量，但建议前端控制在合理范围。

### 6.2 PUT /resumes/projects/{projectId}

接口说明：修改项目经历。

| 项目 | 内容 |
|---|---|
| 请求方式 | `PUT` |
| URL | `/resumes/projects/{projectId}` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `resume`、`resume_project` |

请求参数同 `POST /resumes/{resumeId}/projects`，但不包含 `resumeId`。

业务规则：

1. 只能修改当前用户自己简历下的项目经历。
2. 项目经历必须存在且未删除。
3. 不能通过该接口把项目转移到其他用户简历下。
4. 如果需要调整所属简历，V1 不提供，后续再扩展。

### 6.3 DELETE /resumes/projects/{projectId}

接口说明：删除项目经历。

| 项目 | 内容 |
|---|---|
| 请求方式 | `DELETE` |
| URL | `/resumes/projects/{projectId}` |
| 需要登录 | 是 |
| 管理员权限 | 否 |
| 关联表 | `resume`、`resume_project` |

业务规则：

1. 只能删除当前用户自己简历下的项目经历。
2. 使用逻辑删除。
3. 删除后简历详情不再展示该项目。

---

## 7. 内部 Feign 接口设计

### 7.1 GET /inner/resumes/{id}

调用方：interview-service。

用途：

1. 创建面试时查询简历详情。
2. 生成面试上下文。
3. 生成面试报告时可读取简历基础信息。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/inner/resumes/{id}` |
| 是否对前端暴露 | 否 |
| 关联表 | `resume`、`resume_project` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 简历 ID，路径参数 |
| `userId` | `long` | 是 | 建议作为 query 参数或请求头透传，用于校验简历归属 |
| `includeProjects` | `boolean` | 否 | 是否内嵌项目经历，默认可为 `false` |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `id` | `long` | 简历 ID |
| `userId` | `long` | 用户 ID |
| `resumeName` | `string` | 简历名称 |
| `targetPosition` | `string` | 求职方向 |
| `skills` | `string` | 技能栈 |
| `workSummary` | `string` | 工作经历摘要 |
| `education` | `string` | 教育经历 |
| `isDefault` | `int` | 是否默认简历 |
| `status` | `int` | 简历状态 |
| `projects` | `array<InnerResumeProjectVO>` | 项目经历，可选内嵌 |

业务规则：

1. 必须校验简历归属 `userId`。
2. 不允许 interview-service 查询其他用户简历。
3. 不返回已删除简历。
4. 内部接口仍需遵守业务规则。
5. 创建面试时建议 interview-service 保存必要的简历快照，避免后续简历修改影响历史面试回放。

### 7.2 GET /inner/resumes/{id}/projects

调用方：interview-service。

用途：

1. 查询简历下项目经历。
2. 生成项目深挖类面试问题上下文。

| 项目 | 内容 |
|---|---|
| 请求方式 | `GET` |
| URL | `/inner/resumes/{id}/projects` |
| 是否对前端暴露 | 否 |
| 关联表 | `resume`、`resume_project` |

请求参数：

| 参数 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 简历 ID，路径参数 |
| `userId` | `long` | 是 | 建议作为 query 参数或请求头透传，用于校验简历归属 |

响应字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `projectId` | `long` | 项目经历 ID |
| `resumeId` | `long` | 简历 ID |
| `projectName` | `string` | 项目名称 |
| `projectTime` | `string` | 项目时间 |
| `projectBackground` | `string` | 项目背景 |
| `techStack` | `string` | 技术栈 |
| `responsibility` | `string` | 个人职责 |
| `coreFeatures` | `string` | 核心功能 |
| `technicalChallenges` | `string` | 技术难点 |
| `optimizationResult` | `string` | 优化成果 |
| `extraInfo` | `string` | 补充说明 |
| `sort` | `int` | 排序值 |

业务规则：

1. 必须校验简历归属 `userId`。
2. 只返回未删除项目经历。
3. 不允许跨用户查询。

### 7.3 GET /inner/users/{userId}/default-resume

调用方：interview-service。

用途：创建面试时，如果前端未传 `resumeId`，可查询用户默认简历。

V1 定位：

1. V1 推荐创建面试时前端显式传 `resumeId`。
2. 该接口可选实现。
3. 不纳入第一版前后端联调清单。
4. 如果实现，必须只返回该 `userId` 的默认简历。
5. 如果没有默认简历，返回空或明确错误码。

---

## 8. DTO / VO 设计

### 8.1 用户端简历 DTO / VO

#### ResumeQueryRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `keyword` | `string` | 否 | 简历名称、求职方向关键字 |
| `pageNo` | `int` | 否 | 当前页码 |
| `pageSize` | `int` | 否 | 每页数量 |

#### ResumeListVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 简历 ID |
| `resumeName` | `string` | 是 | 简历名称 |
| `targetPosition` | `string` | 否 | 求职方向 |
| `skills` | `string` | 否 | 技能栈摘要 |
| `isDefault` | `int` | 是 | 是否默认简历 |
| `status` | `int` | 是 | 简历状态 |
| `projectCount` | `int` | 是 | 项目经历数量 |
| `updatedAt` | `string` | 是 | 更新时间 |
| `createdAt` | `string` | 是 | 创建时间 |

#### ResumeSaveRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `resumeName` | `string` | 是 | 简历名称 |
| `targetPosition` | `string` | 否 | 求职方向 |
| `skills` | `string` | 是 | 技能栈 |
| `workSummary` | `string` | 否 | 工作经历摘要 |
| `education` | `string` | 否 | 教育经历 |

#### ResumeDetailVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 简历 ID |
| `resumeName` | `string` | 是 | 简历名称 |
| `targetPosition` | `string` | 否 | 求职方向 |
| `skills` | `string` | 否 | 技能栈 |
| `workSummary` | `string` | 否 | 工作经历摘要 |
| `education` | `string` | 否 | 教育经历 |
| `isDefault` | `int` | 是 | 是否默认简历 |
| `status` | `int` | 是 | 简历状态 |
| `projects` | `array<ResumeProjectVO>` | 否 | 项目经历列表 |
| `createdAt` | `string` | 是 | 创建时间 |
| `updatedAt` | `string` | 是 | 更新时间 |

#### SetDefaultResumeVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 简历 ID |
| `isDefault` | `int` | 是 | 是否默认简历 |
| `updatedAt` | `string` | 是 | 更新时间 |

### 8.2 项目经历 DTO / VO

#### ResumeProjectSaveRequest

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `projectName` | `string` | 是 | 项目名称 |
| `projectTime` | `string` | 否 | 项目时间 |
| `projectBackground` | `string` | 否 | 项目背景 |
| `techStack` | `string` | 否 | 技术栈 |
| `responsibility` | `string` | 否 | 个人职责 |
| `coreFeatures` | `string` | 否 | 核心功能 |
| `technicalChallenges` | `string` | 否 | 技术难点 |
| `optimizationResult` | `string` | 否 | 优化成果 |
| `extraInfo` | `string` | 否 | 补充说明 |
| `sort` | `int` | 否 | 排序值 |

#### ResumeProjectVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `projectId` | `long` | 是 | 项目经历 ID |
| `resumeId` | `long` | 是 | 简历 ID |
| `projectName` | `string` | 是 | 项目名称 |
| `projectTime` | `string` | 否 | 项目时间 |
| `projectBackground` | `string` | 否 | 项目背景 |
| `techStack` | `string` | 否 | 技术栈 |
| `responsibility` | `string` | 否 | 个人职责 |
| `coreFeatures` | `string` | 否 | 核心功能 |
| `technicalChallenges` | `string` | 否 | 技术难点 |
| `optimizationResult` | `string` | 否 | 优化成果 |
| `extraInfo` | `string` | 否 | 补充说明 |
| `sort` | `int` | 是 | 排序值 |
| `createdAt` | `string` | 是 | 创建时间 |
| `updatedAt` | `string` | 是 | 更新时间 |

### 8.3 内部 Feign DTO / VO

#### InnerResumeDetailVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 是 | 简历 ID |
| `userId` | `long` | 是 | 用户 ID |
| `resumeName` | `string` | 是 | 简历名称 |
| `targetPosition` | `string` | 否 | 求职方向 |
| `skills` | `string` | 否 | 技能栈 |
| `workSummary` | `string` | 否 | 工作经历摘要 |
| `education` | `string` | 否 | 教育经历 |
| `isDefault` | `int` | 是 | 是否默认简历 |
| `status` | `int` | 是 | 简历状态 |
| `projects` | `array<InnerResumeProjectVO>` | 否 | 项目经历列表，可选 |

#### InnerResumeProjectVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `projectId` | `long` | 是 | 项目经历 ID |
| `resumeId` | `long` | 是 | 简历 ID |
| `projectName` | `string` | 是 | 项目名称 |
| `projectTime` | `string` | 否 | 项目时间 |
| `projectBackground` | `string` | 否 | 项目背景 |
| `techStack` | `string` | 否 | 技术栈 |
| `responsibility` | `string` | 否 | 个人职责 |
| `coreFeatures` | `string` | 否 | 核心功能 |
| `technicalChallenges` | `string` | 否 | 技术难点 |
| `optimizationResult` | `string` | 否 | 优化成果 |
| `extraInfo` | `string` | 否 | 补充说明 |
| `sort` | `int` | 是 | 排序值 |

#### InnerDefaultResumeVO

| 字段名 | 类型 | 是否必填 | 说明 |
|---|---|---:|---|
| `id` | `long` | 否 | 默认简历 ID，没有默认简历时可为空 |
| `userId` | `long` | 是 | 用户 ID |
| `resumeName` | `string` | 否 | 简历名称 |
| `targetPosition` | `string` | 否 | 求职方向 |
| `isDefault` | `int` | 否 | 是否默认简历 |
| `status` | `int` | 否 | 简历状态 |

---

## 9. 参数校验规则

| 参数 | 校验规则 |
|---|---|
| `pageNo` | 最小 1 |
| `pageSize` | 最大 100 |
| `keyword` | 最大 100 |
| `resumeName` | 不能为空，最大 128 |
| `targetPosition` | 最大 128 |
| `skills` | 不能为空，按数据库 `text` 字段控制，接口建议最大 5000 |
| `workSummary` | 按数据库 `text` 字段控制，接口建议最大 5000 |
| `education` | 按数据库 `text` 字段控制，接口建议最大 3000 |
| `projectName` | 不能为空，最大 128 |
| `projectTime` | 最大 128 |
| `projectBackground` | 按数据库 `text` 字段控制，接口建议最大 5000 |
| `techStack` | 按数据库 `text` 字段控制，接口建议最大 3000 |
| `responsibility` | 按数据库 `text` 字段控制，接口建议最大 5000 |
| `coreFeatures` | 按数据库 `text` 字段控制，接口建议最大 5000 |
| `technicalChallenges` | 按数据库 `text` 字段控制，接口建议最大 5000 |
| `optimizationResult` | 按数据库 `text` 字段控制，接口建议最大 3000 |
| `extraInfo` | 按数据库 `text` 字段控制，接口建议最大 3000 |
| `sort` | 最小 0 |

字段差异说明：

| 建议校验项 | V1 当前处理 |
|---|---|
| `realName` 最大 50 | 简历表未设计，不在简历接口校验 |
| `phone` 最大 30 | 简历表未设计，不在简历接口校验 |
| `email` 邮箱格式最大 100 | 简历表未设计，由 user-service 用户资料接口校验 |
| `yearsOfExperience` 不能小于 0 | 简历表未设计，由创建面试接口校验 |
| `selfEvaluation` | 简历表未设计，不在简历接口校验 |
| `startDate` / `endDate` | 项目表未设计，V1 使用 `projectTime` 文本字段 |

---

## 10. 错误码设计

简历模块错误码范围：`44000-44999`。

| code | message | 说明 |
|---:|---|---|
| `44001` | 简历不存在 | 简历 ID 不存在 |
| `44002` | 无权访问该简历 | 简历不属于当前用户 |
| `44003` | 简历已删除 | 简历已逻辑删除 |
| `44004` | 默认简历不存在 | 查询默认简历为空 |
| `44005` | 项目经历不存在 | 项目经历 ID 不存在 |
| `44006` | 无权访问该项目经历 | 项目经历不属于当前用户简历 |
| `44007` | 项目经历已删除 | 项目经历已逻辑删除 |
| `44008` | 日期范围非法 | V1 当前不使用起止日期，后续扩展时使用 |
| `44009` | 简历名称不能为空 | `resumeName` 为空 |
| `44010` | 项目名称不能为空 | `projectName` 为空 |
| `44011` | 默认简历设置失败 | 设置默认简历事务失败 |
| `44012` | 简历状态不可用 | 简历已停用或不可用于创建面试 |

复用通用错误码：

| code | message | 说明 |
|---:|---|---|
| `40001` | 参数校验失败 | 请求参数不合法 |
| `41000` | 未登录 | 未携带 Token 或 Token 无效 |
| `41003` | 无访问权限 | 非法访问或权限不足 |

---

## 11. 数据库表关系

本模块涉及：

| 表名 | 说明 |
|---|---|
| `resume` | 简历主体信息 |
| `resume_project` | 简历项目经历 |

关系说明：

1. resume-service 是 `resume`、`resume_project` 表的数据归属服务。
2. interview-service 不直接访问简历表。
3. `resume` 与 `resume_project` 是一对多关系。
4. 用户与 `resume` 是一对多关系，resume-service 只保存 `user_id`，不直接查询 user-service 用户表。
5. 默认简历通过 `resume.is_default` 维护。
6. 删除简历后，项目经历需要同步处理展示状态或逻辑删除。
7. 历史面试不应依赖实时简历数据，interview-service 应保存必要快照。

关系示意：

```text
user_id
  └── resume
        └── resume_project
```

---

## 12. 前端页面对应关系

| 前端页面 | 关联接口 |
|---|---|
| 简历列表页 | `GET /resumes`、`DELETE /resumes/{id}`、`PUT /resumes/{id}/default` |
| 简历新增页 | `POST /resumes` |
| 简历编辑页 | `GET /resumes/{id}`、`PUT /resumes/{id}` |
| 简历详情页 | `GET /resumes/{id}` |
| 项目经历编辑区域 | `POST /resumes/{resumeId}/projects`、`PUT /resumes/projects/{projectId}`、`DELETE /resumes/projects/{projectId}` |
| 创建面试页 | 用户端选择简历；interview-service 内部调用 `/inner/resumes/**` |
| AI 面试房间页 | 间接使用简历上下文 |
| 面试报告页 | 间接使用 interview-service 保存的简历快照 |

---

## 13. 注意事项

1. V1 简历只支持手动录入。
2. V1 不做文件上传。
3. V1 不做 PDF / Word 解析。
4. V1 不做 AI 简历优化。
5. V1 不做简历模板导出。
6. 用户只能操作自己的简历。
7. 默认简历每个用户最多一份。
8. 删除简历使用逻辑删除。
9. 删除默认简历后需要处理新的默认简历。
10. 项目经历必须归属于当前用户自己的简历。
11. `/inner/resumes/**` 不对前端暴露。
12. 创建面试时 interview-service 应保存必要简历快照。
13. V1 推荐创建面试时前端显式传 `resumeId`。
14. 管理端 V1 暂不设计简历管理接口。
15. 当前简历接口字段以 `resume`、`resume_project` 表设计为准，不强行加入数据库未设计字段。
