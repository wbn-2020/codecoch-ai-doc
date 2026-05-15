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

1. V1 代码已经基本做完。
2. V1 文档还没有同步更新。
3. 接下来准备做 V2。
4. 但是在进入 V2 之前，必须先完成 V1 文档封板：
   - 同步 PRD 和实际代码
   - 同步接口文档和实际 Controller / 前端 API
   - 输出 V1 Release Notes
   - 输出 V1 验收清单
   - 明确 V1 已完成项、未完成项、已排除项、V2 后续项

请你先阅读当前仓库中的：

1. `AGENTS.md`
2. `PROJECT_STATE.md`
3. `V1_RELEASE_NOTES_DRAFT.md`
4. `V2_ROADMAP_DRAFT.md`
5. 当前仓库实际代码

然后只做分析，不要马上改代码。请输出：

1. 当前仓库与你读取到的项目状态是否一致。
2. 这个仓库在 V1 文档封板中需要更新哪些内容。
3. 是否存在旧接口、旧字段、旧 PRD 描述与实际代码不一致。
4. 推荐下一步任务清单，按 P0/P1/P2 排序。
5. 如果需要修改，请先给修改计划，等我确认后再执行。

要求：

- 不要依赖旧聊天记录。
- 以仓库内文档和实际代码为准。
- 不要编造不存在的接口。
- 不要擅自扩大 V1 范围。
- 不要把 V2 功能混进 V1 文档。
- 输出必须具体到文件路径、接口路径、页面路径或模块名。
