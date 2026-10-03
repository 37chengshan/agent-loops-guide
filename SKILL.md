---
name: "agent-loops-guide"
description: "设计 agent 循环系统的完整指南：以 Claude 团队 context engineering 演讲为底层逻辑（记忆四代演进、四条生产护栏、离线 dreaming 循环），融合 10+ 开源项目的实战模式（graphify reflect、claude-mem 三层检索、LangGraph checkpoint、Letta 记忆分层）。用于设计长期运行、多 agent 的记忆与自改进系统。"
metadata: { "includeInPrompt": true }
tagline: "让 agent 越跑越聪明的循环系统设计指南：dreaming 三档、护栏模板、检索规范。"
---

# Agent Loops 指导手册

> 底层逻辑来自 Claude 团队 context engineering 演讲：模型变强之后，真正决定产品能不能用的杠杆是 context engineering——给 agent 在任务当下能读到、能改、还能跨会话积累的信息。模式素材来自 10+ 开源项目的实测机制（见 `references/projects.md`）。

一句话闭环：**白天 agent 读记忆、干活、把轨迹留下来 → 夜里 dreaming 从轨迹里抽出可复用规则，写回记忆 → 下一次 session 自动带上这些规则，等于系统在给自己写下一轮指令。**

## 一、记忆的四代演进（够用就停）

| 代 | 做法 | 何时用 | 开源范例 |
|---|---|---|---|
| 1. **CLAUDE.md** | 会话开始注入 markdown | 代码库约定、组织偏好 | superpowers 的初始指令；12-factor **Factor 2: Own your prompts** |
| 2. **Memory tool** | agent 自己决定何时读/写/改 | 动态事实、轻量偏好 | mem0 的 `add`/`search`；Letta 的 self-editing memory blocks |
| 3. **Skills** | front matter 先暴露标题，正文按需加载 | 大知识库 | claude-mem **三层 MCP 检索**（先扫书脊再抽书，见 §五） |
| 4. **文件系统即记忆** | 记忆=markdown 文件，bash/grep 检索 | 长期、多 agent | graphify 的 `memory/` + `LESSONS.md` |

演进线：`CLAUDE.md → memory tool → skills → memory/`
极限提醒：第 1 代文件一长就上下文膨胀、互相冲突——长到维护不动时就该升代，不要硬塞。

## 二、生产四护栏（多 agent 长期跑必做）

1. **Versioning（版本）**：每次改动留版本——哪个 session、哪段 transcript、人还是 agent 写的，可回滚。
   开源实现：LangGraph **checkpointing/time-travel**（最完整）；mem0 v3 **ADD-only**（只追加不覆盖，用"让冲突不可能发生"代替"解决冲突"）；graphify 用 **git merge driver** 做 union-merge。
2. **Concurrency（并发）**：写前对内容 hash，提交时对不上就拒绝并重试（乐观锁）。
   **开源基本空白**——这是本 skill 的补位点。模板见 `references/templates.md`（含可直接用的 Python 实现）。
3. **Permissioning（权限）**：组织级知识只读，agent 草稿可写；画出"谁能写什么"分区表。
   开源只有雏形：Letta 的 core（系统级）vs archival（agent 可写）记忆分区。**同样空白，模板见 references。**
4. **Portability（可移植）**：干净 API 暴露记忆，不锁死 runtime；状态外置（12-factor **Factor 12: stateless reducer**——agent 本身无状态）。

## 三、Dreaming：三档实现

in-band 记忆（会话内自己管）有三硬伤：注意力被任务和整理拆开；看不到跨会话重复错误；记忆过期无人核对。Dreaming 是第二条进程：**批量、异步、单独占资源**，输入=记忆库+一段时间的轨迹，输出=整理过的记忆。

按投入从小到大三档：

**轻量档 — Factor 9（12-factor-agents）**
`Compact Errors into Context Window`：不需要 dreaming 进程，每次失败当场提炼一条教训写回记忆。适合：单人小项目，今天就能加。

**标准档 — graphify reflect**
`save-result --outcome useful|dead_end|corrected` 每次记录 verdict → `reflect` 聚合成 `LESSONS.md` → `reflect --if-stale` 做成幂等（没新输入就 no-op，随时可跑）。适合： nightly cron，一天复盘一次。

**重量档 — 演讲完整版**
Verify（规则还成立吗？过期标 "code changed — re-verify" 自动失效）/ Organize（去重合并）/ Enrich（反复失败补成规则），输出提案 → **人工抽查后合并**（dreaming 只提案不合并，人审计）。适合：多 agent 生产系统。

互补关系：离线 dreaming 做"慢思考"（模式提炼），在线引擎做"快验证"（SuperMemory 的 auto-forget：过期即忘、矛盾即解）。

## 四、Learning edge（长回路，99% 的人缺这条）

短回路修这一轮（gate 打回单个单元）；**长回路修以后每一轮**：accepted 结果 → 提炼成约束 → 写回 **splitter 的 brief**（注意落点：不是 worker 指令，是决定"怎么切分工作"的 brief）。

回退四件套（单元级，不退整批）：UNIT / VERDICT / REASON / EVIDENCE / SCOPE（"只修这个文件"，防扩散）。同一单元 3 次不过 → 问题在 plan，loop 看不见 plan，升级处理。

## 五、检索规范（抄 claude-mem）

第 3 代记忆最具体的工程实践，三层渐进，token 预算写死：

1. `search` — 紧凑索引（~50-100 token/条），先扫书脊
2. `timeline` — 看上下文
3. `get_observations` — 只取需要的全文

号称省 ~10x token。规则：**永远不许跳过第 1 层直接读全文**。

## 六、记忆分层（抄 Letta）

- **热**：常驻上下文（core memory：身份、当前任务、当日约束）
- **温**：按需召回（archival：项目知识、历史决策）
- **冷**： dreaming 归档（LESSONS.md：提炼后的规则，不过期检查）

选型规则：**要溯源选图谱，要召回选向量**（graphify benchmark：LOCOMO recall@10 图谱 0.497 vs 纯向量 0.048）。

## 七、Gate（门禁）

按**爆炸半径**开，不按置信度（置信度是模型唯一能操纵的变量）：
- 可逆且收敛（改文案、加测试）→ 先开
- 可逆但影响面大（公共 util、schema）→ 确定性检查 + 干净轨迹
- 难撤销（迁移、删除、动钱）→ **永远不开**，不是阈值高，是直接关闭

证据阅读顺序：确定性结果 → 本轮轨迹 → 该节点的历史回滚率 → 模型的自我评估（最后）。

## 八、落地顺序

1. 先建 **gate**（能大声失败的东西；没 gate 的 graph 只是更快地生产未经验证的输出）
2. 再画 lane（并行/依赖）
3. learning edge 最后（得先有 accepted 结果才能提炼约束）
4. 人只放在**后果最大 + 最不可逆**的一步：批 merge，不看中间过程

## 九、局限（诚实）

- 收益依赖 transcript 质量；dreaming 会写错 → 版本 + 人工抽查缺一不可
- 改善的是上下文，不是模型能力
- "97% fewer first-pass errors"类数字是客户侧观察，不是受控实验

## references/

- `projects.md` — 10 个开源项目的 loop 机制卡片 + 概念对照表
- `dreaming-levels.md` — 三档 dreaming 的详细设计与 cron 模板
- `templates.md` — 可直接抄的代码模板（hash 乐观锁、verdict 记录、权限分区表）
