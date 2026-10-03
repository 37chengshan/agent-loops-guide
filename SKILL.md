---
name: agent-loops-guide
description: Use when designing, debugging, or reviewing long-running or multi-agent workflows where work is chained, retried, verified, parallelized, remembered across sessions, or repeatedly fails in similar ways.
metadata:
  includeInPrompt: true
---

# Agent Loops Guide

## Overview

目标不是让人更勤快地盯 Agent，而是把人从中间步骤里拿掉，只留在**最高后果、最难撤销**的决策点。

这个 skill 用一条架构公式约束设计：

```text
可靠 Agent 系统
= Graph（决定做哪些工作、依赖关系）
+ Loop（让单个工作单元自己收敛）
+ Gate（按爆炸半径决定能否通过）
+ Learning Edge（把已验证结果变成下一轮约束）
+ Memory / Dreaming（让约束跨 session 存活并被整理）
```

一句话：**loop 在 node 内，graph 在 node 间；correction edge 修这一轮，learning edge 修以后每一轮。**

## When to Use

出现这些症状时加载本 skill：

- Agent 每一步都要人确认，离开人就不可靠
- pipeline 很长、很慢，但说不清哪些步骤真有依赖
- 多 Agent 并行后只是得到多份相似意见
- 失败会整批重跑，越修越乱
- 用“模型置信度”决定自动放行
- 同类错误跨 session 重复出现
- 已经有 memory / vector DB，但系统并没有越来越会做
- 想设计长期运行、可回滚、可审计的 agent workflow

不适合：一次性、完全确定性的短脚本。那种情况直接写代码，不要引入 Agent 架构。

## Five Iron Laws

### 1. 先写 Check，再写 Worker

一个 loop 只有四步：

```text
produce -> check -> correct -> repeat until green
```

**check 必须能失败。** 优先使用程序可判定条件：测试退出码、schema、diff 范围、source coverage、数值约束、静态检查。

以下不算 check：

- “看起来不错”
- “模型说自己很有把握”
- “没有报错”
- 另一个模型泛泛地说“通过”

若唯一可用的是模型判断，必须把 rubric 写成可观察证据，并把模型判断放在确定性证据之后。

### 2. 只有真实数据依赖才算 Edge

对 graph 中每一条箭头问：

> 下一个节点具体读取上一个节点的哪个输出变量？

答不出来，就没有 edge；可以并行，或者根本不该串起来。

不要把自然语言里的“然后”自动翻译成依赖。

### 3. Deterministic Work 用 Code Node

节点只有四类：

| Node | 作用 |
|---|---|
| Splitter | 把任务切成互相尽量独立的 unit |
| Worker | 对一个 unit 做需要推理的工作 |
| Code node | merge / sort / dedupe / compare / validate 等确定性变换 |
| Gate | 基于证据决定 accept / return / escalate |

若一个变换不需要“judge / decide / assess / summarize”，优先写代码，不要付模型的成本、延迟和方差。

### 4. Return the Unit, Not the Batch

一个 unit 失败，只退那个 unit。返回包必须带：

```text
UNIT     哪个工作单元
VERDICT  red / corrected / accepted
REASON   为什么失败
EVIDENCE 可定位证据
SCOPE    只允许改哪里
```

默认最多 correction 3 次。三次仍失败，升级到 plan / splitter 层；不要让局部 loop 无限重试一个它看不见的规划错误。

### 5. Gate 看 Blast Radius，不看 Confidence

按“错了以后有多难撤销”开 lane：

| 风险 | 例子 | 策略 |
|---|---|---|
| 可逆且局部 | 文案、测试、隔离函数 | 可自动通过确定性 gate |
| 可逆但影响面大 | 公共 util、schema、共享配置 | 更强 checks + 轨迹 + 必要的人审 |
| 难撤销 | 数据迁移、删除、生产写入、资金动作 | lane 默认关闭，显式人工批准 |

证据阅读顺序：**确定性结果 -> 本轮 trajectory -> 历史 rollback / failure -> 模型自评。**

## Required Design Procedure

使用本 skill 设计或审查系统时，按下面顺序输出，不要从“选模型 / 选向量库 / 加 memory”开始。

### Step 1 — 定义 Unit 与 Green Condition

先写：

```text
UNIT: 一个最小、可独立验证、可独立返工的工作单元
INPUT: ...
OUTPUT: ...
GREEN: 一个或多个可失败条件
MAX_ATTEMPTS: 默认 3
```

如果 GREEN 写不出来，先不要实现 loop。

### Step 2 — 画 Graph，只保留 True Dependencies

列出节点和 edge；对每条 edge 写明跨过去的变量。删除假依赖，找出能 fan-out 的工作。

优先按**blast radius / 独立可验证性**切 unit，而不是机械地按文件夹、章节或输入顺序切。

### Step 3 — 把 Loop 放进 Fallible Node

每个需要推理且可能出错的 Worker 自己负责：produce -> check -> correct。Graph 不负责替单个节点“多想几遍”。

### Step 4 — 设计两条 Return Path

- **Correction edge（短）**：gate 把失败 unit 退给产生它的节点，修当前 run。
- **Learning edge（长）**：accepted / corrected 结果提炼成 constraint，写回 splitter brief，改变未来 run 的切分和约束。

只有 correction edge 的系统会越来越快，但不会越来越聪明。

### Step 5 — 最后才加 Memory / Dreaming

Memory 的任务不是替代 graph，而是让 learning edge 跨 session 存活。

最小演进：

```text
静态指令文件 -> 可读写 memory tool -> 按需加载 skill -> 文件系统 / 外部状态
```

跨 session 重复足够多以后再加 dreaming：

- 轻量：失败后当场压缩成一条 lesson
- 标准：定期聚合 verdict / trajectory，产出 LESSONS
- 重量：Verify / Organize / Enrich，产出 proposal，由人或受控 gate 合并

详细实现见 `references/dreaming-levels.md`。

### Step 6 — 给 Memory 加生产护栏

长期、多 Agent 系统至少要有：

- Versioning：谁、何时、基于什么证据改了什么
- Concurrency：hash / optimistic lock，冲突时重读重试
- Permissioning：共享规则、agent 草稿、dreaming proposal 分权
- Portability：状态外置，接口清晰，不和某个 runtime 强绑定

模板见 `references/templates.md`。

### Step 7 — 把 Human 放在最高后果的一步

人不要做图里的“常驻 reviewer node”。人应该批准：merge、ship、删除、迁移、生产写入、资金动作等高后果节点。

目标不是 human-free，而是 **human-at-the-right-boundary**。

## Output Contract

当本 skill 被用于方案设计或 review，答案至少给出：

1. **Unit**：最小工作单元是什么
2. **Graph**：节点、真实依赖、可并行 lane
3. **Checks**：每个关键节点的 machine-failable green condition
4. **Correction**：失败 unit 如何返回，scope 如何限制
5. **Gate**：按 blast radius 的 lane 策略
6. **Learning Edge**：accepted/corrected 结果写回哪里
7. **Memory/Dreaming**：是否需要、用哪一档、如何版本化
8. **Human Approval Point**：唯一或少数人工批准点
9. **Path Metrics**：attempts、return rate、rollback、latency、token/cost、首次通过率

不要只评价最终答案；**测路径，而不只测落点。**

## Common Failure Modes

| 失败模式 | 修正 |
|---|---|
| 先做 Agent，再补 review | 先写可失败 check |
| 所有节点都是 LLM | 确定性变换换成 code node |
| 把“然后”当 edge | 必须指出跨 edge 的变量 |
| 多 Agent 共用同一上下文互相抄答案 | worker 用独立 context，merge 后再汇总 |
| 一处失败整批返工 | return unit, not batch |
| 无限 self-correction | cap attempts，升级 plan |
| 用 confidence 自动放行 | 按 blast radius 分 lane |
| dreaming 直接改生产规则 | 只产 proposal，经过版本/权限/gate |
| memory 越积越多 | 分热/温/冷层，定期 verify / compact |
| verdict 只写报告不改变后续执行 | verdict 必须改变 route、scope 或未来 constraint |

## References

- `references/loop-graph-playbook.md` — 公式、node/edge 设计、gate 与 learning edge 的完整推导
- `references/projects.md` — 10+ 开源项目的 loop / memory 机制卡片与概念对照
- `references/dreaming-levels.md` — dreaming 三档设计与 cron 模板
- `references/templates.md` — loop contract、edge audit、hash 乐观锁、verdict、权限与三层检索模板

## Final Rule

如果一个失败没有改变**当前 route**，它只是报告；如果一个被确认的教训没有改变**未来 brief**，系统迟早会再撞上它。
