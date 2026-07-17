# 01 - Skill 加载（SKILL.md）

> 来源：Claude Code 官方的 `~/.claude/skills/<name>/SKILL.md` + `learn-claude-code/s07_skill_loading/`
>
> [[07 - Skill Loading]] 启动时扫描 skills 目录，把每个 SKILL.md 的 frontmatter 注册成可用技能。**技能不是工具**——技能是"打包的提示词 + 工作流"，让 Claude Code 知道"遇到 X 类问题时该这么干"。

## 目录结构

```
~/.claude/skills/
├── code-reviewer/
│   ├── SKILL.md              ← 技能定义（frontmatter + body）
│   ├── review-checklist.md   ← 技能资源（被 body 引用）
│   └── examples/
│       └── sample-review.md
├── commit-helper/
│   └── SKILL.md
└── refactor-guide/
    └── SKILL.md
```

---

## SKILL.md 文件格式

```markdown
---
name: code-reviewer
description: Reviews code changes for bugs, security issues, and best practices. Use when user asks to review PR, code changes, or wants feedback on quality.
allowed_tools:
  - Read
  - Grep
  - Glob
---

# Code Reviewer Skill

When activated, follow this workflow:

1. **Read all changed files** in the PR/diff
2. **Check for bugs**: logic errors, off-by-one, null dereferences
3. **Check for security**: SQL injection, XSS, hardcoded secrets
4. **Check for best practices**: naming, error handling, DRY violations

## Output Format

For each issue found:
- **File**: `path/to/file.py:42`
- **Severity**: Critical / Warning / Suggestion
- **Issue**: What's wrong
- **Fix**: Concrete suggestion

## References

See [review-checklist.md](./review-checklist.md) for the full checklist.
```

---

## frontmatter 字段

| 字段 | 必填 | 含义 |
|---|---|---|
| `name` | ✓ | 技能 ID（目录名一致） |
| `description` | ✓ | **触发条件**——Claude 看到这个判断"该激活吗" |
| `allowed_tools` | 可选 | 该技能能调哪些工具（白名单） |

**关键**：`description` 决定技能何时被激活。Claude 会在每轮把所有技能的 description 喂给模型，模型判断"现在用哪个"。

---

## 实际工作流

```
用户：帮我 review 这个 PR
       ↓
Claude 看到所有技能的 description：
  - code-reviewer: "...Use when user asks to review PR..."
  - commit-helper: "...Use when user wants to commit..."
  - refactor-guide: "...Use when user wants to refactor..."
       ↓
Claude 判断：code-reviewer 匹配
       ↓
激活 code-reviewer：把 SKILL.md 的 body 加进 system prompt
       ↓
Claude 按 body 描述的工作流执行（读文件 → 检查 bug → 检查安全 → 输出）
```

---

## 跟工具的差异

| 维度 | 工具（tool） | 技能（skill） |
|---|---|---|
| 形式 | Python/JS 函数 | markdown 提示词 |
| 调用 | 模型显式 tool_use | 模型隐式遵循 body 指令 |
| 输出 | 字符串返回 | 没有"返回"——按指令改 prompt |
| 复用 | 跨任务通用 | 任务特定（review / commit / refactor） |

**类比**：工具是"手"（动作），技能是"剧本"（流程）。

---

## 改造成 PuinClaw 时

PuinClaw 可以加自定义 skill：

```
~/.claude/skills/puinclaw-dev/
└── SKILL.md
    ---
    name: puinclaw-dev
    description: Helps with PuinClaw development. Use when user asks about agent_loop, lanes, or workspace config.
    ---
    
    # PuinClaw Dev Skill
    
    When activated:
    1. Read PUINCLAW-SETUP.md for current state
    2. Read packages/mom/src/agent.ts for agent core
    3. Cross-reference with claw0 design (Sessions/Channels/Intelligence)
```

---

## 相关

- [[07 - Skill Loading]] —— SkillsManager 怎么扫描 / 注册 / 激活
- [[10 - System Prompt]] —— 技能激活后 body 加进 prompt 哪一层
