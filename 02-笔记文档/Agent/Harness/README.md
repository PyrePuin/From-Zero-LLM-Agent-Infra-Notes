# Harness

这一文件夹的笔记**只关注一个主题**：Agent Harness——围绕 LLM 构建的环境、状态管理、验证与控制机制。模型决定写什么代码，Harness 决定它何时、何地、如何写，以及它的输出是否可靠。

三个子目录按学习顺序构成一条主线：**骨架（Learn-Claude-Code）→ 服务（Claw-Theory）→ 内核（pi-mono--PuinClaw）**。循环骨架从始至终不变，扩展方向是“把 Agent 从 CLI 跑成生产服务，再深入到可复用的内核”。

## 推荐课程

> [!TIP]
> **目前推荐的 Harness 工程课程：[learn-harness-engineering](https://github.com/PyrePuin/learn-harness-engineering)（fork 自 [walkinglabs/learn-harness-engineering](https://github.com/walkinglabs/learn-harness-engineering)）**，整体内容简洁，讲解了现代harness的思想、机制，适合从整体思路上首先了解harness，并附带了一个动手实践的课程。
>
> 一个 project-based 的 Harness 工程教程：从 0 到 1 构建让 AI coding agent 可靠工作的环境、状态管理、验证与控制机制（14 讲 + 8 个项目 + 模板资源库，MIT License，15 种语言）。
>
> - 我 fork 的仓库：[PyrePuin/learn-harness-engineering](https://github.com/PyrePuin/learn-harness-engineering)
> - 原仓库：[walkinglabs/learn-harness-engineering](https://github.com/walkinglabs/learn-harness-engineering)
> - 中文讲义（文档网站）：[learn-harness-engineering 中文文档](https://walkinglabs.github.io/learn-harness-engineering/zh/)
>
> 核心观点与本仓库一致：**最强的模型，没有合适的 Harness 依然会在真实工程任务上失败**。课程给出五子系统框架（instructions / state / verification / scope / lifecycle），并拆解了 Pi、Claude Code、Codex、DeepSeek 四个前沿产品的 Harness 设计。这门课的内容十分实用，可以作为一个总览。本仓库的剩余笔记，也是基于 learn-claude-code、learn-openclaw、pi-mono 等学习过程中的总结笔记，可以看作是对harness机制更加详细的讲解，会深入到agentloop内部讲解机制。

## 子目录一览

```text
Agent/Harness/
├── Learn-Claude-Code/    ← Phase 1-6：Claude Code 骨架（agent loop / tools / hooks / memory / subagent / skill）
├── Claw-Theory/          ← Phase 7：产品级常驻 Agent（多通道 / 路由 / 心跳 / 投递 / 重试 / 并发）
└── pi-mono--PuinClaw/    ← Phase 8-9：pi-mono 内核（agent / ai / coding-agent 三包）+ 面试准备
```

### [Learn-Claude-Code/](Learn-Claude-Code/) —— 骨架：一个 Agent Harness 的解剖

只关注一个特定实现：Anthropic 官方 CLI **Claude Code** 的开源复刻教程 [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)。Phase 1-6 逐课讲解**每一课加的是什么、为什么加、这是什么机制、原本的 Claude Code 是怎么做的**。

- **Phase 1 基础机制**：最小骨架长什么样（agent loop + tools + hooks）
- **Phase 2 上下文治理**：怎么跑长任务不爆
- **Phase 3 记忆与恢复**：怎么跨会话连续
- **Phase 4 长时间任务**：怎么跑后台 / 定时任务
- **Phase 5 多智能体**：怎么让多个 Agent 协作
- **Phase 6 扩展与组装**：怎么接入外部世界（MCP / skill / 生态）


### [Claw-Theory/](Claw-Theory/) —— 服务：产品级常驻 Agent 的解剖

只关注一个特定实现：[shareAI-lab/claw0](https://github.com/shareAI-lab/claw0)（10 节，约 7,000 行 Python）。Phase 7 把 Learn-Claude-Code 的 CLI Agent 推进到**产品级常驻服务**——能挂多个 IM 通道、能后台心跳、能可靠投递、能并发处理。learn-claude-code 教的是“Agent 在一个终端里”，claw0 教的是“Agent 跑成一个服务”。

- **s04-s10 依次展开**：Channels（多通道接入）→ Gateway（5 级路由）→ Intelligence（8 层提示词）→ Heartbeat（心跳 + cron）→ Delivery（可靠投递）→ Resilience（重试洋葱）→ Concurrency（命名 lane）
- 每一节只引入一个新概念，前面所有节的代码保持不动。10 节学完，能直接读 OpenClaw 生产代码库。

状态：⏳ in progress（s04 已完成，s05-s10 依次推进）。

### [pi-mono--PuinClaw/](pi-mono--PuinClaw/) —— 内核与面试：pi-mono 三包 + Harness 面试

关注 [pi-mono](https://github.com/badlogic/pi-mono)（Mario Zechner 的极简 coding agent monorepo）。Phase 8 自底向上拆三包：**agent**（Agent runtime：loop + 状态机）→ **ai**（LLM 调用层：streamFn + providers）→ **coding-agent**（真正的 coding CLI：工具 + session），最后用一篇综合串起三层联动与数据流。Phase 9 是 Harness / Agent Infra 方向的面试准备。

- **Phase 8 pi-mono 核心**：`00 - 阅读路线`（常驻导览）+ agent / ai / coding-agent 三篇 + `04 - 综合:三层联动与数据流`
- **Phase 9 面试准备**：面试总览、pm2 与 IM 接口、Agent 评估与提示词优化、Agent Teams、长跑与自治代码库、Agentic Sandbox、Harness、综合面试 QA

状态：⏳ in progress（agent / ai 完成，coding-agent 推进中；Phase 9 备考中）。

## 阅读顺序

**按子目录顺序读**：Learn-Claude-Code（骨架）→ Claw-Theory（服务）→ pi-mono--PuinClaw（内核）。每个子目录内部再按其 README 的 Phase 顺序推进。骨架不变，扩展方向是“从 CLI 跑成生产服务，再深入到可复用的内核”。

```text
Learn-Claude-Code        Claw-Theory              pi-mono--PuinClaw
   Phase 1-6                Phase 7                    Phase 8-9
 ┌───────────────┐      ┌───────────────────┐      ┌──────────────────┐
 │  agent loop   │      │  channels/gateway │      │  agent/ai/coding │
 │  tools/hooks  │ ───→ │  heartbeat/deliv. │ ───→ │  三包自底向上拆解 │
 │  memory/subag │ 骨架 │  retry/concurrenc │ 服务 │  + 面试准备       │
 └───────────────┘      └───────────────────┘      └──────────────────┘
```

## 每篇笔记的固定结构

三个子目录的单课笔记统一为 12 节固定骨架（重点关注 / 加了什么 / 演进与动机 / 核心抽象 / 架构图 / 代码骨架总览 / Q&A 等），设计原则一致：

- 每节只回答一个核心问题（表格先扫概览，文字再深入）
- 代码骨架总览放在最后，读到那自然就懂——前面先建立心智模型
- Mermaid 图全部按 `mermaid-style` 规范（classDef 五色 + 批量赋色）
- `00 - 导览` 类文件不套此模板——它是 Phase 级别的导览，不是单课笔记

遇到具体疑问翻各子目录的 **对话精华 QA**；遇到数据文件不知道长什么样翻各子目录的 **数据样例**。

## 取舍声明

这套笔记**不是**：

- 不是 API 文档翻译（看 [Anthropic 官方文档](https://docs.anthropic.com/)、[claw0 原仓库](https://github.com/shareAI-lab/claw0)、[pi-mono 原仓库](https://github.com/badlogic/pi-mono) 更准）。
- 不是逐行源码注释（看原仓库代码更直接）。
- 不教你如何安装、运行、配置 token（原仓库 README 已经写得很清楚）。

它是**一份心智模型笔记**。读完之后，你应该能在脑子里画出每个 Harness 的结构图，并能解释每一块为什么必须存在。
