---
type: project
status: active
domain: LLM Post-Training
created: 2026-08-05
updated: 2026-08-05
aliases:
  - veRL 框架学习
tags:
  - veRL
  - GRPO
  - Agentic-RL
---

# veRL 框架学习

本目录按照“先掌握框架最小闭环，再阅读 Agentic RL 改造实例”的顺序组织。

## 阅读顺序

1. [veRL 最初步使用笔记](./01-veRL最初步使用笔记.md)：从数据、奖励函数、配置到启动与训练记录，建立最小可运行闭环。
2. [Agentic RL 实例——code-r1 学习笔记](./Agentic%20RL实例——code-r1学习/Agentic%20RL实例——code-r1学习笔记.md)：理解 code-r1 如何把代码执行环境接入 veRL 的 GRPO 训练链路。

## 本地源码

- [code-r1 源码副本](./Agentic%20RL实例——code-r1学习/code-r1/)

> [!NOTE]
> 基础笔记依据 2026-08-05 获取的 veRL 官方仓库版本 `2781c1e` 整理；code-r1 源码副本来自 `wyf3/llm_related` 的提交 `a492338`。两个项目所基于的 veRL 版本不同，配置层级和已有 Agent 能力不能直接混用。
