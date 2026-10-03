# Dreaming 三档详细设计

## 轻量档：Factor 9（今天就能加）

**机制**：`Compact Errors into Context Window`——每次失败当场提炼一条教训，写回记忆。不需要独立进程。

**模板**（失败处理时执行）：
```
失败：{发生了什么}
教训（一句话）：{下次如何避免}
写入：{记忆文件的哪一节}
```

**适用**：单人小项目、cron 单任务。成本≈0。

## 标准档：graphify reflect（nightly cron）

**机制**：
1. 每次工作单元结束记 verdict：`save-result --outcome useful|dead_end|corrected --reason ... --evidence ...`
2. 每夜 `reflect`：把当天的 outcomes 聚合成 `LESSONS.md`（去重、合并、提炼）
3. `reflect --if-stale`：输入无变化则 no-op——**幂等**，随时可跑，不怕重复调度

**cron 模板**：
```md
# dreaming-nightly
输入（只读）：近24h 的运行轨迹（manifests/logs/reviews）+ 现有记忆文档 + 上次的 proposals
任务：
  1. 找重复失败模式（同一模式 ≥3 次，或一次造成线上事故）
  2. 找过期规则（引用了已变更的定价/接口/流程）
  3. 找重复规则（同一意思散在多处）
输出：proposals-YYYYMMDD.md，每条带证据（id/计数/行号），按"证据强度×影响面"排序
铁律：只提案，不合并；弱证据标出
```

**适用**：多任务流水线、每天有稳定产出。成本：一次 nightly agent 运行。

## 重量档：演讲完整版 Verify/Organize/Enrich

**机制**：独立 agent + 独立资源，批量处理一段时间（如一周）的全部轨迹。

- **Verify**：逐条核对现有规则。自动化手段：源码/配置变了就标 "code changed — re-verify"（抄 graphify）；定价/接口变更触发相关规则复核。
- **Organize**：去重、合并、按主题重排。信号：同一意思出现在 ≥2 个文件。
- **Enrich**：新规则。证据门槛：同一失败模式 ≥3 次，或单次造成线上事故/资损/超发。

**合并流程**：
1. dreaming 输出 proposals（带证据、带拟写入位置、带风险评估）
2. 人工抽查（或主 agent 按授权执行）：重点看 Enrich 类——dreaming 会写错
3. 合并后记版本：谁、何时、依据哪次 dreaming

**适用**：多 agent 生产系统。成本：独立资源 + 人工抽查时间。

## 互补：在线快验证

离线 dreaming 做"慢思考"（模式提炼），在线做"快验证"：
- SuperMemory 模式：写入时即做矛盾检测、过期自动遗忘
- mem0 模式：检索时做时间推理（"这条记忆还成立吗"在读时判断）

两者不互斥：在线防"读到过期东西"，离线负责"长出新规则"。
