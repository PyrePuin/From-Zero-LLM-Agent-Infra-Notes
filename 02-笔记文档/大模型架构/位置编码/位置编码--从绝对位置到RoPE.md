---
type: concept
status: seed
domain: 大模型架构/位置编码
created: 2026-07-22
updated: 2026-08-09
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

#### 3.1.1 偏置加在哪里

Relative Position Bias（RPB）不把位置向量加到 token embedding，也不修改 Value。它在 Query-Key 内容分数计算完成后、Softmax 之前，向 Attention logits 添加一个只与相对距离有关的偏置。

对第 $`h`$ 个 Attention head，先计算内容匹配分数：

```math
c_{hij}
=
\frac{q_{hi}^T k_{hj}}{\sqrt{d_h}}
```

再加入位置偏置：

```math
\mathrm{score}_{hij}
=
c_{hij}+b_h(i-j)
```

最后才沿 Key 维度做 Softmax：

```math
\alpha_{hij}
=
\mathrm{softmax}_j\left(\mathrm{score}_{hij}\right)
```

因此，RPB 的完整作用路径是：

```text
Q/K 内容点积
    ↓
加上相对距离偏置
    ↓
Softmax 得到注意力权重
    ↓
对 Value 加权求和
```

它改变的是“允许关注的 Key 中，更偏好哪些相对位置”，而不是直接改变 token 的内容表示。

#### 3.1.2 每个 head 学习一组距离偏置

RPB 通常不是给每个 head 只学习一个标量，而是让每个 head 学习一组“相对距离到偏置”的映射。例如，第 $`h`$ 个 head 可以有：

| 相对距离 | 可学习偏置 |
| ---: | ---: |
| $`0`$ | $`b_h(0)`$ |
| $`1`$ | $`b_h(1)`$ |
| $`2`$ | $`b_h(2)`$ |
| $`3`$ | $`b_h(3)`$ |

这些偏置会和模型的其他参数一起通过反向传播更新。某个 head 可以学会偏好相邻 token，另一个 head 也可以学会降低局部位置的分数、保留更远距离的关系；偏置不一定单调地随距离减小。

如果有 $`H`$ 个 head、$`B`$ 个距离 bucket，那么这组参数可以理解为一个 $`B\times H`$ 的偏置表。它是否在 Transformer 层之间共享，取决于具体模型实现。

#### 3.1.3 为什么需要距离分桶

如果为每个可能的距离都保存独立参数，最大上下文越长，偏置表就越大；训练中没有出现过的新距离也没有学到对应参数。常见做法是先把相对距离 $`i-j`$ 映射到 bucket，再查表：

```math
\mathrm{score}_{hij}
=
c_{hij}+b_h\left(r(i-j)\right)
```

其中 $`r`$ 是距离分桶函数。T5 的思路是：近距离保留较细粒度，远距离逐渐合并到较粗的区间。例如：

| 原始距离 | bucket 示例 |
| --- | --- |
| $`0,1,2,3`$ | 分别使用独立 bucket |
| $`4\text{-}7`$ | 共用一个 bucket |
| $`8\text{-}15`$ | 共用一个 bucket |
| 更远距离 | 逐渐合并到少数 bucket |

近距离通常与局部语法关系密切，需要精确区分；远距离则只保留“较远”或“非常远”等粗粒度信息。双向 Attention 还可以为正、负相对距离分配不同 bucket，从而直接区分“在前面”和“在后面”；因果 Attention 只允许 $`j\le i`$，通常只需处理当前位置和过去位置。

假设三个 Key 的内容分数都是 $`4`$，某个 head 对距离 $`0,1,2`$ 学到的偏置分别为 $`0.3,0.1,-0.4`$，最终 logits 就会变成 $`4.3,4.1,3.6`$。偏置只是修改注意力竞争的起点，并不会硬性禁止模型关注某个距离。

### 3.2 ALiBi

#### 3.2.1 从可学习偏置表变成固定直线

ALiBi（Attention with Linear Biases）可以看成一种受到强约束的 Relative Position Bias。它不再为不同距离学习独立偏置，而是规定：距离每增加一步，就按固定斜率多扣一点分。

对因果 Attention 中允许访问的位置 $`j\le i`$：

```math
\mathrm{score}_{hij}
=
\frac{q_{hi}^T k_{hj}}{\sqrt{d_h}}
-m_h(i-j)
```

其中，$`m_h>0`$ 是第 $`h`$ 个 head 的斜率，$`i-j`$ 是 Query 与 Key 的距离。ALiBi 的位置偏置就是：

```math
b_h(i-j)=-m_h(i-j)
```

在原始 ALiBi 中，$`m_h`$ 在训练前设定并保持固定，不通过反向传播学习。因此，每个 head 只需一个斜率，就能为任意距离生成偏置，不需要保存位置 embedding 或距离查找表。

假设某个 head 的 $`m_h=0.2`$，且几个 Key 的内容分数都是 $`3`$：

| 距离 $`i-j`$ | ALiBi 偏置 | 最终 logit |
| ---: | ---: | ---: |
| $`0`$ | $`0`$ | $`3.0`$ |
| $`1`$ | $`-0.2`$ | $`2.8`$ |
| $`2`$ | $`-0.4`$ | $`2.6`$ |
| $`3`$ | $`-0.6`$ | $`2.4`$ |

这给 Attention 一个明确的局部性先验：内容相关性相同时，距离近的 token 更容易得到较高权重。但它不是硬窗口；如果远处 token 的内容分数足够高，仍然可以抵消距离惩罚并获得较大注意力。

#### 3.2.2 不同 head 为什么使用不同斜率

如果所有 head 使用相同的斜率，它们会拥有相同的距离偏好。ALiBi 为不同 head 分配不同大小的 $`m_h`$，让多个 head 覆盖不同尺度：

- 较大的 $`m_h`$：远距离分数下降得更快，形成更局部的软感受野。
- 较小的 $`m_h`$：远距离惩罚较弱，更有机会保留跨句、跨段关系。

原始论文使用按几何级数排列的固定斜率。例如 8 个 head 可使用从 $`1/2`$ 到 $`1/256`$ 的一组斜率。斜率本身不学习，真正通过训练更新的是各个 head 的 $`W_Q`$、$`W_K`$、$`W_V`$ 等参数。

因此，更准确的理解是：**不同斜率先为各个 head 设置不同强度的距离归纳偏置，再让它们在各自的偏好下学习内容模式。**这种设计鼓励从局部到全局的多头分工，但不保证每个 head 一定形成某种固定语义角色。

#### 3.2.3 ALiBi 与 causal mask 的区别

Causal mask 决定“能不能看”。对未来位置 $`j>i`$，分数会被设为：

```math
\mathrm{score}_{hij}=-\infty
```

ALiBi 决定“允许看的位置中更倾向看哪里”。对当前位置和过去位置 $`j\le i`$，加入线性距离惩罚：

```math
\mathrm{score}_{hij}
=
\frac{q_{hi}^T k_{hj}}{\sqrt{d_h}}
-m_h(i-j)
```

所以，未来 token 被 causal mask 完全禁止；过去 token 都允许参与 Attention，只是越远通常需要越强的内容相关性来抵消 ALiBi 惩罚。

#### 3.2.4 为什么它能够用于长度外推

ALiBi 的偏置是距离的线性函数。即使推理时出现训练中没有见过的更大距离，也可以直接代入公式计算，不会超出有限的位置 embedding 表。分桶式 RPB 也能把新距离映射到最远的 bucket，但多个远距离会共享同一个粗粒度偏置；ALiBi 则会继续随距离线性增加惩罚。

但“公式可以计算新距离”不等于模型一定能可靠处理任意长上下文。序列长度增大后，内容分布、Attention 模式、训练数据和推理实现仍可能发生分布外变化。ALiBi 提供的是较简单的长度外推机制和距离归纳偏置，而不是完整的长上下文能力保证。

两种方法可以概括为：

| 对比 | Relative Position Bias | ALiBi |
| --- | --- | --- |
| 每个 head 的位置参数 | 一组可学习的距离或 bucket 偏置 | 一个固定斜率 $`m_h`$ |
| 偏置形状 | 可学习、可非线性、未必单调 | 线性且随距离单调减小 |
| 新距离的处理 | 依赖分桶、截断或额外参数 | 直接代入线性公式 |
| 主要归纳偏置 | 由训练数据决定 | 明确偏好近距离 |

### 3.3 纯相对位置方法的局限

这里的“纯相对位置”是指：位置机制只向模型提供 $`i-j`$ 或由它变换得到的距离信息，而不再单独提供位置 $`i`$、$`j`$ 的绝对坐标。它与 Attention 的两两比较过程很契合，但也有以下边界。

#### 3.3.1 缺少显式的绝对位置锚点

如果把 Query 和 Key 同时向后平移 $`c`$ 个位置，它们的相对距离不会改变：

```math
(i+c)-(j+c)=i-j
```

因此，仅观察相对距离的位置机制无法直接区分“这一对 token 原来位于第 2、5 位”还是“平移后位于第 12、15 位”。它更擅长表达“相距 3 个 token”，却不能直接表达“当前 token 就在序列第 12 位”“这是第一个位置”或“距离序列末尾还有多少步”。

这不等于整个模型绝对无法推断绝对位置。Causal mask、BOS/EOS、Padding、序列边界和 token 内容都可能间接提供锚点；更准确的说法是：**纯相对位置机制本身没有显式给出绝对坐标。**

[DeBERTa](https://arxiv.org/abs/2006.03654) 就体现了这种互补关系：它在 Transformer 层中使用相对位置信息，又在增强掩码解码器中补入绝对位置，以帮助区分局部上下文相似、但位于句子不同位置的 Mask token。

#### 3.3.2 标量偏置对同一距离使用同一位置先验

对前面介绍的 RPB 和 ALiBi，位置项可以统一写成：

```math
b_h(i-j)
```

在同一个 head 中，只要两个 Query-Key 对的相对距离相同，它们就会得到相同的位置偏置，不论对应的是局部语法关系、跨句引用还是普通相邻词。这意味着位置项本身不能根据 token 内容动态决定“这一次距离 5 很重要，另一次距离 5 不重要”。

不过，最终 Attention logit 仍然包含 $`q^Tk`$ 的内容分数，因此完整 Attention 依然可以根据内容做不同选择。这个局限主要针对 T5 式 RPB、ALiBi 这类**标量 logit 偏置**；[Shaw 等人的相对位置表示](https://arxiv.org/abs/1803.02155)会让相对位置向量与 Query 等内容表示发生交互，表达能力更强，但实现也更复杂。

#### 3.3.3 是否有方向性取决于距离如何定义

纯相对位置方法并不天然“没有方向性”。如果使用有符号距离 $`i-j`$，正负号就能区分两个方向；双向 Attention 中也可以让正、负距离进入不同 bucket。

如果只使用绝对距离：

```math
|i-j|=|j-i|
```

那么“前方 3 个位置”和“后方 3 个位置”会得到相同的位置表示，方向信息才会丢失。如果是因果 Attention，Causal mask 已经禁止访问未来位置，允许访问的相对方向只有当前和过去，因此方向还会被 mask 隐式限定。

所以，准确结论是：**纯相对位置缺少的是绝对坐标，不一定缺少方向；方向性由有符号距离、分桶方式和 Attention mask 共同决定。**

#### 3.3.4 相对表示不自动保证长度外推

相对位置不依赖固定的绝对位置编号，通常比可学习绝对位置表更容易处理新长度，但“能够计算新距离”仍不等于“能够可靠理解更长序列”。

- RPB 如果使用截断或分桶，超出训练范围的许多远距离可能全部落入最后一个 bucket，只保留粗粒度的“很远”。
- ALiBi 可以继续计算任意距离的线性偏置，但距离越远，负偏置也越大；某些需要远距离检索的任务可能要求内容分数克服很强的局部性先验。
- 模型在更长序列上还会遇到 Attention 分布、数据模式和推理长度的分布变化。

因此，长度外推是位置机制、训练长度、数据和任务共同作用的结果。相关实验证据也表明，不同相对位置方法在不同长度泛化任务上的表现并不一致，不能仅凭“使用了相对距离”就推出模型一定能够外推，参见[位置编码对长度泛化影响的系统研究](https://arxiv.org/abs/2305.19466)。

#### 3.3.5 表达能力与实现成本存在取舍

更丰富的相对位置表示可以让内容和位置发生更细致的交互，但通常需要额外的位置向量、投影或成对计算；更简单的标量偏置只需给 Attention logits 加一个数，计算和缓存更轻量，却只能调整注意力的高低，无法旋转或改变 Q/K 表示本身。

这个取舍也解释了 RPB、ALiBi 和后文 RoPE 的差别：

| 方法 | 位置表达能力 | 主要代价或约束 |
| --- | --- | --- |
| 相对位置向量 | 可与内容表示交互 | 参数、计算和实现更复杂 |
| RPB | 每个 head 可学习非线性的距离偏好 | 同一 bucket 共享标量偏置，远距离可能被合并 |
| ALiBi | 极简、可直接计算新距离 | 固定线性形式，强制施加局部性先验 |
| RoPE | 在 Q/K 点积中形成相对相位 | 长度外推仍受频率和训练范围影响 |

> [!warning]
> “纯相对位置”不等于“没有方向”，也不等于“模型不能获得任何绝对位置信息”。它只表示位置机制本身以相对距离为核心，没有直接给出绝对坐标；方向和绝对锚点还可能来自距离符号、Attention mask 与边界 token。

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
