# 位置编码概念笔记改造设计

## 目标

将当前最新版《Transformer 位置编码：从绝对位置到 RoPE》改造成符合“概念笔记极简模板”的独立概念笔记，并放入仓库：

```text
02-笔记文档/大模型基础/结构/位置编码--从绝对位置到RoPE.md
```

完成后提交并直接推送到 `origin/main`。

## 内容来源

- 以工作目录中的最新版 `Transformer 位置编码：从绝对位置到 RoPE.md` 为正文基线，保留用户已有的小改动。
- 参考模板为 `jiaran-king/Re-Zero---Starting-LLM-/06-模板/概念笔记极简模板.md`。
- 标题下保留原小红书笔记链接，明确参考出处。

## 文档结构

1. YAML 元数据：`type`、`status`、`domain`、`created`、`updated`、`aliases`、`tags`。
2. 一级标题与参考出处。
3. `[!note]`：用一段话说明位置编码是什么、解决什么问题以及为什么值得学习。
4. 正文：按理解顺序保留必要内容，不为了形式强行拆分过多小节：
   - Self-Attention 为什么需要位置信息。
   - Learned PE 与 Sinusoidal PE。
   - Relative Position Bias、ALiBi 与 RoPE。
   - RoPE 的关键公式及其如何把相对位置写入 Q/K 点积。
   - YaRN、DCA、ABF 等长上下文扩展的定位与对比。
5. `相关概念`。
6. `[!warning]`：集中说明最容易混淆或被原笔记过度简化的结论。

## 写作约束

- 采用方案 B：保持模板简洁，但不删除理解 RoPE 所需的公式、机制和对比表。
- 不上传七张小红书原图，不创建图片目录，不保留原图索引。
- 不使用依赖原图才能理解的表达。
- 不把笔记扩写成源码教程，不新增 PyTorch 实现。
- 数学公式使用仓库现有的 GitHub Markdown 兼容格式。
- 不修改其他笔记，不处理仓库中无关的 `.DS_Store`。

## 仓库变更范围

最终正文阶段只新增：

```text
02-笔记文档/大模型基础/结构/位置编码--从绝对位置到RoPE.md
```

本设计文档另存于：

```text
docs/superpowers/specs/2026-07-22-position-encoding-concept-note-design.md
```

不更新主题索引或 README，除非后续得到用户明确授权。

## 验收条件

- 文件路径和名称与设计完全一致。
- YAML 字段完整，日期为 `2026-07-22`。
- 参考出处存在且链接正确。
- 不包含任何本地图片引用或原图索引。
- 代码围栏和数学定界符成对。
- Markdown 中不含损坏的 LaTeX 命令或不可见控制字符。
- Git 暂存与提交不包含 `.DS_Store` 或其他无关改动。
- 推送后本地 `HEAD` 与 `origin/main` 一致。
