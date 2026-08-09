---
type: concept
status: seed
domain: 大模型架构/位置编码
created: 2026-08-09
updated: 2026-08-09
aliases: [YaRN, Yet another RoPE extensioN, NTK-by-parts]
tags: [LLM, Transformer, RoPE, YaRN, long-context]
---

# YaRN--高效扩展RoPE上下文窗口

> 主要参考：
> - [YaRN：Efficient Context Window Extension of Large Language Models](https://arxiv.org/abs/2309.00071)
> - [YaRN 作者代码仓库](https://github.com/jquesnelle/yarn)
> - [Position Interpolation](https://arxiv.org/abs/2306.15595)
> - [Hugging Face Transformers：RoPE utilities](https://huggingface.co/docs/transformers/en/internal/rope_utils)
>
> 前置知识：[位置编码--从绝对位置到RoPE](./位置编码--从绝对位置到RoPE.md)

> [!note]
> YaRN（Yet another RoPE extensioN）是一种面向 RoPE 模型的长上下文扩展方法。它不是把所有 RoPE 频率统一缩放，而是根据每个频率维度在原训练长度内旋转的圈数，分别选择完整插值、保持不变或平滑混合；随后再通过 Attention scaling 调整长上下文下的注意力分布。它的核心可以概括为：**NTK-by-parts interpolation + Attention scaling。**

## 1. YaRN 要解决什么问题

设一个使用 RoPE 的模型只在最大长度 $`L`$ 内训练。推理时如果直接输入远长于 $`L`$ 的序列，虽然 RoPE 仍能计算任意位置的旋转角，但模型会遇到训练中没有见过的相对距离、旋转相位和 Attention 分布。

YaRN 的目标是把上下文从原长度 $`L`$ 扩展到目标长度 $`L'`$：

```math
L'>L
```

定义扩展倍数：

```math
s=\frac{L'}{L}
```

例如，从 4K 扩展到 32K 时：

```math
s=\frac{32768}{4096}=8
```

需要解决两个相互关联的问题：

1. **RoPE 频率失配。** 新增长距离会进入原训练范围之外，但如果粗暴压缩全部频率，又会损伤局部位置分辨率。
2. **Attention 分布变化。** 上下文变长后，更多 Key 参与 Softmax 竞争，注意力概率可能变得更分散。

YaRN 分别使用 NTK-by-parts interpolation 和 Attention scaling 处理这两个问题。

## 2. RoPE 的频率与波长

### 2.1 RoPE 频率

设每个 Attention head 的维度为 $`D`$，第 $`d`$ 个二维子空间的 RoPE 频率为：

```math
\theta_d=b^{-2d/D}
```

其中，$`b`$ 是 RoPE base，原始 RoPE 常使用 $`b=10000`$。

位置 $`m`$ 在该维度上的旋转角为：

```math
m\theta_d
```

不同维度使用不同频率，使模型可以同时表示局部和较长尺度的位置变化。

### 2.2 波长

频率 $`\theta_d`$ 对应的波长为：

```math
\lambda_d
=
\frac{2\pi}{\theta_d}
=
2\pi b^{2d/D}
```

$`\lambda_d`$ 表示该维度完成一次 $`2\pi`$ 旋转需要跨越多少个 token：

- $`\lambda_d`$ 小：频率高、旋转快，少量 token 就能转完一圈。
- $`\lambda_d`$ 大：频率低、旋转慢，需要很多 token 才能转完一圈。

YaRN 后面会根据波长相对于训练长度 $`L`$ 的大小，判断每个维度应该怎样缩放。

## 3. Position Interpolation：统一压缩位置

### 3.1 基本做法

Position Interpolation（PI）不让新位置直接落到原训练范围之外，而是把目标长度 $`L'`$ 内的位置统一压缩回 $`L`$：

```math
g(m)=\frac{m}{s}
```

原来的旋转角：

```math
m\theta_d
```

变为：

```math
\frac{m}{s}\theta_d
```

这也可以理解为保持位置 $`m`$ 不变，同时把所有频率统一除以 $`s`$：

```math
\theta_d'=\frac{\theta_d}{s}
```

例如，从 4K 扩展到 32K 时，$`s=8`$。新序列中的位置 24000 会映射为：

```math
\frac{24000}{8}=3000
```

映射后的位置仍处于原训练范围内。

> [!warning]
> YaRN 论文 2.3 节的公式将 PI 写成了 $`g(m)=s\cdot m`$，但其附录 A.1 给出的实际公式是 $`g(m)=mL/L'=m/s`$；后者才与位置压缩的定义一致。

### 3.2 统一插值的问题

PI 对所有频率一视同仁。扩展 $`s`$ 倍，就把所有频率都缩小 $`s`$ 倍，包括负责细粒度局部位置变化的高频维度。

这会压缩原有的局部角度差。例如，扩展 8 倍后，原来相距 8 个 token 的旋转角差，被压缩为原来相距 1 个 token 的角度差。随着 $`s`$ 增大，模型原来学到的局部位置分辨率可能受到破坏。

因此，PI 的主要问题是：

> **它避免了新位置直接外推，却以统一压缩全部频率为代价，损失了部分高频位置信息。**

## 4. NTK-aware：高频少缩放，低频多缩放

### 4.1 核心动机

NTK-aware interpolation 受到 Neural Tangent Kernel 与 Fourier Features 相关分析的启发。它认为，统一插值会削弱高频特征，而神经网络不一定能通过少量微调轻易恢复这些信息。

因此，NTK-aware 不再让所有 $`\theta_d`$ 统一除以 $`s`$，而是：

- 高频维度少缩放，保留局部位置分辨率。
- 低频维度多缩放，扩大能够覆盖的距离范围。

这里的“NTK-aware”表示设计受到 NTK 频谱分析启发，并不表示训练时需要显式计算 Neural Tangent Kernel。

### 4.2 通过增大 base 实现非均匀缩放

一种实现方式是把原始 base $`b`$ 改为：

```math
b'
=
b\cdot s^{D/(D-2)}
```

再用新 base 计算频率：

```math
\theta_d'={b'}^{-2d/D}
```

因为不同维度的指数不同，增大 base 的影响也不同：

- 最高频维度基本不变。
- 中间频率降低一部分。
- 最低频维度接近缩小到原来的 $`1/s`$。

这比 PI 的“一刀切”保留了更多高频信息。

### 4.3 NTK-aware 的不足

NTK-aware 指出了正确方向，但仍有两个主要问题：

1. **合适的 base 难以确定。** 给定扩展倍数后，最佳 base 往往仍需要实验搜索。
2. **没有显式判断每个维度的实际位置功能。** 它按照维度连续改变频率，却没有根据该维度在训练范围内究竟转过多少圈，明确划分应该插值还是保留。

NTK-by-parts 会把第二点变成一个可以直接计算的分段规则。

## 5. NTK-by-parts：按照旋转圈数分段

### 5.1 旋转圈数

定义第 $`d`$ 个维度在原训练长度 $`L`$ 内完成的旋转圈数：

```math
r(d)
=
\frac{L}{\lambda_d}
=
\frac{L}{2\pi b^{2d/D}}
```

$`r(d)`$ 把抽象的频率变成一个更直观的问题：该维度在模型预训练时究竟转过多少圈？

### 5.2 高频维度为什么尽量保持不变

如果波长远小于训练长度：

```math
\lambda_d\ll L
```

那么 $`r(d)`$ 很大，该维度在训练范围内已经旋转过很多圈。同一个相位会出现在多个绝对位置，模型难以仅凭它确定唯一的绝对坐标，但可以利用相位差表示局部相对距离。

因此，论文将这类高频维度视为主要承载局部相对位置，扩展时尽量保持原频率。

### 5.3 低频维度为什么需要插值

如果波长大于或接近训练长度：

```math
\lambda_d\ge L
```

那么 $`r(d)`$ 较小，该维度在训练期间可能连一圈都没有转完。以序列起点为锚点时，不同位置对应近似唯一的相位，模型可能把它用作绝对位置线索。

扩展到 $`L'`$ 后，这些相位容易进入原训练范围之外。因此，论文主张对低频、长波长维度进行完整插值：

```math
\theta_d'=\frac{\theta_d}{s}
```

> [!important]
> “高频偏相对位置、低频偏绝对位置”是 YaRN 根据旋转周期提出的分析和归纳假设，不是每个模型、每个维度都必须严格遵守的数学定理。

### 5.4 中间频率进行平滑混合

论文引入两个旋转圈数边界 $`\alpha`$ 和 $`\beta`$。对 LLaMA 系列，论文给出的经验值为：

```math
\alpha=1,
\qquad
\beta=32
```

处理规则为：

- 当 $`r(d)<\alpha`$：完整插值，使用 $`\theta_d/s`$。
- 当 $`r(d)>\beta`$：不插值，保留 $`\theta_d`$。
- 当 $`\alpha\le r(d)\le\beta`$：在二者之间平滑混合。

中间区间的混合系数为：

```math
\gamma(r)
=
\frac{r-\alpha}{\beta-\alpha}
```

在下边界以下令 $`\gamma=0`$，在上边界以上令 $`\gamma=1`$。最终频率为：

```math
h(\theta_d)
=
\left(1-\gamma(r(d))\right)\frac{\theta_d}{s}
+
\gamma(r(d))\theta_d
```

因此：

| 频率区域 | 旋转特征 | 缩放策略 | 主要目的 |
| --- | --- | --- | --- |
| 低频、长波长 | 训练期旋转不足或很少 | 完整插值 | 避免绝对位置相位外推 |
| 中间频率 | 旋转圈数居中 | 加权混合 | 平滑连接两端策略 |
| 高频、短波长 | 训练期旋转很多圈 | 保持原频率 | 保留局部相对位置能力 |

这套方法称为 NTK-by-parts interpolation。

## 6. Attention scaling：避免注意力过度分散

### 6.1 长上下文下的 Attention 分布变化

上下文变长后，会有更多 Key 参与 Softmax。即使 Attention logits 的统计尺度没有显著变化，概率质量也可能分散到更多 token 上，使注意力熵增大，重要 token 获得的权重下降。

YaRN 在 Softmax 前加入温度 $`t`$：

```math
\mathrm{softmax}_j
\left(
\frac{q_m^T k_n}{t\sqrt{D}}
\right)
```

当 $`t<1`$ 时，logits 之间的差异会被放大，Softmax 分布更加尖锐。它不会预先决定应该关注哪个 token，而是进一步突出内容分数原本较高的 token，并压低分数较低的 token。

### 6.2 通过缩放 Q/K 实现

可以同时把 Query 和 Key 乘以：

```math
\sqrt{\frac{1}{t}}
```

于是点积变为：

```math
\left(\sqrt{\frac{1}{t}}q\right)^T
\left(\sqrt{\frac{1}{t}}k\right)
=
\frac{1}{t}q^Tk
```

论文把这个系数合并到预计算的 RoPE 数值中，从而避免单独修改 Attention 主体计算。

对于 LLaMA 和 Llama 2，论文给出的经验公式为：

```math
\sqrt{\frac{1}{t}}
=
0.1\ln(s)+1
```

例如，扩展 8 倍时：

```math
\sqrt{\frac{1}{t}}
=
1+0.1\ln(8)
\approx
1.208
```

Q 和 K 都乘以约 $`1.208`$，点积约放大到原来的 $`1.208^2\approx1.46`$，从而使 Attention 更集中。

## 7. YaRN 的完整定义

YaRN 不是单独一种新的位置编码公式，而是两个方法的组合：

```math
\mathrm{YaRN}
=
\mathrm{NTK\text{-}by\text{-}parts}
+
\mathrm{Attention\ scaling}
```

完整作用路径可以概括为：

```text
目标扩展倍数 s = L' / L
    ↓
计算每个 RoPE 维度的波长 λ_d
    ↓
计算训练长度内的旋转圈数 r(d) = L / λ_d
    ↓
低频完整插值，中频平滑混合，高频保持不变
    ↓
生成新的 RoPE 频率
    ↓
对 Q/K 施加 Attention scaling
    ↓
按目标场景选择是否进行长上下文微调，并进行专门评测
```

## 8. Dynamic Scaling

### 8.1 固定缩放

一种做法是在整个推理过程中固定使用目标扩展倍数：

```math
s=\frac{L'}{L}
```

即使当前序列仍短于原训练长度，也使用已经缩放的 RoPE。这样实现简单，但可能降低原始短上下文能力；如果实际长度超过目标 $`L'`$，还可能出现明显退化。

### 8.2 动态缩放

Dynamic Scaling 根据当前序列长度 $`l'`$ 动态计算：

```math
s
=
\max\left(1,\frac{l'}{L}\right)
```

当 $`l'\le L`$ 时，$`s=1`$，保持原始 RoPE；只有超过原训练长度后，缩放倍数才随当前长度逐渐增大。

这可以减少短上下文性能损失，并让模型超过目标长度后更平缓地退化。Dynamic Scaling 可以与不同 RoPE scaling 方法组合；与 NTK-aware 结合时通常称为 Dynamic NTK。

### 8.3 KV Cache 一致性

自回归推理通常会缓存已经计算的 Key 和 Value。但在 Dynamic Scaling 中，序列长度增加会改变 $`s`$，同一个历史 token 所需的 RoPE 旋转也可能随之改变。

如果缓存的是使用旧 $`s`$ 旋转后的 Key，后续计算就可能不一致。因此，动态缩放实现需要保证历史 Key 与当前缩放规则一致，例如重新计算旋转结果，或者采用不会在生成过程中改变已有 Key 旋转的实现策略。

## 9. YaRN 解决不了什么

YaRN 主要处理 RoPE 频率和 Attention 分布在长上下文下的失配，但它不是完整的长上下文系统方案。

它不直接解决：

- Self-Attention 随序列长度增长的平方计算量。
- KV Cache 随 token 数增加的显存占用。
- 模型缺少长文档训练数据的问题。
- 长距离检索、组合推理和指令遵循能力不足。
- “上下文能够放入模型，但模型没有真正利用”的有效上下文问题。

因此，修改 YaRN 配置只表示模型的位置机制支持更长范围，不代表模型一定具备相同长度的有效理解能力。仍需配套长上下文训练或微调，并通过困惑度、Passkey Retrieval、Needle-in-a-Haystack、RULER 和真实任务进行验证。

## 10. 核心总结

YaRN 的推理链条是：

```text
PI 统一压缩全部频率
    ↓
高频局部位置能力受损
    ↓
NTK-aware 改为高频少缩放、低频多缩放
    ↓
但 base 难选，频率边界不明确
    ↓
NTK-by-parts 按训练期旋转圈数明确分段
    ↓
Attention scaling 修正长上下文下过度分散的注意力
    ↓
组合成 YaRN
```

最简洁的理解是：

> **YaRN 根据不同 RoPE 维度在预训练长度内的旋转圈数，对高频维度保持原状、对低频维度进行插值、对中间频率进行加权混合，从而兼顾局部相对位置能力和低频位置的分布内映射；随后通过 Attention scaling 放大 logits 差异，缓解长上下文下注意力概率过度分散的问题。**

> [!warning]
> - NTK-aware 不是完整的 YaRN，而是通向 NTK-by-parts 的中间方案。
> - YaRN 的核心是 NTK-by-parts 与 Attention scaling 的组合。
> - “公式可以扩展到更长位置”不等于模型能够可靠使用同样长的上下文。
> - Dynamic Scaling 是可组合的推理策略，不是 YaRN 核心定义中不可缺少的第三部分。
