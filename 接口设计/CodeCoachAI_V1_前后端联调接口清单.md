# CodeCoachAI V1 前后端联调接口清单

## 1. 文档说明

| 项目 | 内容 |
|---|---|
| 文档名称 | CodeCoachAI V1 前后端联调接口清单 |
| 项目阶段 | V1 |
| 文档用途 | 指导 V1 前后端接口联调、接口优先级、页面接口映射和开发顺序规划 |
| 最高依据 | `接口设计/CodeCoachAI_V1_接口设计总览.md` 及各模块接口设计文档 |
| 适用对象 | 后端开发、前端开发、接口联调、自测验收 |

参考文档：

1. `接口设计/CodeCoachAI_V1_接口设计总览.md`
2. `接口设计/CodeCoachAI_V1_认证用户接口设计.md`
3. `接口设计/CodeCoachAI_V1_题库接口设计.md`
4. `接口设计/CodeCoachAI_V1_简历接口设计.md`
5. `接口设计/CodeCoachAI_V1_面试接口设计.md`
6. `接口设计/CodeCoachAI_V1_AI接口设计.md`
7. `接口设计/CodeCoachAI_V1_系统管理接口设计.md`
8. `接口设计/CodeCoachAI_V1_OpenFeign内部接口总表.md`
9. `架构设计/CodeCoachAI_V1_微服务架构设计.md`
10. `PRD/CodeCoachAI_PRD_V1_微服务分层架构版.md`

本文档只整理：

1. 前端用户端需要直接调用的接口。
2. 管理端需要直接调用的接口。
3. 后端内部 Spring Cloud OpenFeign 接口的边界说明和联调依赖。

边界说明：

1. `/inner/**` 接口不进入前端直接调用清单，只在后端内部接口章节单独列出。
2. 前端所有请求统一经过 Gateway。
3. Gateway 不对外暴露 `/inner/**`。
4. AI 能力接口只通过 `/inner/ai/**` 给 interview-service 内部调用。
5. 前端不调用 `/ai/**`，V1 不提供用户端 `/ai/**` 路由。
6. 管理端可以访问 `/admin/ai/**`，仅用于 Prompt 模板和 AI 调用日志管理。
7. 本文档不生成代码，不生成接口实现，不生成 DTO / VO 类，不生成 SQL。

---

## 2. 联调优先级定义

| 优先级 | 定义 | 联调策略 |
|---|---|---|
| P0 | V1 核心闭环必须接口，不完成则 AI 模拟面试主流程无法跑通 | 第一阶段优先完成并端到端联通 |
| P1 | V1 体验增强接口，不影响主流程，但影响页面完整性和使用体验 | 第二阶段补齐 |
| P2 | 管理端、统计类或可选接口，可在第一轮开发中 mock、简化或后置 | 第三阶段处理 |

补充说明：

1. 第一阶段开发优先完成 P0。
2. 第二阶段补齐 P1。
3. 第三阶段处理 P2。
4. `/inner/**` 虽然不由前端调用，但部分内部接口属于 P0 后端核心链路。
5. P0 前端接口联调前，需要先保证对应后端内部 Spring Cloud OpenFeign P0 链路可用。

---

## 3. 前端页面与接口总览

### 3.1 用户端页面

| 页面 | 接口 | 方法 | 服务 | 优先级 | 说明 |
|---|---|---|---|---|---|
| 登录页 | `/auth/login` | POST | auth-service | P0 | 用户或管理员登录 |
| 登录页 | `/auth/current-user` | GET | auth-service | P0 | 刷新页面后恢复登录态 |
| 注册页 | `/auth/register` | POST | auth-service | P0 | 注册普通用户 |
| 用户首页 / 工作台 | `/auth/current-user` | GET | auth-service | P0 | 展示当前用户基础信息 |
| 用户首页 / 工作台 | `/users/overview` | GET | user-service | P2 | 学习概览，可先简化返回 |
| 个人中心页 | `/users/profile` | GET | user-service | P1 | 查询个人资料 |
| 个人中心页 | `/users/profile` | PUT | user-service | P1 | 修改昵称、头像、邮箱 |
| 个人中心页 | `/users/password` | PUT | user-service | P1 | 修改密码 |
| 题库列表页 | `/questions` | GET | question-service | P0 | 查询题目列表 |
| 题目详情页 | `/questions/{id}` | GET | question-service | P0 | 查看题目详情 |
| 题目详情页 | `/questions/{id}/answers` | POST | question-service | P0 | 提交刷题答案，不调用 AI 评分 |
| 题目详情页 | `/questions/{id}/favorite` | POST | question-service | P1 | 收藏题目 |
| 题目详情页 | `/questions/{id}/favorite` | DELETE | question-service | P1 | 取消收藏 |
| 题目详情页 | `/questions/{id}/mastery` | PUT | question-service | P1 | 修改掌握状态 |
| 错题本页 | `/questions/wrong-records` | GET | question-service | P1 | 查询当前用户错题 |
| 收藏题目页 | `/questions/favorites` | GET | question-service | P1 | 查询当前用户收藏题 |
| 简历列表页 | `/resumes` | GET | resume-service | P0 | 查询当前用户简历 |
| 简历列表页 | `/resumes` | POST | resume-service | P0 | 新增简历 |
| 简历列表页 | `/resumes/{id}` | DELETE | resume-service | P1 | 删除简历 |
| 简历列表页 | `/resumes/{id}/default` | PUT | resume-service | P1 | 设置默认简历 |
| 简历编辑页 | `/resumes/{id}` | GET | resume-service | P0 | 查询简历详情和项目经历 |
| 简历编辑页 | `/resumes/{id}` | PUT | resume-service | P0 | 修改简历基础信息 |
| 项目经历编辑区 | `/resumes/{resumeId}/projects` | POST | resume-service | P0 | 新增项目经历 |
| 项目经历编辑区 | `/resumes/projects/{projectId}` | PUT | resume-service | P0 | 修改项目经历 |
| 项目经历编辑区 | `/resumes/projects/{projectId}` | DELETE | resume-service | P0 | 删除项目经历，保证项目编辑闭环 |
| 创建面试页 | `/resumes` | GET | resume-service | P0 | 选择用于面试的简历 |
| 创建面试页 | `/interviews` | POST | interview-service | P0 | 创建 AI 模拟面试 |
| AI 面试房间页 | `/interviews/{id}/start` | POST | interview-service | P0 | 开始面试 |
| AI 面试房间页 | `/interviews/{id}/current` | GET | interview-service | P0 | 获取当前题或追问题 |
| AI 面试房间页 | `/interviews/{id}/answer` | POST | interview-service | P0 | 提交回答，返回评分、点评和 nextAction |
| AI 面试房间页 | `/interviews/{id}/finish` | POST | interview-service | P0 | 结束面试，可同步生成报告 |
| 面试历史页 | `/interviews` | GET | interview-service | P0 | 查询当前用户面试历史 |
| 面试详情页 | `/interviews/{id}` | GET | interview-service | P1 | 查看面试阶段、问题和回答 |
| 面试报告页 | `/interviews/{id}/report` | GET | interview-service | P0 | 查看面试报告 |
| 面试报告页 | `/interviews/{id}/report/retry` | POST | interview-service | P1 | 报告生成失败或未生成时重试 |

### 3.2 管理端页面

| 页面 | 接口 | 方法 | 服务 | 优先级 | 说明 |
|---|---|---|---|---|---|
| 管理端登录页 | `/auth/login` | POST | auth-service | P0 | 管理员登录，登录后需要具备 ADMIN 角色 |
| 管理端登录页 | `/auth/current-user` | GET | auth-service | P0 | 获取当前管理员信息 |
| 管理端首页 | `/admin/system/overview` | GET | system-service | P2 | 简化统计，可先 mock 或返回默认值 |
| 用户管理页 | `/admin/users` | GET | user-service | P2 | 用户分页列表 |
| 用户管理页 | `/admin/users/{id}/status` | PUT | user-service | P2 | 启用或禁用用户 |
| 角色列表，V1 简化 | `/admin/roles` | GET | user-service | P2 | 查询 USER、ADMIN 基础角色 |
| 题目管理页 | `/admin/questions` | GET | question-service | P1 | 管理端题目列表 |
| 题目管理页 | `/admin/questions` | POST | question-service | P1 | 新增题目 |
| 题目管理页 | `/admin/questions/{id}` | PUT | question-service | P1 | 修改题目 |
| 题目管理页 | `/admin/questions/{id}/status` | PUT | question-service | P1 | 启用或禁用题目 |
| 题目管理页 | `/admin/questions/{id}` | DELETE | question-service | P1 | 删除题目 |
| 分类管理页 | `/admin/question-categories` | GET | question-service | P1 | 分类列表 |
| 分类管理页 | `/admin/question-categories` | POST | question-service | P1 | 新增分类 |
| 分类管理页 | `/admin/question-categories/{id}` | PUT | question-service | P1 | 修改分类 |
| 分类管理页 | `/admin/question-categories/{id}/status` | PUT | question-service | P1 | 启用或禁用分类 |
| 分类管理页 | `/admin/question-categories/{id}` | DELETE | question-service | P1 | 删除分类 |
| 标签管理页 | `/admin/question-tags` | GET | question-service | P1 | 标签列表 |
| 标签管理页 | `/admin/question-tags` | POST | question-service | P1 | 新增标签 |
| 标签管理页 | `/admin/question-tags/{id}` | PUT | question-service | P1 | 修改标签 |
| 标签管理页 | `/admin/question-tags/{id}/status` | PUT | question-service | P1 | 启用或禁用标签 |
| 标签管理页 | `/admin/question-tags/{id}` | DELETE | question-service | P1 | 删除标签 |
| 问题组管理页 | `/admin/question-groups` | GET | question-service | P1 | 问题组列表 |
| 问题组管理页 | `/admin/question-groups` | POST | question-service | P1 | 新增问题组 |
| 问题组管理页 | `/admin/question-groups/{id}` | PUT | question-service | P1 | 修改问题组 |
| 问题组管理页 | `/admin/question-groups/{id}/status` | PUT | question-service | P1 | 启用或禁用问题组 |
| 问题组管理页 | `/admin/question-groups/{id}` | DELETE | question-service | P1 | 删除问题组 |
| Prompt 模板管理页 | `/admin/ai/prompts` | GET | ai-service | P1 | Prompt 模板列表 |
| Prompt 模板管理页 | `/admin/ai/prompts/{id}` | GET | ai-service | P1 | Prompt 模板详情 |
| Prompt 模板管理页 | `/admin/ai/prompts` | POST | ai-service | P1 | 新增 Prompt 模板 |
| Prompt 模板管理页 | `/admin/ai/prompts/{id}` | PUT | ai-service | P1 | 修改 Prompt 模板 |
| Prompt 模板管理页 | `/admin/ai/prompts/{id}/status` | PUT | ai-service | P1 | 启用或禁用 Prompt 模板 |
| Prompt 模板管理页 | `/admin/ai/prompts/{id}` | DELETE | ai-service | P2 | 删除 Prompt 模板，可后置 |
| AI 调用日志页 | `/admin/ai/logs` | GET | ai-service | P2 | AI 调用日志列表 |
| AI 调用日志页 | `/admin/ai/logs/{id}` | GET | ai-service | P2 | AI 调用日志详情 |
| 系统配置页 | `/admin/configs` | GET | system-service | P2 | 系统配置列表 |
| 系统配置页 | `/admin/configs` | POST | system-service | P2 | 新增非核心配置 |
| 系统配置页 | `/admin/configs/{key}` | GET | system-service | P2 | 系统配置详情 |
| 系统配置页 | `/admin/configs/{key}` | PUT | system-service | P2 | 修改系统配置 |
| 系统配置页 | `/admin/configs/{key}/status` | PUT | system-service | P2 | 启用或禁用系统配置 |
| 系统配置页 | `/admin/configs/{key}` | DELETE | system-service | P2 | 删除非核心配置 |
| 管理端布局菜单 | `/admin/menus` | GET | system-service | P2 | V1 可选，前端可先使用静态路由 |

---

## 4. 用户端接口清单

### 4.1 auth-service 用户认证接口

| 优先级 | 方法 | URL | 接口名称 | 前端页面 | 说明 |
|---|---|---|---|---|---|
| P0 | POST | `/auth/register` | 用户注册 | 注册页 | 创建普通 USER 用户 |
| P0 | POST | `/auth/login` | 用户登录 | 登录页、管理端登录页 | 登录成功返回 Token 和用户信息 |
| P1 | POST | `/auth/logout` | 用户退出 | 用户端布局、管理端布局 | 当前用户退出登录 |
| P0 | GET | `/auth/current-user` | 当前用户 | 登录态恢复、布局、权限判断 | 获取当前登录用户和角色 |
| P2 | POST | `/auth/refresh-token` | 刷新 Token | 全局请求层 | V1 可选，不纳入第一轮核心联调 |

### 4.2 user-service 用户接口

| 优先级 | 方法 | URL | 接口名称 | 前端页面 | 说明 |
|---|---|---|---|---|---|
| P1 | GET | `/users/profile` | 查询个人资料 | 个人中心页 | 查询当前用户资料 |
| P1 | PUT | `/users/profile` | 修改个人资料 | 个人中心页 | 修改昵称、头像、邮箱等 |
| P1 | PUT | `/users/password` | 修改密码 | 个人中心页 | 修改当前用户密码 |
| P2 | GET | `/users/overview` | 用户学习概览 | 用户首页 / 工作台 | 可第一阶段简化返回或暂不展示 |

`GET /users/overview` 说明：

1. 完整设计可由 user-service 通过 Spring Cloud OpenFeign 聚合 question-service、resume-service、interview-service 的用户统计。
2. V1 第一阶段可简化实现，只返回 user-service 可直接获得的数据，其他统计字段返回 0 或前端暂不展示。
3. 该接口不阻塞 AI 模拟面试核心闭环。

### 4.3 question-service 用户题库接口

| 优先级 | 方法 | URL | 接口名称 | 前端页面 | 说明 |
|---|---|---|---|---|---|
| P0 | GET | `/questions` | 题目列表 | 题库列表页 | 按分类、标签、难度查询 |
| P0 | GET | `/questions/{id}` | 题目详情 | 题目详情页 | 查看题目内容、答案、解析 |
| P0 | POST | `/questions/{id}/answers` | 提交答案 | 题目详情页 | 保存用户刷题记录、错题和掌握状态 |
| P1 | POST | `/questions/{id}/favorite` | 收藏题目 | 题目详情页 | 收藏当前题目 |
| P1 | DELETE | `/questions/{id}/favorite` | 取消收藏 | 题目详情页、收藏题目页 | 取消收藏当前题目 |
| P1 | GET | `/questions/favorites` | 收藏题目列表 | 收藏题目页 | 查询当前用户收藏题 |
| P1 | GET | `/questions/wrong-records` | 错题列表 | 错题本页 | 查询当前用户错题 |
| P1 | PUT | `/questions/{id}/mastery` | 修改掌握状态 | 题目详情页、错题本页 | 设置 MASTERED / VAGUE / UNKNOWN |

注意：

1. 用户提交答案接口必须是 `POST /questions/{id}/answers`。
2. 错题列表接口必须是 `GET /questions/wrong-records`。
3. 刷题答案 V1 不调用 AI 评分，只保存用户答题记录、错题、收藏和掌握状态。
4. question-service 负责题库、刷题、错题、收藏、掌握状态。

### 4.4 resume-service 用户简历接口

| 优先级 | 方法 | URL | 接口名称 | 前端页面 | 说明 |
|---|---|---|---|---|---|
| P0 | GET | `/resumes` | 简历列表 | 简历列表页、创建面试页 | 查询当前用户简历 |
| P0 | POST | `/resumes` | 新增简历 | 简历列表页、简历编辑页 | 手动创建简历 |
| P0 | GET | `/resumes/{id}` | 简历详情 | 简历编辑页、创建面试页 | 查询简历基础信息和项目经历 |
| P0 | PUT | `/resumes/{id}` | 修改简历 | 简历编辑页 | 修改简历基础信息 |
| P1 | DELETE | `/resumes/{id}` | 删除简历 | 简历列表页 | 删除当前用户自己的简历 |
| P1 | PUT | `/resumes/{id}/default` | 设置默认简历 | 简历列表页 | 设置默认简历 |
| P0 | POST | `/resumes/{resumeId}/projects` | 新增项目经历 | 项目经历编辑区 | 项目深挖面试依赖项目经历 |
| P0 | PUT | `/resumes/projects/{projectId}` | 修改项目经历 | 项目经历编辑区 | 修改项目经历 |
| P0 | DELETE | `/resumes/projects/{projectId}` | 删除项目经历 | 项目经历编辑区 | 保证项目经历编辑闭环 |

说明：

1. resume-service 负责简历和项目经历。
2. V1 不做文件上传解析。
3. V1 不做简历 AI 优化。
4. 创建面试时，interview-service 通过内部接口读取简历和项目经历，并保存必要快照。

### 4.5 interview-service 用户面试接口

| 优先级 | 方法 | URL | 接口名称 | 前端页面 | 说明 |
|---|---|---|---|---|---|
| P0 | POST | `/interviews` | 创建面试 | 创建面试页 | 创建面试会话和阶段 |
| P0 | POST | `/interviews/{id}/start` | 开始面试 | AI 面试房间页 | 开始面试并准备首题 |
| P0 | GET | `/interviews/{id}/current` | 当前问题 | AI 面试房间页 | 获取当前题、追问题或阶段状态 |
| P0 | POST | `/interviews/{id}/answer` | 提交回答 | AI 面试房间页 | 保存回答，调用 AI 评分，返回 nextAction |
| P0 | POST | `/interviews/{id}/finish` | 结束面试 | AI 面试房间页 | 结束问答流程，可同步生成报告 |
| P1 | POST | `/interviews/{id}/report/retry` | 重试生成报告 | 面试报告页 | 报告失败或未生成时重试 |
| P0 | GET | `/interviews` | 面试历史 | 面试历史页 | 查询当前用户历史面试 |
| P1 | GET | `/interviews/{id}` | 面试详情 | 面试详情页 | 查看阶段、问题、回答和评分 |
| P0 | GET | `/interviews/{id}/report` | 面试报告 | 面试报告页 | 查看面试报告 |

`POST /interviews/{id}/answer` 必须返回：

| 字段 | 说明 |
|---|---|
| AI 评分 | 本次回答评分，建议 0-100 |
| AI 点评 | 对用户回答的点评、优点、不足和建议 |
| `nextAction` | 下一步动作 |
| 下一题或追问题 | 根据 `nextAction` 返回下一题、追问题或结束提示 |

`nextAction` 枚举：

| 值 | 说明 |
|---|---|
| `FOLLOW_UP` | 继续追问 |
| `NEXT_QUESTION` | 进入下一题 |
| `NEXT_STAGE` | 进入下一阶段 |
| `FINISH` | 问答流程可结束 |

面试和报告规则：

1. `POST /interviews/{id}/answer` 只负责保存用户回答、调用 AI 评分、生成追问或下一步动作。
2. `POST /interviews/{id}/answer` 不直接生成面试报告。
3. 当 `nextAction = FINISH` 时，前端继续调用 `POST /interviews/{id}/finish`。
4. `POST /interviews/{id}/finish` 负责结束面试流程，V1 可同步生成报告。
5. 报告生成成功时，`reportStatus = GENERATED`。
6. 报告生成失败时，`reportStatus = FAILED`，并记录失败原因。
7. 面试主状态仍建议保持 `COMPLETED`，表示问答流程已完成。
8. 报告生成失败后，可调用 `POST /interviews/{id}/report/retry`。
9. `POST /interviews/{id}/report/retry` 不重复生成多份有效报告。

---

## 5. 管理端接口清单

### 5.1 user-service 管理端接口

| 优先级 | 方法 | URL | 接口名称 | 管理端页面 | 说明 |
|---|---|---|---|---|---|
| P2 | GET | `/admin/users` | 管理端用户列表 | 用户管理页 | 用户分页查询 |
| P2 | PUT | `/admin/users/{id}/status` | 修改用户状态 | 用户管理页 | 启用或禁用用户 |
| P2 | GET | `/admin/roles` | 角色列表 | 角色列表、用户管理页 | 查询 USER、ADMIN 基础角色 |

说明：

1. `/admin/roles` 归属 user-service，不归 system-service。
2. user-service 管理 `sys_user`、`sys_role`、`sys_user_role`。
3. V1 只需要 `USER`、`ADMIN` 基础角色。
4. V1 不设计复杂 RBAC、菜单权限、按钮权限和数据权限。

### 5.2 question-service 管理端题库接口

题目接口：

| 优先级 | 方法 | URL | 接口名称 | 说明 |
|---|---|---|---|---|
| P1 | GET | `/admin/questions` | 管理端题目列表 | 题目分页查询 |
| P1 | POST | `/admin/questions` | 新增题目 | 创建题目 |
| P1 | PUT | `/admin/questions/{id}` | 修改题目 | 编辑题目 |
| P1 | PUT | `/admin/questions/{id}/status` | 启用 / 禁用题目 | 修改题目状态 |
| P1 | DELETE | `/admin/questions/{id}` | 删除题目 | 逻辑删除题目 |

分类接口：

| 优先级 | 方法 | URL | 接口名称 | 说明 |
|---|---|---|---|---|
| P1 | GET | `/admin/question-categories` | 分类列表 | 分类查询 |
| P1 | POST | `/admin/question-categories` | 新增分类 | 创建分类 |
| P1 | PUT | `/admin/question-categories/{id}` | 修改分类 | 编辑分类 |
| P1 | PUT | `/admin/question-categories/{id}/status` | 启用 / 禁用分类 | 修改分类状态 |
| P1 | DELETE | `/admin/question-categories/{id}` | 删除分类 | 删除分类 |

标签接口：

| 优先级 | 方法 | URL | 接口名称 | 说明 |
|---|---|---|---|---|
| P1 | GET | `/admin/question-tags` | 标签列表 | 标签查询 |
| P1 | POST | `/admin/question-tags` | 新增标签 | 创建标签 |
| P1 | PUT | `/admin/question-tags/{id}` | 修改标签 | 编辑标签 |
| P1 | PUT | `/admin/question-tags/{id}/status` | 启用 / 禁用标签 | 修改标签状态 |
| P1 | DELETE | `/admin/question-tags/{id}` | 删除标签 | 删除标签 |

问题组接口：

| 优先级 | 方法 | URL | 接口名称 | 说明 |
|---|---|---|---|---|
| P1 | GET | `/admin/question-groups` | 问题组列表 | 问题组查询 |
| P1 | POST | `/admin/question-groups` | 新增问题组 | 创建问题组 |
| P1 | PUT | `/admin/question-groups/{id}` | 修改问题组 | 编辑问题组 |
| P1 | PUT | `/admin/question-groups/{id}/status` | 启用 / 禁用问题组 | 修改问题组状态 |
| P1 | DELETE | `/admin/question-groups/{id}` | 删除问题组 | 删除问题组 |

说明：

1. 分类、标签、问题组、题目管理是 V1 题库基础数据能力。
2. 如果第一阶段使用初始化数据，管理端 CRUD 可以后置到 P1。
3. 如果第一阶段没有初始化题库数据，则至少需要先完成题库管理端新增和启用能力。
4. status 接口用于启用 / 禁用，不建议前端通过普通修改接口隐式改变状态。

### 5.3 ai-service 管理端接口

Prompt 模板接口：

| 优先级 | 方法 | URL | 接口名称 | 说明 |
|---|---|---|---|---|
| P1 | GET | `/admin/ai/prompts` | Prompt 模板列表 | 分页查询 Prompt |
| P1 | GET | `/admin/ai/prompts/{id}` | Prompt 模板详情 | 查看模板内容、变量和状态 |
| P1 | POST | `/admin/ai/prompts` | 新增 Prompt 模板 | 新增模板 |
| P1 | PUT | `/admin/ai/prompts/{id}` | 修改 Prompt 模板 | 编辑模板 |
| P1 | PUT | `/admin/ai/prompts/{id}/status` | 启用 / 禁用 Prompt 模板 | 启用当前模板并处理同类型启用冲突 |
| P2 | DELETE | `/admin/ai/prompts/{id}` | 删除 Prompt 模板 | 可后置，优先使用禁用 |

AI 调用日志接口：

| 优先级 | 方法 | URL | 接口名称 | 说明 |
|---|---|---|---|---|
| P2 | GET | `/admin/ai/logs` | AI 调用日志列表 | 管理端分页筛选 |
| P2 | GET | `/admin/ai/logs/{id}` | AI 调用日志详情 | 查看请求、Prompt、响应和错误信息 |

说明：

1. Prompt 模板列表、详情、修改、启用禁用为 P1。
2. AI 日志列表、详情为 P2，可在核心链路跑通后补齐。
3. 如果第一阶段使用初始化 Prompt，也可以先后置管理端 Prompt CRUD。
4. AI 面试链路必须能读取已启用 Prompt。
5. 管理端 `/admin/ai/**` 需要 `ADMIN` 角色。
6. 用户端前端永远不直接访问 AI 能力接口。

### 5.4 system-service 管理端接口

管理端首页：

| 优先级 | 方法 | URL | 接口名称 | 说明 |
|---|---|---|---|---|
| P2 | GET | `/admin/system/overview` | 管理端首页统计 | 可先返回简化统计或 mock 数据 |

系统配置：

| 优先级 | 方法 | URL | 接口名称 | 说明 |
|---|---|---|---|---|
| P2 | GET | `/admin/configs` | 系统配置列表 | 配置分页查询 |
| P2 | POST | `/admin/configs` | 新增系统配置 | 新增非核心配置 |
| P2 | GET | `/admin/configs/{key}` | 系统配置详情 | 配置编辑页回显 |
| P2 | PUT | `/admin/configs/{key}` | 修改系统配置 | 修改可编辑配置 |
| P2 | PUT | `/admin/configs/{key}/status` | 启用 / 禁用系统配置 | 修改配置状态 |
| P2 | DELETE | `/admin/configs/{key}` | 删除系统配置 | 删除非核心配置 |

可选菜单：

| 优先级 | 方法 | URL | 接口名称 | 说明 |
|---|---|---|---|---|
| P2 | GET | `/admin/menus` | 管理端菜单列表 | V1 可选，如果使用前端静态路由可以不实现 |

说明：

1. `GET /admin/system/overview` 可先返回简化统计或 mock 数据。
2. 系统配置相关接口均为 P2，不阻塞核心面试闭环。
3. `GET /admin/menus` 为 V1 可选，不纳入核心联调。
4. system-service 不负责 `/admin/roles`。
5. system-service V1 主要负责 `system_config` 和管理端首页简化统计。
6. 敏感配置不应明文返回，API Key 不建议通过普通接口管理明文。

---

## 6. 后端内部 OpenFeign 接口清单

本章节只列出后端内部 Spring Cloud OpenFeign 接口，不纳入前端直接调用清单。

### 6.1 auth-service 调 user-service

| 调用方 | 被调用方 | 方法 | URL | 优先级 | 用途 |
|---|---|---|---|---|---|
| auth-service | user-service | GET | `/inner/users/by-username` | P0 | 登录时查询用户认证信息 |
| auth-service | user-service | POST | `/inner/users` | P0 | 注册时创建普通用户 |
| auth-service | user-service | GET | `/inner/users/{id}/roles` | P0 | 查询用户角色 |
| auth-service | user-service | GET | `/inner/users/{id}` | P1 | 内部查询用户基础信息，可选 |

### 6.2 interview-service 调 question-service

| 调用方 | 被调用方 | 方法 | URL | 优先级 | 用途 |
|---|---|---|---|---|---|
| interview-service | question-service | POST | `/inner/questions/pick-for-interview` | P0 | 创建面试或进入下一阶段时抽题 |
| interview-service | question-service | GET | `/inner/questions/{id}` | P0 | 查询题目详情，用于题目快照和 AI 上下文 |
| interview-service | question-service | POST | `/inner/questions/recommend-for-report` | P1 | 报告推荐题，V1 可选 |

### 6.3 interview-service 调 resume-service

| 调用方 | 被调用方 | 方法 | URL | 优先级 | 用途 |
|---|---|---|---|---|---|
| interview-service | resume-service | GET | `/inner/resumes/{id}` | P0 | 创建面试时查询简历详情 |
| interview-service | resume-service | GET | `/inner/resumes/{id}/projects` | P0 | 查询简历项目经历 |
| interview-service | resume-service | GET | `/inner/users/{userId}/default-resume` | P1 | 查询用户默认简历，V1 可选 |

### 6.4 interview-service 调 ai-service

| 调用方 | 被调用方 | 方法 | URL | 优先级 | 用途 |
|---|---|---|---|---|---|
| interview-service | ai-service | POST | `/inner/ai/interview/question` | P1 | 生成或润色面试问题 |
| interview-service | ai-service | POST | `/inner/ai/interview/evaluate` | P0 | 对用户回答进行评分和点评 |
| interview-service | ai-service | POST | `/inner/ai/interview/follow-up` | P0 | 生成动态追问 |
| interview-service | ai-service | POST | `/inner/ai/interview/report` | P0 | 生成面试报告内容 |

强调：

1. `/inner/ai/**` 接口不允许前端调用。
2. AI 能力只允许 interview-service 内部调用。
3. ai-service 不保存面试消息和报告。
4. interview-service 负责面试流程编排、消息保存和报告归属。

### 6.5 user-service 调其他服务，用户概览统计，可选

| 调用方 | 被调用方 | 方法 | URL | 优先级 | 用途 |
|---|---|---|---|---|---|
| user-service | question-service | GET | `/inner/questions/users/{userId}/statistics` | P2 | 查询当前用户刷题、错题、收藏、掌握状态统计 |
| user-service | resume-service | GET | `/inner/resumes/users/{userId}/statistics` | P2 | 查询当前用户简历数量和默认简历 |
| user-service | interview-service | GET | `/inner/interviews/users/{userId}/statistics` | P2 | 查询当前用户面试统计 |

说明：

1. 这些接口用于 `GET /users/overview` 聚合统计。
2. V1 第一阶段可先返回简化统计，不阻塞核心闭环。

### 6.6 system-service 调其他服务，管理端首页统计，可选

| 调用方 | 被调用方 | 建议接口 | 优先级 | 用途 |
|---|---|---|---|---|
| system-service | user-service | `GET /inner/users/statistics` | P2 | 用户统计 |
| system-service | question-service | `GET /inner/questions/statistics` | P2 | 题库统计 |
| system-service | resume-service | `GET /inner/resumes/statistics` | P2 | 简历统计 |
| system-service | interview-service | `GET /inner/interviews/statistics` | P2 | 面试统计 |
| system-service | ai-service | `GET /inner/ai/statistics` | P2 | AI 调用统计 |

说明：

1. 这些接口只允许 system-service 通过 Spring Cloud OpenFeign 内部调用。
2. 不对前端开放。
3. Gateway 不对外暴露。
4. `GET /admin/system/overview` 可先返回简化统计。
5. 这些接口不阻塞 P0 主链路。

---

## 7. P0 核心闭环接口顺序

建议按以下真实用户流程进行 P0 联调：

| 顺序 | 方法 | URL | 服务 | 页面 / 链路 | 说明 |
|---:|---|---|---|---|---|
| 1 | POST | `/auth/register` | auth-service | 注册页 | 注册普通用户 |
| 2 | POST | `/auth/login` | auth-service | 登录页 | 登录获取 Token |
| 3 | GET | `/auth/current-user` | auth-service | 全局登录态 | 获取当前用户和角色 |
| 4 | GET | `/questions` | question-service | 题库列表页 | 查询题目列表 |
| 5 | GET | `/questions/{id}` | question-service | 题目详情页 | 查询题目详情 |
| 6 | POST | `/questions/{id}/answers` | question-service | 题目详情页 | 提交刷题答案 |
| 7 | GET | `/resumes` | resume-service | 简历列表页 | 查询简历 |
| 8 | POST | `/resumes` | resume-service | 简历编辑页 | 新增简历 |
| 9 | GET | `/resumes/{id}` | resume-service | 简历编辑页 | 查询简历详情 |
| 10 | PUT | `/resumes/{id}` | resume-service | 简历编辑页 | 修改简历 |
| 11 | POST | `/resumes/{resumeId}/projects` | resume-service | 项目经历编辑区 | 新增项目经历 |
| 12 | PUT | `/resumes/projects/{projectId}` | resume-service | 项目经历编辑区 | 修改项目经历 |
| 13 | POST | `/interviews` | interview-service | 创建面试页 | 创建面试 |
| 14 | POST | `/interviews/{id}/start` | interview-service | AI 面试房间页 | 开始面试 |
| 15 | GET | `/interviews/{id}/current` | interview-service | AI 面试房间页 | 获取当前题 |
| 16 | POST | `/interviews/{id}/answer` | interview-service | AI 面试房间页 | 提交回答，返回评分和 nextAction |
| 17 | POST | `/interviews/{id}/finish` | interview-service | AI 面试房间页 | 结束面试并生成报告 |
| 18 | GET | `/interviews/{id}/report` | interview-service | 面试报告页 | 查看报告 |
| 19 | GET | `/interviews` | interview-service | 面试历史页 | 查看历史记录 |

后端前置要求：

1. 后端需要先保证对应的内部 Spring Cloud OpenFeign P0 链路可用。
2. interview-service 依赖 question-service、resume-service、ai-service。
3. ai-service 依赖已启用 Prompt 模板。
4. question-service 需要已有启用题目、分类、标签或问题组数据。
5. resume-service 需要能返回简历和项目经历。
6. 如果 AI 暂不可用，开发阶段可临时使用 mock AI 实现，但接口结构必须保持一致。
7. AI 评分失败时不应伪造评分。
8. 报告生成失败时，interview-service 应记录 `reportStatus = FAILED`，后续允许重试。

---

## 8. P1 补充接口顺序

P1 接口建议在 P0 主链路跑通后补齐：

| 顺序 | 方法 | URL | 服务 | 说明 |
|---:|---|---|---|---|
| 1 | POST | `/auth/logout` | auth-service | 退出登录 |
| 2 | GET | `/users/profile` | user-service | 查询个人资料 |
| 3 | PUT | `/users/profile` | user-service | 修改个人资料 |
| 4 | PUT | `/users/password` | user-service | 修改密码 |
| 5 | POST | `/questions/{id}/favorite` | question-service | 收藏题目 |
| 6 | DELETE | `/questions/{id}/favorite` | question-service | 取消收藏 |
| 7 | GET | `/questions/favorites` | question-service | 收藏题目列表 |
| 8 | GET | `/questions/wrong-records` | question-service | 错题列表 |
| 9 | PUT | `/questions/{id}/mastery` | question-service | 修改掌握状态 |
| 10 | DELETE | `/resumes/{id}` | resume-service | 删除简历 |
| 11 | PUT | `/resumes/{id}/default` | resume-service | 设置默认简历 |
| 12 | GET | `/interviews/{id}` | interview-service | 面试详情 |
| 13 | POST | `/interviews/{id}/report/retry` | interview-service | 报告失败重试 |
| 14 | GET | `/admin/questions` | question-service | 管理端题目列表 |
| 15 | POST | `/admin/questions` | question-service | 新增题目 |
| 16 | PUT | `/admin/questions/{id}` | question-service | 修改题目 |
| 17 | PUT | `/admin/questions/{id}/status` | question-service | 启用或禁用题目 |
| 18 | DELETE | `/admin/questions/{id}` | question-service | 删除题目 |
| 19 | GET | `/admin/question-categories` | question-service | 分类列表 |
| 20 | GET | `/admin/question-tags` | question-service | 标签列表 |
| 21 | GET | `/admin/question-groups` | question-service | 问题组列表 |
| 22 | POST / PUT / DELETE | `/admin/question-categories/**` | question-service | 分类维护 |
| 23 | POST / PUT / DELETE | `/admin/question-tags/**` | question-service | 标签维护 |
| 24 | POST / PUT / DELETE | `/admin/question-groups/**` | question-service | 问题组维护 |
| 25 | GET | `/admin/ai/prompts` | ai-service | Prompt 模板列表 |
| 26 | GET | `/admin/ai/prompts/{id}` | ai-service | Prompt 模板详情 |
| 27 | POST | `/admin/ai/prompts` | ai-service | 新增 Prompt |
| 28 | PUT | `/admin/ai/prompts/{id}` | ai-service | 修改 Prompt |
| 29 | PUT | `/admin/ai/prompts/{id}/status` | ai-service | 启用或禁用 Prompt |

说明：

1. Prompt 模板管理接口虽然是管理端能力，但 AI 面试链路需要已启用 Prompt。
2. 如果使用初始化 Prompt，可先不做 Prompt 管理页面，但后端仍需能读取启用模板。
3. 题库管理端 CRUD 如果有初始化数据，可后置到 P1。

---

## 9. P2 可后置接口

| 方法 | URL | 服务 | 说明 |
|---|---|---|---|
| POST | `/auth/refresh-token` | auth-service | 刷新 Token，V1 可选 |
| GET | `/users/overview` | user-service | 用户学习概览，可简化返回 |
| GET | `/admin/users` | user-service | 管理端用户列表 |
| PUT | `/admin/users/{id}/status` | user-service | 修改用户状态 |
| GET | `/admin/roles` | user-service | 基础角色列表 |
| GET | `/admin/ai/logs` | ai-service | AI 调用日志列表 |
| GET | `/admin/ai/logs/{id}` | ai-service | AI 调用日志详情 |
| DELETE | `/admin/ai/prompts/{id}` | ai-service | 删除 Prompt 模板，可后置 |
| GET | `/admin/system/overview` | system-service | 管理端首页简化统计 |
| GET | `/admin/configs` | system-service | 系统配置列表 |
| POST | `/admin/configs` | system-service | 新增系统配置 |
| GET | `/admin/configs/{key}` | system-service | 系统配置详情 |
| PUT | `/admin/configs/{key}` | system-service | 修改系统配置 |
| PUT | `/admin/configs/{key}/status` | system-service | 启用或禁用系统配置 |
| DELETE | `/admin/configs/{key}` | system-service | 删除系统配置 |
| GET | `/admin/menus` | system-service | V1 可选菜单接口 |

说明：

1. 这些接口不阻塞 AI 模拟面试核心闭环。
2. 第一阶段可 mock、简化或后置。
3. system-service V1 保持轻量，不负责复杂 RBAC。
4. `/admin/roles` 归属 user-service，不归 system-service。

---

## 10. 不允许出现在前端联调清单中的接口

以下内容仅作为禁止项列出，不能作为前端可调用接口使用。

1. 不允许前端直接调用 `/inner/**`。
2. 不允许前端调用 `/inner/ai/**`。
3. 不设计 `/ai/**` 用户端接口。
4. 不使用以下旧路径：

| 禁止使用的旧路径 | 当前正式路径或处理方式 |
|---|---|
| `POST /questions/answer` | 使用 `POST /questions/{id}/answers` |
| `GET /questions/wrongs` | 使用 `GET /questions/wrong-records` |
| `POST /ai/interview/question` | 仅后端内部使用 `POST /inner/ai/interview/question` |
| `POST /ai/interview/evaluate` | 仅后端内部使用 `POST /inner/ai/interview/evaluate` |
| `POST /ai/interview/follow-up` | 仅后端内部使用 `POST /inner/ai/interview/follow-up` |
| `POST /ai/interview/report` | 仅后端内部使用 `POST /inner/ai/interview/report` |

补充说明：

1. `/inner/**` 只能在后端服务之间通过 Spring Cloud OpenFeign 调用。
2. 前端联调接口清单不应包含 `/inner/**` 作为直接调用接口。
3. Gateway 路由配置应排除 `/inner/**`。
4. AI 能力接口只允许 interview-service 调用。

---

## 11. 接口联调前置条件

| 序号 | 前置条件 | 说明 |
|---:|---|---|
| 1 | Gateway 路由配置完成 | 前端所有请求统一经过 Gateway |
| 2 | 登录认证和 Token 透传完成 | 请求头使用 `Authorization: Bearer {token}` |
| 3 | common 返回结构完成 | 统一响应、异常处理、分页结构可用 |
| 4 | common-security 完成 | 下游服务可获取当前登录用户 |
| 5 | common-feign 完成 | Spring Cloud OpenFeign 请求头透传和异常处理可用 |
| 6 | 数据库初始化数据完成 | 默认用户、管理员、角色、题库基础数据、Prompt、系统配置 |
| 7 | 内部接口可调用 | P0 Spring Cloud OpenFeign 链路可用 |
| 8 | ai-service 可调用 | 至少支持 mock 或真实模型调用 |
| 9 | Prompt 模板存在且启用 | `ANSWER_EVALUATE`、`FOLLOW_UP`、`INTERVIEW_REPORT` 至少可用 |
| 10 | interview-service 可持久化面试数据 | 能保存面试会话、阶段、消息和报告状态 |

初始化数据建议：

1. 默认用户 / 管理员。
2. 默认角色 `USER` / `ADMIN`。
3. 题目分类。
4. 题目标签。
5. 问题组。
6. 示例题目。
7. Prompt 模板。
8. 系统配置。

---

## 12. 开发建议

### 第一阶段：后端骨架和认证链路

建议优先完成：

1. common-core
2. common-web
3. common-security
4. common-mybatis
5. common-feign
6. gateway
7. auth-service
8. user-service

目标：

1. 项目能启动。
2. Gateway 能路由。
3. 注册、登录、当前用户接口可用。
4. Token 透传和用户上下文可用。
5. auth-service 通过 Spring Cloud OpenFeign 调 user-service。

### 第二阶段：基础业务数据

建议完成：

1. question-service 用户端题库查询和刷题接口。
2. question-service 管理端题库初始化或基础 CRUD。
3. resume-service 简历和项目经历接口。
4. ai-service Prompt 模板读取。
5. ai-service mock AI 能力。

目标：

1. 用户能维护简历和项目经历。
2. 题库有可用题目。
3. ai-service 能返回结构稳定的评分、追问和报告内容。

### 第三阶段：面试核心闭环

建议完成：

1. interview-service。
2. Spring Cloud OpenFeign 联通 question-service。
3. Spring Cloud OpenFeign 联通 resume-service。
4. Spring Cloud OpenFeign 联通 ai-service。
5. 完成 `create / start / current / answer / finish / report`。

目标：

1. 用户可以创建 AI 模拟面试。
2. AI 面试房间可以持续问答。
3. `POST /interviews/{id}/answer` 返回评分、点评、`nextAction` 和下一题或追问题。
4. `POST /interviews/{id}/finish` 可以生成或记录报告失败状态。
5. 用户可以查看历史记录和报告。

### 第四阶段：管理端和体验增强

建议补齐：

1. 管理端题库维护。
2. Prompt 管理。
3. AI 调用日志。
4. 系统配置。
5. 用户概览。
6. 管理端首页。
7. 收藏、错题、掌握状态等体验增强能力。

目标：

1. 管理端可以维护题库基础数据。
2. 管理员可以调整 Prompt 模板。
3. AI 调用失败可通过日志排查。
4. P1 / P2 页面逐步完整。

---

## 13. 最终结论

若本文档审查通过，即可进入 CodeCoachAI-java 后端微服务骨架开发阶段。

第一轮代码开发建议从 common + gateway + auth + user 开始；不建议一开始直接开发 interview-service，因为它依赖 question-service、resume-service、ai-service 的内部接口。
