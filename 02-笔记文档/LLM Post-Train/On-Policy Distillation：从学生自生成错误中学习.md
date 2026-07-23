---
type: paper-note
status: active
domain: LLM Post-Training
created: 2026-07-22
updated: 2026-07-23
aliases: [OPD, On-Policy Distillation, GKD, Generalized Knowledge Distillation]
tags: [LLM, Post-Training, Knowledge-Distillation, On-Policy, GKD]
---

# On-Policy Distillation：从学生自生成错误中学习

> 参考论文：[On-Policy Distillation of Language Models: Learning from Self-Generated Mistakes](https://arxiv.org/abs/2306.13649)
>
> - 作者：Rishabh Agarwal、Nino Vieillard、Yongchao Zhou、Piotr Stanczyk、Sabela Ramos、Matthieu Geist、Olivier Bachem
> - 会议：ICLR 2024
> - 机构：Google DeepMind、Mila、University of Toronto

> [!note]
> 论文提出 Generalized Knowledge Distillation（GKD）：让学生模型在自己生成的序列上接受教师模型的 Token 级分布监督，以减轻自回归生成中的训练—推理分布偏移；同时允许自由选择固定数据与 On-Policy 数据的比例，以及 Forward KL、Reverse KL、JSD 等分布差异。

## 1. 论文解决什么问题

知识蒸馏用容量更大的教师模型指导容量更小的学生模型，目标是在降低推理成本和内存占用的同时，尽可能保留教师能力。

论文认为，自回归语言模型的常见蒸馏方法主要存在两个问题。

### 1.1 固定训练序列造成训练—推理分布偏移

传统蒸馏通常依赖固定输出序列。这些序列可能来自：

- 数据集中的标准答案；
- 教师模型提前生成的答案。

训练期间，学生模型总是在这些固定序列的前缀上预测下一个 Token；推理期间，学生模型却必须基于自己先前生成的 Token 继续预测。

```text
训练：固定答案前缀 → Student 预测下一个 Token
推理：Student 自生成前缀 → Student 预测下一个 Token
```

一旦学生在早期生成了错误 Token，后续预测会进入训练数据没有覆盖的状态，错误还可能沿自回归过程逐步累积。

### 1.2 学生容量可能不足以覆盖教师分布

常用蒸馏目标是 Forward KL，它要求学生尽量覆盖教师分布中的所有模式。当学生容量显著小于教师时，学生可能无法完整表达教师的复杂分布，进而在低质量 Token 上分配概率。

论文因此希望同时解决两个问题：

1. 让训练前缀更接近学生推理时真正遇到的前缀；
2. 允许根据任务和学生容量选择不同的分布差异函数。

## 2. 核心思想

GKD 把自回归蒸馏视为带有交互式专家的模仿学习：

```text
Student 根据输入生成序列
           ↓
得到 Student 实际访问的前缀状态
           ↓
Teacher 在这些前缀上给出下一 Token 的完整概率分布
           ↓
Student 匹配 Teacher 的分布
```

这里的“从自身错误中学习”不表示学生把自己生成的 Token 当作正确标签。学生生成的序列用于确定训练轨迹，真正的监督信号仍来自教师模型。

随着训练推进，学生的策略不断变化，Student 生成的训练序列也随之变化。教师因此能够持续针对当前学生真正会进入的状态提供反馈。

## 3. 预备知识

### 3.1 自回归生成

设输入序列为 $`x`$，输出序列为：

```math
y=(y_1,y_2,\ldots,y_{L_y})
```

词表为 $`V`$，包含 $`M`$ 个 Token。在生成第 $`n`$ 个 Token 时，模型根据输入 $`x`$ 和已有前缀 $`y_{1:n-1}`$，输出整个词表上的概率分布：

```math
p(\cdot\mid y_{1:n-1},x)\in(0,1)^M
```

论文把 $`p(y_n\mid y_{1:n-1},x)`$ 简写为 $`p(y_n\mid x)`$。完整序列由模型逐 Token 采样得到：

```math
y\sim p(\cdot\mid x)
```

模型通过带温度 $`\gamma`$ 的 Softmax 将 Logit 转换为概率：

```math
p(y_n\mid x)=\frac{\exp(z_n/\gamma)}{\sum_{i=1}^{M}\exp(z_i/\gamma)}
```

温度越高，概率分布越平坦，生成随机性越强；温度越低，模型越倾向选择最大概率 Token。论文在训练中保持学生温度为 1，评估时使用 Greedy Sampling 或 Temperature Sampling。

### 3.2 Token 级分布差异

设教师模型为 $`p_T`$，带参数 $`\theta`$ 的学生模型为 $`p_S^\theta`$。给定同一个输入和输出前缀，第 $`n`$ 个位置上的教师、学生分布分别为：

```math
p_T(\cdot\mid y_{1:n-1},x)
```

```math
p_S^\theta(\cdot\mid y_{1:n-1},x)
```

对输出序列 $`y`$，论文把序列级差异定义为所有位置 Token 级差异的平均：

```math
D(p_T\Vert p_S^\theta)(y\mid x)
=
\frac{1}{L_y}
\sum_{n=1}^{L_y}
D\left(
p_T(\cdot\mid y_{1:n-1},x)
\Vert
p_S^\theta(\cdot\mid y_{1:n-1},x)
\right)
```

这里比较的是每个位置上的完整词表概率分布，不只是 Teacher 和 Student 最终选中的 Token。

## 4. 论文中的蒸馏基线

### 4.1 Supervised Fine-Tuning

当只有固定的输入—输出数据 $`(X,Y)`$，但无法查询教师模型时，可以最大化固定答案在学生模型下的似然：

```math
L_{\mathrm{SFT}}(\theta)
=
\mathbb{E}_{(x,y)\sim(X,Y)}
\left[-\log p_S^\theta(y\mid x)\right]
```

SFT 使用固定答案中的目标 Token 作为 Hard Label，不使用教师的完整词表分布。

### 4.2 Sequence-Level Knowledge Distillation

SeqKD 先让教师模型生成高概率输出序列，再把这些序列作为固定答案训练学生。对于学生而言，后续优化过程与在教师合成数据上执行 SFT 相同。

SeqKD 传递的是教师最终生成的序列，没有保留教师在每个位置上的完整 Token 概率分布。

### 4.3 Supervised Knowledge Distillation

Supervised KD 使用固定输出序列提供前缀，并在这些前缀上让学生匹配教师的 Token 级分布。论文写作：

```math
L_{\mathrm{SD}}(\theta)
=
\mathbb{E}_{(x,y)\sim(X,Y)}
\left[
D_{\mathrm{KL}}
\left(
p_T\Vert p_S^\theta
\right)(y\mid x)
\right]
```
也就是在每个token上，计算完整词表分布 KL，然后对输出序列的所有有效 Token 位置取平均。

它比 SFT 和 SeqKD 提供更丰富的 Soft Label，但训练轨迹仍由固定数据决定，因此没有消除训练—推理状态分布偏移。

## 5. Generalized Knowledge Distillation

### 5.1 On-Policy KD

On-Policy KD 不使用固定输出序列，而是先让当前学生模型生成：

```math
y\sim p_S(\cdot\mid x)
```

然后在学生生成序列的每个中间前缀 $`y_{1:n-1}`$ 上，计算教师与学生的 Token 级分布差异。采用 Forward KL 时，On-Policy Loss 为：

```math
L_{\mathrm{OD}}(\theta)
=
\mathbb{E}_{x\sim X}
\mathbb{E}_{y\sim p_S(\cdot\mid x)}
\left[
D_{\mathrm{KL}}
\left(
p_T\Vert p_S^\theta
\right)(y\mid x)
\right]
```

论文不通过学生的离散采样过程反向传播。Student 生成的序列在本次更新中被当作固定上下文，梯度仅来自教师与学生 Token 分布之间的差异。作者认为这样可以提高训练稳定性和计算效率。

训练生成时，论文使用 $`\gamma=1`$ 的采样温度，以提高学生轨迹的多样性。

### 5.2 GKD 的混合目标

GKD 进一步统一固定数据和 On-Policy 数据，并允许使用任意分布差异 $`D`$：

```math
L_{\mathrm{GKD}}(\theta)
=
(1-\lambda)
\mathbb{E}_{(x,y)\sim(X,Y)}
\left[D(p_T\Vert p_S^\theta)(y\mid x)\right]
+
\lambda
\mathbb{E}_{x\sim X}
\mathbb{E}_{y\sim p_S(\cdot\mid x)}
\left[D(p_T\Vert p_S^\theta)(y\mid x)\right]
```

$`\lambda`$ 表示 Student Data Fraction，即 Student 自生成序列在训练数据中的比例：

| $`\lambda`$ | 轨迹来源 | 对应方法 |
| ---: | --- | --- |
| 0 | 全部来自固定数据 | Supervised GKD；采用 Forward KL 时对应 Supervised KD |
| 0.5 | 固定序列与 Student 序列混合 | Mixed GKD |
| 1 | 全部来自 Student 自生成序列 | On-Policy GKD |

因此，论文标题强调的 On-Policy Distillation 是 GKD 框架中 $`\lambda=1`$ 的重要实例；GKD 本身是更一般的统一框架。

### 5.3 Algorithm 1

论文的训练算法可以概括为：

```text
输入：冻结的 Teacher、可训练的 Student、数据集 (X,Y)
超参数：Student Data Fraction λ、差异函数 D、学习率 η

对每个训练 Step：
1. 采样 u ~ Uniform(0,1)。
2. 如果 u <= λ：
   - 从 X 采样 Prompt x；
   - 使用当前 Student 生成 y；
   - 组成 On-Policy Batch。
3. 否则：
   - 从固定数据集 (X,Y) 采样 Batch。
4. Teacher 和 Student 在相同的 y 前缀上计算完整词表分布。
5. 对每个 Token 位置计算 D，并对序列和 Batch 求平均。
6. 只更新 Student 参数 θ。
```

论文假设 Student 已经能够生成质量足以接受教师反馈的序列。因此实验不是从随机初始化开始，而是从经过 SFT 的学生检查点开始。这与 RLHF 常见的“先 SFT，再在线优化”流程类似。

## 6. 散度选择

### 6.1 Forward KL

设教师分布为 $`P`$、学生分布为 $`Q`$：

```math
D_{\mathrm{KL}}(P\Vert Q)
=
\sum_c P(c)\log\frac{P(c)}{Q(c)}
```

Forward KL 要求学生覆盖教师分布支持的不同模式，具有 Mode-Covering 倾向。论文指出，当学生容量不足时，完整覆盖教师分布可能使学生在教师低概率 Token 上分配概率，从而影响采样质量。

### 6.2 Reverse KL

```math
D_{\mathrm{KL}}(Q\Vert P)
=
\sum_c Q(c)\log\frac{Q(c)}{P(c)}
```

Reverse KL 更关注学生已经分配概率的区域，鼓励学生集中到教师高概率模式，具有 Mode-Seeking 倾向。它可能减少低质量生成，但通常会降低输出多样性。

### 6.3 Generalized JSD

论文使用广义 Jensen-Shannon Divergence 在 Forward KL 与 Reverse KL 之间插值。定义混合分布：

```math
M_\beta=\beta P+(1-\beta)Q
```

则：

```math
D_{\mathrm{JSD}(\beta)}(P\Vert Q)
=
\beta D_{\mathrm{KL}}(P\Vert M_\beta)
+
(1-\beta)D_{\mathrm{KL}}(Q\Vert M_\beta)
```

当 $`\beta`$ 接近 0 时，其梯度行为接近 Forward KL；当 $`\beta`$ 接近 1 时，其梯度行为接近 Reverse KL。论文实验比较了 JSD(0.1)、JSD(0.5) 和 JSD(0.9)。

论文没有给出适用于所有任务的最佳散度，而是认为选择取决于：

- Student 与 Teacher 的容量差异；
- 任务需要的输出多样性；
- 推理时使用 Greedy Sampling 还是 Temperature Sampling。

## 7. 与强化学习结合

On-Policy GKD 和 RL 都可以使用 Student 自己生成的序列，因此可以共享 Rollout，但它们提供的监督信号不同：

- RL 用标量奖励评价完整序列，告诉 Student“这条结果整体好不好”；
- GKD 在每个前缀上比较完整词表分布，告诉 Student“这一步应当怎样接近 Teacher”。

论文考虑同时最大化标量奖励 $`r(y)`$ 并匹配教师分布：

```math
\mathbb{E}_{x\sim X}
\left[
(1-\alpha)
\mathbb{E}_{y\sim p_S^\theta(\cdot\mid x)}[r(y)]
-
\alpha
\mathbb{E}_{y\sim p_S(\cdot\mid x)}
[D(p_T\Vert p_S^\theta)(y\mid x)]
\right]
```

$`\alpha`$ 控制蒸馏目标相对 RL 奖励的强度：

- $`\alpha=0`$ 时只优化 RL 奖励；
- $`\alpha=1`$ 时只执行蒸馏；
- $`0<\alpha<1`$ 时在奖励与教师能力之间折中。

### 7.1 一次联合训练怎样进行

对于输入 $`x`$，当前 Student 先生成一条序列：

```math
y\sim p_S^\theta(\cdot\mid x)
```

同一条 Rollout 随后走向两条计算路径：

```text
                         ┌─ Reward Model / 奖励函数 → RL Loss
Student Rollout：x → y ─┤
                         └─ Teacher Token 分布 → GKD Loss
```

具体过程是：

1. 奖励函数对完整输出 $`y`$ 打分，RL 使用策略梯度提高高奖励序列的生成概率；
2. Teacher 与 Student 读取 $`y`$ 的相同前缀，逐位置计算 KL 或 JSD；
3. 将 RL Loss 与 GKD Loss 加权求和，只更新 Student。

从最小化损失的角度，可以简写为：

```math
L_{\mathrm{joint}}
=
(1-\alpha)L_{\mathrm{RL}}
+
\alpha L_{\mathrm{GKD}}
```

RL 与 GKD 对离散 Rollout 的处理不同。RL 需要用 Reward 或 Advantage 形成策略梯度；GKD 不对采样动作反向传播，而是把生成完成的 $`y`$ 当作固定前缀，直接对 Student 的 Token 分布反向传播。二者虽然共享 Rollout，产生梯度的方式并不相同。

### 7.2 为什么要与 RL 结合

OPD 主要解决“如何在 Student 真正访问的状态上模仿 Teacher”。它能转移 Teacher 已有的知识和生成行为，却不能直接保证模型获得 Teacher 本身没有充分表现出来的属性，例如更强的事实一致性、安全性或人类偏好。RL 可以把这些目标写成标量奖励，直接提高高奖励行为的生成概率。

但只使用 RL 也有风险：序列级奖励比较稀疏，Student 可能为了提高单一指标而出现 Reward Hacking、输出退化，或者损失原有任务能力。OPD 的逐 Token 分布监督更加稠密，可以在 RL 改变策略的同时持续把 Student 拉回 Teacher 认可的行为区域。

| 方法 | 提供的信号 | 主要作用 | 为模型保留或增强的能力 |
| --- | --- | --- | --- |
| RL | 完整序列的标量 Reward / Advantage | 将生成策略推向外部目标 | 事实性、安全性、偏好符合度等目标行为 |
| OPD | 每个前缀上的 Teacher 完整 Token 分布 | 提供细粒度指导和能力锚点 | Teacher 的语言能力、任务质量和生成稳定性 |
| 联合训练 | 两种信号的加权组合 | 在目标优化与能力保持之间折中 | 在增强目标行为的同时减少能力退化 |

#### OPD 如何缓解 Reward Hacking

假设 RL 奖励只衡量摘要是否存在事实错误，Student 可能学会生成极短、几乎没有信息的摘要：它不容易犯错，因而能取得高奖励，却不是真正高质量的摘要。

OPD 会在这条 Student Rollout 的实际前缀上继续查询 Teacher。如果 Student 强烈倾向提前输出结束 Token，而 Teacher 仍然倾向生成有效信息，两者的 KL 或 JSD 就会增大，联合损失会阻止 Student 仅靠偏离正常生成行为来提高奖励。由于监督发生在 Student 真正采用的投机路径上，On-Policy 蒸馏比只约束固定答案前缀更容易覆盖新出现的 Reward Hacking 状态。

不过，OPD 只能缓解而不能保证消除 Reward Hacking。如果 Teacher 也采用相同的投机行为、投机策略仍位于 Teacher 的高概率区域，或者蒸馏权重过低，OPD 仍可能无法形成有效约束。

两者还可以复用同一批 Student Rollout：RL 用它计算序列奖励，OPD 用它构造 Teacher 与 Student 的分布匹配状态，因此不需要为两个目标分别生成训练序列。需要注意，RL 主要是在奖励定义的方向上重塑模型行为，并不会凭空注入奖励模型和训练数据中不存在的知识。

若目标只是模仿 Teacher，并没有额外奖励需要优化，就没有必要加入 RL。在 XSum 实验中，作者把基于文本蕴含的事实性奖励与 GKD 结合，在摘要质量和事实一致性之间形成由 $`\alpha`$ 控制的权衡。

## 8. 实验设置

### 8.1 Teacher 与 Student

论文主要使用同一 T5 模型家族：

| 角色 | 模型 | 参数量 |
| --- | --- | ---: |
| Teacher | T5-XL | 约 3B |
| Student | T5-Small | 77M |
| Student | T5-Base | 250M |
| Student | T5-Large | 800M |

Teacher 与三种 Student 的参数规模比分别约为 38 倍、12 倍和 3.8 倍。所有方法都从同一个经过监督微调的 Student 检查点开始。

### 8.2 GKD 变量与基线

论文比较：

- 散度：Forward KL、Reverse KL、JSD(0.1)、JSD(0.5)、JSD(0.9)；
- Student Data Fraction：$`\lambda=0`$、$`0.5`$、$`1`$；
- 基线：SeqKD、Supervised KD、ImitKD、f-distill。

ImitKD 和 f-distill 可以分别看作采用 Forward KL 与 Total Variation Distance、且使用混合数据的特定 GKD 实例。

## 9. 实验结果

### 9.1 XSum：抽象式摘要

论文在 XSum 上用 ROUGE-2 评价摘要质量，主要结论为：

- On-Policy GKD 在不同 Student 尺寸上持续优于 SeqKD 和 Supervised KD；
- On-Policy GKD with JSD(0.9) 在 Greedy Sampling 与 Temperature Sampling 下均优于论文比较的额外基线；
- 只使用 5% XSum 数据的 On-Policy GKD，不使用 Ground-Truth 摘要，也超过了使用完整训练集的 Supervised KD 和 ImitKD；
- On-Policy 与 Mixed 变体通常优于只使用固定数据的 Supervised 变体；
- 从 Forward KL 逐渐过渡到 Reverse KL，生成多样性下降，Mode-Seeking 散度在高温采样时通常获得更高质量；温度下降后，不同散度之间的质量差距缩小。

论文还在 XSum 上将 On-Policy GKD 与基于文本蕴含奖励的 RLAIF 结合。结果显示，该组合能在提高摘要质量的同时显著改善事实一致性，并形成由 $`\alpha`$ 控制的质量—事实性权衡。

### 9.2 WMT14 En-De：机器翻译

论文用 BLEU 评价英语到德语的翻译质量，主要结论为：

- 纯 On-Policy 数据和 Mixed 数据持续优于只使用固定数据的 GKD；
- 只使用 Student 自生成序列的变体整体表现最好；
- 广义 JSD 通常优于 Forward KL 或 Reverse KL；
- Student 规模增大后，不同散度之间的性能差距缩小。

### 9.3 GSM8K：算术推理

论文用经过 SFT 的 FLAN-T5 系列作为起点，通过四个 Few-Shot CoT 示例提示模型，并用外部计算器检查最终答案。主要结论为：

- 只使用 Student 自生成 CoT 的 On-Policy 训练优于固定 CoT 数据或混合数据；
- 当 On-Policy 数据比例超过 25% 后，性能通常随其比例增加而提高；
- On-Policy GKD 在不同 Student 尺寸上优于 SeqKD、Supervised KD、ImitKD 和 f-distill；
- Forward KL 和 Reverse KL 在该任务上都表现良好；
- 附录中的 Self-Distillation 实验也显示 On-Policy GKD 优于监督式方法。

### 9.4 FLAN：任务无关的指令蒸馏

论文用 FLAN T5-XL 蒸馏 FLAN T5-Base，训练数据为包含 536 万样本、覆盖 62 个任务的 FLAN2021，并在训练中未出现的 MMLU 与 BBH 上进行 Few-Shot 评估。

主要结论为：

- On-Policy GKD with Reverse KL 显著优于 Supervised KD 与 ImitKD；
- Reverse KL 明显优于 Forward KL；
- 作者推测，Reverse KL 的 Mode-Seeking 特性有助于模型聚焦指令要求的核心意图和行为。

论文报告，相比初始 Student，On-Policy GKD 在 Held-Out 评估上带来约 2 个百分点的 BBH 绝对准确率提升和约 1 个百分点的 MMLU 绝对准确率提升。

### 9.5 汇总结果

论文将 On-Policy GKD 相对初始 Student 的性能增益，与常用 KD 基线的性能增益进行比较；跨不同尺寸的 T5 Student 平均后，报告：

- 摘要任务的相对增益约为基线的 2.1 倍；
- 机器翻译任务约为 1.7 倍；
- 算术推理任务约为 1.9 倍。

这些数字表示“相对初始 Student 的改进幅度之比”，不是任务指标本身直接提升了 2.1 倍、1.7 倍或 1.9 倍。

## 10. 计算成本与适用前提

### 10.1 额外计算

On-Policy GKD 需要：

1. Student 在线生成训练序列；
2. Teacher 在 Student 前缀上计算 Token 级分布；
3. Student 再执行用于反向传播的前向计算。

论文在 GSM8K 上报告，相比从固定输出数据采样，Student Rollout 带来的计算开销约为：

| Teacher/Student 参数规模比 | 额外开销倍率 |
| ---: | ---: |
| 38 倍 | 1.8 倍 |
| 12 倍 | 2.0 倍 |
| 3.8 倍 | 2.2 倍 |

Student 越接近 Teacher 尺寸，在线生成在整体成本中的占比越高。论文认为，在大量真实请求带来的长期推理成本面前，蒸馏阶段的额外训练成本可能是值得的。

### 10.2 适用前提与论文边界

- Student 需要先经过 SFT，具备生成可接受序列的能力；
- 训练期间需要查询 Teacher 的 Token 级概率，而不只是 Teacher 的最终答案；
- On-Policy 数据会随着 Student 更新而变化，不能完全离线预生成；
- 论文实验主要基于同一 T5/FLAN-T5 模型家族，未验证跨 Tokenizer、跨模型家族蒸馏；
- 最佳散度依赖任务、Student 容量和推理解码方式；
- On-Policy GKD 改善的是 Student 实际访问状态上的模仿学习，但不能消除 Teacher 自身错误或幻觉；
- 论文不通过采样过程反向传播，因此 GKD 本身不是策略梯度方法。

## 11. 论文的核心结论

1. 自回归模型的蒸馏不仅要考虑“模仿什么分布”，还要考虑“在哪些前缀状态上模仿”。
2. 使用 Student 自生成序列可以缩小训练前缀与推理前缀之间的分布差异。
3. 固定数据蒸馏和 On-Policy 蒸馏可以通过 $`\lambda`$ 统一在 GKD 框架中。
4. Forward KL、Reverse KL 与 JSD 对质量和多样性的影响不同，不存在对所有任务都最优的固定选择。
5. On-Policy GKD 可以与 RL/RLAIF 共享 Student Rollout，在优化标量奖励的同时维持教师能力。
6. 论文在摘要、翻译、算术推理与任务无关指令蒸馏中，均观察到 On-Policy 数据相对固定数据的优势。

> [!important]
> GKD 的关键不是把 Student 自己生成的 Token 当作训练标签，而是使用 Student 生成的序列确定训练状态，再让 Teacher 在这些状态上提供完整 Token 分布监督。

## 12. 深入理解 Forward KL 与 Reverse KL

前面的第 6 节给出了两种 KL 的定义，本节进一步解释：它们究竟在关注什么、为什么会表现出不同的训练倾向，以及这种差异在 GKD 中有什么作用。

可以先这样概括：

- **Forward KL 更强调 Teacher 的完整要求。** 它按 Teacher 概率加权，推动 Student 在 Teacher 认可的各个方向上都尽量贴近 Teacher，即使其中一些模式对小模型较难学习；
- **Reverse KL 更强调 Student 当前能够表达的区域。** 它按 Student 概率加权，允许容量有限的 Student 集中选择自己能够覆盖、同时又得到 Teacher 认可的模式，而不必勉强覆盖 Teacher 的全部模式。

不过，Reverse KL 并不会显式判断 Student 的能力上限。上述“在能力范围内满足 Teacher”是它在容量受限时常表现出的优化倾向：Student 几乎没有覆盖的区域权重较小，而 Student 已经投入较多概率的区域会被重点检查是否符合 Teacher。

### 12.1 先固定一个前缀来看

在某个输入 $`x`$ 和前缀 $`y_{1:n-1}`$ 上，记教师、学生对下一个 Token 的完整词表分布为：

```math
P(c)=p_T(c\mid y_{1:n-1},x)
```

```math
Q(c)=p_S^\theta(c\mid y_{1:n-1},x)
```

其中 $`c`$ 遍历词表中的所有 Token。Forward KL 与 Reverse KL 比较的是同一个前缀下的这两个完整分布，不是只比较教师或学生最终采样到的那个 Token。

两者的全局最优解其实相同：只要学生能够无约束地表示教师分布，最优结果都是 $`Q=P`$。它们的区别主要出现在学生容量有限、训练时间有限，或者学生无法同时拟合教师所有模式时。

### 12.2 Forward KL：由教师分布决定关注重点

Forward KL 写作：

```math
D_{\mathrm{KL}}(P\Vert Q)
=
\sum_c P(c)\log\frac{P(c)}{Q(c)}
```

求和中每一项由教师概率 $`P(c)`$ 加权。因此，教师认为重要的 Token 会得到更大的训练权重。

由于教师固定，第一项与学生参数无关，最小化 Forward KL 等价于最小化教师 Soft Label 对学生的交叉熵：

```math
D_{\mathrm{KL}}(P\Vert Q)
=
\sum_c P(c)\log P(c)
-
\sum_c P(c)\log Q(c)
```

在这个前缀上，Student 会先为整个词表输出一组未经 Softmax 的 Logit：

```math
z=(z_1,z_2,\ldots,z_M)
```

其中 $`z_c`$ 是词表中 Token $`c`$ 对应的一个标量分数，经过 Softmax 后得到：

```math
Q(c)=\frac{\exp(z_c)}{\sum_j\exp(z_j)}
```

$`z_c`$ 不是一个独立的模型参数，而是 Student 在当前前缀上的输出。损失对它的偏导数表示：如果稍微改变 Token $`c`$ 的 Logit，损失会怎样变化。Forward KL 对 $`z_c`$ 的梯度具有非常直观的形式：

```math
\frac{\partial D_{\mathrm{KL}}(P\Vert Q)}{\partial z_c}
=
Q(c)-P(c)
```

这意味着：

- 学生概率低于教师概率时，梯度为负，梯度下降会提高该 Token 的 Logit；
- 学生概率高于教师概率时，梯度为正，梯度下降会降低该 Token 的 Logit；
- 只要教师给某个 Token 分配了明显概率，学生就很难完全忽略它。

反向传播还会继续通过链式法则，把对所有 Logit 的梯度传回 Student 参数 $`\theta`$。由于 Softmax 将整个词表耦合在一起，改变一个 Logit 也会相对改变其他 Token 的概率。

因此，Forward KL 常被称为具有 **Mode-Covering** 倾向：它希望学生覆盖教师支持的多种合理输出。它通常更有利于保留教师的分布信息和生成多样性，但容量较小的学生也可能被迫把有限概率分散到多个模式上。

> [!note]
> “Mode-Covering”不是说 Forward KL 的最优结果一定比教师更分散，而是说：当学生无法精确表示教师的多个模式时，Forward KL 对漏掉教师模式的惩罚通常更强。

### 12.3 Reverse KL：由学生当前分布决定关注重点

Reverse KL 交换了两个分布的位置：

```math
D_{\mathrm{KL}}(Q\Vert P)
=
\sum_c Q(c)\log\frac{Q(c)}{P(c)}
```

这次每一项由学生概率 $`Q(c)`$ 加权。学生当前愿意生成的 Token 得到较大权重，而学生几乎不生成的 Token 对损失的直接影响也较小。

它对学生 Logit 的梯度为：

```math
\frac{\partial D_{\mathrm{KL}}(Q\Vert P)}{\partial z_c}
=
Q(c)
\left(
\log\frac{Q(c)}{P(c)}
-D_{\mathrm{KL}}(Q\Vert P)
\right)
```

式子外部的 $`Q(c)`$ 很关键：如果学生已经给某个 Token 极低概率，该 Token 产生的直接梯度也会很小。相反，如果学生把大量概率放在教师认为不合理的 Token 上，$`Q(c)/P(c)`$ 会很大，损失会强烈推动学生撤回这部分概率。

因此，Reverse KL 常被称为具有 **Mode-Seeking** 倾向：当教师分布存在多个模式、而学生无法全部覆盖时，学生可以集中拟合其中一个高概率模式，而不是勉强覆盖所有模式。

它通常有助于：

- 压低学生已经在生成、但教师认为质量较低的 Token；
- 让输出分布更尖锐，集中到少数高置信度模式；
- 在学生容量明显小于教师时，减少“什么都学一点但都学不好”的情况。

代价是学生可能忽略教师分布中的次要但仍然合理的模式，从而降低生成多样性。

### 12.4 一个直观例子

假设在同一个前缀后，教师认为三个候选 Token 都有不同程度的合理性：

| Token | 教师概率 $`P`$ | 含义 |
| --- | ---: | --- |
| A | 0.60 | 最主要的表达 |
| B | 0.35 | 次要但仍合理的表达 |
| C | 0.05 | 低概率表达 |

如果学生容量有限，难以同时表示 A 和 B 两种模式：

- Forward KL 会持续惩罚学生漏掉 B，因为 B 在教师分布中仍有 0.35 的明显概率；
- Reverse KL 更允许学生主要集中在 A，只要学生不要把大量概率放到教师低概率的 C 上；
- 因此前者倾向“把合理答案尽量都覆盖到”，后者倾向“选定一个高质量答案并集中做好”。

这只是帮助理解优化行为的简化例子。在真实语言模型中，比较发生在整个词表上，并且会在序列的每个有效 Token 位置重复进行。

### 12.5 两种 KL 在 GKD 中控制的不是同一件事

GKD 同时包含两个容易混淆的选择：

1. **训练前缀从哪里来**：由 $`\lambda`$ 决定使用固定数据、Student On-Policy 数据，或二者混合；
2. **在每个前缀上怎样匹配 Teacher**：由 Forward KL、Reverse KL 或 JSD 决定。

可以把二者理解为：

```text
On-Policy / Supervised：决定“到哪些状态上学习”
Forward KL / Reverse KL：决定“在每个状态上重点修正什么”
```

因此，On-Policy 不等于 Reverse KL。完全可以在学生自生成前缀上使用 Forward KL，也可以在固定前缀上使用 Reverse KL。论文的 GKD 框架就是把这两个维度解耦后统一研究。

在 On-Policy GKD 中，Student Rollout 已经让训练聚焦于学生真实访问的前缀；进入某个前缀后：

- Forward KL 再要求学生尽可能覆盖 Teacher 在该状态下给出的完整分布；
- Reverse KL 更重点检查学生当前高概率区域是否得到 Teacher 支持，并把概率推向 Teacher 的高概率模式。

### 12.6 与解码方式的关系

散度的选择会影响学生分布的形状，而推理解码方式决定如何从这个分布中取出 Token。

- **Greedy Sampling** 每次只选择最高概率 Token。只要最高概率项一致，分布尾部的差异不一定会明显反映到最终输出中；
- **Temperature Sampling** 会实际采样其他候选 Token。此时分布尾部是否包含低质量候选、概率是否过度分散，会更直接影响生成质量；
- Reverse KL 或偏向 Reverse KL 的 JSD 往往生成更集中的分布，因此在温度采样下可能更有优势，但多样性通常更低；
- Forward KL 更强调覆盖教师分布，在需要多样表达时更有吸引力，但小模型未必有足够容量忠实覆盖所有模式。

论文实验也表明，不存在跨任务统一最优的散度：XSum、WMT、GSM8K 和 FLAN 上的最佳选择并不完全相同，而且结果还会受到学生尺寸与评估解码方式影响。

### 12.7 为什么还需要 JSD

Forward KL 与 Reverse KL 分别代表两种侧重点，但实际任务可能既不希望学生漏掉过多合理模式，也不希望学生在低质量尾部分配过多概率。Generalized JSD 提供了连续的折中方式：

- 更接近 Forward KL 时，增强对 Teacher 多种模式的覆盖；
- 更接近 Reverse KL 时，增强对 Student 当前高概率区域的约束；
- 中间取值则在质量集中度与分布覆盖度之间折中。

论文使用 $`\beta`$ 调整这种倾向，但 $`\beta`$ 不是“越大越好”或“越小越好”的性能旋钮，仍需结合任务、学生容量和推理解码方式验证。

### 12.8 实际选择时可以怎样判断

| 情况 | 更值得优先尝试的散度 | 原因 |
| --- | --- | --- |
| 希望保留 Teacher 的多种合理表达 | Forward KL 或偏 Forward 的 JSD | 更重视覆盖 Teacher 支持的模式 |
| Student 远小于 Teacher，难以覆盖完整分布 | Reverse KL 或偏 Reverse 的 JSD | 允许集中学习少数高概率模式 |
| 推理主要采用 Temperature Sampling，担心低质量尾部 | Reverse KL 或偏 Reverse 的 JSD | 更强地压低 Student 已占据的 Teacher 低概率区域 |
| 任务输出较确定，例如部分推理或分类式生成 | Reverse KL、Forward KL 都应实测 | 多样性价值较低，但不同任务的优化表现仍可能不同 |
| 尚不清楚任务更需要覆盖还是集中 | JSD 中间值并进行消融 | 在两种倾向之间建立可调折中 |

> [!warning]
> “Forward KL 保多样性、Reverse KL 保质量”只是常见优化倾向，不是无条件成立的结论。两者在分布可完全拟合时具有相同最优点；真实差异来自模型容量、参数化方式、有限训练过程、数据状态分布和解码策略的共同作用。

这一节最需要记住的是：**Forward KL 主要由 Teacher 的概率加权，担心 Student 漏掉 Teacher 认可的模式；Reverse KL 主要由 Student 的概率加权，担心 Student 把概率放在 Teacher 不认可的区域。**
