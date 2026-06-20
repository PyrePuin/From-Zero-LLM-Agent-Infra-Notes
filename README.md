# From Zero --- LLM Agent Infra Notes

从零拆解一个 LLM Agent 应该长成什么样。

本仓库的笔记**只关注一个特定实现路径**：Anthropic 官方 CLI **Claude Code** 的开源复刻教程 [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)。每一个 phase 都按"原教程顺序"展开，逐课讲解**它每一课加的是什么、为什么加、这是什么机制、原本的 Claude Code 是怎么做的**。

## 仓库结构

```
Agent/
  Harness/             ← Harness（外壳）层：包在模型外面的工程代码
    Claude Code/       ← 用 Claude Code 这一个具体 harness 做切片
      README.md
      Phase 1 - 基础机制/
        00 - 综合总结.md   ← 每个 Phase 的"整体逻辑"笔记
        01 - Agent Loop.md
        02 - Tool Use.md
        03 - Permission.md
        04 - Hooks.md
      Phase 2 - 上下文治理/
        00 - 综合总结.md
        05 - TodoWrite.md
        06 - Subagent.md
        08 - Context Compact.md
      Phase 3 - 记忆与恢复/
        00 - 综合总结.md
        09 - Memory.md
        10 - System Prompt.md
        11 - Error Recovery.md
      Phase 4 - 长时间任务/
        00 - 综合总结.md
        12 - Task System.md
        13 - Background Tasks.md
        14 - Cron Scheduler.md
      Phase 5 - 多智能体/
        15 - Agent Teams.md
        16 - Team Protocols.md    ← s17/s18 暂未完成
      Phase 6 - 扩展与组装/
        07 - Skill Loading.md     ← s19/s20 暂未完成
      对话精华 QA/
        对话精华.md
```

## 命名约定

- **Agent**：一个能自主调用工具完成任务的 LLM 系统。
- **Harness**：包在模型外面的工程代码（循环、工具派发、权限、hooks、记忆……）。模型本身（权重、推理）由 API 提供，harness 提供"环境"。
- **Claude Code**：Anthropic 官方的 CLI harness，是本仓库的切片对象。

## 阅读顺序

1. 先读 [Harness/Claude Code/README.md](Agent/Harness/Claude%20Code/README.md)：了解整体框架。
2. 每个 Phase 先读 `00 - 综合总结.md`：建立"这一组课为什么放在一起"的整体感。
3. 再按顺序读 01, 02, …：单课笔记里有更多细节。
4. 最后看 `对话精华 QA`：里面记录了学习过程中卡过的点。

## 这套笔记的取向

- **重概念、重机制、重逻辑**，轻实现。代码片段只在每篇末尾出现一次，作为"落地长什么样"的对照。
- **每篇都回答四个问题**：这一步加了什么 / 为什么加 / 这是什么机制 / Claude Code 怎么做。
- **顺序读**：每一课都在前一课的基础上做加法，没有跳跃。

读完之后，你应该能在脑子里画出一个 LLM Agent 的结构图，并能解释每一块为什么必须存在。
