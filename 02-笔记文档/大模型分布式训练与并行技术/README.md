# 大模型分布式训练与并行技术

本主题围绕大模型分布式训练中的并行策略、系统实现与通信优化展开，后续学习笔记主要来自猛猿的“图解大模型训练”系列。

## 一、基础并行策略

1. [图解大模型训练之：流水线并行（Pipeline Parallelism），以 GPipe 为例](https://zhuanlan.zhihu.com/p/613196255)
2. [图解大模型训练之：数据并行上篇（DP、DDP 与 ZeRO）](https://zhuanlan.zhihu.com/p/617133971)
3. [图解大模型训练之：数据并行下篇（ZeRO，零冗余优化）](https://zhuanlan.zhihu.com/p/618865052)
4. [图解大模型系列之：张量模型并行，Megatron-LM](https://zhuanlan.zhihu.com/p/622212228)

## 二、Megatron 源码与训练系统

5. [Megatron 源码解读 1：分布式环境初始化](https://zhuanlan.zhihu.com/p/629121480)
6. [Megatron 源码解读 2：模型并行](https://zhuanlan.zhihu.com/p/634377071)
7. [Megatron 源码解读 3：分布式混合精度训练](https://zhuanlan.zhihu.com/p/662700424)

## 三、MoE 并行训练

8. [DeepSpeed-Megatron MoE 并行训练（原理篇）](https://zhuanlan.zhihu.com/p/681154742)
9. [DeepSpeed-Megatron MoE 并行训练（源码解读篇）](https://zhuanlan.zhihu.com/p/681692152)

## 四、序列与上下文并行

10. [序列并行 1：Megatron SP](https://zhuanlan.zhihu.com/p/4083427292)
11. [序列并行 2：DeepSpeed Ulysses](https://zhuanlan.zhihu.com/p/4496065391)
12. [序列并行 3：Ring Attention](https://zhuanlan.zhihu.com/p/4963530231)
13. [序列并行 4：Megatron Context Parallel](https://zhuanlan.zhihu.com/p/5502876106)

## 五、通信优化与机制辨析

14. [图解 Megatron TP 中的计算通信 Overlap](https://zhuanlan.zhihu.com/p/16594218518)
15. [探索一个关于 DeepSpeed ZeRO-3 的认知误区](https://zhuanlan.zhihu.com/p/20115278338)
