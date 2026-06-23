# 08 - Error Recovery 状态

> 来源：`learn-claude-code/s11_error_recovery/code.py` L79-87
>
> [[11 - Error Recovery]] 用 `RecoveryState` 跨迭代记忆"上次失败时模型在干什么"，决定这次怎么救。**完全内存**——不落盘。

## RecoveryState 内存对象

```python
@dataclass
class RecoveryState:
    consecutive_errors: int = 0           # 连续错误计数
    last_error_type: str | None = None    # 最近错误类型
    has_escalated: bool = False           # 是否已升级 max_tokens
    escalation_level: int = 0             # 0=8K, 1=32K, 2=64K
    last_compact_at: float = 0.0          # 上次压缩时间
    has_compacted: bool = False           # 是否已 reactive_compact 过
    retry_count_this_turn: int = 0        # 当前 turn 重试次数
```

---

## 实际状态变化轨迹

### 初始状态（健康）

```python
RecoveryState(
    consecutive_errors=0,
    last_error_type=None,
    has_escalated=False,
    escalation_level=0,
    last_compact_at=0.0,
    has_compacted=False,
    retry_count_this_turn=0,
)
```

### 第一次失败（context overflow）

```python
RecoveryState(
    consecutive_errors=1,
    last_error_type="context_length_exceeded",  # ← 改
    has_escalated=False,
    escalation_level=0,
    last_compact_at=0.0,
    has_compacted=False,
    retry_count_this_turn=1,                    # ← 改
)
```

**模型会做的事**：触发 `reactive_compact`（丢尾部 5 条），重试。

### reactive_compact 后再失败

```python
RecoveryState(
    consecutive_errors=2,
    last_error_type="context_length_exceeded",
    has_escalated=False,
    escalation_level=0,
    last_compact_at=1718956200.0,              # ← 改
    has_compacted=True,                         # ← 改（不再压）
    retry_count_this_turn=2,
)
```

**模型会做的事**：升级 max_tokens（8K → 32K），重试。

### max_tokens 升级后再失败

```python
RecoveryState(
    consecutive_errors=3,
    last_error_type="context_length_exceeded",
    has_escalated=True,                         # ← 改
    escalation_level=1,                         # ← 改（8K → 32K）
    last_compact_at=1718956200.0,
    has_compacted=True,
    retry_count_this_turn=3,
)
```

**模型会做的事**：升级到 64K（level=2），重试。还不行就抛异常。

### 终于成功

```python
RecoveryState(
    consecutive_errors=0,                       # ← 清零
    last_error_type=None,                        # ← 清
    has_escalated=True,                          # 保留（这一轮内有效）
    escalation_level=1,                          # 保留
    last_compact_at=1718956200.0,
    has_compacted=True,
    retry_count_this_turn=0,                     # ← 清零
)
```

**下一轮开始**：`retry_count_this_turn` 重置；但 `has_escalated` 跨轮保留（避免反复升级）。

---

## 状态驱动的恢复策略

```python
# s11_error_recovery.py 的 with_retry（简化）
def with_retry(state: RecoveryState, fn, messages, *args):
    while True:
        try:
            response = call_api(messages, max_tokens=MAX_TOKENS[state.escalation_level])
            state.consecutive_errors = 0
            state.retry_count_this_turn = 0
            return response

        except Exception as exc:
            state.consecutive_errors += 1
            state.retry_count_this_turn += 1
            state.last_error_type = type(exc).__name__

            # 策略 1：rate_limit → 退避重试
            if is_rate_limit(exc):
                time.sleep(backoff_with_jitter(state.retry_count_this_turn))
                continue

            # 策略 2：context overflow → 压缩或升级
            if is_context_overflow(exc):
                if not state.has_compacted:
                    messages = reactive_compact(messages)
                    state.has_compacted = True
                    continue
                elif not state.has_escalated:
                    state.has_escalated = True
                    state.escalation_level += 1
                    continue

            # 策略 3：超过上限 → 抛
            if state.retry_count_this_turn >= MAX_RETRIES:
                raise

            # 策略 4：未知错误 → 退避重试
            time.sleep(backoff_with_jitter(state.retry_count_this_turn))
```

---

## MAX_TOKENS 升级阶梯

```python
MAX_TOKENS = [
    8192,       # level 0：默认（便宜）
    32768,      # level 1：升级 1 次
    65536,      # level 2：升级 2 次（最大）
]
```

**升级触发**：context overflow + 已经 reactive_compact 过 + 还没升级过。

**为什么阶梯式**：先用最便宜的配额（8K），不够再升级——大部分任务用 8K 就够，少数需要 32K，极少数需要 64K。

---

## 退避 + jitter 公式

```python
def backoff_with_jitter(attempt: int) -> float:
    base = 2 ** attempt              # 1, 2, 4, 8, 16... 秒
    jitter = random.uniform(0.8, 1.2)  # ±20%
    return base * jitter
```

**示例轨迹**：
- attempt 1: `2 * [0.8~1.2] = 1.6~2.4s`
- attempt 2: `4 * [0.8~1.2] = 3.2~4.8s`
- attempt 3: `8 * [0.8~1.2] = 6.4~9.6s`

**jitter 防雷群效应**——50 个并发请求同时失败，不会都 2 秒后重试，散布在 1.6-2.4 秒之间。

---

## 跟 claw0 [[09 - Resilience]] 的对比

| 维度 | s11 RecoveryState | s09 ProfileManager |
|---|---|---|
| 状态 | **每 agent 一个** RecoveryState | **每个 key 一个** AuthProfile |
| 跨迭代 | ✓（state 持久） | 部分（cooldown_until 时间戳） |
| 跨 turn | ✓ | ✗ |
| 升级机制 | max_tokens 升级 | 模型 fallback |
| 压缩机制 | reactive_compact（丢尾部） | 两阶段（truncate + LLM 摘要） |
| 适用场景 | 单 key CLI | 多 key 常驻 bot |

详见 [[09 - Resilience|claw0 s09 vs learn-claude-code s11]]。

---

## 改造成 PuinClaw 时

PuinClaw 目前**没有错误恢复层**——pi-mono 内置的 retry 就是 `with_retry` 的简化版。如果要加：

```typescript
// PuinClaw 的 RecoveryState（推演）
interface RecoveryState {
  consecutiveErrors: number;
  lastErrorType: string | null;
  hasEscalated: boolean;
  escalationLevel: number;  // 0=8K, 1=32K, 2=64K
  hasCompacted: boolean;
  retryCountThisTurn: number;
}
```

跟 s11 几乎一致——s11 的设计很通用，可以直接搬。

---

## 相关

- [[11 - Error Recovery]] —— 完整实现、4 种错误策略
- [[08 - Context Compact]] —— reactive_compact 的"主动压缩"姊妹（reactive 是错误后的）
- [[09 - Resilience|claw0 s09]] —— 对照设计（多 key 轮换 vs 单 key 自愈）
