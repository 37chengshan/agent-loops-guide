# 可直接抄的代码与设计模板

## 0. Loop Contract（先写这个，再写 Worker）

```yaml
unit: "handlers slice"
input: "porting brief + source files"
output: "ported handlers"
green:
  - "pytest tests/handlers exits 0"
  - "diff touches handlers/** only"
max_attempts: 3
on_red:
  route: "return_this_unit"
  required_fields: [unit, verdict, reason, evidence, scope]
```

规则：

- `green` 必须能明确失败；“看起来不错”“模型有信心”“没报错”不算。
- `max_attempts` 默认 3；还不过就升级到 plan / splitter，不允许无限 self-correction。
- `scope` 是 correction 的硬边界，防止“顺手修旁边问题”。

## 0.5 Edge Audit（删掉假依赖）

对 graph 里每条箭头填：

| From | To | To 实际读取 From 的哪个输出变量？ | 保留 edge？ |
|---|---|---|---|
| summarize-file | check-weather | 无 | 否 |
| build | test | build artifact | 是 |
| worker-a | merge | result-a | 是 |

判断式：

```text
if cannot_name(variable_crossing_edge):
    remove_edge()
```

不要把自然语言里的“然后”自动翻译成依赖。

## 1. Hash 乐观锁（Concurrency）

共享 memory / state 写入前读 hash，提交时发现版本变化就拒绝并重试：

```python
import hashlib, json
from pathlib import Path

class VersionedState:
    def __init__(self, path):
        self.path = Path(path)
        self.log = self.path.with_suffix(".jsonl")

    def _hash(self, data):
        return hashlib.sha256(
            json.dumps(data, sort_keys=True, ensure_ascii=False).encode()
        ).hexdigest()[:16]

    def read(self):
        data = json.loads(self.path.read_text(encoding="utf-8"))
        return data, self._hash(data)

    def write(self, data, expected_hash, actor, note=""):
        cur, cur_hash = self.read()
        if expected_hash != cur_hash:
            raise ConcurrentWriteError(
                f"expected {expected_hash}, actual {cur_hash}: state changed; reread and retry"
            )

        new_hash = self._hash(data)
        if new_hash != cur_hash:
            self.path.write_text(
                json.dumps(data, ensure_ascii=False, indent=2),
                encoding="utf-8",
            )
            with open(self.log, "a", encoding="utf-8") as f:
                f.write(json.dumps({
                    "actor": actor,
                    "note": note,
                    "prev": cur_hash,
                    "new": new_hash,
                }, ensure_ascii=False) + "\n")
        return new_hash

class ConcurrentWriteError(Exception):
    pass
```

调用模式：

```text
read -> modify -> write(expected_hash)
```

捕获 `ConcurrentWriteError` 后重读重试；连续 3 次仍冲突就告警，说明存在系统性争用。

## 2. Verdict / Correction Packet（Learning Edge 输入）

每个 unit 结束至少记：

```jsonl
{"unit":"handlers","verdict":"red","reason":"test_auth_redirect failed","evidence":"expected 302, got 200; handlers/auth.py:88","scope":"fix handlers/auth.py only","attempt":1}
{"unit":"handlers","verdict":"corrected","reason":"redirect status fixed","evidence":"pytest tests/handlers exits 0","scope":"handlers/auth.py","attempt":2}
{"unit":"utils","verdict":"useful","reason":"first-pass green","evidence":"pytest tests/utils exits 0","scope":"-","attempt":1}
```

推荐 verdict：

- `useful`：一次通过
- `corrected`：被打回后通过
- `dead_end`：这条路线本身不通
- `red`：当前 attempt 失败，准备 correction

`corrected` 扎堆的位置就是 learning / dreaming 应该优先提炼 constraint 的地方。

## 3. Gate Lane 表（按 Blast Radius）

```yaml
lanes:
  reversible_local:
    examples: [copy, tests, isolated_function]
    auto_open: true
    requires: [deterministic_checks]

  reversible_wide:
    examples: [shared_util, schema, shared_config]
    auto_open: conditional
    requires: [deterministic_checks, clean_trajectory, rollback_history]

  hard_to_reverse:
    examples: [migration, delete, production_write, money_movement]
    auto_open: false
    requires: [explicit_human_approval]
```

注意：第三档不是“置信度阈值更高”，而是 lane 默认没有自动开启路径。

## 4. 写权限分区表（Permissioning）

| 角色 | 可写 | 只读 | 禁止 |
|---|---|---|---|
| 采集者 | 原始数据区（append-only） | 规则文档 | 改共享规则 |
| 建造者 | 自己的工作目录 | 模板/规则 | 改共享状态 |
| 审查者 | verdict（经 orchestrator 落盘） | 产物 | 直接改产物 |
| orchestrator | 状态文件（经 VersionedState） | 一切 | 绕过 helper 手写 |
| dreaming | `proposals/*.md` | 轨迹、规则、历史 verdict | 直接改生产规则 |

原则：共享状态的写入口尽量单一；dreaming 默认只有提案权。

## 5. Learning Edge 记录模板

accepted / corrected 以后，不要只存结果；要导出未来约束：

```yaml
source_unit: "utils slice"
verdict: accepted
evidence: "tests green on first pass"
derived_constraint: "adapters must preserve keyword args exactly"
lands_in: "splitter brief"
applies_to: "all future adapter slices"
status: "proposed"
```

只有当这条 constraint 真正进入未来 brief / route 时，learning edge 才算闭合。

## 6. 三层检索（抄 claude-mem 的渐进读取思想）

```text
L1 索引：每条记忆 <=100 token，一行标题 + 紧凑摘要，全量扫
L2 时间线：看命中条目的前后上下文、谁写的、何时写的、关联事件
L3 全文：只取 L2 确认需要的少量正文
```

规则：默认不跳过 L1 直接读全文。

L1 未命中但后来事实证明“本来应该有记忆”时，记录一次 `dead_end`，交给 dreaming 判断：是检索词设计错了，还是确实缺了一条 memory。

## 7. Path Metrics 最小记录

```json
{
  "unit": "handlers",
  "attempts": 2,
  "first_pass_green": false,
  "correction_count": 1,
  "scope_violation": false,
  "latency_ms": 8421,
  "tokens": 12640,
  "rolled_back": false,
  "learning_constraint_emitted": true
}
```

不要只评 final answer。至少长期看：first-pass green rate、attempts/unit、correction rate、rollback、latency、cost/accepted-unit，以及历史 constraint 被实际复用的次数。
