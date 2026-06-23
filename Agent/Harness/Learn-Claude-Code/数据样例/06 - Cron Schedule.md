# 06 - Cron Schedule

> 来源：`learn-claude-code/s14_cron_scheduler/` + Claude Code 内置 cron
>
> [[14 - Cron Scheduler]] 让 Claude Code 支持"延迟任务 / 定时任务"——比如"每天 9 点跑日报"、"3 小时后提醒我"。

## 文件位置

```
.claude/cron.json
```

---

## cron.json 结构

```json
{
  "jobs": [
    {
      "id": "daily-summary",
      "name": "Daily Summary",
      "enabled": true,
      "schedule": {
        "type": "cron",
        "expression": "0 9 * * *",
        "timezone": "Asia/Shanghai"
      },
      "action": {
        "type": "prompt",
        "prompt": "Generate today's summary: what was completed, what's blocked, what's next."
      },
      "delete_after_run": false,
      "created_at": "2026-06-20T10:00:00Z",
      "last_run_at": "2026-06-23T09:00:00Z",
      "next_run_at": "2026-06-24T09:00:00Z"
    },
    {
      "id": "remind-standup",
      "name": "Standup Reminder",
      "enabled": true,
      "schedule": {
        "type": "at",
        "time": "2026-06-24T10:00:00+08:00"
      },
      "action": {
        "type": "prompt",
        "prompt": "Remind user about standup meeting in 5 minutes."
      },
      "delete_after_run": true
    },
    {
      "id": "weekly-retrospective",
      "name": "Weekly Retro",
      "enabled": true,
      "schedule": {
        "type": "cron",
        "expression": "0 18 * * 5",
        "timezone": "Asia/Shanghai"
      },
      "action": {
        "type": "prompt",
        "prompt": "Generate weekly retrospective: accomplishments, learnings, next week plan."
      },
      "delete_after_run": false
    }
  ]
}
```

---

## 字段语义

### 单个 job

| 字段 | 必填 | 含义 |
|---|---|---|
| `id` | ✓ | 唯一 ID |
| `name` | ✓ | 可读名字 |
| `enabled` | ✓ | 是否启用 |
| `schedule` | ✓ | 调度配置 |
| `action` | ✓ | 触发时执行什么 |
| `delete_after_run` | ✓ | 跑完是否删除 |
| `created_at` | ✓ | 创建时间 |
| `last_run_at` | 运行时 | 上次跑时间 |
| `next_run_at` | 运行时 | 下次该跑时间 |

### schedule 的 2 种类型

#### 1. `cron` —— 标准 cron 表达式

```json
{
  "type": "cron",
  "expression": "0 9 * * *",       // 标准 5 字段
  "timezone": "Asia/Shanghai"       // 可选
}
```

#### 2. `at` —— 一次性

```json
{
  "type": "at",
  "time": "2026-06-24T10:00:00+08:00"   // ISO 8601
}
```

**只触发一次**——跑完后 `delete_after_run=true` 会删除。

### action 的 2 种类型

#### 1. `prompt` —— 投递一条 user message

```json
{
  "type": "prompt",
  "prompt": "Generate today's summary..."
}
```

**效果**：把 `prompt` 作为 user input 投递到 agent_loop，跑完整 agent（含 tool_use）。**最常用**。

#### 2. `command` —— 执行 slash command

```json
{
  "type": "command",
  "command": "/commit"
}
```

**效果**：直接执行 slash command，不让 LLM 思考。

---

## 跟 claw0 [[07 - Heartbeat & Cron]] 的对比

| 维度 | learn-claude-code s14 | claw0 s07 |
|---|---|---|
| 路径 | 进 agent_loop（共享 messages） | 直接调 `run_agent_single_turn` |
| 锁 | **共享 agent_lock**（用户和 cron 互斥） | 不抢锁（独立 messages） |
| 语义 | "用户委托的延迟指令" | "agent 主动的告知行为" |
| 失败处理 | consecutive_errors 自动禁用 | 同 |
| schedule 类型 | cron + at | cron + at + every + daily/weekly |
| 锚点对齐 | 不强调 | **anchor 对齐**（防漂移） |

**核心差异**：s14 是**对称互斥**（用户和 cron 排同一个队），s07 是**不对称隔离**（cron 走独立路径）。详见 [[07 - Heartbeat & Cron#vs learn-claude-code s14：两种 cron 范式]]。

---

## 实际运行流程

```
10:00:00  CronScheduler 启动后台线程
10:00:01  扫描 cron.json，计算每个 job 的 next_run_at
10:00:02  sleep 直到最近的 next_run_at
   ...
09:00:00  daily-summary 到期
09:00:00  acquire(agent_lock)  ← 抢锁
09:00:00  if 锁被用户占着，等待
09:00:30  用户释放锁
09:00:30  投递 "Generate today's summary..." 作为 user message
09:00:30  跑 agent_loop（含 tool_use 多轮）
09:00:45  完成，更新 last_run_at + next_run_at
09:00:45  release(agent_lock)
```

**关键**：用户消息和 cron 在**同一个 lock 后面排队**——cron 跑时用户得等，用户跑时 cron 得等。

---

## ScheduleNext / ListScheduledTasks 工具

```json
{
  "name": "ScheduleNext",
  "description": "Schedule a task for future execution",
  "parameters": {
    "name": "string",
    "schedule": {...},
    "action": {...},
    "delete_after_run": "bool"
  }
}
```

模型自主调用——用户说"明天 9 点提醒我 X"时，模型调 ScheduleNext 建一个 at 任务。

---

## 改造成 PuinClaw 时

PuinClaw 目前用 goal-runner 的 `request_pause` / `mark_complete` 替代 cron——单 goal 内部没有"定时"概念。如果要加（比如"每天检查 goal 进度"），参考 s07 实现：

```typescript
// PuinClaw cron（推演）
class CronService {
  async tick() {
    const jobs = await this.loadJobs();
    for (const job of jobs) {
      if (this.shouldRun(job)) {
        await this.runJob(job);  // 直接调 agent.prompt()
      }
    }
  }
}
```

---

## 相关

- [[14 - Cron Scheduler]] —— 完整实现、对称互斥设计
- [[13 - Background Tasks]] —— cron 跟 background task 的关系
- [[07 - Heartbeat & Cron|claw0 s07]] —— 对照设计（不对称隔离）
