---
type: concept
status: active
domain: 推理与部署
created: 2026-07-22
updated: 2026-07-22
aliases: [KV Cache, Key-Value Cache, 键值缓存]
tags: [LLM, Transformer, Attention, KV-Cache, Prefill, Decode]
---

# KV Cache：大模型推理加速的核心机制

> 参考出处：[小红书笔记《KV Cache：大模型推理加速的核心机制》](https://www.xiaohongshu.com/explore/69d3dfda0000000022001cd8?xsec_token=AB69TlzuSCI0dsbZQBPqkyqrHoaUv3FyFGGZ7ZooaPCCA=&xsec_source=pc_user)
>
> - 作者：无敌嘉然大王
> - 发布时间：2026-04-07
> - 标签：大模型、KV Cache、Transformer、算法

> [!note]
> 在自回归推理中，把历史 token 的 Key 和 Value 缓存起来复用，避免每一步都把前文重新计算一遍。

## 1. 是什么，以及为什么需要

KV Cache 是 Transformer 推理阶段用于加速生成的一种缓存机制。核心想法很简单：已经算过的历史 token 的 K 和 V，不要重复计算。

大语言模型生成文本时采用自回归方式——一次只生成一个 token：

```text
生成第 2 个 token → 参考第 1 个
生成第 3 个 token → 参考前 2 个
生成第 N 个 token → 参考前 N-1 个
```

在 Self-Attention 中，每个 token 会生成三组向量：

| 向量 | 角色 |
| --- | --- |
| Q（Query） | “我想查什么” |
| K（Key） | “我能被匹配到什么” |
| V（Value） | “我携带的实际信息” |

当前 token 用自己的 Q 与历史 token 的 K 做匹配，再对历史 V 加权求和。所以历史 token 的 K/V 会被反复使用。

每生成一个新 token，如果没有 KV Cache，都要把前面整段序列重新过一遍来重新计算历史 K/V——这就是纯粹的重复计算。KV Cache 的做法是把历史 K/V 保存下来直接复用，只为新 token 计算新的 Q/K/V。

> [!important]
> KV Cache 解决的不是“Attention 很慢”，而是自回归生成时对历史前缀的重复编码太浪费。

## 2. 缓存了什么，以及生成时如何工作

KV Cache 只缓存 K 和 V，不缓存 Q。这不是随意的选择，而是由它们在 Attention 中的不同角色决定的。

在自回归 Decode 中，生成第 $`t`$ 个 token 时要做的是：

1. 得到新位置的 hidden state；
2. 用它投影出当前这一个位置的 $`Q_t`$、$`K_t`$、$`V_t`$；
3. 用 $`Q_t`$ 和历史所有位置的 $`K_1,K_2,\ldots,K_t`$ 做 Attention；
4. 再对对应的 V 加权求和，得到当前输出。

所以这一步真正参与“查询”的，只有当前新位置的 $`Q_t`$。历史位置的 $`Q_1,Q_2,\ldots,Q_{t-1}`$ 是之前各步生成自己输出时用过的，一旦那一步结束，就不会再被后续步骤重复使用。

相反，历史位置的 K 和 V 会被后面每一个新 token 反复访问，所以它们值得缓存：

- Q 是当前步骤的一次性查询向量，用完即弃；
- K/V 是会被未来很多步重复读取的历史记忆，值得缓存。

### 每次 Decode 只新增一行

在第 $`t`$ 步，不会重新计算历史 token 的 $`Q_1\sim Q_{t-1}`$，也不会重新计算历史的 $`K_1\sim K_{t-1}`$、$`V_1\sim V_{t-1}`$，因为它们已经在 Cache 中。只计算新 token 对应的一行：$`Q_t`$、$`K_t`$、$`V_t`$。

然后把 $`K_t`$、$`V_t`$ 追加进 Cache，执行：

```math
\mathrm{Attn}\left(Q_t,[K_1,\ldots,K_t],[V_1,\ldots,V_t]\right)
```

从矩阵视角看：Q 只取当前新增的一行，K/V 是历史缓存加上新增的一行，不是整个 Q 矩阵全部重算。

### 训练与推理的计算模式

训练时，输入整段序列 $`X\in\mathbb{R}^{T\times d}`$，一次性算出完整的 Q/K/V 矩阵：

```math
Q=XW_Q,\qquad K=XW_K,\qquad V=XW_V
```

推理时使用 KV Cache，就变成增量模式——只对新位置 $`x_t`$ 做投影：

```math
Q_t=x_tW_Q,\qquad K_t=x_tW_K,\qquad V_t=x_tW_V
```

然后：

```math
K_{\mathrm{all}}=[K_{\mathrm{cache}},K_t],\qquad
V_{\mathrm{all}}=[V_{\mathrm{cache}},V_t]
```

### 历史隐藏状态也不重算

使用 KV Cache 后，模型每一层在当前 step 只处理新 token 的那条路径，逐层向上算出当前 token 在各层的表示，再投影出该层的 $`Q_t`$、$`K_t`$、$`V_t`$。历史 token 在各层对应的 K/V 已经缓存好了，不需要整段回放。

可以把整个 Decode 理解成一个检索系统：历史部分提供可查询的“数据库”（KV Cache），当前 token 提供这一次的“检索请求”（$`Q_t`$）。数据库要持久化，请求本身是一次性的——这就是只缓存 K/V 的根本原因。

### 推理的两个阶段

推理过程分为两个阶段：

|  | Prefill（预填充） | Decode（解码） |
| --- | --- | --- |
| 做什么 | 一次性处理用户输入的完整 prompt | 逐 token 生成后续内容 |
| 例子 | 处理“今天天气不错，我想去” | 依次生成“公”→“园”→“散”→“步” |
| 计算特点 | 一次处理大量 token，计算量大但并行度高 | 每步只处理 1 个 token，不断循环 |
| 对 Cache 的作用 | 建立初始 KV Cache | 持续追加并复用 Cache |

Decode 阶段的每一步：

1. 为新增 token 计算 Q/K/V；
2. 将新的 K、V 拼接到缓存末尾；
3. 用当前 Q 对全部历史 K/V 做 Attention；
4. 输出预测的下一个 token。

没有 KV Cache，就像每写一句新话都要把前文重新抄一遍再理解；有了 KV Cache，前文的整理结果已经保留，只需补充新内容继续往下写。

## 3. 加速了什么，以及没有加速什么

- 加速了：历史 token 的 K/V 不再重复计算，推理延迟显著降低；
- 没有加速：当前 token 仍需与全部历史 K/V 做 Attention 计算。

> [!important]
> KV Cache 节省的是“历史前缀被反复编码”的成本，没有消除“当前 token 依赖长历史上下文”的成本。

这也解释了为什么长上下文场景下模型仍然会慢：长 prompt 在 Prefill 阶段本身就计算量大；Decode 阶段每生成一个 token，仍需访问越来越长的历史缓存。KV Cache 不是让长上下文变便宜，而是避免了其中最浪费的重复计算部分。

## 4. 为什么主要在推理阶段使用

|  | 推理（Inference） | 训练（Training） |
| --- | --- | --- |
| 计算模式 | 逐 token 自回归，一次来一个 | 整段序列一次性送入，并行计算所有位置 |
| 有没有重复计算问题 | 有，历史前缀被反复编码 | 没有，所有位置同时计算 |
| KV Cache 有没有用 | 有显著收益 | 几乎无收益 |

KV Cache 是推理优化，不是训练优化。

## 5. 工程上的存储与显存开销

KV Cache 不是只存一份。Transformer 每一层 Attention 都会产生各自的 K 和 V，因此每层都要维护独立的缓存。

### 单层缓存的张量形状

```text
K: [B, H, T, D]
V: [B, H, T, D]
```

| 符号 | 含义 | 影响 |
| --- | --- | --- |
| $`B`$ | Batch Size | 批量越大，缓存越大 |
| $`H`$ | Attention Head 数 | 头越多，缓存越大 |
| $`T`$ | 已缓存的历史序列长度 | 序列越长，缓存越大 |
| $`D`$ | 每个 Head 的维度 | 维度越高，缓存越大 |

### 总开销

```math
\text{KV Cache 大小}
\propto
2\times L\times B\times H\times T\times D\times\mathrm{dtype\_size}
```

其中 $`L`$ 为模型层数，系数 2 来自 K 和 V 两份缓存。

每多生成一个 token，每一层都要多存一份 K 和 V，显存占用随上下文长度近似线性增长。这也是为什么长上下文推理非常吃显存，以及为什么大量推理优化工作都围绕 KV Cache 的显存管理展开。

## 6. 与 Causal Mask、RoPE 的关系

### 与 Causal Mask

- Causal Mask：保证当前位置只能看见自己和过去，不能看到未来——负责正确性；
- KV Cache：保存过去位置的 K/V 供后续复用——负责效率。

两者职责独立、互不干扰。

### 与 RoPE（旋转位置编码）

KV Cache 与 RoPE 天然兼容，典型流程是：

1. 为当前 token 生成 Q/K；
2. 对 Q/K 应用 RoPE；
3. 将处理后的 K 存入 Cache。

缓存中的 K 本身已携带位置信息，后续无需对历史 token 重新做 RoPE，只需对新增 token 按当前位置继续处理即可。

## 7. 核心逻辑的伪代码

```python
cache_k, cache_v = None, None

for step in range(max_new_tokens):
    # 仅为新增 token 计算 Q/K/V
    q, k, v = model.project(hidden_state_of_new_token)

    # 拼接历史缓存
    if cache_k is None:
        all_k, all_v = k, v  # 第一步，无历史
    else:
        all_k = concat(cache_k, k)  # 沿序列维度拼接
        all_v = concat(cache_v, v)

    # 当前 Q 对全部历史 K/V 做 Attention
    out = attention(q, all_k, all_v)

    # 更新缓存
    cache_k, cache_v = all_k, all_v

    # 预测下一个 token
    next_token = lm_head(out)
```

真实工程实现中通常不会每次执行 `concat`，而是预分配缓存空间。同时还要处理多层 Cache、batch 内不同样本长度不一致、Beam Search 时的 Cache 复制与重排等问题，但核心逻辑与上面一致。

## 8. 常见误区

### 误区 1：有了 KV Cache，每一步生成的代价就几乎固定了

历史 K/V 不需要重复计算了，但当前 token 仍要和全部历史 K/V 做 Attention，所以每步代价仍随序列长度增长。

### 误区 2：KV Cache 也能加速训练

KV Cache 的价值几乎全部体现在推理的自回归生成阶段；训练时整段序列并行计算，没有重复编码的问题。

### 误区 3：KV Cache 消除了长上下文的代价

KV Cache 只是避免了最浪费的重复计算部分。长 prompt 的 Prefill 计算量、Decode 阶段逐步增长的 Attention 开销，都是它无法消除的。
