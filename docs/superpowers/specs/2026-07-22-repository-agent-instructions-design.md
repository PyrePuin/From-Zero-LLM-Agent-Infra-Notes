# 仓库代理写作规范设计

## 目标

在仓库根目录新增 `AGENTS.md`，为后续编辑学习笔记的代理提供可自动发现的仓库级规则，重点固化 GitHub Markdown 数学公式规范，避免再次引入无法渲染或容易被翻译层破坏的 LaTeX。

同时修复当前位置编码笔记中仍使用的 `bmatrix` 环境，使新增规则与现有正文保持一致。

## 产物

- 新建：`AGENTS.md`
- 修改：`02-笔记文档/大模型基础/结构/位置编码--从绝对位置到RoPE.md`
- 新建设计文档：`docs/superpowers/specs/2026-07-22-repository-agent-instructions-design.md`

## AGENTS.md 内容范围

### 仓库与变更纪律

- 本仓库以中文 Markdown 学习笔记为主。
- 编辑前检查 `git status`，保留不属于当前任务的用户改动。
- 只暂存任务相关文件，不提交 `.DS_Store`。
- 新笔记遵循仓库既有 YAML、标题、Callout、相对链接与目录命名习惯。

### GitHub 数学公式规范

- 行内公式统一使用 GitHub 的 ``$`...`$`` 形式。
- 块公式统一使用 `math` fenced code block，不使用 `$$...$$`。
- 常见基础命令可以使用，例如 `\frac`、`\sqrt`、`\sum`、`\mathrm`、`\mathsf`、`\left`、`\right` 和希腊字母。
- 禁止 `\operatorname`、`\tag`、`\label`、`\ref`、`\newcommand`、`\def`。
- 避免所有 `\begin{...}` LaTeX environment，尤其是 `aligned`、`cases`、`matrix`、`bmatrix`；多行公式拆成多个 `math` 块，分段函数拆成自然语言条件加单式公式，矩阵使用紧凑向量/行列表达或直接写分量公式。
- 表格中的公式同样使用 ``$`...`$``。

### 提交前验证

- 扫描 `$$`、禁用宏和 `\begin{...}`。
- 检查 Markdown 代码围栏为偶数。
- 检查 ``$``` 与 ```$`` 数量相等。
- 运行 `git diff --check`。
- 检查暂存文件列表，排除 `.DS_Store` 和无关文件。

## 当前位置编码笔记修复

删除二维旋转矩阵的 `\begin{bmatrix}...\end{bmatrix}` 写法。保留旋转含义，但改为分别写出二维分量：

```math
x'_{2i}=x_{2i}\cos\phi_i(p)-x_{2i+1}\sin\phi_i(p)
```

```math
x'_{2i+1}=x_{2i}\sin\phi_i(p)+x_{2i+1}\cos\phi_i(p)
```

其余正文和公式不改。

## 非目标

- 不增加 GitHub Actions 或独立检查脚本。
- 不批量改写仓库其他笔记。
- 不调整目录、README、模板或图片。

## 验收条件

- 根目录存在 `AGENTS.md`，规则对全仓库生效。
- 规范明确覆盖行内公式、块公式、允许/禁止宏、LaTeX environment 与验证命令。
- `02-笔记文档` 中不存在 `\operatorname`、`$$` 或 `\begin{...}`。
- 位置编码笔记不再依赖 `bmatrix`，数学语义不变。
- 代码围栏和行内公式定界符成对。
- 提交不包含 `.DS_Store` 或其他无关文件。
- 推送后本地 `HEAD` 与 `origin/main` 一致。
