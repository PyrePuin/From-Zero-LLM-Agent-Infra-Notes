# Repository Agent Instructions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add repository-wide agent instructions for GitHub-compatible Markdown math and bring the position-encoding note into compliance.

**Architecture:** A root `AGENTS.md` defines authoring and verification rules for every note. The only existing violation, a `bmatrix` environment in the position-encoding note, is replaced with two scalar component equations.

**Tech Stack:** Markdown, GitHub math fences, Git, shell static checks.

## Global Constraints

- Create root `AGENTS.md`.
- Modify only `02-笔记文档/大模型基础/结构/位置编码--从绝对位置到RoPE.md` for content compliance.
- Do not add CI or a standalone script.
- Do not stage `.DS_Store`.
- Push directly to `origin/main`.

---

### Task 1: Add repository-wide instructions

**Files:**
- Create: `AGENTS.md`

**Interfaces:**
- Consumes: existing repository conventions established by commits `135b676`, `798c2b4`, `8f95bd5`, and `b9ac450`.
- Produces: instructions automatically discoverable by repository agents.

- [ ] **Step 1: Write `AGENTS.md`**

Include repository language and scope, safe Git discipline, note conventions, inline and block math syntax, allowed commands, forbidden commands and environments, replacement patterns, and exact verification commands.

- [ ] **Step 2: Verify required rules exist**

Assert that `AGENTS.md` contains all of: ``$`...`$``, `math` fences, `\operatorname`, `\begin{`, `.DS_Store`, `git diff --check`, and staged-file inspection.

### Task 2: Remove the remaining LaTeX environment

**Files:**
- Modify: `02-笔记文档/大模型基础/结构/位置编码--从绝对位置到RoPE.md`

**Interfaces:**
- Consumes: the current two-dimensional RoPE rotation matrix.
- Produces: equivalent component equations without `bmatrix`.

- [ ] **Step 1: Verify the compatibility check fails before editing**

Run a scan for `\begin{` under `02-笔记文档`; it must fail on the position-encoding note's `bmatrix`.

- [ ] **Step 2: Replace the matrix environment**

Replace the matrix block with:

```math
x'_{2i}=x_{2i}\cos\phi_i(p)-x_{2i+1}\sin\phi_i(p)
```

and:

```math
x'_{2i+1}=x_{2i}\sin\phi_i(p)+x_{2i+1}\cos\phi_i(p)
```

- [ ] **Step 3: Verify the compatibility check passes**

Assert that all `02-笔记文档` files contain zero `$$`, zero `\begin{`, and zero forbidden macros; confirm code fences and inline-math delimiters are paired.

### Task 3: Commit and publish

**Files:**
- Include: `AGENTS.md`
- Include: `02-笔记文档/大模型基础/结构/位置编码--从绝对位置到RoPE.md`
- Include: `docs/superpowers/plans/2026-07-22-repository-agent-instructions.md`

**Interfaces:**
- Consumes: Tasks 1 and 2.
- Produces: one focused commit on `main`, synchronized with `origin/main`.

- [ ] **Step 1: Run `git diff --check` and inspect exact scope**
- [ ] **Step 2: Stage only the three approved files and commit**
- [ ] **Step 3: Push `main`, fetch it, and verify local `HEAD` equals `origin/main`**
