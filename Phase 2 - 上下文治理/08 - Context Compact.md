---
type: concept
status: seed
domain: llm-agent
created: 2026-06-18
updated: 2026-06-18
aliases:
  - context compact
  - 上下文压缩
  - compaction
tags:
  - agent
  - context
  - compaction
  - s08
---

# 08 - Context Compact

> [!note]
> 任何 LLM 都有上下文上限（200K、1M、8M……）。一个跑了 50 轮的 Agent 会自然逼近这个上限——除非主动压缩。s08 设计了**四层渐进式压缩管线**：从最便宜的"丢中间消息"，到最贵的"让 LLM 写总结"，按代价从低到高依次触发。这是 Agent 能跑长任务的命门。

## 这一步加了什么

四层压缩，**从便宜到贵**：

| 层 | 名字 | 触发条件 | 做什么 | 成本 |
|---|---|---|---|---|
| L1 | **snip_compact** | messages > 50 条 | 砍掉中间最老的几条消息 | 几乎零 |
| L2 | **micro_compact** | 任何工具调用后 | 把旧 tool_result 替换成占位符 | 几乎零 |
| L3 | **tool_result_budget** | 单条 tool_result > 200K 字符 | 把大结果落盘，content 改成路径 | 一次磁盘 IO |
| L4 | **compact_history** | 总 token 逼近上限 | 让 LLM 把整个历史写总结 | 一次完整 API 调用 |

加上一个**反应式兜底**：API 返回 `prompt_too_long` 错误时，立即强制触发 L4。

## 为什么是分层而不是一刀切

最直观的做法是：上下文快满了 → 调 LLM 总结历史 → 用总结替换历史。

但这个做法有几个问题：

### 1. 贵

每次压缩都是一次完整的 API 调用（输入几十 K token，输出几 K 总结）。如果一跑长就压缩，开销指数级上升。

### 2. 慢

LLM 总结一次要几秒到几十秒。Agent 频繁停顿会让用户体验崩坏。

### 3. 信息损失不可控

LLM 总结会丢细节。如果只是因为"中间几条 grep 输出太大"，根本不需要 LLM 来总结——直接删掉就好。LLM 总结应该是**最后的手段**，不是第一手段。

### 4. 触发条件难定

什么时候算"快满了"？token 数？消息数？字符数？不同任务模式不一样。

分层方案的核心思想：**用最便宜的方法解决 80% 的情况，把昂贵的 LLM 总结留给真正复杂的 20%**。

## 这是一个什么机制

这是 **Tiered Degradation / Progressive Compression**，在系统设计里到处都是：

- CPU 缓存：L1 → L2 → L3 → 内存 → 磁盘，越往下越慢但容量越大。
- 日志系统：DEBUG → INFO → WARN → ERROR，默认级别以下全砍掉。
- 垃圾回收：分代 GC，新生代用 copying（便宜），老生代用 mark-sweep（贵）。

每一层的设计原则都是：

1. **本层先尝试**：能用便宜方法解决就不升级。
2. **本层失败时升级**：进入下一层。
3. **越往下越激进**：早期是"删冗余"，后期是"重新表达"。

### 每一层的具体语义

#### L1 - snip_compact（条数裁剪）

什么时候触发：messages 超过 50 条。

做什么：保留最早几条（用户最初目标）和最近几十条（最近上下文），中间砍掉。

为什么有效：长任务里中间的工具调用往往是为了"探索"，主 Agent 早已基于它们做出了决定，留下结论就够了。

为什么不直接砍最新：**最近上下文最重要**。模型在 ReAct 循环里需要看到"我刚才查了什么，结果是什么"才能决定下一步。

为什么不砍最早：**最初目标定义了任务**。砍掉就忘了用户要什么。

#### L2 - micro_compact（工具结果占位）

什么时候触发：每轮工具调用后，作为 PostToolUse 的一部分。

做什么：把"超过 N 轮之前的 tool_result"的 content 替换成 `"(see prior tool output)"` 之类的占位符。

为什么有效：tool_result 是上下文里**最大**的块（一次 grep 输出可能 50K token）。但它的影响会随时间衰减——几轮前的输出，主 Agent 早就消化了。

为什么是替换不是删除：API 协议要求每个 `tool_use` 必须有配对的 `tool_result`，否则报错。所以只能**改内容**，不能删块。

> [!note]
> **实现细节（Python 引用语义）**：
>
> 这一层有个微妙的实现技巧。`messages[i]["content"]` 是 block 列表，每个 block 是 dict。`micro_compact` 直接修改 block dict 里的 `content` 字段，**所有引用这个 dict 的地方都自动看到新值**——不需要"找到 messages 里对应的位置回写"。
>
> 这是因为 Python 的变量是引用（类似 C 的指针）：dict 是堆上的对象，赋值只是复制指针。多个变量指向同一个 dict 时，改 dict 的字段对所有变量可见。
>
> 但**重新赋值是另一回事**：`block = new_block` 只改了局部变量，原 dict 不变。所以这一层必须**改字段**（`block["content"] = "..."`）而不是**换对象**（`block = {...}`）。

#### L3 - tool_result_budget（大结果落盘）

什么时候触发：单条 tool_result 的 content 超过 200K 字符。

做什么：把 content 写到磁盘文件（`/tmp/tool_result_xxx.txt`），把 tool_result 的 content 改成 `"Result saved to /tmp/tool_result_xxx.txt (use read_file to access)"`。

为什么有效：很多工具输出天然巨大（一个 SQL 查询返回 50K 行）。这种"罕见但合法"的大块不该靠 L1 / L2 反复触发，直接落盘一次到位。

为什么不让模型直接看：模型也看不过来 200K 字符——它的注意力会被淹没。落盘后**模型需要时再 read_file 部分**，反而更精准。

#### L4 - compact_history（LLM 总结）

什么时候触发：总 token 数逼近上下文上限（比如 150K / 200K）。

做什么：

1. 把当前 messages 发给一个 LLM 调用，要求生成结构化总结：用户目标、已完成的工作、关键发现、未完成的子任务、重要文件路径。
2. 用总结替换整个 messages（保留最近几轮作为"工作记忆"）。
3. 之后续跑。

为什么这层贵但必要：前 3 层都是**机械式**的——它们不知道"什么是重要的"。只有 LLM 能语义判断"这个 grep 输出虽然老但是关键"。L4 是最后的语义压缩。

为什么不一直用 L4：太贵。一次 L4 调用 = 一次完整推理。频繁触发成本爆炸。

#### 反应式兜底（reactive_compact）

API 偶尔会返回 `prompt_too_long` 错误——你预估的 token 数和实际可能差几千。这时候不能让 Agent 死掉。catch 这个错误，强制跑一次 L4，然后重试。

这是 **Fail-Fast Recovery**：宁可花一次贵的总结，也不能让任务挂掉。

## 压缩什么不能碰

无论哪一层，都不要碰：

- **当前 todos（[[05 - TodoWrite]]）**：模型的外置记忆，丢了它就忘了任务。
- **最近 N 轮（最近的 5 - 10 条）**：模型在 ReAct 中的工作记忆。
- **最初的用户请求**：任务的目标定义。
- **system prompt**：永远不压缩。

这些都是"压缩不变量"——保护它们就是保护任务连续性。

## 原本的 Claude Code 怎么做的

Claude Code 的压缩在公开文档里描述得很清楚，核心思想一致：

### 1. 自动压缩

接近上下文上限时（系统会监控），自动触发 LLM 总结。总结会被**显式标记**（注入一条 system-reminder 告诉模型"以下是历史总结"），让模型知道这不是原始历史。

### 2. 用户可见

压缩发生时，CLI 会显示"Compressing context…"。**用户必须知道发生了什么**——模型可能因为压缩丢失关键信息，用户看到提示能主动补回（比如再说一遍"目标是 X"）。

### 3. /compact 命令

用户可以主动触发压缩，而不是等系统触发。在长任务里这是个有用的"保存点"——压缩完再继续，节省后续 token。

### 4. /clear 命令

更激进——直接清空所有历史，从空白开始。等价于"重启会话"。

### 5. todo 的特殊保护

Claude Code 的 todo / task 列表在压缩时被**单独保存**，压缩完作为新一轮 system-reminder 重新注入。这保证了无论压缩多少次，任务上下文都不会丢。

## 设计要点

### 1. 触发条件用"水位线"而不是单点

不要写"超过 150K 触发压缩"。应该写"超过 130K 准备触发，超过 150K 强制触发，超过 180K 紧急触发"。这样能避免边界抖动（在 150K 上下反复触发）。

### 2. 每层都要可观测

每次压缩都应该打日志：哪一层触发、压缩前后多少 token、压掉了什么。否则用户不知道为什么 Agent 突然忘了某个细节。

### 3. 压缩后注入 system-reminder

模型看到"以下是历史总结"的标记，会知道记忆可能不全，更倾向于 ask user 而不是假设。这是诚实的工程实践。

### 4. 压缩时机选在"自然的回合边界"

不要在工具调用中途压缩——可能丢失工具结果。最好在 stop_reason 不是 tool_use 的回合（模型说完了话）之后压缩。

## 实现对照（s08/code.py）

执行顺序（每次循环开始时跑一遍）：

```python
def agent_loop(messages: list):
    while True:
        # 四层管线，从便宜到贵
        tool_result_budget(messages)   # L3: 大结果落盘
        snip_compact(messages)         # L1: 条数裁剪
        micro_compact(messages)        # L2: 工具结果占位
        if estimate_tokens(messages) > CONTEXT_LIMIT:
            compact_history(messages)  # L4: LLM 总结

        try:
            response = client.messages.create(...)
        except APIError as e:
            if "prompt_too_long" in str(e):
                reactive_compact(messages)  # 反应式兜底
                continue
            raise
        ...
```

L2 的引用技巧（注意是改字段不是换对象）：

```python
def micro_compact(messages: list):
    KEEP_RECENT = 10
    for i, msg in enumerate(messages):
        if i >= len(messages) - KEEP_RECENT:
            break
        if not isinstance(msg.get("content"), list):
            continue
        for block in msg["content"]:
            # block 是 dict 引用，改字段会影响原 messages
            if block.get("type") == "tool_result":
                if not block["content"].startswith("("):
                    block["content"] = "(prior tool output elided)"
```

L4 的 LLM 总结：

```python
def compact_history(messages: list):
    summary_prompt = "Summarize the conversation so far: ..."
    response = client.messages.create(
        model=MODEL,
        messages=[{"role": "user", "content": summary_prompt + str(messages)}],
        max_tokens=4000,
    )
    summary = response.content[0].text
    # 替换 messages：总结 + 最近几轮
    recent = messages[-KEEP_RECENT:]
    messages.clear()
    messages.append({"role": "user",
                     "content": f"<summary>{summary}</summary>"})
    messages.extend(recent)
```

注意 `messages.clear()` + `messages.extend()` 而不是 `messages = [...]`——因为外部持有 messages 的引用（主循环的 history 变量），直接换对象会让外部失效。**继续引用语义的考量**。

## 相关概念

- [[05 - TodoWrite]]：todo 是压缩不变量，必须保护。
- [[06 - Subagent]]：subagent 是"预期会爆就先隔离"的主动策略，compact 是"已经爆了再压缩"的被动策略。互补。
- [[01 - Agent Loop]]：所有压缩都发生在 agent_loop 的回合边界。
- [[04 - Hooks]]：PostToolUse 是 L2 / L3 触发的天然位置。

> [!warning]
> 几个容易踩的坑：
>
> 1. **一次压缩太多**：L4 把整个历史塞进总结，丢失关键路径。保留最近 N 轮不被总结。
> 2. **压缩 todo**：任务列表被压没，模型忘了在做什么。
> 3. **改对象而不是改字段**：micro_compact 写成 `block = {...}` 不会生效，必须 `block["content"] = ...`。
> 4. **没有反应式兜底**：API 报 prompt_too_long 时直接挂掉。生产环境必须 catch + 强制压缩 + 重试。
> 5. **触发条件单点**：在 150K 边界抖动，反复触发。要用水位线 + 滞回。
