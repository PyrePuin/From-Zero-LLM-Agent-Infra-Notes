---
type: concept
status: active
domain: 推理与部署
created: 2026-07-22
updated: 2026-07-22
aliases: [大模型资源估算, 参数量估算, FLOPs 估算, 显存估算]
tags: [LLM, Transformer, 参数量, FLOPs, 显存, KV-Cache]
---

# 大模型资源估算：参数量、FLOPs 与显存

> 参考出处：[小红书笔记](https://www.xiaohongshu.com/explore/69de59a3000000001a0243ce?xsec_token=AB2bCSk3JM9jM34YLxzsvdMVQ3808IQRCENomwXLyVdvM=&xsec_source=pc_user)

> [!note]
> 本文以 decoder-only Transformer 为例，从参数量、计算量和显存占用三个角度建立资源估算框架。公式以标准 MHA 和普通两层 FFN 为基线，并补充 SwiGLU、GQA/MQA、KV Cache 等现代变体。

## 1. 全局框架

| 层级 | 主题 | 核心问题 |
| --- | --- | --- |
| 第一层 | 参数量 | Embedding、Attention、FFN、Norm 和 LM Head 各有多少参数？ |
| 第二层 | 计算量 | 前向、训练、Prefill 和 Decode 分别需要多少 FLOPs？ |
| 第三层 | 显存占用 | 训练时的参数、梯度、优化器状态、激活，以及推理时的权重和 KV Cache 各占多少显存？ |

### 1.1 统一符号

| 符号 | 含义 |
| --- | --- |
| $`V`$ | 词表大小（vocabulary size） |
| $`d`$ | 隐藏维度（hidden size） |
| $`L`$ | Transformer Block 层数 |
| $`d_{ff}`$ | FFN 中间维度 |
| $`T`$ | 序列长度或当前上下文长度 |
| $`T_0`$ | 推理时的 Prompt 长度 |
| $`h`$ | Query Head 数 |
| $`n_{kv}`$ | KV Head 数；MHA 中 $`n_{kv}=h`$ |
| $`d_h`$ | 每个 Head 的维度，$`d=h d_h`$ |
| $`B`$ | Batch Size |
| $`N`$ | Decode 阶段新生成的 Token 数 |
| $`P`$ | 模型总参数量 |
| $`b_w,b_g,b_a,b_{kv}`$ | 权重、梯度、激活和 KV Cache 中每个元素的字节数 |

常见精度对应的字节数：FP32 为 4 Byte，FP16/BF16 为 2 Byte，INT8 为 1 Byte，INT4 为 0.5 Byte。

> [!important]
> - 参数量 = 参数元素个数，与 Batch Size 和序列长度无关。
> - FLOPs = 完成计算所需的浮点操作次数。
> - 显存占用 = 元素个数 × 每个元素的字节数。

## 2. 参数量估算

### 2.1 Embedding

Token Embedding 为每个 Token 保存一个 $`d`$ 维向量：

```math
P_{\mathrm{token\_emb}}=Vd
```

只有可学习的绝对位置编码才会额外引入位置参数。设最大长度为 $`T_{max}`$：

```math
P_{\mathrm{pos\_emb}}=T_{max}d
```

RoPE、正弦位置编码和 ALiBi 不需要为每个位置保存独立的可学习向量，因此没有这一项参数。

### 2.2 Self-Attention

标准 MHA 包含 $`W_Q,W_K,W_V,W_O`$ 四个 $`d\times d`$ 线性投影。忽略 Bias：

```math
P_{\mathrm{attn}}=4d^2
```

在 GQA/MQA 中，K/V 的输出维度缩小为 $`n_{kv}d_h`$：

```math
P_{\mathrm{attn}}=2d^2+2d n_{kv}d_h
```

其中两个 $`d^2`$ 分别来自 Q 投影和输出投影。当 $`n_{kv}=h`$ 时，该式退化为标准 MHA 的 $`4d^2`$。

### 2.3 FFN

普通两层 FFN 的投影方向为 $`d\rightarrow d_{ff}\rightarrow d`$：

```math
P_{\mathrm{ffn}}=2dd_{ff}
```

若采用经典设定 $`d_{ff}=4d`$：

```math
P_{\mathrm{ffn}}=8d^2
```

SwiGLU 包含门控投影、上投影和下投影，共三个矩阵：

```math
P_{\mathrm{SwiGLU}}=3dd_{ff}
```

为了让参数量接近普通 FFN，SwiGLU 常将实际中间维度设为约 $`\frac{8}{3}d`$，并根据硬件友好的倍数取整。此时参数量仍约为 $`8d^2`$。

> [!warning]
> 不能把 SwiGLU 直接写成 $`\frac{8}{3}dd_{ff}`$。$`\frac{8}{3}d`$ 描述的是常用的中间维度，而在中间维度已经记作 $`d_{ff}`$ 时，三个投影的参数量应为 $`3dd_{ff}`$。

### 2.4 Norm 与单层 Block

每层若有两个 LayerNorm，Scale 和 Bias 共约 $`4d`$ 个参数；两个 RMSNorm 只有 Scale，共约 $`2d`$ 个参数。它们相对 $`d^2`$ 通常是小项，手算时可以忽略。

标准 MHA 加普通 FFN 的单层参数量约为：

```math
P_{\mathrm{block}}\approx4d^2+2dd_{ff}
```

当 $`d_{ff}=4d`$：

```math
P_{\mathrm{block}}\approx12d^2
```

$`L`$ 层合计：

```math
P_{\mathrm{blocks}}\approx L\left(4d^2+2dd_{ff}\right)
```

### 2.5 LM Head 与总参数量

LM Head 把隐藏状态投影到词表：

```math
P_{\mathrm{lm\_head}}=dV
```

很多模型让 LM Head 与 Token Embedding 共享权重。共享后，LM Head 不再新增参数，但生成词表 Logits 的计算仍然存在。

忽略位置编码、Norm 和 Bias，不共享 LM Head 时：

```math
P_{\mathrm{total}}\approx2Vd+L\left(4d^2+2dd_{ff}\right)
```

当 $`d_{ff}=4d`$：

```math
P_{\mathrm{total}}\approx2Vd+12Ld^2
```

共享 LM Head 时：

```math
P_{\mathrm{total}}\approx Vd+12Ld^2
```

## 3. FLOPs 估算

### 3.1 计数约定

矩阵 $`A\in\mathbb{R}^{m\times n}`$ 与 $`B\in\mathbb{R}^{n\times p}`$ 相乘时，输出共有 $`mp`$ 个元素。每个元素需要约 $`n`$ 次乘法和 $`n`$ 次加法，因此：

```math
F_{\mathrm{matmul}}\approx2mnp
```

这里把一次乘法和一次加法分别计为一个 FLOP。若使用“一次乘加算一个操作”的口径，绝对数值会减半，但各部分之间的比例不变。

下面只统计主要矩阵乘法，忽略 Embedding 查表、Softmax、Norm、激活函数和采样等小项。

### 3.2 单层 Block 的前向 FLOPs

输入形状为 $`B\times T\times d`$。

Q/K/V/O 四个线性投影：

```math
F_{\mathrm{QKVO}}=8BTd^2
```

Attention 分数矩阵 $`QK^{\mathsf T}`$：

```math
F_{QK^{\mathsf T}}=2BT^2d
```

Attention 权重与 V 相乘：

```math
F_{\mathrm{AttnV}}=2BT^2d
```

普通两层 FFN：

```math
F_{\mathrm{FFN}}=4BTdd_{ff}
```

因此，单层 Block 的主要前向计算量为：

```math
F_{\mathrm{block}}=8BTd^2+4BT^2d+4BTdd_{ff}
```

当 $`d_{ff}=4d`$：

```math
F_{\mathrm{block}}=24BTd^2+4BT^2d
```

> [!note]
> 因果 Attention 实际只使用下三角区域。上式按常见的稠密矩阵乘法口径估算，可视为易于手算的上界；具体 Kernel 是否跳过被 Mask 的区域会影响实际 FLOPs。

### 3.3 整个模型的一次前向

包含 $`L`$ 层 Block 和 LM Head：

```math
F_{\mathrm{fwd}}\approx L\left(8BTd^2+4BT^2d+4BTdd_{ff}\right)+2BTdV
```

当 $`d_{ff}=4d`$：

```math
F_{\mathrm{fwd}}\approx L\left(24BTd^2+4BT^2d\right)+2BTdV
```

对于以参数矩阵乘法为主、Attention 二次项不是主导项的模型，也常使用：

```math
F_{\mathrm{fwd}}\approx2PBT
```

### 3.4 训练 FLOPs

对一次矩阵乘法而言，反向传播通常还要分别计算输入梯度和参数梯度，两者计算量都与前向同量级。因此：

```math
F_{\mathrm{backward}}\approx2F_{\mathrm{fwd}}
```

完整训练步骤包含一次前向和一次反向：

```math
F_{\mathrm{train}}\approx3F_{\mathrm{fwd}}
```

结合 $`F_{\mathrm{fwd}}\approx2PBT`$，可得到常见估算：

```math
F_{\mathrm{train}}\approx6PBT
```

> [!warning]
> “训练约为前向的 3 倍”是矩阵乘法主导时的近似。梯度检查点会在反向阶段重算部分前向，使实际计算量进一步增加。

### 3.5 Prefill

Prompt 长度为 $`T_0`$ 时，Prefill 对整个 Prompt 并行执行前向，并建立 KV Cache：

```math
F_{\mathrm{prefill}}\approx L\left(8BT_0d^2+4BT_0^2d+4BT_0dd_{ff}\right)+2BT_0dV
```

当 $`d_{ff}=4d`$：

```math
F_{\mathrm{prefill}}\approx L\left(24BT_0d^2+4BT_0^2d\right)+2BT_0dV
```

Prefill 可以利用 Token 维度上的并行性，但 Attention 仍包含随 Prompt 长度二次增长的项。

### 3.6 Decode 单步

设当前上下文长度为 $`T`$。生成一个新 Token 时，输入形状退化为 $`B\times1\times d`$，历史 K/V 从 KV Cache 中读取。

单层主要计算量为：

```math
F_{\mathrm{block,decode}}=8Bd^2+4BTd+4Bdd_{ff}
```

整个模型单步计算量为：

```math
F_{\mathrm{decode\_step}}\approx L\left(8Bd^2+4BTd+4Bdd_{ff}\right)+2BdV
```

> [!important]
> KV Cache 避免了每一步重新编码全部历史 Token。单步 Decode 的 Attention 从重新计算完整前缀时的二次复杂度降为随当前上下文长度线性增长，但 Attention Scores 仍需在当前 Query 与全部历史 Key 之间现场计算。

### 3.7 连续生成多个 Token

连续生成 $`N`$ 个 Token 时，第 $`i`$ 步的上下文长度为：

```math
T_i=T_0+i-1
```

这些上下文长度之和为：

```math
\sum_{i=1}^{N}T_i=NT_0+\frac{N(N-1)}{2}
```

总 Decode FLOPs 约为：

```math
F_{\mathrm{decode,total}}\approx BLN\left(8d^2+4dd_{ff}\right)+4BLd\sum_{i=1}^{N}T_i+2BNdV
```

完整推理计算量为：

```math
F_{\mathrm{infer}}=F_{\mathrm{prefill}}+F_{\mathrm{decode,total}}
```

单步 Decode 对当前上下文长度是线性的；但连续生成 $`N`$ 个 Token 时，所有步骤累加后仍包含 $`N^2`$ 项。

### 3.8 Scaling 规律

| 变量 | 增长规律 | 说明 |
| --- | --- | --- |
| $`B`$ | $`O(B)`$ | Batch Size 线性放大计算量 |
| $`L`$ | $`O(L)`$ | 层数线性放大计算量 |
| $`d`$ | 主要为 $`O(d^2)`$ | Q/K/V/O 和 FFN 投影通常是主要计算来源 |
| $`T`$ | $`O(T)+O(T^2)`$ | 投影和 FFN 为线性项，完整序列 Attention 为二次项 |
| 单步 Decode 的 $`T`$ | $`O(T)`$ | 当前 Query 仍需读取并匹配全部历史 K/V |

## 4. 显存估算

训练与推理的显存组成不同：

```math
M_{\mathrm{train}}=M_{\mathrm{param}}+M_{\mathrm{grad}}+M_{\mathrm{opt}}+M_{\mathrm{act}}+M_{\mathrm{runtime}}
```

```math
M_{\mathrm{infer}}=M_{\mathrm{param}}+M_{\mathrm{KV}}+M_{\mathrm{runtime}}
```

### 4.1 参数与梯度

```math
M_{\mathrm{param}}=Pb_w
```

```math
M_{\mathrm{grad}}=Pb_g
```

例如，7B 模型采用 BF16 权重时，仅模型权重约为 $`7\times10^9\times2=14`$ GB。这里使用十进制 GB；按 GiB 计算时数值会略小。

### 4.2 Adam/AdamW 优化器状态

Adam/AdamW 通常保存 FP32 一阶动量 $`m`$ 和二阶动量 $`v`$：

```math
M_{m,v}=8P\ \mathrm{Byte}
```

传统 FP16 混合精度训练还可能维护一份 FP32 Master Weights：

```math
M_{\mathrm{master}}=4P\ \mathrm{Byte}
```

因此，采用 FP16/BF16 权重、FP16/BF16 梯度和 AdamW 时：

| 分项 | 每参数字节数 | 3B 模型示例 |
| --- | ---: | ---: |
| FP16/BF16 权重 | 2 Byte | 6 GB |
| FP16/BF16 梯度 | 2 Byte | 6 GB |
| FP32 一阶、二阶动量 | 8 Byte | 24 GB |
| FP32 Master Weights（若保留） | 4 Byte | 12 GB |
| 合计（不保留 Master Weights） | 12 Byte | 36 GB |
| 合计（保留 Master Weights） | 16 Byte | 48 GB |

> [!note]
> BF16 训练经常可以不保留 FP32 Master Weights；传统 FP16 混合精度通常会保留。具体实现还可能以 FP32 保存梯度，因此不能只根据“混合精度”四个字确定总显存。

### 4.3 激活值

训练时反向传播需要保存前向中间结果。粗略估算为：

```math
M_{\mathrm{act}}\approx cBTLd b_a
```

$`c`$ 是实现相关常数，受模型结构、保存哪些中间结果、FlashAttention 和 Gradient Checkpointing 等因素影响。

朴素 Attention 如果显式保存每层的 Attention Map，还会产生：

```math
M_{\mathrm{attn\_map}}\approx BhT^2b_a
```

它随序列长度平方增长。FlashAttention 通过分块计算避免在显存中物化完整的 $`T\times T`$ Attention 矩阵；Gradient Checkpointing 则少保存激活，在反向传播时重新计算，以算力换显存。

### 4.4 KV Cache

标准 MHA 中，每个 Token 在每一层都要缓存一份 Key 和一份 Value，每份总维度为 $`d`$：

```math
M_{\mathrm{KV,MHA}}=2BLTd b_{kv}
```

在 GQA/MQA 中，KV Head 数为 $`n_{kv}`$：

```math
M_{\mathrm{KV,GQA}}=2BLT n_{kv}d_h b_{kv}
```

KV Cache 随当前上下文长度和并发 Batch Size 线性增长，是长上下文、高并发推理的重要显存与带宽瓶颈。

> [!note]
> MLA、GQA 和 MQA 从模型结构上缩小需要缓存的 KV 表示；KV Cache 量化降低每个元素的字节数；PagedAttention 优化的是缓存的分配和管理方式。它们解决的是不同层面的问题。

### 4.5 显存小结

| 场景 | 主要组成 | 常见主导项 |
| --- | --- | --- |
| 全参数训练 | 参数、梯度、优化器状态、激活、运行时缓冲 | Adam 状态与激活 |
| 推理 | 模型权重、KV Cache、运行时缓冲 | 权重与 KV Cache |

## 5. 最该记住的公式

参数量主项：

```math
P_{\mathrm{block}}\approx4d^2+2dd_{ff}
```

标准 FFN 且 $`d_{ff}=4d`$ 时：

```math
P_{\mathrm{block}}\approx12d^2
```

矩阵乘法：

```math
F_{\mathrm{matmul}}\approx2mnp
```

训练计算量：

```math
F_{\mathrm{train}}\approx3F_{\mathrm{fwd}}\approx6PBT
```

训练静态显存：

```math
M_{\mathrm{static}}\approx12P\mathrel{\sim}16P\ \mathrm{Byte}
```

标准 MHA 的 KV Cache：

```math
M_{\mathrm{KV}}=2BLTd b_{kv}
```

## 6. 易错点

- Head 数 $`h`$ 不改变标准 MHA 的总参数量；GQA/MQA 减少 KV Head 后例外。
- LM Head 与 Embedding 共享权重时不新增参数，但计算词表 Logits 的 FLOPs 仍然存在。
- SwiGLU 有三个投影，参数量是 $`3dd_{ff}`$；约 $`\frac{8}{3}d`$ 是其常用中间维度，不是参数量系数。
- 完整训练约为一次前向的 3 倍；反向传播本身约为一次前向的 2 倍。
- 训练显存不能只看模型权重。优化器状态和激活值往往才是主要开销。
- KV Cache 只缓存 K/V，Attention Scores 每一步仍需根据当前 Q 与历史 K 现场计算。
- KV Cache 令单步 Decode 的 Attention 对当前上下文长度呈 $`O(T)`$，但生成多个 Token 的累计计算仍包含二次项。
- 所有公式都是用于容量规划的近似值；Kernel、并行策略、量化方式、显存碎片和框架实现都会影响实际峰值。
