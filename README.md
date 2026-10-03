# agent-loops-guide

**从“盯着 Agent 每一步”到“只批准最后一个高后果步骤”的系统设计 skill。**

这个仓库把 loop、graph、gate、learning edge、memory 和 dreaming 串成一套可执行的架构方法。核心不是“让模型多反思几次”，而是先把**工作形状、失败条件、返回路径和长期学习位置**设计对。

## 核心公式

```text
可靠 Agent 系统
= Graph（决定做哪些 unit、哪些真有依赖）
+ Loop（produce -> check -> correct，直到 green）
+ Gate（按 blast radius 放行）
+ Learning Edge（把验证过的结果写回未来 brief）
+ Memory / Dreaming（让这些约束跨 session 存活并整理）
```

最重要的两句话：

> **Loop lives inside a node. Graph lives between nodes.**
>
> **Correction edge 修当前 run；learning edge 修未来 run。**

这套思路来自你提供的 *Loops and Graphs* 长文/演讲框架，并结合仓库原有的 context engineering、memory、dreaming 与 10+ 开源项目机制整理成一个可被 Agent 直接调用的 skill。它是工程方法的综合整理，不代表 Anthropic/Claude 的官方规范。

## 解决什么问题

当你遇到下面任一情况，这个 skill 就有用：

- Agent 每一步都要人确认
- pipeline 很慢，却说不清哪些步骤真的依赖前一步
- 多 Agent 并行后只是得到重复意见
- 一个 unit 失败导致整批返工
- 用“模型置信度”决定是否自动放行
- memory 越做越复杂，但错误仍跨 session 重复
- 想做长期运行、可回滚、可审计的 Agent 系统

## 仓库结构

```text
SKILL.md                         # 主 skill：触发条件 + 决策程序 + 输出契约
references/
  loop-graph-playbook.md         # Loop / Graph / Gate / Learning Edge 完整 playbook
  projects.md                    # 10+ 开源项目的机制卡片 + 概念对照表
  dreaming-levels.md             # Dreaming 三档详细设计 + cron 模板
  templates.md                   # Loop contract、edge audit、hash 锁、verdict、权限、检索模板
LICENSE                          # MIT
```

## 从哪里开始

如果你是 Agent / coding agent：直接加载 `SKILL.md`。

如果你是人类设计者：按这个顺序读：

1. `SKILL.md` — 先掌握决策程序
2. `references/loop-graph-playbook.md` — 看完整推导和反例
3. `references/templates.md` — 直接抄 contract / edge audit / verdict 结构
4. 需要长期记忆时再看 `references/dreaming-levels.md`
5. 需要选实现模式时查 `references/projects.md`

## 最小落地顺序

```text
1. 定义 unit
2. 写 machine-failable GREEN 条件
3. 画 graph，只保留真实数据依赖
4. 把 fallible loop 放进 worker node
5. 设计 correction edge
6. 按 blast radius 设计 gate
7. 再加 learning edge
8. 最后才加 memory / dreaming
9. 人只留在最高后果、最难撤销的节点
```

不要从“该用哪个模型”“要不要向量库”“要不要 multi-agent”开始。

## 一个 30 秒自检

对你现有 workflow 问 7 个问题：

- 每个 unit 的 GREEN 条件能被程序或明确 rubric 判失败吗？
- 每条 edge 都能指出“跨过去的变量”吗？
- merge / sort / dedupe / compare 是否还在浪费 LLM？
- 一个失败会不会把已经正确的 unit 一起打回？
- retry 是否有上限，超过后是否升级到 plan/splitter？
- gate 是按 blast radius，还是按模型 confidence？
- accepted/corrected 的结果有没有改变下一轮 splitter brief？

只要有两三个答不上来，问题通常不在 prompt，而在系统结构。

## 原有资产仍然保留

这个版本没有丢掉仓库原来的核心：

- **记忆四代演进**：静态指令 -> memory tool -> skills -> 文件系统/外部状态
- **生产四护栏**：Versioning / Concurrency / Permissioning / Portability
- **Dreaming 三档**：轻量压缩 -> nightly reflect -> Verify/Organize/Enrich
- **三层检索**：索引 -> 时间线 -> 全文
- **Learning edge**：把 accepted/corrected 结果变成未来约束

区别是：这些能力现在被放回更大的执行架构里——**memory 不再是起点，而是 learning edge 的持久化层。**

## 使用方式

把整个目录放进你所用 Agent runtime 支持的 skills 目录，或直接把 `SKILL.md` 作为项目级设计/审查指南加载。

也可以直接用一句话调用它：

```text
用 agent-loops-guide 审查这个 Agent workflow：先找 unit / fake edges / weak checks，再给 correction edge、gate、learning edge 和 human approval point。
```

## License

MIT License.
