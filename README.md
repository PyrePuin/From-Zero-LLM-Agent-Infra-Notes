# From Zero --- LLM Agent Infra Notes

从零拆解一个 LLM Agent 应该长成什么样：以 [shareAI-lab/learn-claude-code](https://github.com/shareAI-lab/learn-claude-code) 为底本，逐课讲解**它每一课加的是什么、为什么加、这是什么机制、原本的 Claude Code 怎么做的**。

笔记原则：

- **重概念、重机制、重逻辑，轻实现**。代码片段只在每篇末尾出现一次，作为"落地长什么样"的对照。
- **顺序读**。每一课都在前一课的基础上做"加法"，没有跳跃。
- **每篇都回答四个问题**：
  1. 这一步加了什么？
  2. 为什么需要加？
  3. 这是一种什么机制？（它的名字、它的同类）
  4. 真正的 Claude Code 是怎么做的？

## 阶段划分

| 阶段 | 课程 | 主题 |
|---|---|---|
| Phase 1 | s01 - s04 | 基础机制：Agent Loop、Tool Use、Permission、Hooks |
| Phase 2 | s05, s06, s08 | 上下文治理：TodoWrite、Subagent、Context Compact |
| Phase 3 | s09 - s11 | 长期记忆与系统提示：Memory、System Prompt、Error Recovery |
| Phase 4 | s12 - s14 | 任务编排：Task System、Background Tasks、Cron |
| Phase 5 | s15 - s18 | 多智能体：Teams、Protocols、Autonomous、Worktree |
| Phase 6 | s07, s19, s20 | 生态与整合：Skill Loading、MCP、Comprehensive |

## 目录结构

```
Phase 1 - 基础机制/
  01 - Agent Loop.md
  02 - Tool Use.md
  03 - Permission.md
  04 - Hooks.md
Phase 2 - 上下文治理/
  05 - TodoWrite.md
  06 - Subagent.md
  08 - Context Compact.md
对话精华 QA/
  对话精华.md
```

## 阅读顺序建议

1. 先快速过一遍 Phase 1 四篇，理解一个 Agent 最小骨架长什么样。
2. 再读 Phase 2 三篇，理解骨架跑起来之后最先出现的"上下文不够用"问题怎么治。
3. 最后看 `对话精华 QA`，里面记录了学习过程中一些容易卡住的点。

## 这套笔记不是什么

- 不是 API 文档翻译。
- 不是逐行源码注释。
- 不教你如何安装、运行（仓库 README 已经写得很清楚）。

它是**一份心智模型笔记**：读完之后，你应该能在脑子里画出一个 LLM Agent 的结构图，并能解释每一块为什么必须存在。
