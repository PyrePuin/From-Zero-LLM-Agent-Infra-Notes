---
type: roadmap
status: active
domain: 大模型架构/位置编码
created: 2026-08-09
updated: 2026-08-09
aliases: [位置编码学习路线, Position Encoding Roadmap]
tags: [LLM, Transformer, Position-Encoding, RoPE, Long-Context]
---

# 位置编码学习路线

> [!note]
> 本目录从“Attention 为什么需要位置”出发，逐步进入绝对位置、相对位置、RoPE 数学原理、RoPE 工程调用链和 YaRN 长上下文扩展。推荐按下面的顺序阅读，不需要为了编号而改动旧文件名和相对链接。

## 1. 推荐阅读顺序

| 顺序 | 笔记 | 主要内容 | 读完后应能回答 |
| ---: | --- | --- | --- |
| 1 | [位置编码--从绝对位置到RoPE](./位置编码--从绝对位置到RoPE.md) | Learned PE、Sinusoidal PE、Relative Position Bias、ALiBi、RoPE 的概念演进 | 位置信息可以在哪些地方进入 Attention？RoPE 为什么将位置写入 Q/K？ |
| 2 | [RoPE的工程实现原理](./RoPE的工程实现原理.md) | 逆频率、cos/sin、`rotate_half`、真实张量形状、KV Cache、Prefill 和 Decode | RoPE 在代码中什么时候作用于 Q/K？为什么 Cache 中的 Key 已经带有位置？ |
| 3 | [YaRN--高效扩展RoPE上下文窗口](./YaRN--高效扩展RoPE上下文窗口.md) | Position Interpolation、NTK-aware、NTK-by-parts、Attention Scaling、动态扩展 | 为什么不能对所有 RoPE 频率一刀切缩放？YaRN 怎样分别处理高频和低频？ |

## 2. 每篇笔记的定位

### 2.1 第一篇：建立方法地图

[位置编码--从绝对位置到RoPE](./位置编码--从绝对位置到RoPE.md) 先解决“为什么需要位置编码”，再区分三类入口：

```text
输入表示：Learned PE / Sinusoidal PE
Attention Logits：Relative Position Bias / ALiBi
Query 与 Key：RoPE
```

它是整个目录的概念前置，应先理解位置信息加在哪里，再进入具体公式和代码。

### 2.2 第二篇：把数学映射到代码

[RoPE的工程实现原理](./RoPE的工程实现原理.md) 从真实张量形状出发，追踪：

```text
inv_freq
→ position_ids
→ cos / sin
→ rotate_half
→ apply_rotary_pos_emb
→ KV Cache
→ Prefill / Decode
```

阅读时要把注意力放在旋转前后的形状、Cache 一致性和实现变体，而不只是记住 `rotate_half` 函数。

### 2.3 第三篇：理解长上下文扩展

[YaRN--高效扩展RoPE上下文窗口](./YaRN--高效扩展RoPE上下文窗口.md) 以前两篇为基础，进一步处理：

- 训练长度之外的位置和旋转相位分布外问题。
- 高频局部位置分辨率与低频长距离覆盖之间的冲突。
- 上下文变长后 Softmax 竞争和 Attention 分布变化。
- 动态缩放与 KV Cache 位置语义一致性。

## 3. 两条学习路线

### 3.1 原理路线

```text
Attention 的排列等变性
→ 绝对位置编码
→ 相对位置偏置
→ RoPE 相对点积
→ 频率、波长与长度外推
→ YaRN
```

### 3.2 工程路线

```text
position_ids / cache_position
→ RoPE 参数和 cos/sin
→ Q/K 旋转
→ KV Cache 写入和复用
→ Prefill / Decode 一致性
→ 长上下文缩放和 Cache 约束
```

## 4. 建议的前置知识

进入本目录前，建议已经理解：

- Token Embedding 和隐藏状态。
- Multi-Head Self-Attention 中 Q、K、V 的形状与点积。
- Attention Mask 与 Causal Mask 的作用。
- Prefill、逐 Token Decode 和 KV Cache 的基本区别。
- 二维向量旋转、正弦与余弦的基本几何含义。

ViT 使用可学习绝对位置编码的例子，可以参考[01-ViT：把图像变成 Token 的视觉 Transformer](../../多模态VLM/01-ViT.md)。

## 5. 完成本目录后的检查问题

1. 不带位置信息的 Self-Attention 对 Token 重排有什么性质？
2. Learned PE 和 Sinusoidal PE 为什么属于绝对位置编码？
3. Relative Position Bias、ALiBi 和 RoPE 分别在 Attention 的哪一步注入位置？
4. RoPE 为什么只作用于 Q/K，通常不旋转 Value？
5. RoPE 旋转为什么不改变向量范数？
6. 为什么两个位置旋转后的 Q/K 点积可以表达相对距离？
7. 为什么可以计算更远位置的 RoPE，却不等于模型能稳定处理更长上下文？
8. Position Interpolation 为什么可能损伤高频局部位置分辨率？
9. YaRN 如何使用原训练长度内的旋转圈数，决定不同频率应该保留、插值还是混合？
10. 动态缩放为什么必须考虑已写入 KV Cache 的 Key 所使用的位置规则？
