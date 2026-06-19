---
type: concept
status: seed
domain: llm-agent
created: 2026-06-18
updated: 2026-06-18
aliases:
  - TodoWrite
  - todo
  - 任务列表
tags:
  - agent
  - planning
  - s05
---

# 05 - TodoWrite

> [!note]
> 任务稍微一长，模型就会"漂"——做一半忘了原目标，或者半路跳到无关的事情上。TodoWrite 给模型一个**外置的、可写的、跨轮可见的任务列表**：先列出来再执行，每完成一个打勾。它不是给用户看的项目管理工具，是给**模型自己**用的认知支架。

## 这一步加了什么

- 一个 **`todo_write` 工具**：参数是 `[{content, status}, ...]`，status 是 pending / in_progress / completed。
- 一块 **`CURRENT_TODOS` 全局内存**：保存当前任务列表。
- 一段 **SYSTEM prompt**：要求"多步任务开始前先用 todo_write 规划"。
- 一个 **nag 机制**：如果模型连续 3 轮没调用 todo_write，循环里自动追加 `<reminder>Update your todos.</reminder>`。

## 为什么需要加

### 1. 模型在长任务里会漂

LLM 的注意力是"近因偏向"的——越近的 token 影响越大。一个跨 20 轮的任务，到第 15 轮时，第 1 轮的目标可能已经被中间的工具输出冲淡。模型会：

- 做完一步忘了下一步是什么。
- 中途被某个工具输出引偏，做了无关的事情。
- 在循环里反复检查同一个文件，因为忘了自己已经查过。

### 2. 自然语言的"在心里计划"没用

你可以在 SYSTEM prompt 里写"开始前先在心里规划任务"。模型会输出一段"好的我先做 A 再做 B……"，但这段话在 5 轮之后就被压缩掉了（s08 会讲上下文压缩），等于没说。

**记忆必须外置**——写到一个不会被压缩的位置。`CURRENT_TODOS` 就是这个位置。

### 3. 任务状态对用户也有价值

用户看到 Agent 在跑，但不知道它跑到哪了。一个 todo 列表实时渲染（CLI 里用颜色框出来），用户能：

- 知道进度（5 个任务完成 3 个）。
- 发现偏航（"我让它做 X，它怎么在搞 Y"）。
- 提前打断（看到方向错了就 Ctrl+C）。

## 这是一个什么机制

这是 **External Working Memory（外部工作记忆）** 模式。认知科学里叫"扩展心智"——把短期记忆外置到纸笔或屏幕上，腾出脑力做实际推理。

在 Agent 设计里，它有几个具体形态：

### 1. 工具语义 = 模型用它的方式

todo_write 不是一个"动作工具"（不像 bash 真的执行什么）。它是一个**状态修改工具**：调用它的副作用是更新 `CURRENT_TODOS`。模型把它当成"在便签上写下下一步"来用。

这种"非动作工具"很重要——它给模型一种**声明式的表达方式**，比让它每次都输出一段文字描述任务进度高效得多。

### 2. SYSTEM prompt + 工具 + nag 三件套

光给工具不够。模型可能根本不调用它。所以要三管齐下：

- **SYSTEM prompt 引导**："复杂任务先用 todo_write 规划"。模型读 SYSTEM 时就建立了"该用"的预期。
- **工具暴露**：让模型"能"用。
- **nag 兜底**：如果它忘了用，循环里强制塞 reminder。

这是教 LLM 用工具的**通用模式**：引导 + 能力 + 强制。三件缺一效果都打折。

### 3. 状态机：pending → in_progress → completed

todo 的 status 字段强制模型用"状态机思维"：

- **pending**：还没开始。
- **in_progress**：正在做（**同一时间应该只有一个**）。
- **completed**：做完。

这逼模型把任务切成离散步骤、明确"我现在在哪一步"，而不是模糊地"我在处理这件事"。

## 原本的 Claude Code 怎么做的

Claude Code 的等价物是 **TaskCreate / TaskUpdate / TaskGet / TaskList** 一组工具。比 s05 多了几个能力：

### 1. 四个工具而不是一个

- **TaskCreate**：新建任务。
- **TaskUpdate**：改状态、改 owner（多 Agent 时用）。
- **TaskGet**：读单个任务详情。
- **TaskList**：列所有任务。

拆开的好处是**支持多 Agent 协作**（s15 团队）：一个 Agent 创建任务，另一个 Agent claim 它、改 owner、完成后更新状态。s05 的单 Agent 没这个需求，所以一个 todo_write 够了。

### 2. blockedBy / blocks 依赖

任务可以声明依赖："任务 B 必须等任务 A 完成才能开始"。这逼模型在规划阶段就想清楚顺序，避免乱开并行。

### 3. owner 字段

每个任务可以指派 owner（"claude" / "explore-agent" / 用户）。多 Agent 场景下，一个 Agent 看任务列表时知道哪些是自己该做的。

### 4. 任务被注入 system reminder

Claude Code 会在循环里**定期把当前任务列表注入到对话**（作为 system-reminder），让模型始终看到最新状态。这相当于把 s05 的 nag 做得更精细：不是"提醒你更新"，是直接把列表拍在它脸上。

### 5. activeForm（spinner 文字）

每个任务可以有个 `activeForm` 字段，比如"Running tests"。CLI 用它显示在 spinner 上，用户实时看到"Agent 现在在跑测试"而不是"Agent 在跑"。这是**给用户的工作记忆**。

## 设计要点

### 1. todo 是给模型的，不是给用户的

很多新手会把 todo 做成项目管理工具（加 deadline、加优先级、加负责人）。这是过度设计。**todo 的唯一目的是帮模型记住自己该做什么**。任何对模型没用的字段都是噪音。

### 2. 状态语义要严

`in_progress` 同时只能有一个。如果模型把三个都设成 in_progress，等于没设。s05 的 `_normalize_todos` 只校验 status 字段合法，但更严的版本应该校验"in_progress 不超过 1 个"。

### 3. nag 频率要适中

s05 是 3 轮一次。太频繁（每轮）会打断模型思路；太稀疏（10 轮）已经晚了。3 - 5 轮是常见值。

### 4. 任务粒度要可控

模型可能把"写一个 web 服务"列成一个 todo（太大），也可能列成 50 个（太碎）。SYSTEM prompt 里应该给个大致引导："每个 todo 应该能在 3 - 10 个工具调用内完成"。

## 实现对照（s05/code.py）

工具定义：

```python
TOOLS.append({
    "name": "todo_write",
    "description": "Create and manage a task list for your current coding session.",
    "input_schema": {
        "type": "object",
        "properties": {
            "todos": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "content": {"type": "string"},
                        "status": {"type": "string",
                                    "enum": ["pending", "in_progress", "completed"]}
                    },
                    "required": ["content", "status"]
                }
            }
        },
        "required": ["todos"]
    }
})
```

Handler + 渲染：

```python
CURRENT_TODOS: list[dict] = []

def run_todo_write(todos: list) -> str:
    global CURRENT_TODOS
    todos, error = _normalize_todos(todos)
    if error:
        return error
    CURRENT_TODOS = todos
    # 渲染到 CLI（彩色）
    lines = ["\n## Current Tasks"]
    for t in CURRENT_TODOS:
        icon = {"pending": " ", "in_progress": "▸", "completed": "✓"}[t["status"]]
        lines.append(f"  [{icon}] {t['content']}")
    print("\n".join(lines))
    return f"Updated {len(CURRENT_TODOS)} tasks"
```

nag 机制：

```python
rounds_since_todo = 0

def agent_loop(messages: list):
    global rounds_since_todo
    while True:
        if rounds_since_todo >= 3 and messages:
            messages.append({"role": "user",
                             "content": "<reminder>Update your todos.</reminder>"})
            rounds_since_todo = 0
        ...
        # 每调用一次非 todo_write 的工具，计数 +1
        if block.name == "todo_write":
            rounds_since_todo = 0
        else:
            rounds_since_todo += 1
```

几个细节：

- `_normalize_todos` 是 schema 之外的二次校验。LLM 偶尔会传字符串（`"[{...}]"`），需要 json.loads 兜底。
- 渲染用 ANSI 颜色，让用户在 CLI 里一眼看到进度。
- nag 用 `<reminder>` 包裹，模仿 system-reminder 的语义，让模型识别为"系统提示"。

## 相关概念

- [[04 - Hooks]]：nag 的注入点在循环里，未来可以做成 Stop hook 的反向用法（强制再进一轮）。
- [[08 - Context Compact]]：todo 列表是**不应该被压缩**的外置记忆。压缩策略要避开它。
- [[06 - Subagent]]：子 Agent 应该有自己的 todo，不和主 Agent 共享——否则状态会串。

> [!warning]
> 几个容易踩的坑：
>
> 1. **让 todo 承担项目管理职责**：加 deadline、加 assignee，最后变成 JIRA，模型用不明白。
> 2. **同时多个 in_progress**：失去"现在做哪个"的语义，模型注意力分散。
> 3. **nag 太频繁**：每轮都提醒，等于背景噪音，模型学会忽略。
> 4. **不渲染给用户**：todo 只存在内存里，用户看不到进度，体感上"Agent 在乱跑"。
