---
type: concept
status: seed
domain: 大模型基础/结构
created: 2026-07-22
updated: 2026-07-22
aliases: [Position Encoding, Positional Encoding, RoPE, 旋转位置编码]
tags: [LLM, Transformer, position-encoding, RoPE, long-context]
---

# 位置编码--从绝对位置到RoPE

> 参考出处：[小红书笔记《Transformer 位置编码：从绝对位置到 RoPE》](https://www.xiaohongshu.com/explore/69da6f02000000001a035718?xsec_token=AB_i20gUT1YweDCG1YoqNTSkYcJSvQ2W9bywIOMYmeDwM=&xsec_source=pc_user)

> [!note]
> 位置编码为 token 序列注入顺序或距离信息，解决纯 Self-Attention 无法区分不同排列的问题。绝对位置编码描述“token 在哪里”，相对位置方法强调“两个 token 相距多远”；RoPE 通过旋转 Query 和 Key，让相对位置信息自然进入点积，因此成为当前大语言模型的主流方案。理解它也是理解长上下文扩展的基础。

## 为什么 Attention 需要位置编码

不带位置信息的 Self-Attention 对 token 排列是**等变的**：如果以相同方式重排输入 token，输出也只会跟着重排。它能比较 token 内容，却无法仅凭内容判断某个 token 是第 1 个还是第 10 个。

因此，Transformer 必须额外引入顺序信号。常见做法可以按信息进入模型的位置分成三类：

| 路线 | 位置作用在哪里 | 代表方法 |
| --- | --- | --- |
| 输入表示 | 将位置向量加到 token embedding | Learned PE、Sinusoidal PE |
| Attention logits | 给注意力分数增加距离相关偏置 | Relative Position Bias、ALiBi |
| Query / Key | 根据位置旋转 Q、K，使相对位置进入点积 | RoPE |

## 绝对位置编码

### Learned PE

Learned PE 为位置 $`0,1,2,\ldots`$ 分别学习一个向量，并将它加到对应 token embedding 上。它的优点是简单、灵活；缺点是训练长度之外的位置没有学过对应表示，因此长度外推能力较弱。

### Sinusoidal PE

原始 Transformer 使用不同频率的正弦、余弦函数构造位置向量：

```math
PE_{(pos,2i)}
=
\sin\left(\frac{pos}{10000^{2i/d}}\right)
```

```math
PE_{(pos,2i+1)}
=
\cos\left(\frac{pos}{10000^{2i/d}}\right)
```

它没有可训练参数，并且能为训练长度之外的位置继续生成编码。不过，“公式可以计算到更远位置”并不等于模型一定能稳定处理任意长序列，实际外推仍取决于训练分布和模型行为。

## 相对位置方法

相对位置方法不只关心“当前是第几个 token”，而是强调 Query 位置 $`i`$ 与 Key 位置 $`j`$ 的相对距离。这与 Attention 比较 token 两两关系的过程更自然。

### Relative Position Bias

Relative Position Bias 直接向 Attention logits 添加距离相关项：

```math
\mathrm{score}_{ij}
=
\frac{q_i^T k_j}{\sqrt{d_h}}
+ b(i-j)
```

偏置 $`b(i-j)`$ 可以通过查表或距离分桶得到。T5 使用的就是分桶式相对位置偏置：近距离保留较细粒度，远距离合并到较粗的区间。

### ALiBi

ALiBi 为不同 Attention head 设置不同斜率，并对较远位置施加更大的线性惩罚。对因果注意力中的 $`j\le i`$，可写成：

```math
\mathrm{score}_{ij}
=
\frac{q_i^T k_j}{\sqrt{d_h}}
-m_h(i-j)
```

它不需要位置 embedding，参数少、实现轻量，也具有一定长度外推能力；但它通过线性偏置影响分数，表达方式与 RoPE 不同。

## RoPE：把位置写入 Q/K 的旋转

RoPE 不把位置向量加到输入 embedding，也不额外给 logits 加偏置。它把每个 Attention head 的维度两两成对，并根据位置对 Query 和 Key 做二维旋转。

对第 $`i`$ 个二维子空间，角频率为：

```math
\omega_i=\theta^{-2i/d_h}
```

位置 $`p`$ 对应的旋转角为：

```math
\phi_i(p)=p\omega_i
```

旋转矩阵为：

```math
R_i(p)=
\begin{bmatrix}
\cos\phi_i(p) & -\sin\phi_i(p)\\
\sin\phi_i(p) & \cos\phi_i(p)
\end{bmatrix}
```

于是对应的 Query 和 Key 变为：

```math
q_i^{(p)}=R_i(p)q_i,
\qquad
k_i^{(s)}=R_i(s)k_i
```

二维旋转矩阵满足 $`R(p)^T R(s)=R(s-p)`$，因此旋转后的点积为：

```math
\left(q_i^{(p)}\right)^T k_i^{(s)}
=
q_i^T R_i(s-p)k_i
```

点积中的位置项只依赖相对距离 $`s-p`$。这就是 RoPE 的核心：**Q、K 各自使用绝对位置进行旋转，但二者点积呈现相对位置关系。**

### RoPE 为什么成为主流

- 相对位置信息直接参与 Q/K 点积，与 Attention 的计算形式结合自然。
- 不需要为每个位置保存独立 embedding。
- 不改变 Value，可较容易用于 MHA、GQA、MQA、MLA 等注意力结构。
- 可通过调整位置尺度、频率或注意力计算扩展到更长上下文。

需要记住的关键区分是：

> **RoPE 改的是 Q/K 本身；Relative Position Bias 和 ALiBi 改的是 Attention logits。**

## RoPE 为什么需要长上下文扩展

模型只在有限长度内训练时，直接输入远超训练范围的位置会让旋转相位进入模型未见过的分布。问题并不能简单概括为“所有频率都转得太快”，而是不同频率维度的相位和 Attention 模式共同发生分布外变化。

常见扩展方法分别调整位置、频率或 Attention 的相对位置范围。

### YaRN

YaRN 属于 RoPE scaling 方法。它不会简单地把所有频率统一缩放，而是按频率区间混合插值与外推，并配合 Attention scaling，使模型在扩展上下文时尽量保留短距离能力。

可以把它理解为：不同频率维度采用不同强度的长度扩展，而不是仅做一次统一的 $`p\mapsto p/s`$。

### DCA

DCA（Dual Chunk Attention）把长序列划分为 chunk，并分别处理块内、跨块和相邻块的相对位置，使 RoPE 看到的距离尽量保持在可处理范围。它主要改变推理时的 Attention 组织与位置编号方式，不等同于重新训练一种位置编码。

### ABF

ABF（Adaptive Base Frequency）保留 RoPE 形式，但增大频率基数 $`\theta`$：

```math
\omega_i=\theta^{-2i/d_h}
```

当 $`\theta`$ 增大时，除最高频的首个维度对外，其余维度的频率会按不同程度降低，从而改变远距离位置的相位变化速度。“Adaptive”通常指为目标上下文长度选择合适的 base，并不是每个 token 都实时调整 base。

### 方法定位

| 方法 | 主要改变 | 使用位置 |
| --- | --- | --- |
| YaRN | 不同频率维度的位置/频率缩放，并配合 Attention scaling | 训练或微调 + 推理 |
| DCA | 分块 Attention 与块内、块间相对位置组织 | 主要在推理阶段 |
| ABF | RoPE 的频率基数 $`\theta`$ | 模型配置与训练/推理 |

长上下文能力不是只修改一个配置值就能保证的。训练数据长度、RoPE scaling、Attention 组织、KV Cache、推理实现和长上下文评测需要配套考虑。

## 其他变体与学习顺序

- **xPos**：在旋转位置编码上加入与位置相关的缩放，改善长距离衰减特性。
- **LongRoPE**：搜索或优化不同维度、不同位置区间的非均匀缩放因子，并结合渐进式长度扩展；不能简单等同于“只增大 base”。
- **二维位置编码**：在视觉 Transformer 中同时编码行、列位置，用于图像网格等二维结构。

从大语言模型角度，可以按下面的顺序学习：

```text
Sinusoidal PE
→ Learned PE
→ Relative Position Bias
→ ALiBi
→ RoPE
→ Position Interpolation / NTK-aware Scaling
→ YaRN / DCA / ABF / LongRoPE
```

## 相关概念

- Self-Attention
- Transformer
- MHA、MQA、GQA 与 MLA
- KV Cache
- 长上下文训练与推理
- Position Interpolation
- NTK-aware RoPE Scaling

> [!warning]
> - 不带位置编码的 Self-Attention 更准确地说是对排列**等变**，不是所有输出都“排列不变”。
> - RoPE 不是直接给 token 写入一个“相对位置向量”，而是让 Q/K 旋转后的点积只依赖相对位置差。
> - 增大 RoPE base 不会让全部频率等比例变慢；不同维度受到的影响不同。
> - YaRN、DCA、ABF、LongRoPE 解决问题的层面不同，不能只根据名称互换。修改配置后仍需要长上下文训练或微调以及专门评测来确认效果。
