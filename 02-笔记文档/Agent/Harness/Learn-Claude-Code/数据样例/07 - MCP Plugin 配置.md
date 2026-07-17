# 07 - MCP Plugin 配置

> 来源：`learn-claude-code/s19_mcp_plugin/` + Claude Code 官方 MCP 支持
>
> [[19 - MCP Plugin]] 让 Claude Code 通过 **Model Context Protocol** 接入外部工具服务器——把"自有工具"变成"插件生态"。

## 文件位置

```
.claude/mcp.json              ← 项目级（团队共享）
~/.claude/mcp.json            ← 用户级（个人专属）
```

**优先级**：项目级覆盖用户级同名 server。

---

## mcp.json 结构

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxxxxxxxxxx"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"],
      "env": {}
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/puin/Documents"],
      "env": {}
    },
    "my-private-server": {
      "command": "node",
      "args": ["/path/to/my/server.js"],
      "env": {
        "API_KEY": "..."
      }
    }
  }
}
```

---

## 字段语义

### 顶层

```json
{
  "mcpServers": {
    "<server-name>": { ... }
  }
}
```

`<server-name>` 是用户起的名字，用于引用（日志、调试）。

### 单个 server 配置

| 字段 | 必填 | 含义 |
|---|---|---|
| `command` | ✓ | 启动 MCP server 的命令（如 `npx` / `node` / `python`） |
| `args` | ✓ | 命令参数数组 |
| `env` | 可选 | 环境变量（API key 等敏感信息） |

**典型模式**：`npx -y @modelcontextprotocol/server-XXX`——直接从 npm 拉 MCP server。

---

## MCP 协议核心

```
Claude Code (MCP client)
    │
    │ spawn subprocess（command + args）
    ▼
MCP Server (subprocess)
    │
    │ JSON-RPC over stdio
    ▼
列出工具 / 调用工具 / 获取资源
```

**关键**：每个 MCP server 是一个**独立进程**——通过 stdio 跟 Claude Code 通信。崩了不影响 Claude Code。

---

## 启动流程

```python
# 简化版（伪代码）
class McpPlugin:
    def load_config(self, path):
        return json.loads(path.read_text())

    def start_server(self, name, config):
        proc = subprocess.Popen(
            [config["command"], *config["args"]],
            stdin=subprocess.PIPE,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            env={**os.environ, **config.get("env", {})}
        )
        # 握手：发 initialize，等响应
        self._send(proc, {"method": "initialize", "params": {...}})
        resp = self._recv(proc)
        # 列工具
        self._send(proc, {"method": "tools/list"})
        tools = self._recv(proc)
        return McpConnection(proc, tools)

    def call_tool(self, conn, tool_name, args):
        self._send(conn.proc, {
            "method": "tools/call",
            "params": {"name": tool_name, "arguments": args}
        })
        return self._recv(conn.proc)
```

---

## 工具发现（tools/list）

启动后 Claude Code 调 `tools/list` 拿 server 提供的所有工具，注入到 system prompt 第 2 段（Tool Definitions）：

```json
{
  "method": "tools/list",
  "result": {
    "tools": [
      {
        "name": "create_issue",
        "description": "Create a GitHub issue",
        "inputSchema": {
          "type": "object",
          "properties": {
            "title": {"type": "string"},
            "body":  {"type": "string"}
          }
        }
      },
      {
        "name": "search_repos",
        "description": "Search GitHub repositories",
        "inputSchema": {...}
      }
    ]
  }
}
```

**对模型来说**：MCP 工具跟内置工具**完全等价**——只是来源不同。

---

## 工具调用（tools/call）

模型决定调用时：

```json
// Claude Code → MCP server
{
  "method": "tools/call",
  "params": {
    "name": "create_issue",
    "arguments": {
      "title": "Bug: login fails on Safari",
      "body": "..."
    }
  }
}

// MCP server → Claude Code
{
  "result": {
    "content": [
      {"type": "text", "text": "Issue #42 created: https://github.com/..."}
    ]
  }
}
```

---

## 跟内置工具的差异

| 维度 | 内置工具 | MCP 工具 |
|---|---|---|
| 来源 | Claude Code 代码 | 外部进程 |
| 启动 | 同进程 | 子进程（独立崩溃域） |
| 配置 | 硬编码 | `mcp.json` 动态 |
| 工具发现 | 编译时 | 运行时（`tools/list`） |
| 协议 | 函数调用 | JSON-RPC over stdio |
| 生态 | 内置 | 任意第三方 |

**类比**：内置工具是"自带技能"，MCP 是"装插件扩展技能"。

---

## 常见 MCP server

| server | 提供能力 |
|---|---|
| `@modelcontextprotocol/server-github` | GitHub issue / PR / repo 操作 |
| `@modelcontextprotocol/server-postgres` | PostgreSQL 查询 |
| `@modelcontextprotocol/server-filesystem` | 文件系统访问（受限路径） |
| `@modelcontextprotocol/server-slack` | Slack 消息发送 |
| `@modelcontextprotocol/server-puppeteer` | 浏览器自动化 |

---

## 安全注意事项

**`env` 字段常含敏感信息**（API token / 密码）。建议：

```json
// .gitignore
.claude/mcp.json          ← 不提交
.claude/mcp.json.example  ← 提交模板（敏感字段留空）
```

模板：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "<your-token-here>"
      }
    }
  }
}
```

---

## 改造成 PuinClaw 时

PuinClaw 基于 pi-mono，pi-mono 已经内置 MCP 支持。配置位置：

```
/Users/mr.puin/LLMLearning/Agent/pi-mono-main/.claude/mcp.json
```

或 PuinClaw 自己加 MCP 接入（参考 s19 实现）。

---

## 相关

- [[19 - MCP Plugin]] —— 完整实现、JSON-RPC 协议细节
- [[20 - Comprehensive Agent]] —— MCP 怎么跟其他能力（memory / skill / task）整合
- [[02 - Tool Use]] —— 内置工具的调用机制（MCP 是它的扩展）
