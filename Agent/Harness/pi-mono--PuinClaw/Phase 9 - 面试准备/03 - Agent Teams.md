---
type: concept
status: seed
domain: llm-agent
created: 2026-06-25
updated: 2026-06-25
series: pi-mono
tags:
  - 面试
  - multi-agent
  - agent-teams
  - coordination
aliases:
  - Agent Teams
  - 多智能体协作
---

# 03 - Agent Teams

> [!note]
> 单 agent 能力有上限,**Agent Teams** 是"用规模换能力"的范式——
> 多个 agent 协作完成一个 agent 做不了的任务。
> 代表案例:**Anthropic 用 16 个 Claude 编译了一个 C 编译器**。

## 1. 概念

```
Agent Teams = 多个 agent 协作完成大任务

不是简单的"多线程跑同一个任务",
而是有结构、有分工、有通信的协作。
```

### 跟"单 agent 多 turn"区别

| | 单 agent 多 turn | Agent Teams |
|---|---|---|
| 上下文 | 共享一个 transcript | 各自独立 transcript |
| 角色 | 一个 agent 干所有事 | 分工不同(lead/worker/reviewer) |
| 失败 | 一处错全错 | 局部失败可重试 |
| 协调 | 不需要 | 需要协议 |

## 2. Anthropic 案例(必背!)

```
┌──────────────────────────────────────────────────────┐
│ Anthropic:Building a C compiler with parallel Claude │
│                                                      │
│ 目标:从零用 Rust 写一个 C 编译器                    │
│ 规模:16 个 Claude Opus 4.6 并行                     │
│ 时长:~2 周                                          │
│ 花费:~$20,000 API 成本                              │
│ 会话:~2,000 次 Claude Code sessions                 │
│ 结果:能编译 PostgreSQL / SQLite / Linux kernel     │
│                                                      │
│ 核心做法:                                           │
│   ① 把编译器拆成多个 module(lexer/parser/...)     │
│   ② 每个 module 由一个 agent 负责                   │
│   ③ 用 git 仓库作为协作介质                         │
│   ④ 用 GCC 做参考编译器,隔离失败到具体文件          │
│   ⑤ Lead agent 协调 + Reviewer agent 审查           │
└──────────────────────────────────────────────────────┘
```

**关键洞察**:
- **不是 16 个 agent 同时写一份代码** —— 是 16 个 agent 各自负责不同 module
- **代码库本身是协作介质** —— 不需要复杂的消息协议,git pull/push 即可
- **失败隔离很重要** —— 单文件失败不能拖垮整个项目
- **测试驱动** —— GCC 作为参考答案,自动判定对错

## 3. 协作模式(4 种)

```
┌─────────────────┬────────────────────────────────────┐
│ 模式            │ 特点                               │
├─────────────────┼────────────────────────────────────┤
│ ① 层级          │ Lead 拆任务,Worker 执行,         │
│ (Hierarchical)  │ Reviewer 审查                      │
├─────────────────┼────────────────────────────────────┤
│ ② 并行          │ N 个 agent 干独立的子任务,        │
│ (Parallel)      │ 完成后合并                         │
├─────────────────┼────────────────────────────────────┤
│ ③ 流水线        │ Agent A 输出 → Agent B 输入 →     │
│ (Pipeline)      │ Agent C 输入                       │
├─────────────────┼────────────────────────────────────┤
│ ④ 市场          │ 任务看板,agent 抢活               │
│ (Market)        │ Autonomy 高,coordinatation 难    │
└─────────────────┴────────────────────────────────────┘
```

## 4. 实际怎么做(对照 PuinClaw)

### 4.1 学过的对应

```
learn-claude-code (Phase 5):
   s15 Agent Teams        ← Lead-Worker 模式
   s16 Team Protocols     ← 协作协议
   s17 Autonomous Agents  ← 自治 idle poll(看板模式)
   s18 Worktree Isolation ← git worktree 隔离

claw0:
   s06 Agents 配置        ← 多 agent 角色定义
   s10 Concurrency        ← 并发控制

pi-mono:
   pi-agent-core          ← 一个 Agent 实例,Teams = 多实例
```

### 4.2 PuinClaw 实现 Agent Teams 的思路

```
方案 A: Lead-Worker(最简单)
   Lead agent(单实例)接收任务
       ↓
   拆成子任务
       ↓
   spawn N 个 Worker agent(各自独立 state)
       ↓
   每个 Worker 跑完 → 结果回传 Lead
       ↓
   Lead 整合 → 输出

方案 B: Git-based(参考 Anthropic)
   共享一个 git 仓库
   每个 agent 在独立 branch/worktree 工作
   通过 PR / commit 通信
   Lead agent review + merge
```

### 4.3 PuinClaw Teams 架构示例

```
┌──────────────────────────────────────────────────┐
│ Lead Agent                                       │
│  ├─ 接收 /goal "build C compiler"               │
│  ├─ 拆任务:[lexer, parser, codegen, ...]       │
│  └─ spawn workers                               │
└──────────────────────────────────────────────────┘
       │                  │                │
       ▼                  ▼                ▼
┌────────────┐    ┌────────────┐    ┌────────────┐
│ Worker 1   │    │ Worker 2   │    │ Worker 3   │
│  lexer.rs  │    │  parser.rs │    │ codegen.rs │
│            │    │            │    │            │
│ 独立 state │    │ 独立 state │    │ 独立 state │
└────────────┘    └────────────┘    └────────────┘
       │                  │                │
       ▼                  ▼                ▼
            git commit + push 到共享 repo
                      │
                      ▼
              ┌──────────────┐
              │ Reviewer     │
              │  (CI 跑测试)│
              └──────────────┘
                      │
                      ▼
              Lead merge + 下一步
```

## 5. 关键挑战

```
挑战 1: 怎么拆任务?
   ① 按 module 拆(技术边界)— Anthropic 做法
   ② 按文件拆(物理边界)— 简单
   ③ 按功能拆(语义边界)— 难,要 agent 自己拆

挑战 2: 怎么通信?
   ① 通过 git(代码即消息)— 推荐
   ② 通过消息队列(显式协议)— 复杂
   ③ 通过共享文件(JSON 状态)— 中等

挑战 3: 怎么避免冲突?
   ① Worktree 隔离(每个 agent 独立工作树)
   ② 锁机制(文件锁/资源锁)
   ③ 任务无依赖(并行无冲突)

挑战 4: 怎么处理失败?
   ① 单 worker 失败 → 重试 / 换 agent
   ② 整体失败 → Lead 决定回滚还是补救
   ③ Reviewer 拒绝 → 退回 worker 修

挑战 5: 怎么控制成本?
   ① Worker 数量上限(16 是 sweet spot)
   ② 单 worker cost cap
   ③ 总 cost cap(Anthropic 花了 $20k 是有 cap 的)
```

## 6. 面试要点

### 必答清单

```
Q1: Agent Teams 是什么?跟单 agent 区别?
A: 多 agent 协作完成大任务。区别 —
   独立 transcript / 角色分工 / 局部失败可恢复 / 需要协调协议。

Q2: 讲讲 Anthropic 的 16 Claude C 编译器案例?
A: 16 个 Opus 4.6 / 2 周 / $20k / 2000 sessions。
   拆 module + git 协作 + GCC 参考 + Lead/Reviewer。
   结果能编译 PostgreSQL/SQLite/Linux kernel。

Q3: 多 agent 怎么通信?
A: 推荐通过 git(代码即消息)。
   好处:协作介质天然 / 历史可追溯 / 冲突可解决。
   复杂消息才用消息队列。

Q4: Agent Teams 适合什么场景?
A: ① 任务可拆分(modular)
   ② 子任务相对独立
   ③ 有客观验收标准(测试/编译)
   不适合:紧密耦合 / 主观任务 / 小任务

Q5: 怎么避免 agent 互相干扰?
A: ① Worktree 隔离(各自工作树)
   ② 任务设计避免共享文件
   ③ 锁机制
   ④ Reviewer 把关合并
```

### 加分点

```
• 知道 s15 Agent Teams(Lead-Worker)的实现细节
• 知道 s18 Worktree Isolation 的做法
• 提到"代码库本身是协作介质"这个核心洞察
• 提到成本控制(总 cost cap)
• 提到 Reviewer 角色的重要性
```

## 7. 关键资料

| 资料 | 内容 |
|---|---|
| [Building a C compiler with parallel Claudes — Anthropic](https://www.anthropic.com/engineering/building-c-compiler) | 原文 |
| [Sixteen AI agents built a C compiler — Ars Technica](https://arstechnica.com/ai/2026/02/sixteen-claude-ai-agents-working-together-created-a-new-c-compiler/) | 报道 |
| [InfoQ 报道](https://www.infoq.com/news/2026/02/claude-built-c-compiler/) | 中文友好 |
| [Hacker News 讨论](https://news.ycombinator.com/item?id=46903616) | 社区讨论 |
| [[15 - Agent Teams]] | Lead-Worker 模式 |
| [[17 - Autonomous Agents]] | 看板模式 |
| [[18 - Worktree Isolation]] | git 隔离 |

## 8. PuinClaw 实操清单

```
□ 在 PuinClaw 加 spawn_teammate 工具
□ 实现 Lead-Worker 协作(参考 s15)
□ 用 git worktree 做隔离
□ 设计任务看板(tasks.json)
□ 加 Reviewer 角色(独立 agent + CI 测试)
□ 总 cost cap($20)
```

_Generated for PuinClaw 面试准备, 2026-06-25_
