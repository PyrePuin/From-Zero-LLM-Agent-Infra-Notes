# 02 - CRON 调度配置（CRON.json）

> 来源：[`claw0/workspace/CRON.json`](../claw0/workspace/CRON.json)
>
> [[07 - Heartbeat & Cron]] 的 `CronService.load_jobs()` 启动时读取这个文件，把每个 job 解析成内部 job dict，加上 `next_run_at` / `consecutive_errors` 等运行时字段。

## 文件全貌

```json
{
  "jobs": [
    {
      "id": "morning-briefing",
      "name": "Morning Briefing",
      "enabled": true,
      "schedule": {
        "kind": "cron",
        "expr": "0 9 * * *",
        "tz": "Asia/Shanghai"
      },
      "payload": {
        "kind": "agent_turn",
        "message": "Check today's calendar, weather forecast, and any pending reminders. Give a brief morning summary."
      },
      "delete_after_run": false
    },
    {
      "id": "dep-security-scan",
      "name": "Dependency Security Scan",
      "enabled": true,
      "schedule": {
        "kind": "cron",
        "expr": "0 9 * * 1",
        "tz": "Asia/Shanghai"
      },
      "payload": {
        "kind": "agent_turn",
        "message": "Run a security audit on project dependencies. Report any critical vulnerabilities."
      },
      "delete_after_run": false
    },
    {
      "id": "remind-meeting",
      "name": "Meeting Reminder",
      "enabled": true,
      "schedule": {
        "kind": "at",
        "at": "2026-02-25T09:30:00+08:00"
      },
      "payload": {
        "kind": "system_event",
        "text": "Reminder: You have a team meeting at 10:00 AM today."
      },
      "delete_after_run": true
    },
    {
      "id": "health-check",
      "name": "System Health Check",
      "enabled": true,
      "schedule": {
        "kind": "every",
        "every_seconds": 3600,
        "anchor": "2026-02-24T00:00:00+08:00"
      },
      "payload": {
        "kind": "agent_turn",
        "message": "Check system health: memory usage, disk space, and running services. Report only if something needs attention."
      },
      "delete_after_run": false
    }
  ]
}
```

---

## 字段语义

### 顶层

```json
{
  "jobs": [...]   // cron job 列表
}
```

### 单个 job

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `id` | string | ✓ | job 唯一 ID（用于日志和去重） |
| `name` | string | ✓ | 可读名字（日志显示用） |
| `enabled` | bool | ✓ | 是否启用（false 跳过） |
| `schedule` | object | ✓ | 调度配置（4 种 kind） |
| `payload` | object | ✓ | 触发时执行什么（2 种 kind） |
| `delete_after_run` | bool | ✓ | 跑完后是否从配置删除（用于"一次性提醒"） |

### schedule 的 4 种 kind

#### 1. `cron` —— 标准 cron 表达式

```json
"schedule": {
  "kind": "cron",
  "expr": "0 9 * * *",      // 每天 9:00
  "tz": "Asia/Shanghai"      // 可选，默认 UTC
}
```

**expr 格式**：标准 5 字段 cron——`minute hour day-of-month month day-of-week`

例子：
- `"0 9 * * *"` 每天 9:00
- `"0 9 * * 1"` 每周一 9:00
- `"*/30 * * * *"` 每 30 分钟
- `"0 0 1 * *"` 每月 1 号 0:00

#### 2. `at` —— 一次性定时

```json
"schedule": {
  "kind": "at",
  "at": "2026-02-25T09:30:00+08:00"   // ISO 8601 带时区
}
```

**只触发一次**——到达时间后跑，跑完如果 `delete_after_run=true` 就从配置删除。**用于"X 月 X 日提醒我开会"**。

#### 3. `every` —— 间隔触发（带 anchor）

```json
"schedule": {
  "kind": "every",
  "every_seconds": 3600,                          // 每 3600 秒（1 小时）
  "anchor": "2026-02-24T00:00:00+08:00"           // 对齐到这个时间
}
```

**anchor 的作用**：把触发时间对齐到固定点。比如 `every_seconds=3600, anchor=00:00`——无论何时启动 cron 服务，触发点都是整点（00:00 / 01:00 / 02:00...）。**重启后立刻按原计划继续**，不漂移。

详见 [[07 - Heartbeat & Cron#Q6：CronService 的 every 类型为什么对齐到 anchor]]。

#### 4. 教学版未实现的（生产代码有）

`daily` / `weekly` / `monthly` 这些更高级的调度类型教学版没实现，OpenClaw 生产版有。

### payload 的 2 种 kind

#### 1. `agent_turn` —— 让 agent 跑一轮

```json
"payload": {
  "kind": "agent_turn",
  "message": "Check today's calendar..."
}
```

**效果**：`message` 作为 user input 投给 agent，跑完整 agent_loop（含 tool_use）。**最常用的 payload 类型**。

#### 2. `system_event` —— 系统事件（不跑 agent）

```json
"payload": {
  "kind": "system_event",
  "text": "Reminder: You have a team meeting at 10:00 AM today."
}
```

**效果**：`text` 直接投递到对话历史，**不让 agent 处理**——用于"硬提醒"，agent 不消耗 token 思考。

---

## 运行时状态字段（不写进 CRON.json）

`CronService.load_jobs()` 读取后会给每个 job 加上运行时字段（**不写回 CRON.json**）：

```python
# 内存中的 job dict（s07_heartbeat_cron.py L524-533）
{
    "id": "morning-briefing",
    "name": "Morning Briefing",
    "enabled": True,
    "every_seconds": 86400,         # 来自 schedule
    "payload": {...},
    "last_run_at": 0.0,             # 运行时：上次跑的时间
    "next_run_at": <startup + every>,  # 运行时：下次该跑的时间
    "consecutive_errors": 0,        # 运行时：连续失败计数
}
```

**这些字段只在内存**——服务重启会丢失 `last_run_at`（但 `next_run_at` 会基于 schedule 重新计算）。如果需要持久化失败状态，生产代码会单独存到 `cron_state.json`。

---

## 4 种 job 的对比

| job | schedule.kind | payload.kind | delete_after_run | 用途 |
|---|---|---|---|---|
| `morning-briefing` | cron (`0 9 * * *`) | agent_turn | false | 每天早报 |
| `dep-security-scan` | cron (`0 9 * * 1`) | agent_turn | false | 每周一定时任务 |
| `remind-meeting` | at | system_event | **true** | 一次性提醒（跑完删） |
| `health-check` | every (3600s) | agent_turn | false | 每小时健康检查 |

**delete_after_run=true 的语义**：跑完后从 CRON.json 删除——典型用于"X 月 X 日提醒"，到点提醒完就不需要再保留了。

---

## 改造成 PuinClaw 时怎么写

最小化版本（只保留一个 daily job）：

```json
{
  "jobs": [
    {
      "id": "daily-standup",
      "name": "Daily Standup",
      "enabled": true,
      "schedule": {
        "kind": "cron",
        "expr": "0 10 * * *",
        "tz": "Asia/Shanghai"
      },
      "payload": {
        "kind": "agent_turn",
        "message": "Read PuinClaw progress.md and summarize what was done yesterday. Suggest today's top 3 priorities."
      },
      "delete_after_run": false
    }
  ]
}
```

---

## 相关

- [[07 - Heartbeat & Cron]] —— CronService 怎么读这个文件、`_compute_next` 怎么算下次触发
- [[10 - Concurrency]] —— s10 里 cron 怎么进 `lane "cron"`
- [[14 - Cron Scheduler|learn-claude-code s14]] —— 对照设计：s14 的 cron 走 agent_loop 单路径
