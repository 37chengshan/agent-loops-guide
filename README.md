# agent-loops-guide

让 agent 越跑越聪明的循环系统设计指南。

底层逻辑来自 Claude 团队 context engineering 演讲：模型变强之后，真正决定产品能不能用的杠杆是 **context engineering**——给 agent 在任务当下能读到、能改、还能跨会话积累的信息。模式素材来自 10+ 开源项目的实测机制（Mem0、claude-mem、graphify、Letta、LangGraph、SuperMemory、12-factor-agents…）。

## 核心框架

- **记忆四代演进**：`CLAUDE.md → memory tool → skills → 文件系统即记忆`（够用就停）
- **生产四护栏**：Versioning / Concurrency / Permissioning / Portability（多 agent 长期跑必做；其中 Concurrency 的 hash 乐观锁在开源里基本空白，本仓库给了可直接抄的模板）
- **Dreaming 三档**：轻量（Factor 9 失败当场压缩）→ 标准（graphify reflect nightly 聚合）→ 重量（Verify/Organize/Enrich + 人工抽查）
- **Learning edge**：短回路修这一轮，长回路把 accepted 结果提炼成约束写回 splitter 的 brief

## 目录

```
SKILL.md                  # 主指南（Muse skill 格式，可直接装载）
references/
  projects.md             # 10 个开源项目的 loop 机制卡片 + 概念对照表
  dreaming-levels.md      # 三档 dreaming 详细设计 + cron 模板
  templates.md            # 可直接抄的代码：hash 乐观锁、verdict 记录、权限分区表、三层检索
```

## 用法

- 当 Muse skill 用：整个目录放到 `~/workspace/skills/` 即可
- 当设计手册读：从 SKILL.md 开始，§八 有落地顺序

## 诚实声明

- "97% fewer first-pass errors"类数字是客户侧观察，不是受控实验
- Letta 的 memory blocks 机制依据公开文档，未读源码
- star 数为 2026-10-03 实测

MIT License.
