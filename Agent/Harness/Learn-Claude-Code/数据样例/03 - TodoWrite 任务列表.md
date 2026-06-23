# 03 - TodoWrite 任务列表

> 来源：Claude Code 内置的 TodoWrite 工具，数据在内存（教学版）；生产用 `.claude/todos.json`
>
> [[05 - TodoWrite]] 让模型在复杂任务里**自己列 todo 清单**，每完成一项更新状态。本质是给模型一个"显式工作记忆"，避免长任务里漏步。

## todos.json 结构

```json
{
  "todos": [
    {
      "id": "1",
      "content": "Read agent.ts to understand current architecture",
      "status": "completed",
      "priority": "high",
      "created_at": "2026-06-23T10:00:00Z",
      "completed_at": "2026-06-23T10:02:30Z"
    },
    {
      "id": "2",
      "content": "Identify where to inject multi-key profile manager",
      "status": "completed",
      "priority": "high",
      "created_at": "2026-06-23T10:02:35Z",
      "completed_at": "2026-06-23T10:05:00Z"
    },
    {
      "id": "3",
      "content": "Add ProfileManager class with cooldown logic",
      "status": "in_progress",
      "priority": "high",
      "created_at": "2026-06-23T10:05:10Z",
      "completed_at": null
    },
    {
      "id": "4",
      "content": "Wire ProfileManager into agent_loop",
      "status": "pending",
      "priority": "medium",
      "created_at": "2026-06-23T10:05:15Z",
      "completed_at": null
    },
    {
      "id": "5",
      "content": "Add /retry and /health REPL commands",
      "status": "pending",
      "priority": "low",
      "created_at": "2026-06-23T10:05:20Z",
      "completed_at": null
    }
  ],
  "updated_at": "2026-06-23T10:15:00Z"
}
```

---

## 字段语义

### 单个 todo

| 字段 | 类型 | 含义 |
|---|---|---|
| `id` | string | 唯一 ID（递增整数或 UUID） |
| `content` | string | 任务描述（动词开头） |
| `status` | enum | `pending` / `in_progress` / `completed` |
| `priority` | enum | `low` / `medium` / `high` |
| `created_at` | ISO 8601 | 创建时间 |
| `completed_at` | ISO 8601? | 完成时间（pending/in_progress 时为 null） |

### 顶层

| 字段 | 含义 |
|---|---|
| `todos` | 任务数组（顺序即创建顺序） |
| `updated_at` | 最近一次任意 todo 更新时间 |

---

## status 流转

```
pending ──set in_progress──> in_progress ──set completed──> completed
   ↑                              │
   └──────set pending─────────────┘（重新打开）
```

**关键约束**：同一时刻**只能有一个 in_progress**——强迫模型"专注做完一件再做下一件"。

---

## TodoWrite 工具的 schema

```json
{
  "name": "TodoWrite",
  "description": "Update the todo list. Use proactively for multi-step tasks (3+ steps).",
  "parameters": {
    "todos": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "id":       {"type": "string"},
          "content":  {"type": "string"},
          "status":   {"type": "string", "enum": ["pending", "in_progress", "completed"]},
          "priority": {"type": "string", "enum": ["low", "medium", "high"]}
        }
      }
    }
  }
}
```

**注意**：调用 TodoWrite 是**整体替换**——传入完整 todos 数组，不是增量更新。模型每次重写整个列表。

---

## 何时触发 TodoWrite

模型**自主判断**。一般触发条件：
- 任务**有 3+ 步骤**
- 步骤**有明确顺序**
- 用户**要求"系统性"工作**（"重构 X"、"实现 Y"、"修复 Z 并加测试"）

不触发的场景：
- 简单问答（"什么是 closure？"）
- 单步任务（"改这一行的变量名"）
- 探索性问题（"这里有什么文件？"）

---

## TodoWrite 在 system prompt 里的投影

当前 todo list 会被**注入到 system prompt**——让模型每轮都"看到"自己列的清单：

```markdown
# Current Task State

Active todo:
[in_progress] Add ProfileManager class with cooldown logic

Pending:
- Wire ProfileManager into agent_loop
- Add /retry and /health REPL commands

Completed:
- Read agent.ts to understand current architecture
- Identify where to inject multi-key profile manager
```

**这就是 TodoWrite 的核心价值**——给模型一个**持久的工作记忆**，避免在长任务中"忘记自己要干什么"。

---

## 跟 Task System 的差异

[[12 - Task System]] 是更高级的版本，支持：
- 任务间依赖（task A blocks task B）
- 子任务嵌套
- 持久化到磁盘（独立 .md 文件）
- 跨会话恢复

TodoWrite 是**轻量、单会话、内存级**——简单任务用它够，复杂任务升级到 Task System。

---

## 相关

- [[05 - TodoWrite]] —— 完整实现、为什么"显式 todo"比"心里 todo"强
- [[12 - Task System]] —— 升级版，支持依赖和持久化
- [[08 - Context Compact]] —— todo list 不被压缩（高优先级）
