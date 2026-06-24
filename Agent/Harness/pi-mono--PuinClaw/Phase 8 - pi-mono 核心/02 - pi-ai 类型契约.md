---
type: concept
status: seed
domain: llm-agent
created: 2026-06-24
updated: 2026-06-24
series: pi-mono
tags:
  - pi-mono
  - pi-ai
  - typescript
  - 类型契约
aliases:
  - pi-ai types
  - ai package types
---

# 02 - pi-ai 类型契约（types.ts）

> [!note]
> 本篇对应 `packages/ai/src/types.ts`（337 行）。
> 跟 [[01 - pi-agent-core]] 的 types.ts 同款哲学：**纯类型声明，运行时全部消失**。
> 但 pi-ai 的 types.ts 更"实"——它定义的是**整个 pi-mono 跟 LLM 对话的协议层**。
> 读完这一篇，你应该能解释 agent 包里 `streamFn` 到底在调什么、`Message` 在传输时长什么样。

## 重点关注

读完这一篇，你应该能回答：

- pi-ai 把 11 家 LLM provider 收敛成什么统一协议？
- 一个 `Message` 在传输时有哪些形态？跟 agent 包的 `AgentMessage` 什么关系？
- LLM 流式响应（SSE chunk）怎么变成 12 种 `AssistantMessageEvent`？
- `Model<TApi>` 这个泛型在干嘛？为什么不是简单对象？
- OpenAI-compatible 兼容层为什么需要 12+ 个开关？

---

## 1. types.ts 在 pi-mono 里的位置

```
┌─────────────────────────────────────────────────────┐
│  coding-agent (产品层)                              │
│    ├─ tools / sessions / extensions                 │
│    └─ AgentMessage (扩展点,declaration merging)     │
│              │                                      │
│              ▼ convertToLlm: AgentMessage → Message │
│  agent (runtime)                                    │
│    └─ streamFn: (model, context, options) => Stream │
│              │                                      │
│              ▼                                      │
│  ai (本篇) ← 你在这里                              │
│    ├─ types.ts ← 契约(本篇)                         │
│    ├─ stream.ts    (4 个入口函数)                   │
│    ├─ api-registry (provider 注册表)                │
│    └─ providers/*  (11 家具体实现)                  │
└─────────────────────────────────────────────────────┘
```

**核心定位**：types.ts 定义了 pi-ai 跟外界的**全部契约**——
- 上游（agent 包）：用 `StreamFunction` / `Message` / `Context`
- 下游（provider 实现）：实现 `ApiProvider`（见 stream.ts 注册），吐 `AssistantMessageEvent`

类比：types.ts 是**一份"LLM RPC 协议规范"**——11 家 provider 都要按这份协议说话。

---

## 2. 核心概念分组（9 组，按阅读顺序）

### 2.1 身份标识层（先看这个，建立坐标系）

```
KnownApi (10 个)      ← LLM API 的协议族
   │
   ▼
Api = KnownApi | string   ← 扩展点:用户可加自定义 api

KnownProvider (22 个) ← LLM 服务商(品牌)
   │
   ▼
Provider = KnownProvider | string

Model<TApi>           ← 一个具体模型(协议族 + 服务商 + 配置)
```

**关键认知**：
- `Api` 是**协议**（怎么说话）—— anthropic-messages / openai-responses / ...
- `Provider` 是**服务商**（谁提供）—— anthropic / openai / xai / ...
- 一个 `Provider` 可能支持多个 `Api`（比如 OpenAI 同时有 completions 和 responses 两个 api）
- `Model<TApi>` 的泛型把"这个模型走哪个协议"写进类型系统

```
Provider "openai" ─┬─→ api "openai-completions" (老协议)
                   └─→ api "openai-responses"   (新协议)

Provider "anthropic" ─→ api "anthropic-messages" (唯一)
```

### 2.2 思考控制（reasoning 模型专用）

| 类型 | 取值 | 用途 |
|---|---|---|
| `ThinkingLevel` | `"minimal" \| "low" \| "medium" \| "high" \| "xhigh"` | 思考强度(5 档) |
| `ThinkingBudgets` | `{ minimal?, low?, medium?, high? }` | 每档对应的 token 预算 |

> [!note]
> 跟 agent 包的 `ThinkingLevel` 一样,只是 agent 多了个 `"off"`。
> pi-ai 的 `ThinkingLevel` 没有 off——**因为关闭思考等于不调用 reasoning 模型**。

### 2.3 通用请求选项 StreamOptions（12 个字段）

这是 **provider 共享的最小选项集**,所有 provider 都要识别这些字段(不支持的忽略):

```ts
interface StreamOptions {
    temperature?: number;       // 温度
    maxTokens?: number;         // 最大输出 token
    signal?: AbortSignal;       // 取消信号
    apiKey?: string;            // API key
    transport?: Transport;      // "sse" | "websocket" | "auto"
    cacheRetention?: CacheRetention;  // "none" | "short" | "long"
    sessionId?: string;         // 会话 ID(prompt cache 用)
    onPayload?: (payload, model) => ...;  // ★ payload 拦截器
    headers?: Record<string, string>;     // 自定义 HTTP headers
    maxRetryDelayMs?: number;   // 重试延迟上限(60s 默认)
    metadata?: Record<string, unknown>;   // 元数据(Anthropic 用 user_id 等)
}
```

**两个变体**:

```
StreamOptions                  (基础)
   │
   ├─→ ProviderStreamOptions = StreamOptions & Record<string, unknown>
   │   (provider 可塞自己的字段)
   │
   └─→ SimpleStreamOptions = StreamOptions + {
           reasoning?: ThinkingLevel;       // ★ 思考级别
           thinkingBudgets?: ThinkingBudgets;
       }
       (streamSimple/completeSimple 用)
```

> [!important]
> agent 包里 `AgentLoopConfig extends SimpleStreamOptions`——
> 所以 `reasoning` 和 `thinkingBudgets` 能直接从 agent 层传到 provider。
> 这就是为什么 `thinkingLevel: "high"` 在 agent.ts 里设,最终能影响 LLM 请求。

### 2.4 流式契约（最核心,反复看）

#### 2.4.1 StreamFunction 签名

```ts
type StreamFunction<TApi, TOptions> = (
    model: Model<TApi>,
    context: Context,
    options?: TOptions,
) => AssistantMessageEventStream;
```

**契约(代码注释原文)**:
1. 必须返回 `AssistantMessageEventStream`
2. 一旦调用,**请求/模型/运行时错误必须编码进流里,不能抛**
3. 错误终止必须产生 `stopReason: "error" | "aborted"` 的 AssistantMessage,通过流协议发出

> [!warning]
> 第 2 条是反直觉的——**"错误作为数据"**,不是抛异常。
> 这是函数式编程风格,好处是流式消费者用一个 `for await` 就能处理所有情况,不用 try/catch。

#### 2.4.2 AssistantMessageEvent（12 种事件）

```
┌──────────────────────────────────────────────────────────────┐
│ 生命周期事件 (3 种)                                          │
│    start          流开始,带初始 partial                      │
│    done           成功结束,带最终 message                    │
│    error          失败结束,带 error message                  │
├──────────────────────────────────────────────────────────────┤
│  文本块事件 (3 种)                                           │
│    text_start     开始一个 text 块                           │
│    text_delta     文本增量                                   │
│    text_end       文本块结束                                 │
├──────────────────────────────────────────────────────────────┤
│  思考块事件 (3 种)                                           │
│    thinking_start 开始一个 thinking 块                       │
│    thinking_delta 思考增量                                   │
│    thinking_end   思考块结束                                 │
├──────────────────────────────────────────────────────────────┤
│  工具调用块事件 (3 种)                                       │
│    toolcall_start 开始一个 toolCall 块                       │
│    toolcall_delta 工具调用参数增量(JSON 片段)               │
│    toolcall_end   工具调用块结束                             │
└──────────────────────────────────────────────────────────────┘

每个事件都带 partial: AssistantMessage —— 实时快照
```

**设计要点**:
- 每种内容块都有 start/delta/end 三段,跟 SSE 的 chunk 边界对齐
- 所有事件都带 `partial`(当前 AssistantMessage 快照),消费者可以选择**只看 partial** 忽略 delta
- `done` / `error` 是终止事件,之后流必须关闭

### 2.5 内容块类型（assistant 消息里有什么）

```
AssistantMessage.content 是这个联合类型:

TextContent        { type: "text", text, textSignature? }
ThinkingContent    { type: "thinking", thinking, thinkingSignature?, redacted? }
ToolCall           { type: "toolCall", id, name, arguments, thoughtSignature? }
```

**3 种 Signature 字段**(高级):
- `textSignature` —— OpenAI 响应的元数据
- `thinkingSignature` —— OpenAI 的 reasoning item ID(多轮对话要回传)
- `thoughtSignature` —— Google 专有(复用 thought context)

> [!note]
> 这些 Signature 是**多轮对话的"跨轮指纹"**——
> 比如第二轮要把第一轮的 thinking 内容传回去,但 Anthropic 要求原始 signature,
> 所以 pi-ai 在第一轮就把 signature 存在 message 里,第二轮原样回传。

### 2.6 消息类型（3 种 role）

```
UserMessage         { role: "user",       content: string|(Text|Image)[], timestamp }
AssistantMessage    { role: "assistant",  content: (Text|Thinking|ToolCall)[],
                      api, provider, model, responseId?, usage, stopReason, errorMessage?,
                      timestamp }
ToolResultMessage   { role: "toolResult", toolCallId, toolName,
                      content: (Text|Image)[], details?, isError, timestamp }

Message = UserMessage | AssistantMessage | ToolResultMessage
```

#### AssistantMessage 字段速查

| 字段 | 含义 |
|---|---|
| `api` | 用的哪个 API 协议 |
| `provider` | 哪家服务商 |
| `model` | 实际模型名(比如 `claude-opus-4`) |
| `responseId?` | provider 返回的响应 ID(可用来 debug) |
| `usage` | input/output/cacheRead/cacheWrite/cost |
| `stopReason` | `"stop" \| "length" \| "toolUse" \| "error" \| "aborted"` |
| `errorMessage?` | stopReason 是 error/aborted 时填 |

> [!important]
> 跟 agent 包的 `AgentMessage` 区别:
> - agent 包的 `AgentMessage = Message + CustomAgentMessages`(可扩展)
> - pi-ai 的 `Message` 是**传输层格式**,不能扩展
> - 中间靠 `convertToLlm` 转换: `AgentMessage[] → Message[]`

### 2.7 Usage（钱和 token 统计）

```ts
interface Usage {
    input: number;       // 输入 token
    output: number;      // 输出 token
    cacheRead: number;   // 缓存命中读
    cacheWrite: number;  // 缓存写入
    totalTokens: number;
    cost: {
        input: number;
        output: number;
        cacheRead: number;
        cacheWrite: number;
        total: number;  // ★ 直接是美元($)
    };
}
```

> [!tip]
> `cost.total` **直接是美元**——pi-ai 在内部按 Model.cost($/M tokens) 算好了。
> UI 显示成本直接读这个字段,不用自己换算。

### 2.8 Tool 与 Context（请求载荷）

```ts
interface Tool<TParameters = TSchema> {
    name: string;
    description: string;
    parameters: TParameters;   // ★ TypeBox schema(不是 zod)
}

interface Context {
    systemPrompt?: string;
    messages: Message[];
    tools?: Tool[];
}
```

**注意**:这里 `Tool.parameters` 用的是 **TypeBox**(`@sinclair/typebox`),不是 zod 也不是 ajv。
原因:TypeBox 本身就是**类型即对象**——一个 schema 同时是 TS 类型,也是 JSON Schema,运行时校验也直接用。

### 2.9 OpenAI-compatible 兼容层（12+ 个开关）

为什么需要这个?——因为"OpenAI 兼容"是个**模糊概念**:

```
真正的 OpenAI    ─┐
Azure OpenAI     │  大家都说自己"OpenAI 兼容",
OpenRouter       │  但每个都有些不一样:
Groq             │   - 有的支持 store 字段,有的不支持
Cerebras         │   - 有的用 developer role,有的用 system
xAI              │   - 有的支持 reasoning_effort,有的不支持
ZAI              │   - maxTokens 字段名不一致
Qwen             │   - thinking 格式五花八门
Minimax          │   - ...
OpenCode         ─┘
```

```ts
interface OpenAICompletionsCompat {
    supportsStore?: boolean;
    supportsDeveloperRole?: boolean;
    supportsReasoningEffort?: boolean;
    reasoningEffortMap?: Partial<Record<ThinkingLevel, string>>;
    supportsUsageInStreaming?: boolean;
    maxTokensField?: "max_completion_tokens" | "max_tokens";
    requiresToolResultName?: boolean;
    requiresAssistantAfterToolResult?: boolean;
    requiresThinkingAsText?: boolean;
    thinkingFormat?: "openai" | "openrouter" | "zai" | "qwen" | "qwen-chat-template";
    openRouterRouting?: OpenRouterRouting;
    vercelGatewayRouting?: VercelGatewayRouting;
    supportsStrictMode?: boolean;
}
```

> [!warning]
> 不要背这些字段。**只需要知道"有这一层兼容映射"**——
> 看 anthropic.ts / openai-responses.ts 时,知道 provider 实现里会用这些开关;
> 真要写新 provider 时再回来查具体字段。

---

## 3. 跟 agent/types.ts 对照

| 概念 | agent 包 | ai 包 | 关系 |
|---|---|---|---|
| 消息 | `AgentMessage` (可扩展) | `Message` (固定) | `convertToLlm` 转换 |
| 工具 | `AgentTool` (带 execute) | `Tool` (只声明) | agent 的工具**包含** ai 的工具定义 |
| 工具调用 | `AgentToolCall` | `ToolCall` | 结构相同 |
| 工具结果 | `AgentToolResult<T>` | `ToolResultMessage` | agent 的 result 塞进 ai 的 message |
| 思考级别 | `ThinkingLevel` (含 "off") | `ThinkingLevel` (无 off) | agent 多一个 off |
| 流选项 | `SimpleStreamOptions` | `SimpleStreamOptions` | 共用 |
| 流函数 | `StreamFn` | `StreamFunction` | 同源,agent 的 StreamFn 来自 ai 的 StreamFunction |
| 事件 | `AgentEvent` (10 种) | `AssistantMessageEvent` (12 种) | 不同层事件,agent 的 message_update 转发自 ai 的 text_delta |
| 状态 | `AgentState` | (无) | ai 是无状态的,状态由 agent 管 |
| Context | `AgentContext` | `Context` | agent 的 context **包含** ai 的 context + 工具实例 |

> [!important]
> **核心关系**: ai 包是**传输层**,agent 包是**运行时层**。
> ai 不知道"循环"、"状态"、"工具执行",它只知道**怎么把 (model, context, options) 变成事件流**。

---

## 4. 最值得盯的 3 个抽象

如果时间紧,只看这 3 个:

1. **`AssistantMessageEvent`** (L237-249) —— 12 种事件定义了**整个 pi-mono 的事件协议**,所有 UI 渲染、日志、持久化都基于这 12 种
2. **`StreamFunction`** (L125-129) —— 这是 agent 包里 `streamFn` 的真身,理解它就理解 agent 跟 ai 的边界
3. **`Message` 联合** (L213) —— 3 种 role 构成的"传输三元组",pi-mono 的所有对话都基于这 3 种消息

---

## 5. 关键设计决策

### 5.1 为什么 `Api` 和 `Provider` 分开?

```
错误做法: 一个枚举包含所有 provider
   Provider = "anthropic" | "openai" | ...
   (然后每个 provider 内部硬编码协议)

pi-mono 做法: 协议(Api)和服务商(Provider)正交
   Provider "openai" 可同时支持:
     - api "openai-completions"
     - api "openai-responses"
   Provider "openrouter" 可走多个 api
```

好处:加一个新 provider(比如 "groq")如果它兼容 OpenAI 协议,**不用写新代码**,只要在 register-builtins 里挂到现有 api。

### 5.2 为什么 `StreamFunction` 不抛异常?

代码注释 L120-124 原文:
> Once invoked, request/model/runtime failures should be encoded in the returned stream, not thrown.

原因:**流式消费者用一个 for-await-of 就能处理所有情况**——
不用 try/catch 包裹 streamFn(),不用同时监听 error 事件和 try/catch,
错误就是普通的 `event.type === "error"`。

### 5.3 为什么 `AssistantMessage` 要带 `api` / `provider` / `model`?

因为**一个 transcript 可能跨多家 provider**——
比如用户切换模型继续对话,前 5 条是 anthropic,后 3 条是 openai。
每条 AssistantMessage 自己记录是哪家生成的,渲染/统计/调试都不会乱。

### 5.4 为什么 12 种 event 都带 `partial`?

partial 是**当前 AssistantMessage 的实时快照**。
好处:消费者可以选择**忽略所有 delta,只看 partial**——
比如日志系统只关心每个块的最终内容,不关心增量,就只读 partial。

---

## 6. [!warning] 阅读陷阱

| 陷阱 | 说明 |
|---|---|
| 把 `Api` 和 `Provider` 混淆 | Api 是协议,Provider 是服务商,正交关系 |
| 以为 `Tool.parameters` 用 zod | 实际用 TypeBox,因为 TypeBox 同时是类型和 schema |
| 忘了 `Message` 不可扩展 | 想加自定义消息去 agent 包,用 `CustomAgentMessages` |
| 看到 `textSignature` 困惑 | 这是 OpenAI 元数据,Anthropic 用不到,忽略 |
| 想给 `StreamFunction` 加 try/catch | 错误已经编码进流,不用 try |
| 把 `cost.total` 当 token 数 | 它是**美元**,直接显示 |

---

## 7. 读完应该能回答

1. pi-ai 把 11 家 provider 收敛成什么协议? —— `StreamFunction` 签名 + `AssistantMessageEvent` 12 种事件
2. `Model<TApi>` 的泛型在干嘛? —— 把"模型走哪个 api 协议"编码进类型
3. `StreamFunction` 失败了为什么不能抛? —— 错误作为数据,通过 `event.type === "error"` 流出
4. `Message` 跟 `AgentMessage` 什么关系? —— ai 的 Message 是传输层格式,agent 的 AgentMessage 是上层扩展,靠 `convertToLlm` 转换
5. `Usage.cost.total` 是什么单位? —— 美元

---

## 8. 相关概念

- [[01 - pi-agent-core]] —— agent 包(上层)
- [[00 - 阅读路线]] —— 接下来读 stream.ts
- [[06 - Intelligence]] —— claw0 主循环抽象(pi-ai 是它的底层)

_Generated while learning pi-mono, 2026-06-24_
