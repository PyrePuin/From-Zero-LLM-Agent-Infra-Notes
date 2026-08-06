---
type: concept
status: seed
domain: 大模型架构/位置编码
created: 2026-07-22
updated: 2026-08-06
aliases: [Position Encoding, Positional Encoding, RoPE, 旋转位置编码]
tags: [LLM, Transformer, position-encoding, RoPE, long-context]
---

# 位置编码--从绝对位置到RoPE

> 参考出处：[小红书笔记《Transformer 位置编码：从绝对位置到 RoPE》](https://www.xiaohongshu.com/explore/69da6f02000000001a035718?xsec_token=AB_i20gUT1YweDCG1YoqNTSkYcJSvQ2W9bywIOMYmeDwM=&xsec_source=pc_user)

> [!note]
> 位置编码为 token 序列注入顺序或距离信息，解决纯 Self-Attention 无法区分不同排列的问题。绝对位置编码描述“token 在哪里”，相对位置方法强调“两个 token 相距多远”；RoPE 通过旋转 Query 和 Key，让相对位置信息自然进入点积，因此成为当前大语言模型的主流方案。理解它也是理解长上下文扩展的基础。

## 1. 为什么 Attention 需要位置编码

不带位置信息的 Self-Attention 对 token 排列是**等变的**：如果以相同方式重排输入 token，输出也只会跟着重排。它能比较 token 内容，却无法仅凭内容判断某个 token 是第 1 个还是第 10 个。

因此，Transformer 必须额外引入顺序信号。常见做法可以按信息进入模型的位置分成三类：

| 路线 | 位置作用在哪里 | 代表方法 |
| --- | --- | --- |
| 输入表示 | 将位置向量加到 token embedding | Learned PE、Sinusoidal PE |
| Attention logits | 给注意力分数增加距离相关偏置 | Relative Position Bias、ALiBi |
| Query / Key | 根据位置旋转 Q、K，使相对位置进入点积 | RoPE |

## 2. 绝对位置编码

### 2.1 Learned PE

Learned PE 为位置 $`0,1,2,\ldots`$ 分别学习一个向量，并将它加到对应 token embedding 上。它的优点是简单、灵活；缺点是训练长度之外的位置没有学过对应表示，因此长度外推能力较弱。

### 2.2 Sinusoidal PE

#### 2.2.1 编码如何生成

原始 Transformer 使用不同频率的正弦、余弦函数构造位置向量。对序列位置 $`pos`$ 和第 $`i`$ 个二维频率组：

```math
PE_{(pos,2i)}
=
\sin\left(\frac{pos}{10000^{2i/d_{\mathrm{model}}}}\right)
```

```math
PE_{(pos,2i+1)}
=
\cos\left(\frac{pos}{10000^{2i/d_{\mathrm{model}}}}\right)
```

其中，$`d_{\mathrm{model}}`$ 是 token embedding 和模型隐藏状态的维度，并假设它是偶数；$`i=0,1,\ldots,d_{\mathrm{model}}/2-1`$。同一组的正弦与余弦共享角频率：

```math
\omega_i=10000^{-2i/d_{\mathrm{model}}}
```

因此，第 $`i`$ 个二维组可以写成 $`(\sin(pos\omega_i),\cos(pos\omega_i))`$。较小的 $`i`$ 对应较高频率，位置稍微变化，相位就会明显变化；较大的 $`i`$ 对应较低频率，用来描述更缓慢、更长尺度的位置变化。多个频率组合在一起，使每个位置得到一个多尺度的固定编码。

Sinusoidal PE 没有可训练参数，也不需要为每个位置保存一行独立的 embedding 表。给定任意整数位置，都可以继续通过公式计算对应编码。

#### 2.2.2 位置编码加到哪里

Sinusoidal PE 加在 **Transformer 的输入表示** 上，而不是直接加到 Attention logits，也不是只作用于 Query 和 Key。

设位置 $`p`$ 的 token embedding 为 $`e_p`$。在原始 Transformer 中，embedding 会先乘以 $`\sqrt{d_{\mathrm{model}}}`$，再与同维度的位置编码逐元素相加：

```math
h_p^{(0)}=\sqrt{d_{\mathrm{model}}}e_p+PE(p)
```

相加后的 $`h_p^{(0)}`$ 经过 dropout，再进入第一个 Transformer Block。Self-Attention 随后从这份已经混合了“token 内容”和“绝对位置”的表示生成 Query、Key 和 Value：

```math
q_p=h_p^{(0)}W_Q
```

```math
k_p=h_p^{(0)}W_K
```

```math
v_p=h_p^{(0)}W_V
```

所以它的作用路径是：

```text
token embedding + Sinusoidal PE
              ↓
       输入隐藏状态
              ↓
         投影为 Q/K/V
              ↓
       计算 Attention
```

原始做法只在模型输入端加入一次位置编码，后续层依靠隐藏状态和残差连接继续携带位置信息。

#### 2.2.3 它如何表达相对位移

Sinusoidal PE 虽然属于绝对位置编码，但它的频率结构允许模型利用相对位移。对某个频率 $`\omega_i`$，把位置从 $`p`$ 平移到 $`p+\Delta`$ 时：

```math
\sin\left((p+\Delta)\omega_i\right)
=
\sin(p\omega_i)\cos(\Delta\omega_i)
+
\cos(p\omega_i)\sin(\Delta\omega_i)
```

```math
\cos\left((p+\Delta)\omega_i\right)
=
\cos(p\omega_i)\cos(\Delta\omega_i)
-
\sin(p\omega_i)\sin(\Delta\omega_i)
```

对固定的相对位移 $`\Delta`$，新位置的两个分量可以由原位置的两个分量通过一个只依赖 $`\Delta`$ 的线性变换得到。这意味着模型有机会从绝对编码中学习相对位置关系。

但“可以学习”不等于“Attention 天然就只看相对位置”。同一频率组的两个原始位置编码直接做内积时，有：

```math
\sin(p\omega_i)\sin(s\omega_i)
+
\cos(p\omega_i)\cos(s\omega_i)
=
\cos\left((p-s)\omega_i\right)
```

由于余弦是偶函数，$`p-s`$ 与 $`s-p`$ 会得到相同结果。也就是说，**原始位置编码的标准内积能够反映距离，却不能直接区分相对位移的正负方向。**

这不等于 Sinusoidal PE 完全“没有方向性”。正弦与余弦组成的相位仍保留位置变化信息；经过不对称的 $`W_Q`$、$`W_K`$ 投影，或者结合因果掩码，模型仍可能学习“向前”和“向后”的差别。更准确的缺陷是：它没有像有符号相对位置偏置那样，直接向 Attention 提供方向归纳偏置。

#### 2.2.4 主要缺陷

1. **内容与位置通过加法纠缠。** Token embedding 和位置编码占用同一组维度，相加后再共同投影为 Q、K、V。模型需要自己学会哪些变化来自内容、哪些变化来自位置。
2. **相对关系不是 Attention 的显式输入。** Sinusoidal PE 提供的是每个 token 的绝对位置表示。两个 token 相距多远、谁在谁前面，需要 Q/K 投影和后续网络从绝对编码中学习，不像 Relative Position Bias 那样直接写入 logits，也不像 RoPE 那样由 Q/K 点积自然产生相对位移。
3. **原始位置相似度缺少有符号方向。** $`PE(p)^T PE(s)`$ 对 $`p-s`$ 和 $`s-p`$ 对称，因此单靠标准内积无法判断另一个 token 位于当前 token 的前方还是后方；方向差异要依赖学习后的投影或 Attention mask。
4. **位置只在输入端注入一次。** 后续层不会重新加入原始 Sinusoidal PE，而要依靠隐藏状态和残差路径保留位置特征。
5. **公式可外推，不代表模型能可靠外推。** 虽然任意更远位置都能计算出编码，但模型没有在训练长度之外见过这些相位组合及其对应的 Attention 模式，长序列上的性能仍可能明显下降。

## 3. 相对位置方法

相对位置方法不只关心“当前是第几个 token”，而是强调 Query 位置 $`i`$ 与 Key 位置 $`j`$ 的相对距离。这与 Attention 比较 token 两两关系的过程更自然。

### 3.1 Relative Position Bias

Relative Position Bias 直接向 Attention logits 添加距离相关项：

```math
\mathrm{score}_{ij}
=
\frac{q_i^T k_j}{\sqrt{d_h}}
+ b(i-j)
```

偏置 $`b(i-j)`$ 可以通过查表或距离分桶得到。T5 使用的就是分桶式相对位置偏置：近距离保留较细粒度，远距离合并到较粗的区间。

### 3.2 ALiBi

ALiBi 为不同 Attention head 设置不同斜率，并对较远位置施加更大的线性惩罚。对因果注意力中的 $`j\le i`$，可写成：

```math
\mathrm{score}_{ij}
=
\frac{q_i^T k_j}{\sqrt{d_h}}
-m_h(i-j)
```

它不需要位置 embedding，参数少、实现轻量，也具有一定长度外推能力；但它通过线性偏置影响分数，表达方式与 RoPE 不同。

## 4. RoPE：把位置写入 Q/K 的旋转

RoPE 不把位置向量加到输入 embedding，也不额外给 logits 加偏置。它把每个 Attention head 的维度两两成对，并根据位置对 Query 和 Key 做二维旋转。

对第 $`i`$ 个二维子空间，角频率为：

```math
\omega_i=\theta^{-2i/d_h}
```

位置 $`p`$ 对应的旋转角为：

```math
\phi_i(p)=p\omega_i
```

位置 $`p`$ 的二维旋转作用记为 $`R_i(p)`$。为避免依赖矩阵环境，可以直接写成两个分量的变换：

```math
x'_{2i}=x_{2i}\cos\phi_i(p)-x_{2i+1}\sin\phi_i(p)
```

```math
x'_{2i+1}=x_{2i}\sin\phi_i(p)+x_{2i+1}\cos\phi_i(p)
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

### 4.1 RoPE 为什么成为主流

- 相对位置信息直接参与 Q/K 点积，与 Attention 的计算形式结合自然。
- 不需要为每个位置保存独立 embedding。
- 不改变 Value，可较容易用于 MHA、GQA、MQA、MLA 等注意力结构。
- 可通过调整位置尺度、频率或注意力计算扩展到更长上下文。

需要记住的关键区分是：

> **RoPE 改的是 Q/K 本身；Relative Position Bias 和 ALiBi 改的是 Attention logits。**

## 5. RoPE 为什么需要长上下文扩展

模型只在有限长度内训练时，直接输入远超训练范围的位置会让旋转相位进入模型未见过的分布。问题并不能简单概括为“所有频率都转得太快”，而是不同频率维度的相位和 Attention 模式共同发生分布外变化。

常见扩展方法分别调整位置、频率或 Attention 的相对位置范围。

### 5.1 YaRN

YaRN 属于 RoPE scaling 方法。它不会简单地把所有频率统一缩放，而是按频率区间混合插值与外推，并配合 Attention scaling，使模型在扩展上下文时尽量保留短距离能力。

可以把它理解为：不同频率维度采用不同强度的长度扩展，而不是仅做一次统一的 $`p\mapsto p/s`$。

### 5.2 DCA

DCA（Dual Chunk Attention）把长序列划分为 chunk，并分别处理块内、跨块和相邻块的相对位置，使 RoPE 看到的距离尽量保持在可处理范围。它主要改变推理时的 Attention 组织与位置编号方式，不等同于重新训练一种位置编码。

### 5.3 ABF

ABF（Adaptive Base Frequency）保留 RoPE 形式，但增大频率基数 $`\theta`$：

```math
\omega_i=\theta^{-2i/d_h}
```

当 $`\theta`$ 增大时，除最高频的首个维度对外，其余维度的频率会按不同程度降低，从而改变远距离位置的相位变化速度。“Adaptive”通常指为目标上下文长度选择合适的 base，并不是每个 token 都实时调整 base。

### 5.4 方法定位

| 方法 | 主要改变 | 使用位置 |
| --- | --- | --- |
| YaRN | 不同频率维度的位置/频率缩放，并配合 Attention scaling | 训练或微调 + 推理 |
| DCA | 分块 Attention 与块内、块间相对位置组织 | 主要在推理阶段 |
| ABF | RoPE 的频率基数 $`\theta`$ | 模型配置与训练/推理 |

长上下文能力不是只修改一个配置值就能保证的。训练数据长度、RoPE scaling、Attention 组织、KV Cache、推理实现和长上下文评测需要配套考虑。

## 6. 其他变体与学习顺序

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

## 7. 相关概念

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
