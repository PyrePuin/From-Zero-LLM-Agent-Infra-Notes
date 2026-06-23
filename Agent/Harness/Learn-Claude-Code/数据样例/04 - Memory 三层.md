# 04 - Memory 三层

> 来源：`learn-claude-code/s09_memory/code.py` + Claude Code 官方 `~/.claude/memory/`
>
> [[09 - Memory]] 不是单一存储，而是**三层**——不同层级用不同机制，覆盖不同时间尺度的记忆。

## 三层概览

```
┌──────────────────────────────────────────┐
│ 第 1 层：MEMORY.md (Evergreen)           │  ← 长期事实
│ 路径：~/.claude/MEMORY.md                │
│ 加载：每次启动全量加载                    │
│ 写入：手动维护或 LLM 主动写入             │
└──────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────┐
│ 第 2 层：daily JSONL                      │  ← 每日日志
│ 路径：~/.claude/memory/2026-06-23.jsonl  │
│ 加载：按需搜索（TF-IDF 或 LLM side-query）│
│ 写入：每轮自动追加                         │
└──────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────┐
│ 第 3 层：session context                  │  ← 当前对话
│ 路径：内存 messages[]                     │
│ 加载：自动                                 │
│ 写入：每轮自动                             │
└──────────────────────────────────────────┘
```

---

## 第 1 层：MEMORY.md（长期事实）

```markdown
# Evergreen Memory

## User Preferences
- Prefers concise answers
- Timezone: Asia/Shanghai (UTC+8)
- Primary language: Chinese, comfortable with English technical terms

## Important Context
- Working on PuinClaw (pi-mono fork)
- Values code quality and clean abstractions

## Technical Preferences
- TypeScript with explicit types
- Avoid backwards-compatibility hacks
- Default to no comments unless WHY is non-obvious
```

**特点**：
- 全量加载（不搜索）—— 进 system prompt 第 6 段
- 手动维护或 LLM 用 `memory_write` 工具写入
- 受 20k token 截断保护（超了截前半部分）

---

## 第 2 层：daily JSONL（每日日志）

文件：`~/.claude/memory/2026-06-23.jsonl`

```
{"ts": "2026-06-23T10:00:00Z", "type": "user_message", "content": "帮我实现 goal-runner"}
{"ts": "2026-06-23T10:02:00Z", "type": "tool_call", "name": "read_file", "input": {"path": "agent.ts"}}
{"ts": "2026-06-23T10:02:30Z", "type": "tool_result", "summary": "agent.ts 882 lines, env-driven model"}
{"ts": "2026-06-23T10:05:00Z", "type": "memory_write", "content": "agent.ts uses getModel() with env override"}
{"ts": "2026-06-23T10:10:00Z", "type": "user_message", "content": "加 checkpoint"}
{"ts": "2026-06-23T10:15:00Z", "type": "memory_write", "content": "Goal-runner uses state.json for checkpoint"}
```

**特点**：
- 每天一个文件，自动按日期切分
- 每条事件一行（JSONL 格式，跟 [[03 - Sessions]] 的会话持久化一样）
- **不全量加载**——按需搜索

### 搜索机制

`memory_search(query)` 用两种方式找相关记忆：

**方式 A：TF-IDF + 余弦相似度**（便宜版）
```
query = "checkpoint"
↓
对 daily JSONL 里所有 memory_write 条目算 TF-IDF
↓
返回 cosine similarity > 0.3 的前 5 条
↓
返回结果：
- "Goal-runner uses state.json for checkpoint" (score: 0.85)
- "state.json schema: turn, cost, status" (score: 0.72)
```

**方式 B：LLM side-query**（精准版）
```
query = "checkpoint"
↓
side-query to LLM: "Which memories are relevant to 'checkpoint'?"
候选记忆（前 N 条）+ query → LLM 判断相关性
↓
LLM 返回最相关的 3 条
```

s09 教学版用方式 A（免费），生产代码用方式 B（每次约 $0.01）。

---

## 第 3 层：session context（当前对话）

```python
messages = [
    {"role": "user",      "content": "帮我实现 goal-runner"},
    {"role": "assistant", "content": [...]},
    {"role": "user",      "content": "加 checkpoint"},
    {"role": "assistant", "content": [...]},
    {"role": "user",      "content": "现在改成支持 resume"},  # ← 当前
]
```

**特点**：
- 内存里，每轮自动追加
- 直接进 LLM 的 messages 参数（不是 system prompt）
- 超长时触发 [[08 - Context Compact|context compact]]

---

## 三层的协同

```
用户发消息
    ↓
[第 3 层] messages[] 自动包含最近对话
    ↓
[第 1 层] MEMORY.md 全量加载到 system prompt 第 6 段
    ↓
[第 2 层] side-query 搜相关 daily 日志 → 进 system prompt 第 6 段
    ↓
完整 context 喂给 LLM
```

**层级关系**：
- 第 1 层：永远在，不挑相关性（"档案"）
- 第 2 层：按相关性挑（"检索结果"）
- 第 3 层：完整对话历史（"短期记忆"）

---

## memory_write 工具

```json
{
  "name": "memory_write",
  "description": "Store a memory for later recall",
  "parameters": {
    "content": {"type": "string"},
    "type": {"type": "string", "enum": ["fact", "preference", "decision", "incident"]}
  }
}
```

**模型主动调用**——看到"用户说了重要的事"就写。比如：
- 用户："我项目用 TypeScript" → 模型调 memory_write({content: "User uses TypeScript", type: "preference"})
- 用户："明天 10 点提醒我" → 写一条 fact
- 用户："bug 在第 42 行" → 写一条 fact

---

## 跟 claw0 [[06 - Intelligence]] 的对比

| 维度 | learn-claude-code s09 | claw0 s06 |
|---|---|---|
| 长期记忆 | MEMORY.md（全量加载） | MEMORY.md（全量加载） |
| 短期记忆 | session messages | session messages |
| 中期记忆 | daily JSONL + TF-IDF/side-query | daily JSONL + TF-IDF + hash |
| 搜索精度 | LLM side-query（$0.01/次） | 纯算法（$0/次） |
| 搜索频率 | 每 agent_loop 进来时 1 次 | **每轮**都搜 |

详见 [[06 - Intelligence|claw0 s06 vs learn-claude-code s09]]。

---

## 相关

- [[09 - Memory]] —— 完整三层实现、side-query vs 算法搜索
- [[10 - System Prompt]] —— 记忆怎么进 prompt
- [[08 - Context Compact]] —— 记忆太多时怎么压缩
