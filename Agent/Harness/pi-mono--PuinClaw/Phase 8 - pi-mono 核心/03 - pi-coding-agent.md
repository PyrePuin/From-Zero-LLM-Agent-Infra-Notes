---
type: concept
status: seed
domain: llm-agent
created: 2026-06-25
updated: 2026-06-25
series: pi-mono
tags:
  - pi-mono
  - pi-coding-agent
  - product-layer
  - harness
  - typescript
aliases:
  - pi-coding-agent
  - coding-agent
  - 产品层
---

# 03 - pi-coding-agent

> [!note]
> 本篇对应 `packages/coding-agent`(npm 名 `@mariozechner/pi-coding-agent`)。
> 这是 pi-mono 三层架构的**最顶层** —— 把 `pi-agent-core`(runtime)+ `pi-ai`(LLM 适配)封装成一个能直接用的编码 agent。
> 学完 [[01 - pi-agent-core]] + [[02 - pi-ai]] 再看这一篇,你会看到所有抽象如何"产品化"。
> 这一层 ≈ Claude Code / Cursor / Aider 的等价物,也是 [[06 - Harness]](面试) 的代码基础。

## 重点关注

读完这一篇,你应该能回答:

- pi-coding-agent 在 pi-mono 三层架构里担什么角色?
- 41,150 行代码 / 31 个 core 文件 / 3 种 UI 模式,分别是什么?
- `createAgentSession()` 这个唯一入口装配了哪些组件?
- system prompt 是怎么"动态组装"出来的?
- 7 个工具的 `ToolDefinition` 长什么样?
- 为什么工具自带 `promptSnippet` 和 `promptGuidelines`?
- 这一层跟你学过的 learn-claude-code / claw0 怎么对应?
- PuinClaw 改造应该改哪些文件?

---

## 1. 整体定位

```
┌──────────────────────────────────────────────────────────┐
│                  pi-coding-agent(本篇)                  │
│              产品层 — 用户实际用的就是这个                │
└──────────────────────────────────────────────────────────┘
              ↓ 依赖                    ↓ 依赖
   ┌──────────────────┐         ┌──────────────────┐
   │ pi-agent-core    │         │ pi-ai            │
   │ (runtime 双循环) │         │ (11 家 LLM 适配) │
   └──────────────────┘         └──────────────────┘
```

**一句话**:`pi-agent-core` 提供循环 + `pi-ai` 提供 LLM,中间加上工具/会话/扩展/UI,就成了 `pi-coding-agent`。

---

## 2. 顶层结构

```
src/
├── cli.ts / main.ts / index.ts   ← 入口(3 种调用方式)
├── config.ts                     ← 配置发现(~/.pi/agent)
│
├── core/          (31 文件) ★    ← 引擎层(大头)
├── modes/         (3 子目录)     ← 运行模式
├── utils/         (15 文件)      ← 通用工具
└── bun/                          ← Bun runtime 专用
```

总计 **41,150 行 TypeScript**。

---

## 3. `core/` 引擎层(31 文件,7 大类)

```
┌─ 入口与装配 ──────────────────────────────────────┐
│ sdk.ts              ★ createAgentSession 装配总入口
│ index.ts            模块对外导出(38 个导出)
│ agent-session.ts    AgentSession 类(被 sdk 创建)
├─ 提示词与上下文 ──────────────────────────────────┤
│ system-prompt.ts    ★ 系统 prompt 动态组装
│ prompt-templates.ts 自定义 prompt 模板
│ messages.ts         AgentMessage ↔ LLM Message 转换
│ skills.ts           Skills 加载与格式化
│ resource-loader.ts  AGENTS.md/CLAUDE.md 自动发现
├─ 会话与持久化 ────────────────────────────────────┤
│ session-manager.ts  会话保存/恢复(state.json)
│ settings-manager.ts 用户偏好(thinking/steering)
│ auth-storage.ts     API key 安全存储
│ compaction/         ★ 上下文压缩(4 文件)
│   ├ compaction.ts       压缩入口
│   ├ branch-summarization.ts  分支摘要
│   └ utils.ts
├─ 模型管理 ────────────────────────────────────────┤
│ model-registry.ts   已注册的模型表
│ model-resolver.ts   决定首次用哪个模型
├─ 工具(7 个) ─────────────────────────────────────┤
│ tools/              ★ read/write/edit/bash/grep/find/ls
│ bash-executor.ts    bash 工具底层执行器
│ file-mutation-queue 文件写入串行化
├─ 扩展系统 ────────────────────────────────────────┤
│ extensions/         (5 文件)插件机制
│   ├ types.ts          ExtensionAPI 契约
│   ├ loader.ts         发现与加载
│   ├ runner.ts         钩子触发器
│   └ wrapper.ts        包装器
├─ 杂项基础设施 ────────────────────────────────────┤
│ event-bus.ts        内部事件总线
│ exec.ts             子进程封装
│ output-guard.ts     输出安全过滤
│ timings.ts          性能埋点
│ diagnostics.ts      健康检查
│ package-manager.ts  npm 包管理
│ slash-commands.ts   /help /model 等斜杠命令
│ keybindings.ts      快捷键
│ defaults.ts         全局默认值
│ export-html/        会话导出 HTML
│ footer-data-provider.ts  TUI 底栏
│ source-info.ts      资源元信息
│ resolve-config-value.ts  配置值解析
└────────────────────────────────────────────────────┘
```

---

## 4. `modes/` 运行模式(3 种)

```
┌─────────────────────────────────────────────────────────┐
│ interactive/   ★ TUI 全屏交互模式(默认)               │
│   ├ interactive-mode.ts    主控制器                     │
│   ├ components/            UI 组件(输入框/消息流/...)  │
│   └ theme/                 主题 + 语法高亮              │
├─────────────────────────────────────────────────────────┤
│ print-mode.ts  非交互 — 跑一次就退出(CI/headless)     │
│                pi -p "fix the bug"                       │
├─────────────────────────────────────────────────────────┤
│ rpc/           JSON-RPC 模式(给编辑器/IDE 用)         │
│   ├ rpc-mode.ts    主入口                               │
│   ├ rpc-client.ts  客户端                               │
│   ├ rpc-types.ts   协议定义                             │
│   └ jsonl.ts       JSON Lines 传输                      │
└─────────────────────────────────────────────────────────┘
```

对照 learn-claude-code s2 三种模式的产品化实现。

---

## 5. `utils/` 通用工具(15 文件)

```
├─ 进程类
│   child-process.ts   spawn 封装
│   shell.ts           shell 检测
│   sleep.ts           异步 sleep
├─ 剪贴板(截图粘贴)
│   clipboard.ts / clipboard-native.ts / clipboard-image.ts
├─ 图片
│   image-resize.ts    ★ 工具用(给 LLM 前 resize)
│   image-convert.ts   格式转换
│   photon.ts          WASM 图片引擎
│   exif-orientation.ts EXIF 方向修正
│   mime.ts            MIME 检测
├─ 工具类
│   git.ts             git 操作
│   frontmatter.ts     YAML frontmatter 解析
│   changelog.ts       版本变更
│   tools-manager.ts   工具开关管理
```

---

## 6. 深入:`sdk.ts`(装配总入口)

### 6.1 文件结构(3 部分)

```
┌─────────────────────────────────────────────────────┐
│ Part 1: Re-exports (L85-123)                        │
│   把 tools/extensions/skills 的类型和工厂透出      │
│   → SDK 用户不用深入子模块                          │
├─────────────────────────────────────────────────────┤
│ Part 2: CreateAgentSessionOptions (L42-83)          │
│   全部字段可选 — 任何字段不传就用默认              │
│   → "零配置能跑,要定制也行"                       │
├─────────────────────────────────────────────────────┤
│ Part 3: createAgentSession() (L166-360) ★          │
│   真正的组装逻辑                                   │
└─────────────────────────────────────────────────────┘
```

### 6.2 `createAgentSession()` 干了 8 件事

```
① 解析 cwd / agentDir                  (L167-168)
   cwd 默认 process.cwd()
   agentDir 默认 ~/.pi/agent

② 创建基础设施(可选注入)             (L172-184)
   AuthStorage     — 凭证存储
   ModelRegistry   — 模型注册表
   SettingsManager — 配置
   SessionManager  — 会话持久化
   ResourceLoader  — skills/resources 加载

③ 决定使用哪个 model                  (L191-221)
   优先级:options.model >
          已有 session 里恢复的 model >
          findInitialModel(settings 默认)

④ 决定 thinkingLevel                  (L223-240)
   session 历史 > settings 默认 > 'medium'
   如果 model 不支持 reasoning → 强制 'off'

⑤ 选默认 active tools                 (L242-245)
   不传 → [read, bash, edit, write]

⑥ 包装 convertToLlm(blockImages 用)  (L250-284)
   按 setting 动态过滤图片

⑦ new Agent(...) — 真正创建 runtime  (L288-325)
   注入 streamFn/onPayload/transformContext
   这些钩子让 extensions 能介入

⑧ 包成 AgentSession 返回              (L341-359)
   AgentSession = Agent + Managers + Extensions
```

### 6.3 关键设计:**依赖注入 + 零配置默认**

```
传 options  → 全自定义(测试/嵌入别的应用)
不传        → 全部用默认(开箱即用)

         createAgentSession(options)
                    │
        ┌───────────┴───────────┐
        │                       │
   用户传入的               自动创建的
   (override)              (default)
        │                       │
        └─────────→ 合并使用 ←──┘
                    │
                    ▼
              AgentSession
```

---

## 7. 深入:`system-prompt.ts`(Context Engineering 教科书)

### 7.1 行号对照(默认模板 L127-167)

```
L127   ① 角色:"expert coding assistant ... pi harness"
L130   ② Available tools(占位符替换)
L132   ② 补充:可能还有 custom tools
L135   ③ Guidelines(dynamic,占位符替换)
L137-143 ④ Pi 自身文档路径 + 使用规则
L145   (可选)appendSystemPrompt
L150-156 ⑤ Project Context(AGENTS.md / CLAUDE.md)
L159-161 ⑥ Skills(formSkillsForPrompt)
L164   Current date: YYYY-MM-DD
L165   Current working directory: ...
```

### 7.2 6 段结构

```
┌────────────────────────────────────────────┐
│ [L127]    ① 角色 — pi coding agent        │
│                                            │
│ [L130]    ② Available tools               │
│ [L132]       (custom tools 补充说明)       │
│                                            │
│ [L135]    ③ Guidelines(dynamic)           │
│                                            │
│ [L137-143] ④ Pi docs paths + rules        │
│                                            │
│ [L145]    (可选)appendSystemPrompt        │
│                                            │
│ [L150-156] ⑤ Project Context              │
│               ## AGENTS.md                 │
│               ## CLAUDE.md                 │
│                                            │
│ [L159-161] ⑥ Skills(formSkillsForPrompt) │
│                                            │
│ [L164]    Current date: 2026-06-25         │
│ [L165]    Current working directory: ...   │
└────────────────────────────────────────────┘
```

### 7.3 3 个关键设计点

```
① Guidelines 是动态的(L101-119)
   按 selectedTools 组合选不同指南
   例:有 bash 没 grep → "用 bash 做 ls/rg/find"
       有 bash 又有 grep → "优先用 grep 工具"

② 工具可见性 = toolSnippets 过滤(L86)
   没 snippet 的工具 → 不告诉 LLM(虽然仍可用)

③ Context Files 直接拼到 prompt(L150-156)
   AGENTS.md/CLAUDE.md 不需要 LLM 主动读
   → 节省 tool call
```

---

## 8. 深入:`tools/read.ts`(工具样板)

### 8.1 ToolDefinition 的 7 个字段(L120-260)

**所有工具都长这样** —— pi-mono 工具的标准结构:

```
┌──────────────────────────────────────────────────────────┐
│ name:             "read"                                 │
│ label:            "read"     ← TUI 显示用                │
│ description:      长描述   ← LLM 看这个决定是否调用     │
│ promptSnippet:    "Read file contents" ← 进 system prompt│
│ promptGuidelines: ["Use read instead of cat/sed"]        │
│ parameters:       readSchema (typebox) ← 参数 JSON Schema│
│                                                          │
│ execute:           (toolCallId, args, signal) => Result  │
│ renderCall:        TUI 显示调用时怎么渲染                │
│ renderResult:      TUI 显示结果时怎么渲染                │
└──────────────────────────────────────────────────────────┘
```

### 8.2 execute() 的核心流程(L127-249)

```
入口:execute(toolCallId, { path, offset, limit }, signal)
   │
   ├─ ① 解析绝对路径              L134
   ├─ ② 绑定 AbortSignal          L137-146
   ├─ ③ access(absolutePath)      L151
   ├─ ④ detectImageMimeType       L153
   │     │
   │     ├─ 是图片 → L156-184
   │     │   ├─ readFile → buffer → base64
   │     │   ├─ autoResize → 2000x2000(默认开)
   │     │   └─ 返回 [TextContent, ImageContent]
   │     │
   │     └─ 是文本 → L185-237
   │         ├─ readFile → toString("utf-8")
   │         ├─ split("\n") → allLines
   │         ├─ 应用 offset(1→0 indexed)   L192
   │         ├─ 应用 limit(用户显式限制)   L201
   │         ├─ truncateHead(双重限制)     L209
   │         └─ 构造"续读提示"             L216-235
   │
   └─ ⑤ 返回 { content, details }    L241
```

### 8.3 3 个关键设计点

#### (1) ReadOperations 接口 — 远程文件系统可注入(L33-46)

```typescript
export interface ReadOperations {
    readFile: (absolutePath: string) => Promise<Buffer>;
    access: (absolutePath: string) => Promise<void>;
    detectImageMimeType?: (...) => Promise<string | null>;
}
```

**意义**:把 `readFile` 抽象出来,read 工具可以**远程跑** — 注入一个走 SSH 的 `readFile`,LLM 调 `read` 时实际读的是远程机器。

→ 这是 **pi-mono 的沙盒方案基础**(对照 [[05 - Agentic Sandbox]])。

#### (2) 截断策略 — 给 LLM 可执行的下一步(L208-235)

```
truncateHead 返回 4 种情况,每种都给 actionable 提示:

① 首行就超字节限制                       L211-215
   "[Line N is 500KB, exceeds 256KB limit.
    Use bash: sed -n 'Np' path | head -c 256]"
   → 告诉 LLM 改用 bash + sed

② 行数/字节被截断                        L216-226
   "[Showing lines 1-2000 of 5000.
     Use offset=2001 to continue.]"

③ 用户显式 limit 提前结束                L227-231
   "[3000 more lines in file.
     Use offset=2050 to continue.]"

④ 没截断                                L232-235
   直接返回内容
```

**意义**:LLM 不用瞎猜,直接照提示调下一次 `read`。这是 **pi-mono 处理大文件的核心模式** — 不一次读完,而是"按需续读"。

#### (3) 图片自动 resize(L156-184)

```
原始图 → base64 → resizeImage(2000x2000 上限)
              │
              ├─ 成功 → 返回 [文本说明, resized 图片]
              └─ 失败(还是太大)→ 只返回文本"[Image omitted]"
```

**意义**:Anthropic / OpenAI 的 vision API 有 size 限制,pi-mono 在工具层就把图片预处理了,避免 LLM 直接拒收。

#### (4) promptSnippet + promptGuidelines(L124-125)★

```typescript
promptSnippet:    "Read file contents",
promptGuidelines: ["Use read to examine files instead of cat or sed."],
```

**这就是 `system-prompt.ts` 里 toolSnippets 和 guidelines 的来源!**

```
回想 system-prompt.ts L86:
   visibleTools = tools.filter(name => !!toolSnippets?.[name])

回想 system-prompt.ts L114:
   for (const guideline of promptGuidelines ?? []) {...}

→ 工具自带"如何被介绍给 LLM"的元数据
→ 不是在 system-prompt 里硬编码
```

### 8.4 默认导出策略(L263-269)

```
createReadToolDefinition(cwd)  ← 返回 ToolDefinition(给 extensions 用)
       ↓ wrapToolDefinition
createReadTool(cwd)            ← 返回 AgentTool(给 Agent 直接用)

readToolDefinition = createReadToolDefinition(process.cwd())
readTool           = createReadTool(process.cwd())
                                       ↑
                          默认用 process.cwd(),向后兼容
```

---

## 9. 功能矩阵(16 项)

```
┌──────────────────┬─────────────────────────────────────┐
│ 功能             │ 实现位置                            │
├──────────────────┼─────────────────────────────────────┤
│ 1. 跑 agent      │ sdk.ts → agent-session.ts           │
│ 2. 工具调用      │ tools/ 7 个工具                     │
│ 3. 多 LLM 支持   │ 转发到 pi-ai                        │
│ 4. 上下文管理    │ compaction/ + system-prompt.ts      │
│ 5. 会话持久化    │ session-manager.ts                  │
│ 6. 模型切换      │ model-registry.ts + model-resolver  │
│ 7. 项目上下文    │ resource-loader.ts (AGENTS.md)      │
│ 8. 技能系统      │ skills.ts                           │
│ 9. 扩展插件      │ extensions/                         │
│ 10. 3 种 UI 模式 │ modes/                              │
│ 11. 凭证管理     │ auth-storage.ts                     │
│ 12. 斜杠命令     │ slash-commands.ts                   │
│ 13. 截图粘贴     │ utils/clipboard-*                   │
│ 14. 图片支持     │ utils/image-*                       │
│ 15. HTML 导出    │ core/export-html/                   │
│ 16. 主题高亮     │ modes/interactive/theme/            │
└──────────────────┴─────────────────────────────────────┘
```

---

## 10. 跟你学过的对照(完整映射)

```
learn-claude-code:
  s10 System Prompt      → core/system-prompt.ts
  s11 Recovery           → core/session-manager.ts
  s12 Task System        → core/agent-session.ts + tools/
  s13 Background Tasks   → modes/rpc/(异步任务基础)
  s15/17 Agent Teams     → 通过 sdk.ts 多实例 spawn
  s18 Worktree           → utils/git.ts
  s19 MCP Plugin         → core/extensions/

claw0:
  s04 Channels           → modes/interactive + modes/rpc
  s05 Gateway            → core/sdk.ts(路由到 Agent)
  s06 Intelligence       → core/system-prompt.ts + compaction/
  s07 Heartbeat          → core/agent-session.ts(steering)
  s08 Delivery           → core/tools/(操作介质)
  s09 Resilience         → core/session-manager.ts + auth-storage
  s10 Concurrency        → utils/file-mutation-queue

pi-mono:
  pi-ai                  → 通过 messages.ts 转换 + sdk.ts streamFn
  pi-agent-core          → sdk.ts new Agent(...) 创建实例
```

---

## 11. PuinClaw 改造路线

```
要让 pi-mono 变 PuinClaw,改这些就够:

┌─────────────────────────────────────────────────┐
│ 必改                                             │
│  ├─ core/agent-session.ts    /goal 长跑循环     │
│  ├─ modes/ 加 telegram-mode  IM 接入            │
│  └─ tools/bash-executor.ts   Docker 沙盒接入    │
├─────────────────────────────────────────────────┤
│ 增益(规划中)                                   │
│  ├─ extensions/ 加 reviewer    Agent Teams      │
│  ├─ core/compaction/          长跑上下文治理    │
│  └─ modes/rpc/ + webhook      GitHub PRD2PR     │
└─────────────────────────────────────────────────┘

→ 这一层的清晰分层让 PuinClaw 改造有明确"切入点"
→ 不用动 pi-agent-core / pi-ai(下层稳定)
```

---

## 12. 面试要点

### 必答清单

```
Q1: pi-coding-agent 在 pi-mono 里担什么角色?
A: 三层架构最顶层 —— 产品层。封装 pi-agent-core(runtime)
   + pi-ai(LLM 适配) + 7 个工具 + 扩展系统 + 3 种 UI 模式
   + 会话/模型/凭证/auth 管理 + skills/resources/compaction 上下文治理。
   ≈ Claude Code / Cursor / Aider 的等价物。

Q2: createAgentSession() 干了什么?
A: 唯一入口函数,装配 8 件事 ——
   ① cwd/agentDir 解析
   ② 5 个 Managers(AuthStorage/ModelRegistry/Settings/Session/ResourceLoader)
   ③ 决定 model(优先级:options > session > settings 默认)
   ④ 决定 thinkingLevel
   ⑤ 选 active tools
   ⑥ 包装 convertToLlm
   ⑦ new Agent(...) 创建 runtime
   ⑧ 包成 AgentSession 返回

Q3: system prompt 是写死的吗?
A: 不是。6 段中只有 L127-143 写死,其余动态注入 ——
   • Available tools 按 toolSnippets 过滤
   • Guidelines 按 selectedTools 动态组合
   • Project Context 注入 AGENTS.md/CLAUDE.md
   • Skills 注入 formatSkillsForPrompt 结果
   • Date/cwd 实时生成

Q4: 工具的 ToolDefinition 长什么样?
A: 7 字段 —— name / label / description / promptSnippet /
   promptGuidelines / parameters / execute + 2 个 render 方法。
   特点:工具自带 promptSnippet 和 promptGuidelines,
   system-prompt.ts 收集这些元数据动态组装。

Q5: read 工具怎么处理大文件?
A: 双重限制(MAX_LINES + MAX_BYTES),4 种截断分支,
   每种都给 actionable 续读提示("Use offset=N to continue")。
   首行超限甚至建议改用 bash + sed。

Q6: read 工具怎么支持沙盒?
A: ReadOperations 接口把 readFile 抽象出来,可注入 SSH 实现。
   LLM 仍调 read,但实际读的是远程机器。
```

### 加分点

```
• 引用 Harness Engineering 三大支柱,把 coding-agent 定位为
  Context + Harness + Workflow Engineering 的产品化整合
• 提到 toolSnippets/guidelines 自描述工具 → Context Engineering 教科书
• 提到 ReadOperations 接口是沙盒方案基础
• 提到截断策略的 actionable 设计(对照传统软件的 error message)
• 提到图片预处理(auto-resize)避免 LLM 拒收
• 把 PuinClaw 改造路线跟代码层级对上号
```

---

## 13. 1 分钟电梯陈述

```
"pi-coding-agent 是 pi-mono 的产品层,41000 行 TS,
封装 runtime + LLM 适配 + 7 个工具 + 扩展系统 + 3 种 UI 模式。

入口是 sdk.ts 的 createAgentSession(),一个函数装配 5 个
Managers + Agent + AgentSession,零配置能跑,全字段可注入。

亮点 3 个:
① system prompt 动态组装 —— Guidelines 按工具组合选,
   工具自带 promptSnippet 自描述,AGENTS.md 自动注入
② ToolDefinition 7 字段标准结构,read/write/edit/bash 都是变体
③ ReadOperations 接口让工具能远程跑 —— 沙盒方案基础

我的 PuinClaw 就基于这层改 —— /goal 长跑 + Telegram mode +
Docker sandbox + reviewer extension,不动下层 pi-agent-core
和 pi-ai,改造有明确切入点。"
```

---

## 14. 关键资料

| 资料 | 内容 |
|---|---|
| pi-mono 仓库 | `packages/coding-agent/src/` |
| [[01 - pi-agent-core]] | 下层 runtime |
| [[02 - pi-ai]] | 下层 LLM 适配 |
| [[06 - Harness]] | 产品层 = Harness 思路的产品化 |
| [[05 - Agentic Sandbox]] | ReadOperations 接口的延伸 |

---

## 15. 阅读路线状态

```
pi-mono 三层阅读进度:
   ✅ 01 - pi-agent-core(runtime 双循环 + 事件流)
   ✅ 02 - pi-ai(11 家 LLM 适配 + StreamFunction)
   ✅ 03 - pi-coding-agent(本篇,产品层)

下一步深入方向(按需):
   □ agent-session.ts        AgentSession 类细节
   □ compaction/             上下文压缩(长跑必备)
   □ extensions/             插件系统
   □ tools/edit.ts           diff 模式工具
   □ tools/bash.ts           沙盒相关
```

_Generated for PuinClaw 面试准备, 2026-06-25_
