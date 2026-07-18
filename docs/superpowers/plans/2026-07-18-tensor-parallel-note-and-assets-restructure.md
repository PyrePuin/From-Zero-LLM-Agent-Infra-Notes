# 张量并行笔记与配图目录重构实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 新增独立的 Megatron-LM 张量并行教程，统一前四篇文件名，并将所有配图集中迁移到主题目录唯一的 `assets/` 目录。

**Architecture:** 保留四篇 Markdown 平铺在主题根目录，所有图片进入 `assets/<对应文章名>/`。先完成可追踪的机械迁移，再更新引用和写入第四篇，最后用静态检查覆盖文件名、链接、图片、公式和控制字符。

**Tech Stack:** Markdown、Git、Shell 静态检查、知乎页面正文与本地 JPEG 资源。

## Global Constraints

- 四篇文件名必须与批准的名称完全一致。
- 主题根目录只保留一个 `assets/` 配图入口。
- 第四篇必须是可独立阅读的教程，并在文末收录本轮 QA。
- 不创建插图索引，不使用依赖来源文章的叙述口吻。
- 不修改已有图片二进制内容。
- 最终直接推送到 `origin/main`。

---

### Task 1: 收集第四篇正文结构和图片

**Files:**
- Inspect: `02-笔记文档/大模型分布式训练与并行技术/README.md`
- Create: `02-笔记文档/大模型分布式训练与并行技术/assets/04-张量模型并行--Megatron-LM/*.jpg`

**Interfaces:**
- Consumes: 知乎文章 `https://zhuanlan.zhihu.com/p/622212228`
- Produces: 第四篇章节结构、公式口径和本地配图文件。

- [ ] **Step 1: 提取文章的标题层级、正文段落、公式语义和图片 URL**

使用已登录的浏览器读取精确 `article` 容器，记录行/列切分、MLP、Attention、Embedding、Cross Entropy、TP+DP 和实验章节。

- [ ] **Step 2: 下载并稳定命名文章图片**

图片命名使用内容语义，例如：

```text
00-weight-matmul.jpg
01-row-parallel-forward.jpg
02-row-parallel-backward.jpg
03-column-parallel-forward.jpg
04-column-parallel-backward.jpg
05-mlp-tensor-parallel.jpg
06-attention-tensor-parallel.jpg
07-vocab-parallel-embedding.jpg
08-vocab-parallel-cross-entropy.jpg
09-tp-dp-layout.jpg
```

- [ ] **Step 3: 验证图片文件有效**

Run:

```bash
find '02-笔记文档/大模型分布式训练与并行技术/assets/04-张量模型并行--Megatron-LM' -type f -name '*.jpg' -exec file {} +
```

Expected: 每个文件均识别为 JPEG image data，且文件数与选取的正文图片数一致。

### Task 2: 统一前三篇文件和配图目录

**Files:**
- Rename: `01-流水线并行技术.md` → `01-流水线并行技术--以Gpipe为例.md`
- Rename: `02-数据并行技术.md` → `02-数据并行技术--DP与DDP.md`
- Rename: `03-ZeRO零冗余优化技术.md` → `03-数据并行技术--ZeRO与零冗余优化.md`
- Move: 三个旧 `*.assets/` 目录中的图片 → `assets/<文章名>/`

**Interfaces:**
- Consumes: 三篇已有 Markdown 与 40 张本地图片。
- Produces: 批准的统一文件名和单一配图入口。

- [ ] **Step 1: 创建四个文章配图子目录**

Run:

```bash
mkdir -p \
  '02-笔记文档/大模型分布式训练与并行技术/assets/01-流水线并行技术--以Gpipe为例' \
  '02-笔记文档/大模型分布式训练与并行技术/assets/02-数据并行技术--DP与DDP' \
  '02-笔记文档/大模型分布式训练与并行技术/assets/03-数据并行技术--ZeRO与零冗余优化' \
  '02-笔记文档/大模型分布式训练与并行技术/assets/04-张量模型并行--Megatron-LM'
```

- [ ] **Step 2: 移动前三篇图片并重命名 Markdown**

使用明确的源目录和目标目录执行 `mv`，不使用通配符删除；迁移后只对确认已空的旧目录执行 `rmdir`。

- [ ] **Step 3: 检查 Git 将迁移识别为重命名**

Run:

```bash
git status --short
git diff --summary
```

Expected: 只有批准范围内的 Markdown、图片和后续索引变更。

### Task 3: 更新前三篇引用与主题索引

**Files:**
- Modify: `01-流水线并行技术--以Gpipe为例.md`
- Modify: `02-数据并行技术--DP与DDP.md`
- Modify: `03-数据并行技术--ZeRO与零冗余优化.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: Task 2 的最终路径。
- Produces: 全部指向 `./assets/<文章名>/...` 的有效相对链接。

- [ ] **Step 1: 更新三篇一级标题和图片路径**

使用 `apply_patch` 将旧标题改为批准名称，并将：

```text
./01-流水线并行技术.assets/
./02-数据并行技术.assets/
./03-ZeRO零冗余优化技术.assets/
```

分别替换为对应的 `./assets/<文章名>/`。

- [ ] **Step 2: 更新主题 README 的前四篇链接**

README 的基础并行策略区必须精确链接四个新文件，并保留来源链接。

- [ ] **Step 3: 扫描旧路径残留**

Run:

```bash
rg -n '01-流水线并行技术\.assets|02-数据并行技术\.assets|03-ZeRO零冗余优化技术\.assets|01-流水线并行技术\.md|02-数据并行技术\.md|03-ZeRO零冗余优化技术\.md' .
```

Expected: 无旧路径残留。

### Task 4: 编写第四篇独立教程

**Files:**
- Create: `02-笔记文档/大模型分布式训练与并行技术/04-张量模型并行--Megatron-LM.md`

**Interfaces:**
- Consumes: Task 1 的正文结构与图片、已完成的学习问答。
- Produces: 可独立阅读的第四篇教程。

- [ ] **Step 1: 写入元数据、总览和统一符号**

定义：

```text
b: batch size
s: sequence length
h: hidden size
h': MLP intermediate size
V: vocabulary size
n: tensor parallel size
```

- [ ] **Step 2: 写清线性层切分和 MLP 主链路**

正文必须包含：

```text
列切：W=[W1,...,Wn]，forward 输出自然分片，backward 的 dX 求和。
行切：W=[W1;...;Wn]，forward 部分和求和，backward 的 dX 自然分片。
MLP：A 列并行 → 本地 GELU → B 行并行 → All-Reduce。
形状：(b,s,h) → (b,s,h'/n) → (b,s,h)。
```

- [ ] **Step 3: 写清 Attention、Embedding 和 Vocabulary Parallel Cross Entropy**

Cross Entropy 必须从普通公式开始，依次说明局部 logits、全局 max、全局 exp sum、目标 logit 和 loss，形状为：

```text
(b,s,h) → (b,s,V/n) → (b,s) → scalar
```

- [ ] **Step 4: 写清通信量和混合并行**

区分逻辑张量量与 Ring All-Reduce 实际单卡发送量：

```text
一次 Ring All-Reduce：2(n-1)/n * Phi
MLP forward + backward：4(n-1)/n * Phi
```

说明 PP 决定 layer 归属，TP 决定 layer 内 tensor shard，DP 复制相同 PP+TP 分片，ZeRO/Distributed Optimizer 在 DP 组内继续分片状态。

- [ ] **Step 5: 收录 QA**

QA 至少覆盖本轮已讨论的：

```text
行切/列切的 backward 与单卡有何不同
MLP 是否需要恢复完整中间激活
为什么最后是 All-Reduce 而不是 All-Gather
MLP 通信量为什么统计为 4Phi
Cross Entropy 的词表并行与形状变化
Megatron 是否只做 layer 内 tensor 分片
Megatron 与 ZeRO-3 如何组合
PP 是否同时切分参数、梯度和优化器状态
```

### Task 5: 全量验证并发布

**Files:**
- Verify: `02-笔记文档/大模型分布式训练与并行技术/**/*.md`
- Verify: `02-笔记文档/大模型分布式训练与并行技术/assets/**/*`

**Interfaces:**
- Consumes: Tasks 1–4 的完整工作树。
- Produces: 通过静态检查并推送到 `origin/main` 的提交。

- [ ] **Step 1: 验证文件结构**

Run:

```bash
find '02-笔记文档/大模型分布式训练与并行技术' -maxdepth 2 -print | sort
```

Expected: 四篇批准名称的 Markdown、README 和唯一 `assets/` 入口；无 `*.assets/` 目录。

- [ ] **Step 2: 验证图片与 Markdown 链接**

从每个 Markdown 提取相对图片和文档链接，解析到文件系统路径；任何缺失路径都令检查失败。

- [ ] **Step 3: 验证 Markdown 与公式**

检查：

```text
git diff --check
代码围栏数量为偶数
块公式 $$ 定界符为偶数
不存在 ASCII 控制字符
不存在损坏的 rac{ 公式命令
不存在“文章给出|文章介绍|原文给出|原文介绍”
第四篇恰好存在一个 ## QA
不存在插图索引标题
```

- [ ] **Step 4: 检查提交范围并提交**

Run:

```bash
git status --short
git diff --stat
git diff --check
```

Expected: 仅包含设计/计划、四篇笔记、README 和统一 assets 迁移。

Commit:

```bash
git add -- <本次明确路径>
git commit -m 'docs: add Megatron tensor parallel notes'
```

- [ ] **Step 5: 推送并核对远端**

Run:

```bash
git push origin main
git rev-parse HEAD
git rev-parse origin/main
git status --short --branch
```

Expected: 本地 HEAD 与 `origin/main` 一致，工作区干净。
