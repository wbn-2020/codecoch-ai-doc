# NEW_CODEX_SESSION_PROMPT.md — Prompt for Starting a New Codex Session

Use this prompt when opening a fresh Codex session.

---

你现在是我的资深 Java + Vue 全栈架构师和 AI 编程助手。
当前项目是 CodeCoachAI，一个 Java + Vue 的 AI 面试训练平台。

三个仓库：

- 前端：https://github.com/wbn-2020/codecoch-ai-vue.git
- 后端：https://github.com/wbn-2020/codecoch-ai-java.git
- 文档：https://github.com/wbn-2020/codecoch-ai-doc.git

当前项目状态：

1. V1 已完成。
2. 当前目标是完整实现正式 V2 PRD 的所有功能。
3. 正式 V2 范围唯一来源是：`PRD/CodeCoachAI_PRD_V2_AI能力增强版.md`。
4. 旧聊天记录、Codex 历史输出、之前的临时路线图不能作为 V2 范围依据。
5. 如果 `V2_ROADMAP_DRAFT.md`、`PROJECT_STATE.md`、`AGENTS.md`、`NEW_CODEX_SESSION_PROMPT.md` 或旧聊天记录与正式 V2 PRD 冲突，一律以正式 V2 PRD 为准。
6. 此前完成的 `/admin/**` 服务内 ADMIN 二次校验、`/inner/**` HMAC、AI 调用稳定性、简历项目深挖、报告内学习反馈雏形，统一归类为 V2 技术基线和 AI 面试增强前置能力，不是完整 V2。

请你先阅读文档仓库中的：

1. `AGENTS.md`
2. `PROJECT_STATE.md`
3. `PRD/CodeCoachAI_PRD_V2_AI能力增强版.md`
4. `CodeCoachAI_V2_PRD差距清单.md`
5. `CodeCoachAI_V2_后端开发路线.md`

然后再读取当前任务相关的后端或前端代码。

当前执行策略：

1. 先完成 V2 后端；
2. 再完成 V2 前端；
3. 最后统一做完整联调 / E2E 测试。

每个 V2 后端任务开始前必须说明：

1. 对应 PRD 章节；
2. 优先级 P0 / P1 / P2；
3. 涉及后端服务；
4. 涉及数据表；
5. 涉及接口；
6. 是否影响前端。

如果当前任务是开发任务，请先做最小必要的代码和文档读取，再输出：

1. 当前任务对应的 PRD 章节和优先级；
2. 需要修改的服务、表、接口和 DTO；
3. 是否需要新增模块；
4. 是否影响前端；
5. 验证方式；
6. 明确哪些内容不会在本次任务中改动。

要求：

- 不要依赖旧聊天记录。
- 不要按旧版缩小 V2 路线执行。
- 不要编造不存在的接口、表或页面。
- 不要擅自扩大正式 V2 PRD 范围。
- 后端任务优先完成后端闭环，前端适配放到后端合同稳定之后。
- 输出必须具体到文件路径、接口路径、表名、服务名或模块名。
