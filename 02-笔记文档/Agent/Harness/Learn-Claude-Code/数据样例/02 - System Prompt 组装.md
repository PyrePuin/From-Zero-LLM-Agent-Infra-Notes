# 02 - System Prompt 组装（8 段拼接）

> 来源：`learn-claude-code/s10_system_prompt/code.py` 动态拼装
>
> [[10 - System Prompt]] 不写死 prompt，而是**根据当前 context 动态选 8 个段落**拼成完整 system prompt。每段长度和顺序都按"当前需要"调整。

## 8 个段落总览

```
[Tool Results]              ← 最大但最不重要，放最前（容易被压缩）
[Tool Definitions]          ← 工具 schema
[Environment]               ← OS / cwd / shell
[Skills]                    ← 当前激活的技能
[Preferences]               ← 用户偏好
[Memories]                  ← 长期记忆
[Agent Instructions]        ← 核心身份和行为准则
[Output Format]             ← 输出格式约定
```

**顺序逻辑**：信息密度低、易压缩的放前面；核心指令放后面（更靠近用户消息，影响力更强）。

---

## 完整 system prompt 示例

```markdown
# Tool Results
[Previous bash output from grep command - 500 lines of search results...]
[Previous Read output from agent.ts - 200 lines...]

# Tool Definitions

## read_file
Description: Read file contents...
Parameters:
  - path (string, required): Absolute path to file

## write_file
Description: Write content to file...
Parameters:
  - path (string, required)
  - content (string, required)

## bash
Description: Execute shell command...
Parameters:
  - command (string, required)
  - timeout (number, optional)

# Environment
- OS: macOS 14.5
- Working directory: /Users/puin/puinclaw
- Shell: zsh
- Git branch: main

# Skills
The following skills are available:
- code-reviewer: Reviews code for bugs and security
- commit-helper: Helps with git commits
- puinclaw-dev: PuinClaw development assistant

# Preferences
User preferences (from settings):
- Language: Chinese for casual chat, English for code
- Code style: TypeScript with explicit types
- Commit format: Conventional Commits

# Memories
Long-term facts from past sessions:
- Project: math grading system using TypeScript
- User prefers terse responses
- Never regress scorer.py without re-running evaluation

# Agent Instructions
You are Claude Code, an AI assistant integrated into a development environment.

Help the user with software engineering tasks. When given a choice, prefer:
1. Reading files over assuming content
2. Editing existing files over creating new
3. Concrete code over abstract suggestions

When uncertain about scope, ask. When certain, act.

# Output Format
- Use markdown for formatting
- Lead with the answer, elaborate if asked
- For code changes, show diff before applying
```

---

## 段落选择逻辑（动态）

不是每次都把 8 段全拼上。根据 context 选择性加载：

```python
# learn-claude-code s10 的伪代码
def build_system_prompt(context: Context) -> str:
    sections = []

    # Tool Results —— 只在有上一轮 tool 调用时加
    if context.has_previous_tool_results:
        sections.append(format_tool_results(context))

    # Tool Definitions —— 总是加
    sections.append(format_tools(context.available_tools))

    # Environment —— 总是加
    sections.append(format_env(context))

    # Skills —— 只在用户消息暗示技能时加
    relevant_skills = match_skills(context.user_message)
    if relevant_skills:
        sections.append(format_skills(relevant_skills))

    # Preferences —— 总是加（用户偏好稳定）
    sections.append(format_preferences(context.user))

    # Memories —— 用 LLM side-query 选相关记忆
    relevant_memories = side_query_memory(context.user_message)
    if relevant_memories:
        sections.append(format_memories(relevant_memories))

    # Agent Instructions —— 总是加（核心）
    sections.append(AGENT_INSTRUCTIONS_TEMPLATE)

    # Output Format —— 总是加
    sections.append(OUTPUT_FORMAT_TEMPLATE)

    return "\n\n".join(sections)
```

---

## 跟 claw0 [[06 - Intelligence]] 的对比

| 维度 | learn-claude-code s10 | claw0 s06 |
|---|---|---|
| 数据源 | sections 写死在代码里 | 从 workspace/*.md 读 |
| 组装 | 按 context **动态选** section | 固定 8 层顺序，每轮全拼 |
| Memory 层 | LLM side-query（贵但准） | TF-IDF + hash（便宜但粗） |
| 多 agent | 单 agent | 每 agent 独立 workspace |

详见 [[06 - Intelligence|claw0 s06 vs learn-claude-code s10]]。

---

## token 预算分配

典型 system prompt ~2000-4000 tokens，分配大致：

| 段落 | token 占比 | 备注 |
|---|---|---|
| Tool Results | 30-50% | 历史 tool 输出（最长） |
| Tool Definitions | 15-20% | 工具 schema |
| Agent Instructions | 10-15% | 核心身份 |
| Environment | 5% | 简短 |
| Skills | 0-15% | 激活才有 |
| Preferences | 5% | 简短 |
| Memories | 0-10% | 选中的才有 |
| Output Format | 5% | 简短 |

**关键观察**：Tool Results 占大头但最不重要——这就是 [[08 - Context Compact|context compact]] 优先压缩它的原因。

---

## 相关

- [[10 - System Prompt]] —— 8 段选择逻辑、动态组装
- [[08 - Context Compact]] —— prompt 太长时怎么压缩
- [[09 - Memory]] —— Memories 段怎么选相关记忆
- [[07 - Skill Loading]] —— Skills 段怎么决定激活哪个
