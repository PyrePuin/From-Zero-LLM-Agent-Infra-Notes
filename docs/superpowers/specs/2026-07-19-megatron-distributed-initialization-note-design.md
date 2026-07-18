# Megatron 分布式环境初始化笔记设计

## 目标

将 `https://zhuanlan.zhihu.com/p/629121480` 整理为本仓库“分布式训练与并行技术”主题下的第 5 篇独立教程。读者只阅读本地笔记即可理解 Megatron 初始化主链路、DP/TP/PP/MP/Embedding 分组、rank 排列与 ZeRO-R 的作用。

## 产物

- 新建 `02-笔记文档/大模型分布式训练与并行技术/05-Megatron源码解读1--分布式环境初始化.md`。
- 使用 Mermaid 与文本结构图重绘分组关系，不依赖受访问限制的外部配图。
- 将主题 `README.md` 中第 5 条改成本地笔记链接，并保留来源链接。

## 内容结构

1. 初始化解决的问题及预训练主链路。
2. `global rank`、节点内 `local rank`、subgroup rank 的区别。
3. 用 `world_size=16、TP=2、PP=4、DP=2` 建立三维坐标和 rank 布局。
4. 分模块讲解 `initialize_model_parallel()`：组数计算、Virtual PP、DP、MP、TP、PP、Embedding group。
5. 解释每个进程为什么执行全部 `new_group()`，但只保存自己所属的 group 句柄。
6. 解释 ZeRO-R 为什么沿 TP 维度消除冗余激活。
7. QA 收录本轮学习中的关键问题。

## 正确性约束

- MP 是 TP 与 PP 共同构成完整模型副本的逻辑范围，不描述成第四种独立切分算法。
- 普通 DP 复制对应参数分片并同步梯度；只有 ZeRO 才进一步沿 DP 维度分片状态。
- 修正文中第二个 MP group 的 rank 笔误，使用 `[2,3,6,7,10,11,14,15]`。
- `_DATA_PARALLEL_GROUP` 是当前进程所属的一个 group 句柄，不是全部 DP 组列表。
- `get_tensor_model_parallel_rank()` 返回 TP subgroup rank，避免与节点内 `local_rank` 混淆。
- ZeRO-R 只针对 TP 聚合后形成的冗余激活，不泛化为所有 TP 中间量。

## 写作与素材约束

- 采用独立教程口吻，不使用“文章给出”“原文介绍”等依赖性措辞。
- 保留关键源码，避免逐行翻译；以执行链路和 rank 推导组织内容。
- 不建立插图索引。
- Markdown 公式使用 GitHub 可渲染的 `$...$` 与 `$$...$$`。
- 分组与调用链使用仓库内可直接渲染的 Mermaid 或文本图，不创建空素材目录。

## 验收

- 本地 Markdown 与 README 链接存在。
- 公式分隔符与代码围栏成对。
- Mermaid 围栏、普通代码围栏与公式分隔符成对。
- 不包含已知错误和依赖性措辞。
- Git 暂存范围只包含本篇笔记、素材、README、设计与计划文件。
