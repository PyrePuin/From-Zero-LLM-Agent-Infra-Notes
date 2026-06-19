---
type: concept
status: seed
domain: llm-agent
created: 2026-06-18
updated: 2026-06-18
aliases:
  - tool use
  - tool dispatch
tags:
  - agent
  - tools
  - s02
---

# 02 - Tool Use

> [!note]
> s01 的循环只认一个 `bash`。s02 把"工具"抽象成一张表：每个工具一个名字、一个 schema、一个 handler。模型每轮可以并发调用任意多个工具，循环按名字查表派发。这就是 Agent 拥有"行动能力"的标准姿势。

## 这一步加了什么

- 一个 **`TOOL_HANDLERS` 字典**：工具名 → Python 函数。
- 一个 **`TOOLS` 列表**：工具名 + JSON Schema，喂给 API 让模型知道有哪些工具可用。
- 循环里从"写死调 run_bash"变成 **按名字查表派发**：`handler = TOOL_HANDLERS.get(block.name)`。

## 为什么需要加

### 1. 一个工具不够

只用 `bash` 能做所有事吗？理论上可以——文件读写、grep、grep、网络请求都能 shell 出来。但有几个问题：

- **不安全**：模型写 `rm -rf` 你拦不住（s03 解决）。
- **不可靠**：模型对 `cat`、`grep` 输出格式的理解远不如它对结构化工具的理解。
- **不可观测**：`bash` 的输出是一坨字符串，harness 无法区分"读取文件"和"执行命令"，没法针对性打日志、限流、改权限。

把操作拆成 `read_file / write_file / edit_file / glob / bash` 等**语义化工具**，每个工具有清晰的 schema，模型才会用结构化的方式思考。

### 2. 一轮可能要调多个工具

模型经常会在一轮里同时说：

> "我先 `read_file('a.py')` 看看，同时 `read_file('b.py')` 对比一下。"

如果只支持一轮一个工具，第二个就得等下一轮——多花一次往返、多一份 token。所以循环要**遍历 `response.content` 里所有的 `tool_use` block**，并发执行，把所有结果一次性塞回去。

## 这是一个什么机制

这是经典的 **Dispatch Table / Strategy Pattern**：

```
工具名 (字符串)  →  handler (可调用对象)
"bash"          →  run_bash
"read_file"     →  run_read
"write_file"    →  run_write
...
```

加上 schema：

```
TOOLS = [
  {name, description, input_schema},  # 给模型看
  ...
]
TOOL_HANDLERS = {name: handler}       # 给 harness 用
```

两张表共享同一组名字，一张对外（API），一张对内（执行）。

### Schema 的作用

`input_schema` 是 JSON Schema。它做了三件事：

1. **告诉模型有哪些参数**：模型据此生成合法的 `input`。
2. **告诉 API 校验**：API 会拒绝不符合 schema 的调用，省去 harness 自己写校验。
3. **文档化**：好的 `description` 能显著提升模型选对工具的概率。

一个常见的坑：模型可能传 `limit="50"`（字符串）而 schema 要 `integer`。s05 之后会用 `_normalize_todos` 这种手动转换兜底，但**首选是让 schema 严格**——schema 严，模型少犯错。

## 原本的 Claude Code 怎么做的

Claude Code 内置工具大致是这套（部分）：

| 工具 | 作用 | 对应 s02 工具 |
|---|---|---|
| Bash | 跑命令 | bash |
| Read | 读文件（带行号、支持图片、PDF、ipynb） | read_file |
| Write | 覆盖写文件 | write_file |
| Edit | 精确替换（一次一处） | edit_file |
| Glob | 文件名模式匹配 | glob |
| Grep | 内容正则搜索（基于 ripgrep） | —— |
| TodoWrite / TaskCreate | 任务列表 | todo_write（s05） |
| Agent | 子智能体 | task（s06） |
| WebSearch / WebFetch | 联网 | —— |

每个工具的 schema 都很讲究——比如 `Edit` 会拒绝 `old_string` 不唯一的情况，逼模型用更多上下文定位；`Read` 默认 2000 行、可指定 `offset` 和 `limit`。

这些细节决定了 Agent 的**手感**：同样的模型，工具设计得好坏能让效率差好几倍。

## 几个值得记住的设计原则

### 1. 工具越小越专一越好

不要写一个 `do_everything(action, ...)`。一个工具一件事，名字动词开头，schema 字段不超 5 个。

### 2. 失败要返回字符串，不要抛异常

循环里如果 handler 抛异常，整个回合就崩了。最佳实践是**捕获所有异常，把错误字符串塞回去当 tool_result**：

```python
def run_bash(command: str) -> str:
    try:
        ...
    except Exception as e:
        return f"Error: {e}"
```

模型看到 `Error: ...` 会自己修正——这比 harness 替它重试要可靠得多。

### 3. 输出要截断

工具输出可能极大（比如 `cat` 一个 10MB 日志）。直接塞回 `messages` 会瞬间爆上下文。s02 已经有 `out[:50000]` 的兜底，s04 会用 PostToolUse hook 做更精细的截断，s08 会用 budget 把超大输出落盘。

## 实现对照（s02/code.py）

派发部分：

```python
TOOL_HANDLERS = {
    "bash": run_bash,
    "read_file": run_read,
    "write_file": run_write,
    "edit_file": run_edit,
    "glob": run_glob,
}

# 在 agent_loop 里：
for block in response.content:
    if block.type != "tool_use":
        continue
    handler = TOOL_HANDLERS.get(block.name)
    output = handler(**block.input) if handler else f"Unknown: {block.name}"
    results.append({"type": "tool_result",
                     "tool_use_id": block.id, "content": output})
```

几个关键点：

- `handler(**block.input)` 把模型的 JSON input 解包成关键字参数。schema 严的话这里不会出错。
- 未知工具名返回字符串而不是抛错，模型看到 "Unknown: xxx" 会自己停下来。
- 遍历 `response.content` 而不是只取第一个，所以**支持模型并发调用多工具**。

## 相关概念

- [[01 - Agent Loop]]：循环本身没变，只是循环体里"如何执行工具"从硬编码变成了查表。
- [[03 - Permission]]：在 `handler(**block.input)` 之前要插一道闸门。
- [[04 - Hooks]]：把闸门、日志、截断从循环里抽出去，做成可插拔回调。

> [!warning]
> 三个容易踩的坑：
>
> 1. **schema 太松**：`{"type": "object"}` 啥都不约束，模型乱传参数。
> 2. **handler 抛异常**：把异常冒泡到循环，整个回合崩溃。务必 catch 后返回字符串。
> 3. **不截断输出**：一次 `read_file` 一个大文件就把上下文撑爆，下一轮 API 直接 413。
