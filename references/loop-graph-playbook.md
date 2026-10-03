# Loop / Graph / Gate / Learning Edge Playbook

这份 playbook 把核心演讲思路压成一套工程决策语言。不要把它当“多 Agent 框架教程”；它更像一个审查 Agent 系统结构的 checklist。

## 1. Loop 的本质：一个会失败的 Check

最小 loop：

```text
produce -> check -> correct -> repeat until green
```

真正决定 loop 是否存在的不是 retry，而是 **check 能不能明确失败**。

### 好的 GREEN

```text
GREEN  test suite exits 0
GREEN  every factual claim has a source line
GREEN  diff touches only files named in the plan
GREEN  JSON validates against schema
GREEN  latency <= 800 ms under test fixture
```

### 伪 GREEN

```text
NOT A CHECK  output looks good
NOT A CHECK  model confidence > 0.9
NOT A CHECK  another model says PASS
NOT A CHECK  no errors were raised
```

“没有报错”只说明没有触发错误路径，不等于结果正确。

### Loop Contract

实现前先写：

```yaml
unit: "handlers slice"
input: "porting brief + source files"
output: "ported handlers"
green:
  - "pytest tests/handlers exits 0"
  - "diff touches handlers/** only"
max_attempts: 3
on_red: "return this unit with evidence and scope"
```

如果连这个 contract 都写不清楚，先不要调 prompt。

---

## 2. Graph 的本质：工作的形状

Graph 决定：

- 哪些 unit 存在
- 哪些可以并行
- 哪些必须等待
- 哪些其实不需要执行
- 失败后结果往哪里退
- 成功后经验往哪里写回

### Edge Audit

对每条箭头都填这张表：

| From | To | To 读取 From 的哪个变量？ | 真依赖？ |
|---|---|---|---|
| summarize-file | check-weather | 无 | 否，删除 edge |
| build | test | build artifact | 是 |
| worker-A | merge | result-A | 是 |

规则：**命名不出跨 edge 的变量，就没有 edge。**

很多 workflow 最大的提速不是换模型，而是删掉假 edge。

---

## 3. 四种 Node 足够表达大多数 Agent Workflow

### Splitter

把任务切成 unit。Splitter 的质量决定下游工作是否重复、是否能独立验证。

错误切法示例：按文件夹机械切 repository，导致多个 worker 审到相同跨目录依赖。

更好的切法：按 blast radius、责任边界、可独立验证性切。

### Worker

每个 worker 只拿一个 unit、一个 lens、独立 context。

多 auditor 共用同一 context 容易形成“第一份结论 + 三个回声”，不是四个独立视角。

### Code Node

以下工作通常应该是代码：

- merge
- sort / rank by fixed key
- dedupe
- exact compare
- schema validation
- export before/after diff
- deterministic routing

测试：如果描述这个步骤时不需要 `judge / decide / assess / summarize`，优先用代码。

### Gate

Gate 不生产主要内容，只读取 evidence 并决定：

```text
accept
return(unit, reason, evidence, scope)
escalate
```

---

## 4. Loop 在 Node 内，Graph 在 Node 间

这是最重要的边界：

```text
Inside one node:
produce -> check -> correct -> repeat

Between nodes:
split -> fan out -> merge -> gate -> route / return
```

只有 graph、没有 node loop：你会并行地产出大量未验证内容。

只有 loop、没有 graph：你会得到一个很会自纠错的 Agent，但仍按错误的顺序执行错误数量的步骤。

---

## 5. 两条 Return Path

### Correction Edge：短回路

当前 run 中，失败 unit 退给产生它的节点。

返回内容必须完整：

```text
UNIT      handlers slice
VERDICT   red
REASON    test_auth_redirect failed
EVIDENCE  expected 302, got 200; handlers/auth.py:88
SCOPE     fix handlers/auth.py only
```

`SCOPE` 防止“顺手修旁边问题”把局部 correction 膨胀成大 diff。

### Learning Edge：长回路

accepted / corrected 的结果不只是结束；还应该导出一个约束，写回未来的 splitter brief。

```text
ACCEPTED  utils slice ported, green on first pass
DERIVED   adapters must preserve keyword args exactly
LANDS IN  splitter brief for every later slice
```

为什么落在 splitter，而不是只写 worker prompt？

因为高价值教训经常改变的是**以后怎么切任务、哪些先做、哪些不能混在一起**，不只是“worker 更小心一点”。

只有 correction edge 的系统能修错；有 learning edge 的系统才会减少未来同类错误。

---

## 6. Return the Unit, Not the Batch

四个 unit 中一个失败：

错误做法：整批重跑。

后果：

1. 三个本来正确的 unit 被重写
2. 新版本只是“不同”，不一定更好
3. 所有 unit 重新验证
4. 原本一个 red 变成四个不确定结果

正确做法：只退失败 unit。

默认最多 correction 3 次。第 4 次之前必须问：

> 是 worker 执行错了，还是 splitter / plan 从一开始就把 unit 定义错了？

局部 loop 看不见全局 plan，不要让它无限自救。

---

## 7. Gate 按 Blast Radius，而不是 Confidence

Confidence 是弱变量：模型可以影响它，而且校准往往不稳定。

更强的变量是：**如果这个动作错了，恢复成本是什么？**

### Lane A — 可逆且局部

例如：文案、测试、隔离函数。

策略：确定性 checks 通过后可自动放行。

### Lane B — 可逆但影响面大

例如：共享 util、schema、公共配置。

策略：确定性 checks + clean trajectory + 历史 rollback 信号；必要时人工 review。

### Lane C — 难撤销

例如：migration、delete、production write、money movement。

策略：默认关闭。不是“阈值 0.99”，而是**没有自动开启路径**，需要显式授权。

### Gate Evidence Order

```text
1. deterministic checks
2. trajectory of this run
3. historical rollback / failure rate
4. model self-assessment
```

越靠后的证据权重越低。

---

## 8. Human 应该放在哪里

不要让人变成 graph 中间的常驻节点。

应该让人批准：

- merge / ship
- irreversible migration
- destructive delete
- production data write
- financial action
- high-blast-radius policy change

设计目标：

```text
不是 human-free
而是 human-at-the-right-boundary
```

如果每个 worker 后面都有人审，graph 的吞吐上限就是人的阅读速度。

---

## 9. Memory 是 Learning Edge 的持久化层

不要从“选 vector DB”开始。

先有：

1. accepted/corrected verdict
2. 能导出的 constraint
3. 明确 constraint 应该改变哪个未来 brief

然后才问如何跨 session 保存它。

### 四级演进

```text
静态项目指令
-> agent 可读写 memory
-> 大知识按 skill / reference 按需加载
-> 文件系统或外部状态作为长期共享记忆
```

原则：够用就停。

---

## 10. Dreaming 是慢速 Learning Edge

在线 correction 处理当前错误；dreaming 处理跨 run 模式。

### 轻量

失败后立刻压缩一条 lesson。

### 标准

定期聚合 verdict：

```text
useful / dead_end / corrected
-> dedupe
-> cluster repeated failures
-> update LESSONS proposal
```

### 重量

```text
Verify   规则还成立吗？
Organize 去重、合并、分层
Enrich   把反复失败补成更具体的 constraint
```

Dreaming 最安全的默认权限是：**proposal-only**，不直接改生产规则。

---

## 11. 长期 Memory 的四个生产护栏

### Versioning

每次规则变化保留 actor、session、evidence、prev/new version，必须可回滚。

### Concurrency

共享状态写入使用 optimistic lock / content hash；冲突则重读重试。

### Permissioning

把组织规则、agent 草稿、dreaming proposal、orchestrator state 分开权限。

### Portability

Agent runtime 尽量无状态；memory 通过清晰 API / 文件协议暴露，不把系统锁死在单一框架。

---

## 12. 测什么：Path，不只测 Final Answer

至少记录：

- first-pass green rate
- attempts per unit
- correction rate
- scope violations
- rollback rate
- fake-edge / idle wait time
- end-to-end latency
- token / cost per accepted unit
- learning reuse：历史 constraint 实际命中并改变 route 的次数

一个 verdict 如果不改变接下来跑什么，只是报告。

一条 lesson 如果从未在未来 brief 中被检索/应用，只是存档。

---

## 13. 建议的落地顺序

```text
1. 选一个重复 workflow
2. 定义最小 unit
3. 先写 GREEN check
4. 画 splitter / lanes / merge / gate
5. 对所有 edge 做 variable audit
6. 把 deterministic node 换成 code
7. 给 worker 放入 bounded loop
8. 只退失败 unit
9. 按 blast radius 开 gate
10. 记录 verdict + evidence + scope
11. 有足够 accepted/corrected 数据后，再建 learning edge
12. 最后才加 memory / dreaming
13. 把 human 留在最难撤销的一步
```

---

## 14. Skill Self-Test Scenarios

用下面场景检查这个 skill 是否真的改变 Agent 行为。

### Scenario A — “给每一步都加 reviewer Agent”

期望：拒绝把第二个 LLM reviewer 当默认 check；先找 deterministic GREEN。

### Scenario B — “先总结文件，然后查天气”

期望：指出没有数据依赖，删 edge 并行执行。

### Scenario C — “四个 slice 一个失败，整批重跑”

期望：只返回失败 unit，保留三个 green unit。

### Scenario D — “置信度 > 0.95 自动执行数据库迁移”

期望：按 blast radius 把 migration 放进关闭 lane，需要显式人工批准。

### Scenario E — “我们已经有向量库，为什么还重复犯错？”

期望：先检查 learning edge 是否把 accepted/corrected 结果导成 constraint 并落到未来 splitter brief，而不是继续换 embedding。

### Scenario F — “一个 unit 第四次 self-correction”

期望：停止局部 retry，升级检查 plan / splitter。

如果 skill 对这些场景仍然只给 prompt 建议，而没有改变 graph / check / route / gate，说明 skill 还没有真正生效。
