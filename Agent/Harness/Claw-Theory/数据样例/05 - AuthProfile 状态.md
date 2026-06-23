# 05 - AuthProfile 状态（多 key 轮换）

> 来源：[`claw0/sessions/zh/s09_resilience.py`](../claw0/sessions/zh/s09_resilience.py) L173-262
>
> [[09 - Resilience]] 的 `ProfileManager` 在内存里维护多个 `AuthProfile`（每个 API key 一个）。**教学版不落盘**——状态重启会丢失。OpenClaw 生产版会写到 `workspace/profiles.json`。

## AuthProfile 内存对象（教学版）

```python
@dataclass
class AuthProfile:
    name: str                    # "anthropic-primary" / "anthropic-backup-1" / ...
    provider: str                # "anthropic" / "openai" / ...
    api_key: str                 # 实际的 API key
    base_url: str | None = None  # 可选的 base URL override
    cooldown_until: float = 0.0  # 罚站截止时间戳（0 = 未罚站）
    failure_reason: str | None = None  # 最近一次失败原因（rate_limit / auth / ...）
    last_good_at: float = 0.0    # 上次成功时间戳
```

---

## 生产版 profiles.json（持久化版）

如果教学版加落盘，长这样（**claw0 仓库没有这个文件**——下面是按生产模式推演）：

```json
{
  "profiles": [
    {
      "name": "anthropic-primary",
      "provider": "anthropic",
      "api_key": "sk-ant-api03-xxxxxxxxxxxxxxxx",
      "base_url": null,
      "cooldown_until": 0.0,
      "failure_reason": null,
      "last_good_at": 1718956200.123
    },
    {
      "name": "anthropic-backup-1",
      "provider": "anthropic",
      "api_key": "sk-ant-api03-yyyyyyyyyyyyyyyy",
      "base_url": null,
      "cooldown_until": 1718956320.456,
      "failure_reason": "rate_limit",
      "last_good_at": 1718955000.789
    },
    {
      "name": "openrouter-fallback",
      "provider": "openrouter",
      "api_key": "sk-or-v1-zzzzzzzzzzzzzzzz",
      "base_url": "https://openrouter.ai/api/v1",
      "cooldown_until": 0.0,
      "failure_reason": null,
      "last_good_at": 0.0
    }
  ],
  "fallback_models": [
    "claude-3-5-sonnet-20241022",
    "claude-3-5-haiku-20241022"
  ]
}
```

---

## 字段语义

### 单个 profile

| 字段 | 类型 | 含义 |
|---|---|---|
| `name` | string | 可读标签，日志和 cooldown 时引用 |
| `provider` | string | LLM 服务商（决定用哪个 SDK） |
| `api_key` | string | 实际密钥（生产代码应加密存储） |
| `base_url` | string? | 可选 base URL（用于代理 / 自部署 / OpenRouter） |
| `cooldown_until` | float | 罚站截止时间戳（Unix epoch）；0 表示未罚站 |
| `failure_reason` | string? | 最近失败原因；`null` 表示从未失败或最近 mark_success 过 |
| `last_good_at` | float | 上次成功时间戳；0 表示从未成功过 |

### 顶层（生产版）

| 字段 | 含义 |
|---|---|
| `profiles` | API key 列表 |
| `fallback_models` | 模型降级列表（主 key 全失败时换这些 model 再试） |

---

## `cooldown_until` 的几种状态

```
cooldown_until = 0.0
    ↓ 含义：从未罚站过 / 已经刑满释放
    ↓ select_profile 会选它

cooldown_until = 1718956320.456（未来时间）
    ↓ 含义：还在罚站
    ↓ select_profile 跳过它

cooldown_until = 1718955000.123（过去时间）
    ↓ 含义：罚站期满
    ↓ select_profile 会选它（now >= cooldown_until 成立）
```

---

## `failure_reason` 的 6 种值

跟 [[09 - Resilience#6 种错误分类：classify_failure|FailoverReason]] 一一对应：

| reason | cooldown 时长 | 语义 |
|---|---|---|
| `"rate_limit"` | 120s | HTTP 429（限流） |
| `"auth"` | 3600s | HTTP 401（key 无效） |
| `"timeout"` | 60s | 网络超时 |
| `"billing"` | 3600s | HTTP 402（欠费） |
| `"overflow"` | — | 上下文超限（不冷却，走压缩路径） |
| `"unknown"` | 300s | 未识别错误（保守处理） |

---

## 实际运行中的状态变化

### 初始（所有 key 健康）

```json
[
  {"name": "primary",  "cooldown_until": 0.0, "failure_reason": null, ...},
  {"name": "backup-1", "cooldown_until": 0.0, "failure_reason": null, ...},
  {"name": "backup-2", "cooldown_until": 0.0, "failure_reason": null, ...}
]
```

`select_profile()` 返回 `primary`（第一个可用的）。

### Primary 被 429（rate_limit）

```json
[
  {"name": "primary",  "cooldown_until": 1718956320.456, "failure_reason": "rate_limit", ...},
  {"name": "backup-1", "cooldown_until": 0.0, "failure_reason": null, ...},
  {"name": "backup-2", "cooldown_until": 0.0, "failure_reason": null, ...}
]
```

`select_profile()` 跳过 primary，返回 backup-1。

### 2 分钟后（primary 刑满）

```json
[
  {"name": "primary",  "cooldown_until": 1718956320.456, "failure_reason": "rate_limit", ...},
  ...
]
```

注意：`cooldown_until` **不变**（保留历史记录），但因为现在时间已超过它，`select_profile()` 又会选 primary。**这就是"罚站期满自动回归"**。

### Primary 又成功了

```json
[
  {"name": "primary", "cooldown_until": 1718956320.456, "failure_reason": null, "last_good_at": 1718956440.123, ...},
  ...
]
```

`mark_success` **只清 `failure_reason`，不清 `cooldown_until`**——因为成功的 key 本来就没在罚站，cooldown_until 是过去时戳，清它没意义。详见 [[09 - Resilience#Q6：mark_success 为什么不清 cooldown_until]]。

---

## fallback_models 的角色

主 key 全部失败后，**换模型**再试一遍。配置示例：

```json
"fallback_models": [
  "claude-3-5-sonnet-20241022",     // 主用
  "claude-3-5-haiku-20241022"        // 备胎 1
  "claude-3-haiku-20240307"          // 备胎 2（最便宜）
]
```

**降级语义**：Sonnet 挂了 → Haiku 4.5 → Haiku 3。每级模型越来越弱但越来越便宜/可用。详见 [[09 - Resilience#fallback 段：换模型再试一遍]]。

---

## 改造成 PuinClaw 时

PuinClaw 目前用 pi-mono 的 env-driven 单 key（`ANTHROPIC_API_KEY`），**没有轮换**。如果要加（比如想用 2 个 key 避免限流）：

```python
# PuinClaw 加多 key 的最小改动
PROFILES = [
    AuthProfile(name="primary",  api_key=os.environ["ANTHROPIC_API_KEY_PRIMARY"]),
    AuthProfile(name="backup-1", api_key=os.environ["ANTHROPIC_API_KEY_BACKUP_1"]),
]
```

env 加：

```bash
# .env
ANTHROPIC_API_KEY_PRIMARY=sk-ant-...
ANTHROPIC_API_KEY_BACKUP_1=sk-ant-...
```

---

## 相关

- [[09 - Resilience]] —— ProfileManager 完整实现、6 种错误分类、fallback 段
- [[08 - Delivery]] —— 投递层也有 retry_count，但语义不同（消息一次性 vs key 长期资产）
