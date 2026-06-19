---
type: concept
status: seed
domain: llm-agent
created: 2026-06-18
updated: 2026-06-18
aliases:
  - permission
  - 权限系统
tags:
  - agent
  - security
  - s03
---

# 03 - Permission

> [!note]
> Agent 一旦有 `bash`，就能 `rm -rf /`。Permission 这一步要做的事是：**在工具真正执行之前，插一道闸门**——按规则拒绝、放行，或者停下来问用户。它是 Agent 从"理论上能做"到"实际可以放心交给它做"之间的那一道安全网。

## 这一步加了什么

- 一个 **`check_permission(block) -> bool`** 函数：在 `handler(**block.input)` 之前调用。
- 三类决策：
  1. **deny list**（黑名单）：命令里包含 `rm -rf /`、`sudo`、`shutdown` 等 → 直接拒绝。
  2. **rules**（规则）：某些工具/某些参数模式 → 触发"询问用户"。
  3. **allow**：默认放行。
- 一个 **`ask_user`** 机制：把决定权交回人类。

## 为什么必须加

模型不是故意的，但它会犯错。常见的几种危险动作：

- **清理路径写错**：本来想删 `./tmp/`，结果写成 `/tmp/` 或者 `./*`。
- **覆盖关键文件**：`write_file` 覆盖 `.env`、`~/.ssh/` 之类。
- **网络请求**：模型可能主动去 `curl` 一个外部 URL，泄露代码或 token。
- **包管理**：`pip install` 改环境，`npm publish` 发包。

模型自己**无法可靠地判断"这次操作危险"**——它的训练目标是"完成任务"，不是"保守"。所以这个判断必须由 harness 来做。

而且这个判断**不能只在 SYSTEM prompt 里写规则**。模型可能"理解"了规则但还是违规，因为：

- 长对话里规则会被稀释。
- 模型在压力下（任务卡住、用户催）会绕过自己的约束。
- 规则用自然语言写，可解释性差。

**硬代码层 + 模型层**，两道防线都要有。s03 是硬代码层。

## 这是一个什么机制

这是 **Policy Enforcement Point（策略执行点）** 模式，在安全工程里很常见：

```
请求 → [PEP] → 资源
        ↓
       PDP（决策）
        ↓
     allow / deny / ask
```

翻译到 Agent：

- **PEP**：循环里执行 handler 之前的那一行 `if not check_permission(block): continue`。
- **PDP**：`check_permission` 函数本身，按规则算出决策。
- **策略源**：deny list、rules、用户偏好。

### 三种决策的语义

- **allow**：直接执行，不打扰用户。
- **deny**：返回 `"Permission denied"` 字符串当 tool_result。模型会看到这个反馈，自己调整。
- **ask**：弹一个交互问题给用户，用户选 allow / deny / always allow。

`ask` 这一支特别重要——很多操作不是非黑即白：删一个新建的临时文件可以 allow，删一个 git 跟踪的文件得问一下。把决策权交回用户，是 Agent **可信任**的关键。

## 原本的 Claude Code 怎么做的

Claude Code 的权限系统比 s03 复杂得多，但骨架一样：

### 1. 多级规则源

- **allow list** / **deny list**：用户配置（`settings.json`）和项目 CLAUDE.md 里写的。
- **per-tool rules**：比如 `Bash(rm:*)` 表示所有 `rm` 开头的命令。
- **default policy**：未匹配时的默认行为（ask / allow / deny）。

### 2. 交互模式

Claude Code 有几种 permission mode：

- **default**：危险操作 ask。
- **plan**：所有写操作都要 ask（用于规划阶段）。
- **acceptEdits**：编辑类自动 allow，bash 仍 ask。
- **bypassPermissions**：全部 allow（危险，只在隔离环境用）。

### 3. "Always allow" 记忆

用户在 ask 弹窗里可以勾"以后都允许"，这个规则会写回 settings，下次自动放行。这是用户体验的关键——没有它，每次都要点确认会让人发疯。

### 4. Hook 接管

到 s04 你会看到，permission 这套东西其实**整个被搬到 hook 系统里了**。`PreToolUse` hook 返回非 None 就短路执行。这是 Claude Code 真正的扩展方式：权限不是硬代码，是 hook。

## 设计要点

### 1. deny 要尽量短而狠

deny list 应该只放**绝对不允许、不需要解释**的东西（`rm -rf /`、`mkfs`、`dd if=`）。太长的 deny list 维护成本高，而且容易误伤（比如禁止 `git push -f` 会误伤强制覆盖远程的合法场景）。

灰色操作应该走 **ask**，让用户在场景里决定。

### 2. 拒绝信息要回到模型

deny 之后**不能静默丢弃**，要把"Permission denied"作为 tool_result 塞回去。模型看到这个反馈，才知道换一种方式。否则它会以为工具坏了，反复重试。

### 3. permission 是 harness 的责任，不是模型

不要在 SYSTEM prompt 里写"你不许 rm -rf"就完事。模型会违反。**PEP 必须在代码里强制执行**。

### 4. 决策应该可观测

每次 deny / ask 都要打日志。这样事后排查"为什么我的 Agent 没完成任务"时，能看到是哪条规则拦了它。

## 实现对照（s03/code.py）

伪代码（具体见仓库）：

```python
DENY_LIST = ["rm -rf /", "sudo", "shutdown", "reboot", "mkfs", "dd if="]

def check_deny_list(command: str):
    for p in DENY_LIST:
        if p in command:
            return f"Blocked: contains '{p}'"
    return None

def check_permission(block) -> bool:
    if block.name == "bash":
        reason = check_deny_list(block.input.get("command", ""))
        if reason:
            return False
    reason = check_rules(block.name, block.input)
    if reason:
        decision = ask_user(block.name, block.input, reason)
        if decision == "deny":
            return False
    return True

# 在 agent_loop 里：
for block in response.content:
    if block.type != "tool_use":
        continue
    if not check_permission(block):
        results.append({"type": "tool_result",
                         "tool_use_id": block.id,
                         "content": "Permission denied"})
        continue
    output = TOOL_HANDLERS[block.name](**block.input)
    ...
```

注意几点：

- 被拒绝的工具**也要回一个 tool_result**，否则 API 会因为缺 tool_use_id 而报错。
- `check_rules` 这里是 stub，s04 之后会演化成完整的 hook 链。
- `ask_user` 在 CLI 里就是 input prompt；在 GUI 里是按钮；在 CI 里通常返回 deny。

## 相关概念

- [[02 - Tool Use]]：permission 拦的是工具调用，工具的存在是前提。
- [[04 - Hooks]]：s04 会把 permission 重构成 PreToolUse hook，让它和日志、截断等逻辑平级。
- [[06 - Subagent]]：子 Agent 也要跑同一套 permission，否则它绕开主 Agent 的安全网。

> [!warning]
> 几个容易踩的坑：
>
> 1. **静默丢弃被拒工具**：忘了塞 tool_result，API 直接报错"missing tool_use_id"。
> 2. **deny list 用 `==` 而不是 `in`**：模型可能写 `rm -rf /home`，`==` 拦不住。
> 3. **把权限交给模型**：SYSTEM prompt 里写"不要执行危险操作"不靠谱，必须在代码里拦。
> 4. **没有 ask 选项**：只能 allow / deny 会让用户体验很差——灰色地带全靠用户改配置，没人愿意改。
