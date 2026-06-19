---
type: concept
status: seed
domain: llm-agent
created: 2026-06-18
updated: 2026-06-18
aliases:
  - hooks
  - 钩子系统
tags:
  - agent
  - hooks
  - extensibility
  - s04
---

# 04 - Hooks

> [!note]
> s03 把权限检查写进了 agent_loop——但是写死在循环里意味着每次想加新逻辑（日志、截断、统计、用户偏好注入）都要改循环本身。Hooks 把循环的几个时机暴露成"扩展点"：注册一个回调函数，循环到那个时机就自动调用。这是 Agent 从"硬代码"走向"可扩展平台"的关键一步。

## 这一步加了什么

- 一个 **`HOOKS` 字典**：按事件名分桶，每个桶是一组回调。
- 一个 **`register_hook(event, callback)`**：往桶里加回调。
- 一个 **`trigger_hooks(event, *args)`**：依次调用桶里的回调，**任何一个返回非 None 就短路**。
- 四个事件：`UserPromptSubmit` / `PreToolUse` / `PostToolUse` / `Stop`。

s04 之后，s03 的 permission 变成了一个 PreToolUse hook。原本散在循环里的 log、context inject、summary 也都变成了 hook。

## 为什么需要加

### 1. 循环本身要保持简单

s01 - s03 的循环已经开始变臃肿：派发 + 权限 + 日志 + 各种 if。如果继续往里加（统计、限流、审计、用户偏好注入、停止前总结……），最后会变成一坨没人敢动的屎山。

**好的架构是循环只管"骨架"，所有"装饰"都挂在钩子上。**

### 2. 用户需要自定义

不同用户对同一个 Agent 的需求完全不同：

- 企业用户要**审计日志**（每次工具调用都写一行到 SIEM）。
- 个人用户要**快捷别名**（`/foo` 自动展开成某段 prompt）。
- CI 环境要**无交互**（禁用所有 ask，统一 deny）。

如果这些逻辑都写死在主循环，用户没法定制。Hook 让用户**不改源码就能改变行为**。

### 3. 测试更容易

模块化的 hook 可以单独测试。`permission_hook(block)` 就是一个纯函数，给它输入测输出。比测整个 agent_loop 容易多了。

## 这是一个什么机制

这是 **Lifecycle Hooks / Event-Driven Extension** 模式。前端框架（React 的 useEffect、Vue 的 mounted）、操作系统（Unix signal handler）、Web 框架（Express middleware、Django signals）里都有它的影子。

核心要素：

1. **预定义的事件名**：系统在固定的时机触发，不允许自定义事件（不然没法在循环里写）。
2. **回调链**：一个事件可以有多个回调，按注册顺序执行。
3. **短路语义**：某些事件的回调如果返回值，就**提前终止**整条链，后面的不再跑。
4. **副作用优先于返回值**：大多数 hook 的作用是副作用（打日志、改状态），返回值只是控制流。

### 四个事件的语义

```
用户输入 ──→ [UserPromptSubmit] ──→ 进入循环
                                   │
                                   ↓
                              调 API
                                   │
                                   ↓
                          ┌─── for each block:
                          │     [PreToolUse] (可短路)
                          │         ↓
                          │     执行 handler
                          │         ↓
                          │     [PostToolUse]
                          └─────────┘
                                   │
                                   ↓
                          stop_reason != tool_use
                                   │
                                   ↓
                              [Stop] (可强制再进循环)
                                   │
                                   ↓
                                 返回
```

每个事件的典型用途：

- **UserPromptSubmit**：注入上下文（当前目录、git 分支、最近修改的文件）、改写 prompt（别名展开）、阻止某些输入。
- **PreToolUse**：权限检查、限流、参数校验、记录审计日志。**可短路**——返回字符串就当 tool_result 塞回去，跳过执行。
- **PostToolUse**：截断输出（防上下文爆炸）、格式化结果、统计耗时、触发通知。
- **Stop**：会话总结、保存历史、自动追加"还有什么要做吗"的 prompt。

### 短路语义的妙处

注意 `trigger_hooks` 的实现：

```python
def trigger_hooks(event, *args):
    for callback in HOOKS[event]:
        result = callback(*args)
        if result is not None:
            return result
    return None
```

只要有一个回调返回非 None，**后面的回调就不再跑**。这意味着：

- 权限 hook 可以"否决"后续的日志 hook（如果它直接拒绝执行）。
- 多个权限规则可以叠加（先匹配的先生效）。
- 用户可以**优先级排序**：把最严格的规则放最前面。

## 原本的 Claude Code 怎么做的

Claude Code 的 hook 系统就是这个架构的工程化版本：

### 1. 更细的事件分类

除了 s04 的四个，Claude Code 还有：

- **SessionStart** / **SessionEnd**：会话级别的钩子。
- **Notification**：弹通知（比如长任务跑完）。
- **PreCompact**：上下文压缩之前（让你保存重要信息）。
- **SubagentStop**：子 Agent 结束时。

每个事件都有结构化的 input 和 output schema，方便写复杂的 hook。

### 2. Hook 用 shell 命令实现

Claude Code 的 hook 不是 Python 回调，而是 **shell 命令**——配置在 `settings.json` 里：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{"type": "command", "command": "/path/to/my-check.sh"}]
    }]
  }
}
```

Agent 把 block 的 JSON 通过 stdin 喂给 shell 命令，命令的退出码 / stdout 控制 hook 的决策。这样**用户可以用任何语言写 hook**——bash、Python、Node、Go——只要能读 stdin 写 stdout。

### 3. matcher（工具名过滤）

不是所有 hook 都关心所有工具。`matcher: "Bash"` 表示只对 Bash 工具触发。这避免了"日志 hook 对每次 read_file 都跑一遍"的浪费。

### 4. 跨进程隔离

用 shell 命令做 hook 还有一个好处：**hook 崩溃不会拖垮主进程**。一个有 bug 的 Python 回调会让整个 Agent 挂掉；shell 子进程挂了最多这一条 hook 失效。

## 设计要点

### 1. 事件要少而稳定

事件名是**公共 API**。一旦发布就不能随便改——所有 hook 都依赖它。Claude Code 的事件数控制在十几个以内，每个语义明确。新增事件要慎重。

### 2. 回调要幂等可重入

PostToolUse hook 可能因为重试被调用多次。回调里不要做不可逆的事（比如每次都 append 一行日志导致重复）。

### 3. 错误要隔离

一个 hook 抛异常不应该让整个循环挂掉。最佳实践是 trigger_hooks 内部 catch 异常、打日志、继续跑下一个。

### 4. 顺序要可预测

按注册顺序执行是最简单的语义。如果允许优先级，要明确文档化。Claude Code 用配置文件里的数组顺序，简单粗暴。

## 实现对照（s04/code.py）

```python
HOOKS = {"UserPromptSubmit": [], "PreToolUse": [], "PostToolUse": [], "Stop": []}

def register_hook(event: str, callback):
    HOOKS[event].append(callback)

def trigger_hooks(event: str, *args):
    for callback in HOOKS[event]:
        result = callback(*args)
        if result is not None:
            return result
    return None

# 注册各种 hook
register_hook("UserPromptSubmit", context_inject_hook)  # 注入工作目录
register_hook("PreToolUse", permission_hook)            # 权限检查
register_hook("PreToolUse", log_hook)                   # 日志
register_hook("PostToolUse", large_output_hook)         # 截断输出
register_hook("Stop", summary_hook)                     # 停止前总结
```

在循环里的接入点：

```python
# 进入 agent_loop 之前
trigger_hooks("UserPromptSubmit", query)

# 每个工具调用之前
blocked = trigger_hooks("PreToolUse", block)
if blocked:
    results.append({"type": "tool_result",
                     "tool_use_id": block.id,
                     "content": str(blocked)})
    continue

# 执行 handler ...
output = handler(**block.input)

# 工具调用之后
trigger_hooks("PostToolUse", block, output)

# 整个循环结束
force = trigger_hooks("Stop", messages)
if force:
    messages.append({"role": "user", "content": force})
    continue  # 不真的停，再进一轮循环
```

注意 Stop 的 `force` 用法：返回非 None 会**强制再进一轮循环**，可以用来"自动续跑"（比如检测到任务没完成就追加一句 prompt）。

## 相关概念

- [[03 - Permission]]：s03 的 permission 在 s04 里被重构成 PreToolUse hook。
- [[05 - TodoWrite]]：Stop hook 是注入"提醒更新任务"的好地方。
- [[08 - Context Compact]]：PostToolUse hook 是截断大输出的标准位置。
- [[02 - Tool Use]]：PreToolUse / PostToolUse 是工具调用的"包围式"扩展点。

> [!warning]
> 几个容易踩的坑：
>
> 1. **回调里抛异常**：一个挂的 hook 拖垮整个 Agent。trigger_hooks 必须 catch。
> 2. **忘了短路语义**：permission 拒绝了但日志 hook 还在跑——可能产生误导日志。
> 3. **事件越加越多**：每个新功能都加个事件，最后事件之间互相依赖、顺序敏感，维护噩梦。
> 4. **hook 之间隐式通信**：通过全局变量传状态。改成显式参数或 context 对象。
