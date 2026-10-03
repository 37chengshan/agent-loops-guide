# 开源 Loop 项目卡片

> star 数为 2026-10-03 实测；行号引用来自各项目 README（浅克隆）。

## mem0（mem0ai/mem0，~66.5k★）
给 AI Agent 的记忆层（YC S24）。**in-band 记忆**：会话中实时抽取事实、跨会话检索，无 dreaming。
- 四代：第 2 代（`add`/`search` API）
- 亮点：2026-04 新算法 **ADD-only 抽取**（README L57："Single-pass ADD-only extraction — one LLM call, no UPDATE/DELETE. Memories accumulate; nothing is overwritten"）——用"只追加"让更新冲突不可能发生，是 versioning 的务实形态。另有三信号融合检索（semantic+BM25+entity）与时间推理（检索时做过期判断≈在线 Verify）。

## claude-mem（thedotmack/claude-mem，~95k★）
Claude 类 agent 的跨会话持久记忆，基于生命周期钩子自动捕获。
- Loop：in-band 自动化的极致——5 个钩子（SessionStart、UserPromptSubmit、PostToolUse、Stop、SessionEnd，README L250）自动记录 observations，**把"整理记忆"从 agent 注意力里摘出去**（直击 in-band 硬伤 #1）
- 四代：第 2 代→第 4 代之间（钩子自动写 + 本地 Worker + 文件存储）
- 亮点：**三层 MCP 检索**（README L261-276）：`search` 拿紧凑索引（~50-100 token/条）→ `timeline` 看上下文 → `get_observations` 取全文，省 ~10x token——"书架比喻"最干净的代码实现

## graphify（Graphify-Labs/graphify，YC S26）
代码库知识图谱 + 工作记忆与复盘循环。**本次调研里和演讲贴合度最高**。
- Loop：**完整 dreaming + learning edge**；第 4 代（`graphify-out/memory/` + `reflections/LESSONS.md`）
- 护栏：Versioning/Concurrency 都用 **git merge driver** 自动 union-merge（README L662），冲突永不发生
- 亮点（README L726-732）：`save-result --outcome useful|dead_end|corrected` 记 verdict → `reflect` 聚合成 LESSONS.md → `reflect --if-stale` 幂等（无新输入就 no-op）；节点打 `preferred/tentative/contested` 标签（置信度）；源码变了标 "code changed — re-verify"（**Verify 自动化**）

## cognee（topoteretes/cognee，~31.3k★）
免费开源 agent 记忆平台，文档/代码/对话 → 自托管知识图谱。
- in-band 记忆（存图→查图），无 dreaming；第 2 代 API + 第 4 代自托管
- 亮点：Portability 主打（"Runs locally for free — no API key required"）；选型启示：**要溯源选图谱，要召回选向量**

## letta-ai/letta（~25k★，MemGPT 血统）
Stateful Agents 平台，能"学习"的 agent。
- Loop：in-band + 自我编辑——context 满时 agent 自己决定写什么进长期记忆（archival memory），即"memory tool：agent 自己决定何时读写"的标杆
- 亮点：**记忆分层** core（常驻）vs archival（按需召回）→ 本 skill 热/温/冷三层的直接来源；core/archival 天然分区是 Permissioning 雏形

## langchain-ai/langgraph（~42.6k★）
用图编排 agent，生产级状态管理。"graph 住在节点之间"的正统实现。
- 亮点：**checkpointing + time travel**（versioning 最完整的开源实现：每次状态转移存档，可回滚可审计）；**human-in-the-loop interrupts**（"人在最高后果点审批"的代码实现，README L40）

## supermemoryai/supermemory（~31k★）
极速记忆层，带自动矛盾处理。在线版 Verify。
- 亮点（README L386-387）："Extracts facts, **tracks updates, resolves contradictions, auto-forgets expired info**"——矛盾解决+自动遗忘过期 = dreaming 的 Verify 做成在线服务。与离线 dreaming 互补：慢思考 vs 快验证。

## humanlayer/12-factor-agents（~26.5k★）
不是代码，是原则文档（LLM 软件工程的 12-factor）。
- 关键因子：F2 Own your prompts；F5 执行状态与业务状态统一（"记忆即状态文件"的理论依据）；F8 自己掌控控制流；F9 **Compact Errors into Context Window**（轻量 dreaming：失败当场压缩成教训）；F12 stateless reducer（状态外置，portability 哲学基础）
- "Agents as loops"节定义了 loop 标准形状：LLM 定下一步 → 确定性代码执行 → 结果 append 进 context → 循环直到 done

## superpowers（obra，社区热门）
coding agent 完整方法论。节点内的 loop（produce→review→correct）+ splitter（plan 切分）。
- 亮点：reviewer 链 = gate 实例；"plan 写得足够让 junior 照着做" = splitter 决定一切；触发式 skill 加载 = 第 3 代记忆实践样本

---

## 概念对照表

| 演讲框架概念 | 对应项目机制 |
|---|---|
| CLAUDE.md（第 1 代） | superpowers 初始指令；12-factor F2 |
| Memory tool（第 2 代） | mem0 add/search；Letta self-editing blocks；SuperMemory fact extraction |
| Skills（第 3 代） | claude-mem 三层 MCP 检索 |
| 文件系统即记忆（第 4 代） | graphify memory/+LESSONS.md；claude-mem 本地文件 |
| Versioning | LangGraph checkpoint/time-travel；mem0 ADD-only；graphify git merge driver |
| Concurrency | graphify git union-merge（唯一像样实现）；**演讲的 hash 乐观锁在开源里基本空白 → 本 skill 补位** |
| Permissioning | Letta core/archival 分区（雏形）；**普遍缺失 → 本 skill 补位** |
| Portability | cognee 自托管；12-factor F12 |
| Dreaming | graphify reflect（最完整）；SuperMemory auto-forget（在线版） |
| Learning edge | graphify outcome→LESSONS.md；12-factor F9 |
| Gate | superpowers reviewer 链；LangGraph human-in-the-loop |
| In-band 三硬伤之 #1 | claude-mem 生命周期钩子（自动捕获，不占注意力） |
