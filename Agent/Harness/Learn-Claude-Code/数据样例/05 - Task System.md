# 05 - Task System

> 来源：Claude Code 内置 + `learn-claude-code/s12_task_system/`
>
> [[12 - Task System]] 是 TodoWrite 的升级版——支持**依赖关系、子任务、持久化到磁盘**。适合需要"几天/几周才能完成"的长任务。

## 目录结构

```
.claude/tasks/
├── active/
│   ├── refactor-auth-middleware.md       ← 进行中
│   └── add-payment-system.md
├── completed/
│   └── fix-login-bug.md                  ← 已完成
└── blocked/
    └── migrate-to-postgres.md            ← 被阻塞
```

---

## 单个 task 文件

文件：`.claude/tasks/active/refactor-auth-middleware.md`

```markdown
---
id: refactor-auth-middleware
title: Refactor Auth Middleware for Compliance
status: in_progress
priority: high
created_at: 2026-06-20T10:00:00Z
updated_at: 2026-06-23T14:30:00Z
owner: claude
blocks: [add-payment-system]
blocked_by: []
tags: [auth, compliance, refactor]
---

# Refactor Auth Middleware

## Goal
Rewrite the session token storage to meet new legal compliance requirements.

## Context
- Legal flagged current implementation as non-compliant (2026-06-19)
- Affects 3 endpoints: /login, /logout, /refresh
- Deadline: 2026-07-01

## Subtasks

- [x] 1. Audit current implementation
  - [x] Map all session token flows
  - [x] Identify compliance gaps
- [ ] 2. Design new storage schema
- [ ] 3. Implement migration
  - [ ] 3.1 Add new schema
  - [ ] 3.2 Write backfill script
  - [ ] 3.3 Test in staging
- [ ] 4. Deploy
- [ ] 5. Remove old code

## Notes
- 2026-06-21: Audit complete. Found 3 gaps.
- 2026-06-23: Schema design in progress, discussing with security team.
```

---

## frontmatter 字段

| 字段 | 必填 | 含义 |
|---|---|---|
| `id` | ✓ | 任务 ID（文件名一致） |
| `title` | ✓ | 可读标题 |
| `status` | ✓ | `pending` / `in_progress` / `completed` / `blocked` |
| `priority` | ✓ | `low` / `medium` / `high` / `critical` |
| `created_at` | ✓ | ISO 8601 |
| `updated_at` | ✓ | 最近更新时间 |
| `owner` | 可选 | 负责方（claude / 用户 / 团队成员） |
| `blocks` | 可选 | 这个任务阻塞的其他 task ID 列表 |
| `blocked_by` | 可选 | 阻塞这个任务的其他 task ID 列表 |
| `tags` | 可选 | 标签（用于过滤） |

**关键字段**：`blocks` / `blocked_by`——**依赖图**。Task System 会按依赖顺序自动排序。

---

## body 结构

```markdown
# <title>

## Goal              ← 这个任务要达成什么（一句话）
## Context           ← 为什么要做、约束是什么
## Subtasks          ← 拆解步骤（嵌套 checkbox）
## Notes             ← 时间线日志（每次更新追加）
```

**vs TodoWrite**：
- TodoWrite 只有平铺的 todo list
- Task System 有 Goal / Context / Subtasks / Notes **4 个语义段**
- Task System 支持 Subtasks 嵌套（TodoWrite 不支持）

---

## 整体 task graph

多个 task 通过 `blocks` / `blocked_by` 形成 DAG：

```
audit-auth (completed)
     │
     ▼
refactor-auth-middleware (in_progress)  ← blocks 下面两个
     │
     ├─────────────────┐
     ▼                 ▼
add-payment-system   update-rate-limiter
     │                 │
     ▼                 ▼
deploy-v2            deploy-v2          ← 两个都被 deploy-v2 阻塞
                          │
                          ▼
                       deploy-v2
```

**Task System 自动计算"现在能干什么"**——只显示 `blocked_by` 都已完成的 task。

---

## task 的生命周期

```
created (pending)
    ↓ 开始做
in_progress
    ↓ 完成
completed → 移到 completed/ 目录
    ↓
    或被其他事挡住
blocked → 移到 blocked/ 目录
    ↓ 阻塞解除
in_progress
```

**目录 = 状态**：active/ + completed/ + blocked/ 三个目录对应三种状态。移动文件 = 改状态。

---

## TaskCreate / TaskUpdate / TaskList 工具

```json
{
  "name": "TaskCreate",
  "parameters": {
    "subject": "string",
    "description": "string",
    "priority": "low|medium|high",
    "blocks": ["string"],
    "blocked_by": ["string"]
  }
}
```

模型自主调用——遇到"3+ 步骤且步骤间有依赖"时主动建 task。

---

## 跟 TodoWrite 的对比

| 维度 | TodoWrite | Task System |
|---|---|---|
| 持久化 | 内存 | 磁盘（.md 文件） |
| 依赖 | ✗ | ✓（blocks / blocked_by） |
| 嵌套 | ✗ | ✓（Subtasks） |
| 跨会话 | ✗（重启丢） | ✓（落盘） |
| 复杂度 | 简单 | 复杂 |
| 适用 | 单次 3-5 步 | 多日 / 多周 / 多人 |

**选择规则**：
- 5 步以内、当前会话能做完 → TodoWrite
- 跨会话 / 跨天 / 有依赖 → Task System

---

## 改造成 PuinClaw 时

PuinClaw 的 goal-runner 已经有类似的 `progress.md` + `state.json`——是简化版 Task System：

```
my-goal.md           ← Goal + Context
state.json           ← status / turn / cost（task 的 frontmatter）
progress.md          ← 时间线（task 的 Notes）
messages.json        ← 完整对话
```

差别：PuinClaw 是**单 goal 单文件**，Task System 是**多 task 依赖图**。

---

## 相关

- [[12 - Task System]] —— 完整实现、依赖图算法
- [[05 - TodoWrite]] —— 轻量版
- [[13 - Background Tasks]] —— 跟 Task System 配合的"后台跑 task"
