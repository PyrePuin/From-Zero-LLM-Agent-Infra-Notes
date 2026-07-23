# Repository Instructions

These instructions apply to the entire repository.

## Repository Purpose

- This repository contains Chinese Markdown learning notes about large-model training, LLM Agents, and infrastructure.
- Prefer clear, self-contained explanations. A reader should not need the source article or source images to understand a finished note.
- Preserve the established directory structure, naming style, YAML frontmatter, Markdown callouts, tables, Mermaid diagrams, and relative links.
- When a task names a template or source file, read its latest version before editing.

## Change Discipline

- Run `git status -sb` before editing.
- Existing or unrelated changes belong to the user. Do not rewrite, stage, commit, or delete them.
- Use explicit file paths with `git add`; do not use `git add -A` in a mixed worktree.
- Never commit `.DS_Store`.
- Run `git diff --check` and inspect the staged file list before every commit.

## GitHub Markdown Math

GitHub rendering compatibility is required for every Markdown note under `02-笔记文档/`.

### Inline Math

Use GitHub's backtick-delimited inline form:

```text
$`x_i=W_i h`$
```

Use the same form inside tables. Do not use bare `$...$` for inline math in this repository.

### Block Math

Use a fenced `math` block:

````markdown
```math
y=Wx+b
```
````

Do not use `$$...$$`.

### Compatible Commands

Prefer simple KaTeX commands already used successfully in the repository, including:

- Fractions and roots: `\frac`, `\sqrt`.
- Sums and limits: `\sum`, `\prod`, subscripts, and superscripts.
- Delimiters: `\left`, `\right`, `\lVert`, `\rVert`.
- Text styles: `\mathrm`, `\mathsf`, `\mathbf`, `\text` when GitHub renders the specific usage.
- Greek letters and basic operators such as `\theta`, `\phi`, `\omega`, `\times`, `\cdot`, and `\le`.

Keep commands minimal. If ordinary symbols express the same idea, prefer the simpler expression.

### Safe Subscript Notation

Do not place a raw comparison character at the beginning of a braced subscript. GitHub's math translation layer can misparse forms such as:

```text
y_{<n}
y_{>n}
```

and report `Extra open brace or missing close brace`, even though other LaTeX renderers may accept them.

Prefer an explicit index range when it expresses the same meaning:

```text
y_{1:n-1}
```

If a comparison relation is essential, use a tested command such as `\lt` or `\gt` instead of a raw leading `<` or `>`. Keep the explicit index-range form as the repository default.

### Forbidden Commands and Environments

Do not use these commands:

```text
\operatorname
\tag
\label
\ref
\newcommand
\def
```

Do not use LaTeX environments introduced with `\begin{...}`. In particular, avoid:

```text
aligned
cases
matrix
bmatrix
pmatrix
array
```

These constructs have caused GitHub rendering or translation-layer failures in this repository.

Use these replacements:

- Replace `\operatorname{score}` with `\mathrm{score}` or a plain symbol such as `s`.
- Split an `aligned` derivation into multiple `math` blocks.
- Replace `cases` with prose conditions followed by one formula per condition.
- Replace a matrix environment with compact row/column notation or explicit component equations.

## Markdown Verification

Before committing a note, scan all note documents for unsupported syntax:

```bash
rg -n '^\$\$$|\\operatorname|\\tag|\\label|\\ref|\\newcommand|\\def|\\begin\{' '02-笔记文档'
```

Expected result: no output.

Also scan for raw comparison characters at the start of braced subscripts:

```bash
rg -n '_\{[<>]' '02-笔记文档'
```

Expected result: no output. Rewrite matches with an explicit index range such as `y_{1:n-1}`, or use a tested `\lt` / `\gt` relation when the range form would change the meaning.

Check every Markdown file independently for paired code fences and inline-math delimiters:

```bash
while IFS= read -r file; do
  fences=$(rg -c '^```' "$file" || true)
  opens=$(rg -o '\$`' "$file" | wc -l | tr -d ' ')
  closes=$(rg -o '`\$' "$file" | wc -l | tr -d ' ')
  test $((fences % 2)) -eq 0 || { echo "unpaired fence: $file"; exit 1; }
  test "$opens" -eq "$closes" || { echo "unpaired inline math: $file"; exit 1; }
done < <(rg --files '02-笔记文档' -g '*.md')
```

Then verify Git scope:

```bash
git diff --check
git status --short
git diff --cached --name-only
```

The staged list must contain only task-approved files and must not contain `.DS_Store`.
