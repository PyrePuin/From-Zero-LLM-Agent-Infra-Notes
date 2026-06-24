---
type: concept
status: seed
domain: llm-agent
created: 2026-06-25
updated: 2026-06-25
series: pi-mono
tags:
  - 面试
  - harness
  - ai-first
  - prd2pr
aliases:
  - Harness
  - 线束
  - AI First
---

# 06 - Harness

> [!note]
> "Harness" 来自 OpenAI 2026 年 2 月文章
> 《Engineering:leveraging Codex in an agent-first world》。
> 概念模糊但**面试谈资很值**——展现你对 agent 时代工程范式的思考。

## 1. Harness 是什么(我的理解)

```
Harness 字面意思:线束 / 马具 / 挽具(套在马身上控制马的装备)

借喻:Agent 就是那匹马,Harness 就是"套在 agent 身上"
     让它更好跑、更可控的一切工程基础设施。

                  ┌─────────────┐
   Harness  ────→ │   Agent     │ ←──── Harness
   (外围)         │  (LLM 内核) │       (外围)
                  └─────────────┘
                  ↓
              真实代码库 / 任务

包含:
   • Sandbox(沙盒)
   • Tools(工具集)
   • Session 管理
   • Compaction(上下文压缩)
   • Skills / Memory
   • Evaluation pipeline
   • PR/CI 流程
   • Long-running 守护
```

## 2. OpenAI Harness Engineering(原文要点)

```
┌────────────────────────────────────────────────────────┐
│ 文章:Harness engineering: leveraging Codex in an     │
│       agent-first world                               │
│ 时间:2026-02-11                                       │
│ 来源:OpenAI                                           │
└────────────────────────────────────────────────────────┘

震撼点:
   ① 从空 git repo 开始,没有一行预存的人写代码
   ② 连 AGENTS.md 都是 Codex 自己写的
   ③ 整个产品(agent 自己)由 agent 塑造
   ④ "Redefining the role of the engineer"
     — 工程师角色被重新定义
```

### 三大支柱(Three Pillars)

```
┌──────────────────────────────────────────────────────┐
│ ① Context Engineering(上下文工程)                  │
│   不只是 prompt,包括:                              │
│   - System prompt 组装                               │
│   - Memory / Skills 加载                             │
│   - 上下文压缩                                       │
│   - Tool 选择                                        │
│   - File / Repo 注入                                 │
├──────────────────────────────────────────────────────┤
│ ② Harness Engineering(线束工程)                    │
│   "给 agent 一个更好的运行环境":                    │
│   - Sandbox                                          │
│   - Session / State 管理                             │
│   - Long-running 守护                                │
│   - Multi-agent 协调                                 │
│   - 评估 pipeline                                    │
├──────────────────────────────────────────────────────┤
│ ③ Workflow Engineering(工作流工程)                 │
│   把 agent 嵌入真实工作流:                          │
│   - PRD2PR(需求文档 → 代码 → PR)                   │
│   - GitHub 集成                                      │
│   - CI/CD 集成                                       │
│   - Human-in-the-loop                                │
└──────────────────────────────────────────────────────┘
```

## 3. PRD2PR(Harness 的典型实践)

```
PRD2PR = Product Requirement Document → Pull Request

传统流程:
   PM 写 PRD → 工程师读 → 手写代码 → 提 PR → Review → Merge

PRD2PR 流程:
   PM 写 PRD
       │
       ▼
   Agent 读 PRD
       │
       ├─ 拆任务
       ├─ 写代码
       ├─ 写测试
       └─ 跑 CI
       │
       ▼
   自动提 PR(带描述、测试结果)
       │
       ▼
   Human Review(关键)
       │
       ▼
   Merge

→ "PM → PR" 不再有"工程师"这个角色
```

### 已经在做 PRD2PR 的公司

```
• OpenAI Codex(自己内部用)
• Anthropic Claude Code
• Cursor / Cognition Devin
• 一些 YC startup(Plandex / Aider)
• 大厂内部工具(字节 / 阿里 / Google)
```

## 4. "AI First" 战略的反思

```
关键文章:为什么你的"人工智能优先"策略可能是错误的

错误理解:
   "AI First" = 把所有事情都用 AI 做
   
正确理解:
   "AI First" = 设计产品/流程时,先考虑 AI 能做什么,
                再围绕 AI 的能力重新设计工作流
   
   ┌──────────────────────────────────────────┐
   │ 错误做法(表面 AI First):              │
   │   把 AI 塞进传统流程                    │
   │   工程师还是按老方式工作,AI 当工具    │
   │   结果:效率提升有限,组织没变革        │
   ├──────────────────────────────────────────┤
   │ 正确做法(真正 AI First):              │
   │   重新设计流程让 AI 居中                │
   │   人变成 reviewer / 决策者              │
   │   Harness 提供支撑                      │
   │   结果:工作流根本性变化                │
   └──────────────────────────────────────────┘
```

### 例子:编码工作的演进

```
阶段 1(过去):人写代码,AI 当搜索工具
阶段 2(现在):人主导,AI 当副驾驶(Copilot)
阶段 3(PR-First):AI 主导,人 Review PR
阶段 4(Autonomous):AI 全自动,人只设方向
                  ↑ OpenAI Harness Engineering 在这里
```

## 5. Harness 跟你学过的所有东西的关系

```
Harness = 你学过的所有抽象的"产品化整合"

┌─────────────────────────────────────────────────────────┐
│ 你学过的(Harness 的组成部分)                          │
├─────────────────────────────────────────────────────────┤
│ learn-claude-code:                                      │
│   s10 System Prompt → Context Engineering               │
│   s11 Recovery         → Harness 状态持久化             │
│   s12 Task System      → Harness 任务管理               │
│   s13 Background Tasks → Harness 异步执行               │
│   s15/17 Agent Teams   → Harness 多 agent               │
│   s19 MCP Plugin       → Harness 扩展机制               │
│                                                         │
│ claw0:                                                  │
│   s04 Channels         → Harness IM 接口                │
│   s05 Gateway          → Harness 路由                   │
│   s06 Intelligence     → Harness Context 组装           │
│   s07 Heartbeat/Cron   → Harness 长跑                   │
│   s08 Delivery         → Harness 异步交付               │
│   s09 Resilience       → Harness 容错                   │
│   s10 Concurrency      → Harness 并发                   │
│                                                         │
│ pi-mono:                                                │
│   pi-ai                → Harness LLM 适配               │
│   pi-agent-core        → Harness runtime                │
│   pi-coding-agent      → Harness 产品层(就是 Harness)│
│   pi-mom               → Harness 长跑 /goal 实现        │
└─────────────────────────────────────────────────────────┘

→ PuinClaw = 你的个人 Harness 实现
```

## 6. 怎么谈 Harness(面试谈资)

```
面试官:"你觉得 agent 时代工程范式怎么样?"

你:
   "我认为 OpenAI 提的 Harness Engineering 很关键 —
    它把'让 LLM 写代码'升级为'重新设计整个工程流程'。
    
    传统 AI Coding 还停留在'AI 当工具',工程师主导。
    Harness Engineering 把 AI 放在主导位置,人做 Review。
    
    我的 PuinClaw 就是个 mini Harness —
    pi-agent-core 是 runtime,pi-ai 是 LLM 适配,
    pi-mom 是长跑 /goal,加 pm2 + IM 接入 + Docker 沙盒,
    基本就是一个能持续运行的 agent 系统。
    
    下一步我可以加 PRD2PR 流程 — 接 GitHub webhook,
    自动响应 issue,开 PR,人 Review merge。"
```

## 7. 面试要点

### 必答清单

```
Q1: Harness 是什么?
A: OpenAI 2026-02 提出的概念,指"套在 agent 身上"
   让它更好跑的工程基础设施 — 沙盒 / 工具 / 状态 /
   评估 / 长跑 / 多 agent 等。

Q2: Harness Engineering 三大支柱?
A: ① Context Engineering(上下文组装)
   ② Harness Engineering(运行环境)
   ③ Workflow Engineering(嵌入真实工作流)

Q3: PRD2PR 是什么?
A: PRD → Agent 自动写代码 → 提 PR → 人 Review → Merge。
   是 Harness 的典型实践。

Q4: "AI First" 战略常见的错误?
A: 表面 AI First = 把 AI 塞进老流程。
   真正 AI First = 围绕 AI 重新设计流程,人变成 reviewer。

Q5: 你的 PuinClaw 算 Harness 吗?
A: 是 mini Harness — 包含 runtime(pi-agent-core) +
   LLM 适配(pi-ai) + 长跑(pi-mom) + 守护(pm2) +
   IM 接入 + 沙盒(Docker) + 评估 pipeline(DSPy)。
```

### 加分点

```
• 引用 OpenAI 文章细节(空 repo / AGENTS.md 是 Codex 写的)
• 提到"工程师角色被重新定义"
• 提到 PRD2PR 已经在哪些公司落地
• 对照 Anthropic Agent Teams 案例
• 谈你对"AI First 错误理解"的看法
```

## 8. 关键资料

| 资料 | 内容 |
|---|---|
| [Harness engineering: leveraging Codex in an agent-first world](https://openai.com/index/harness-engineering/) | OpenAI 原文 |
| [为什么你的"人工智能优先"策略可能是错误的](https://hbr.org/2025/04/your-ai-first-strategy-might-be-wrong) | HBR 战略反思 |
| [OpenAI Harness 中文版](https://openai.com/zh-Hans-CN/index/harness-engineering/) | 中文翻译 |
| [Build Systems That Make AI Agents Work](https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026) | 三大支柱详解 |

## 9. 你的 PuinClaw 是 Harness 的最佳例证

```
面试 1 分钟电梯陈述:
─────────────────────────────────────────────────────────
"PuinClaw 是我基于 pi-mono 改造的常驻编码 Agent,
是 OpenAI Harness Engineering 思路的产品级实现。

包含完整的 Harness 组件:
• LLM 适配层(pi-ai):支持 11 家 provider
• Runtime(pi-agent-core):双循环 + 工具调度 + 事件流
• 长跑能力(pi-mom /goal):持续推进直到完成
• 守护进程(pm2):7×24 在线 + 自动重启
• IM 接口(Telegram):远程交互
• Docker 沙盒:安全隔离
• Agent Teams(规划):多 agent 协作
• DSPy 评估 pipeline(规划):效果可量化

下一步计划:
• 接 GitHub webhook,实现 PRD2PR
• 加 Reviewer agent,质量保证
• 上下文压缩,支持长跑"
─────────────────────────────────────────────────────────
```

_Generated for PuinClaw 面试准备, 2026-06-25_
