---
type: concept
status: seed
domain: llm-agent
created: 2026-06-18
updated: 2026-06-18
aliases:
  - subagent
  - 子智能体
  - context isolation
tags:
  - agent
  - subagent
  - context
  - s06
---

# 06 - Subagent

> [!note]
> 一个研究类任务可能要跑 30 次工具调用、读 10 个文件、产生几万 token 的中间结果。这些**中间结果对最终决策没用**，但全留在主对话里会把上下文撑爆。Subagent 是一个"独立小循环"——给它一个任务，它在自己干净的 messages[] 里跑完，只把**总结**返回给主 Agent。中间过程全丢弃。

## 这一步加了什么

- 一个 **`task` 工具**：主 Agent 用它来"委托"任务。
- 一个 **`spawn_subagent(description)` 函数**：起一个**全新的 messages 数组**、自己的 system prompt、自己的循环、最多跑 30 轮。
- 一组 **SUB_TOOLS**：子 Agent 能用的工具集——**故意不含 `task`**，防止递归。
- 一个 **extract_text 兜底**：30 轮用完没出结论时，从最后几条 assistant 消息里抢救出文字。

## 为什么需要加

### 1. 探索型任务对上下文是污染

主 Agent 接到"找到所有处理支付的文件"。它得 grep 几次、read 几个、grep 再细查。整个过程可能产生 5 万 token 的中间结果。

但主 Agent 真正需要的只是答案：**"支付相关代码在 payment_service.py 和 checkout.py，主要函数是 charge() 和 refund()"**。

如果让主 Agent 自己跑这些工具，它的 messages 里会塞满无用的 grep 输出。**后续每一轮 API 调用都要为这些垃圾 token 付费**，而且模型可能被中间结果干扰，做出错误判断。

### 2. 隔离 = 可抛弃

子 Agent 用独立 messages，意味着：

- 中间过程可以激进地探索（read 大文件、grep 全仓库）而不用担心污染主对话。
- 主 Agent 只看到总结，注意力集中在"决策"而不是"搜寻"。
- 上下文预算被严格保护——主对话的增长只等于"总结长度"，不是"探索过程长度"。

### 3. 角色专精

子 Agent 可以有自己的 SYSTEM prompt。一个"探索 Agent"被告知"只看代码，不修改"，一个"计划 Agent"被告知"只输出方案不动手"。**角色分化**让单次任务更聚焦。

## 这是一个什么机制

这是 **Process Isolation / Fork-Join** 模式，操作系统里到处都是：

- Unix `fork()` 创建子进程，子进程有独立的内存空间，跑完通过 `pipe` / `exit code` 把结果传回父进程。
- Web 浏览器给每个 tab 一个独立的渲染进程，一个崩了不拖垮其他。
- MapReduce：mapper 独立处理一片数据，reducer 汇总。

Agent 里的 Subagent 完全对应：

- **Fork**：spawn_subagent 复制工具集、新建 messages、起一个新循环。
- **隔离**：子 Agent 的 messages 完全独立，不引用主 Agent 的历史。
- **Join**：子 Agent 跑完，只把最后一段文字返回。主 Agent 把这段文字当 tool_result 接收。

### "无递归"约束

注意 SUB_TOOLS 里**没有 task 工具**。这是故意的：

- 如果子 Agent 也能 spawn sub-subagent，主 Agent 可能不小心触发一个**深度爆炸**的递归。
- 资源消耗不可控——每个 subagent 都有自己的 30 轮限额，3 层嵌套就是 27000 轮潜在工具调用。
- 收益边际递减——两层的"主 → 子"已经足够隔离上下文，三层基本是过度设计。

所以 s06 的 subagent 是**单层**的。这和 Unix 的 `fork` 不一样（fork 可以无限嵌套），但在 Agent 场景下，限制深度比放开更稳健。

### 安全阀：30 轮上限

子 Agent 的循环不是 `while True`，是 `for _ in range(30)`。这是**另一个故意**：

- 模型可能陷入死循环（反复 read 同一个文件）。
- 模型可能策略错误（一个简单任务展开了 40 步）。
- 30 轮是个安全阀，超过就强制停，把当前状态作为返回。

如果 30 轮不够，说明任务**该拆**——主 Agent 应该把它分成两个子任务，而不是给单个 subagent 加预算。

## 原本的 Claude Code 怎么做的

Claude Code 内置了一组 subagent，每个有专门的 `subagent_type`：

### 1. 专门化的 subagent

- **Explore**：快速代码搜索、定位文件、回答"这段逻辑在哪"。
- **Plan**：设计实现方案，输出步骤化计划。
- **general-purpose**：通用研究、多步任务。
- **statusline-setup**：配置状态栏（很窄的专精）。
- **code-reviewer**：审查 diff（独立第二意见）。

每个 subagent 有自己的 system prompt、自己的工具子集。比如 Explore 没有 Edit / Write——它是只读的，不会误改代码。

### 2. description 字段

主 Agent 用 task 工具时传 `description`，子 Agent 看到的第一条 user 消息就是它。Claude Code 的工程实践强调 description 要**自包含**：

- 不要写"基于上面的发现做 X"——子 Agent 看不到"上面"。
- 要写"我们在改支付模块。当前文件是 payment.py。问题是退款逻辑没处理 X。请查找所有调用 refund 的地方并报告。"

这是 subagent 隔离的代价：**主 Agent 必须显式传上下文**。

### 3. 后台运行

Claude Code 的 Agent 工具支持 `run_in_background: true`——子 Agent 在后台跑，主 Agent 继续干别的。完成后通知主 Agent。这对**并行探索**特别有用：主 Agent 可以同时派 3 个 Explore Agent 查不同的方向。

### 4. worktree 隔离

更激进的隔离是 **git worktree**——子 Agent 不光有独立 messages，还有**独立的代码工作目录**。它的修改不影响主 Agent 看到的文件状态。这是 s18 的内容。

## 设计要点

### 1. 子 Agent 的 SYSTEM 要短而专

主 Agent 的 SYSTEM 包含工作目录、可用工具、用户偏好等，可能上千 token。子 Agent 的 SYSTEM 应该极简（s06 里就两句话："你是 X 的编码 Agent。完成任务，返回简洁总结。不要再委托。"）。

因为每轮 API 调用都要重发 SYSTEM，太长浪费 token。子 Agent 通常不需要那些上下文。

### 2. 总结要文字，不要结构化

子 Agent 返回的是 tool_result 的 content，对主 Agent 来说就是一段文字。**不要让它返回 JSON**——主 Agent 看到结构化数据反而要去解析，不如直接看自然语言总结。

如果非要结构化（比如多个候选方案），让子 Agent 返回 markdown 列表，主 Agent 读起来更顺。

### 3. 子 Agent 也跑 hook

s06 里 spawn_subagent 内部也调 `trigger_hooks("PreToolUse", block)`。这是故意的——子 Agent 的工具调用也要过权限闸门，不能因为它"层级低"就绕过安全。

### 4. 失败要优雅

子 Agent 可能：

- 30 轮跑完没结论（fallback 到最后一条 assistant 文字）。
- 工具调用失败（继续跑，让它从错误中学习）。
- SYSTEM 太严导致它拒绝执行（主 Agent 看到"我做不到 X"的反馈，自己调整）。

主 Agent 要把子 Agent 的返回**当成参考而非真相**。子 Agent 可能错——主 Agent 应该有自己的判断。

## 实现对照（s06/code.py）

子 Agent 工具集（注意没有 task）：

```python
SUB_TOOLS = [
    {"name": "bash", ...},
    {"name": "read_file", ...},
    {"name": "write_file", ...},
    {"name": "edit_file", ...},
    {"name": "glob", ...},
    # 没有 "task"
]
SUB_HANDLERS = {"bash": run_bash, "read_file": run_read, ...}
```

子 Agent 的 SYSTEM：

```python
SUB_SYSTEM = (
    f"You are a coding agent at {WORKDIR}. "
    "Complete the task you were given, then return a concise summary. "
    "Do not delegate further."
)
```

Spawn 函数：

```python
def spawn_subagent(description: str) -> str:
    messages = [{"role": "user", "content": description}]  # 全新 messages

    for _ in range(30):  # 安全阀
        response = client.messages.create(
            model=MODEL, system=SUB_SYSTEM,
            messages=messages, tools=SUB_TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            break
        # 执行工具（同样过 hook）
        results = []
        for block in response.content:
            if block.type == "tool_use":
                blocked = trigger_hooks("PreToolUse", block)
                if blocked:
                    results.append({"type": "tool_result",
                                     "tool_use_id": block.id,
                                     "content": str(blocked)})
                    continue
                handler = SUB_HANDLERS.get(block.name)
                output = handler(**block.input) if handler else f"Unknown"
                trigger_hooks("PostToolUse", block, output)
                results.append({"type": "tool_result",
                                 "tool_use_id": block.id, "content": output})
        messages.append({"role": "user", "content": results})

    # 抢救最后一段文字
    result = extract_text(messages[-1]["content"])
    if not result:
        for msg in reversed(messages):
            if msg["role"] == "assistant":
                result = extract_text(msg["content"])
                if result:
                    break
    return result  # 只返回总结，整个 messages 丢弃
```

主 Agent 把它当 tool 接：

```python
TOOLS.append({
    "name": "task",
    "description": "Launch a subagent to handle a complex subtask. Returns only the final conclusion.",
    "input_schema": {"type": "object",
                      "properties": {"description": {"type": "string"}},
                      "required": ["description"]}
})
TOOL_HANDLERS["task"] = spawn_subagent
```

关键细节：

- 子 Agent 的 `messages` 是函数局部变量，函数返回后就被 GC。**这就是隔离的本质**——内存里不存了，自然就影响不到主对话。
- 主 Agent 收到的就是一段字符串（子 Agent 的总结），它会被包成 `tool_result`，长度通常几百到几千 token——比 30 轮探索过程省 90% 上下文。
- 子 Agent 内部循环和主循环**结构完全一样**，只是用了不同的 SYSTEM、tools、messages。这是漂亮的代码复用。

## 相关概念

- [[01 - Agent Loop]]：子 Agent 跑的就是一个 agent_loop，只是绑定了不同的上下文。
- [[02 - Tool Use]]：SUB_TOOLS 是 TOOL_HANDLERS 的子集，去掉 task 防止递归。
- [[08 - Context Compact]]：subagent 是**主动**压缩——预期会爆就先隔离；compact 是**被动**压缩——已经爆了再压缩。两者互补。
- [[04 - Hooks]]：子 Agent 也跑 hook，权限规则一致。

> [!warning]
> 几个容易踩的坑：
>
> 1. **description 太短**："基于上面的探索，再深入查一下"。子 Agent 不知道"上面"是什么。必须自包含。
> 2. **允许递归**：SUB_TOOLS 里塞了 task，结果模型套娃，token 烧穿。
> 3. **没有轮数上限**：子 Agent 陷入死循环跑了几百轮才停。
> 4. **总结当作真相**：主 Agent 完全相信子 Agent 的结论，不再校验。子 Agent 也会错。
