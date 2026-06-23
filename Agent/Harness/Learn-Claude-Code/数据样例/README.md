# 数据样例索引（Learn-Claude-Code）

> 配合 Phase 1-6 各节笔记使用——读到"写 todos.json"或"加载 MEMORY.md"时，来这里看**文件长什么样**。

## 配置类（启动时读）

| 文件 | 来源 | 用途 | 涉及章节 |
|---|---|---|---|
| [[01 - Skill 加载]] | `~/.claude/skills/<name>/SKILL.md` | 技能定义 | [[07 - Skill Loading]] |
| [[02 - System Prompt 组装]] | 动态拼装 | 8 段拼接的完整 prompt | [[10 - System Prompt]] |

## 运行时数据（运行时写）

| 文件 | 来源 | 用途 | 涉及章节 |
|---|---|---|---|
| [[03 - TodoWrite 任务列表]] | `todos.json`（教学版内存） | 任务跟踪 | [[05 - TodoWrite]] |
| [[04 - Memory 三层]] | `MEMORY.md` + `daily/*.jsonl` + side-query | 长期记忆 | [[09 - Memory]] |
| [[05 - Task System]] | `.claude/tasks/<id>.md` | 长时任务 | [[12 - Task System]] |
| [[06 - Cron Schedule]] | `.claude/cron.json` | 定时任务 | [[14 - Cron Scheduler]] |
| [[07 - MCP Plugin 配置]] | `.claude/mcp.json` | MCP 服务接入 | [[19 - MCP Plugin]] |
| [[08 - Error Recovery 状态]] | `RecoveryState`（内存） | 错误恢复状态机 | [[11 - Error Recovery]] |

---

## 怎么用

1. **读笔记时遇到数据文件名** → 来这里找对应样例
2. **想看真实数据** → `learn-claude-code/<section>/` 目录下有代码（submodule 里）
3. **想配置自己的 Claude Code** → 复制样例，按需修改

## 真实出处

Learn-Claude-Code 教学版**主要在内存里**（不写盘），所以样例多为"内存对象 + 推演的磁盘格式"。真实磁盘格式来自 Claude Code 官方版（[`learn-claude-code`](https://github.com/shareAI-lab/learn-claude-code) 的 docs）。

## 与 claw0 数据样例的对比

| 维度 | Learn-Claude-Code（CLI） | claw0（常驻服务） |
|---|---|---|
| 持久化 | 大多在内存，少部分 JSON | 重度持久化（WAL 队列 + session jsonl） |
| 配置文件 | SKILL.md / system prompt 动态 | workspace/*.md（8 个固定文件） |
| 数据生命周期 | 单次任务 | 7×24 长期 |
| 失败容忍 | 用户重跑 | 自动重试 + fallback |
