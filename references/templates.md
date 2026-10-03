# 可直接抄的代码模板

## 1. Hash 乐观锁（Concurrency）

演讲的 hash 方案在开源里基本空白，这是最小可用实现：

```python
import hashlib, json
from pathlib import Path

class VersionedState:
    def __init__(self, path):
        self.path = Path(path)
        self.log = self.path.with_suffix(".jsonl")  # 版本日志

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
                f"expected {expected_hash}, actual {cur_hash}：期间被改过，重读重试")
        new_hash = self._hash(data)
        if new_hash != cur_hash:  # 真有变化才写
            self.path.write_text(json.dumps(data, ensure_ascii=False, indent=1),
                                 encoding="utf-8")
            with open(self.log, "a", encoding="utf-8") as f:
                f.write(json.dumps({"actor": actor, "note": note,
                                    "prev": cur_hash, "new": new_hash},
                                   ensure_ascii=False) + "\n")
        return new_hash

class ConcurrentWriteError(Exception): pass
```

调用方模式：`read → 改 → write(expected_hash)`，捕获 `ConcurrentWriteError` 后重读重试（最多 3 次，还冲突就告警——说明有系统性争用）。

## 2. Verdict 记录（Learning edge 输入）

每次工作单元结束记一条（抄 graphify `save-result`）：

```jsonl
{"unit": "shop-xyz/v3", "verdict": "corrected", "reason": "R2 驳回：编造了客户评价数量",
 "evidence": "review-r2.md:12", "scope": "只改 v3 testimonials 段"}
{"unit": "shop-abc/v1", "verdict": "useful", "reason": "一次通过",
 "evidence": "manifest b20261003-1140", "scope": "-"}
```

verdict 取值：`useful`（一次过）/ `dead_end`（此路不通）/ `corrected`（被打回修好）。
dreaming 的 Enrich 就吃这个：`corrected` 扎堆的地方就是规则缺口。

## 3. 写权限分区表（Permissioning）

| 角色 | 可写 | 只读 | 禁止 |
|---|---|---|---|
| 采集者 | 原始数据区（append-only） | 规则文档 | 改状态 |
| 建造者 | 自己的工作目录 | 模板/规则 | 改共享状态 |
| 审查者 | verdict（经 orchestrator 落盘） | 产物 | 直接改产物 |
| orchestrator | 状态文件（经 VersionedState） | 一切 | 绕过 helper 手写 |
| dreaming | proposals/*.md | 一切 | 直接改规则文档 |

原则：共享状态的写入口只有一个；dreaming 只有提案权。

## 4. 三层检索（抄 claude-mem）

```
L1 索引：每条记忆 ≤100 token（一行：标题 + 5 词摘要），全量扫
L2 时间线：命中条目的前后上下文（什么时候、谁写的、关联事件）
L3 全文：只取 L2 确认需要的那几条
```

规则：永远不许跳过 L1 直接读全文；L1 未命中 → 记一次 dead_end（喂给 dreaming：是不是缺记忆？）。
