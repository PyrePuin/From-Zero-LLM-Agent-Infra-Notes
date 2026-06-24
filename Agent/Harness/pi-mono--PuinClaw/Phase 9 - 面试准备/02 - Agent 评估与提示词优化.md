---
type: concept
status: seed
domain: llm-agent
created: 2026-06-25
updated: 2026-06-25
series: pi-mono
tags:
  - 面试
  - evaluation
  - prompt-optimization
  - dspy
aliases:
  - Agent 评估
  - 提示词自动优化
---

# 02 - Agent 评估与提示词自动优化

> [!note]
> 这是面试里**较难但很亮**的两块。大部分候选人只会说"我调了 prompt 效果变好了",
> 你能讲清楚**怎么量化评估 + 怎么自动优化**,直接拉开档次。
> 工具代表:**DSPy**(Stanford NLP)。

## 1. 为什么 Agent 评估难

```
传统软件测试:
   输入 → 函数 → 输出 (确定性,assert 就行)

Agent 测试:
   输入 → LLM → 输出 (非确定性,同样输入可能不同输出)
                    ↑
              ┌─────┴─────┐
              │ 难点 3 个 │
              └───────────┘
   ① 输出非确定 — 同样输入跑 5 次,5 个结果
   ② 任务复杂 — 多步推理 + 工具调用 + 中间状态
   ③ "好"是主观的 — 怎么定义"对"?
```

## 2. 评估方法论

### 2.1 三种评估方式

```
┌─────────────────┬─────────────────────────────────────┐
│ 方式            │ 特点                                │
├─────────────────┼─────────────────────────────────────┤
│ ① 离线评估      │ 跑固定测试集,算指标                │
│ (offline)       │ 适合:开发期对比 prompt 版本       │
├─────────────────┼─────────────────────────────────────┤
│ ② 在线评估      │ 真实用户用,收集反馈               │
│ (online)        │ 适合:产品上线后监控               │
├─────────────────┼─────────────────────────────────────┤
│ ③ 影子评估      │ 新版本不真正服务用户,只对比输出    │
│ (shadow)        │ 适合:上线前回归测试               │
└─────────────────┴─────────────────────────────────────┘
```

### 2.2 评估指标(4 类)

```
┌──────────────────┬────────────────────────────────────┐
│ 指标类型         │ 示例                               │
├──────────────────┼────────────────────────────────────┤
│ ① 任务完成率     │ 成功 / 失败 / 部分完成             │
│ Task Success     │                                    │
├──────────────────┼────────────────────────────────────┤
│ ② 工具使用       │ 工具选对了吗?调用次数?           │
│ Tool Use         │                                    │
├──────────────────┼────────────────────────────────────┤
│ ③ 效率           │ 多少 turn 完成?用了多少 token?   │
│ Efficiency       │                                    │
├──────────────────┼─────────────────────────────────────┤
│ ④ 成本           │ 花了多少钱?latency 多少?         │
│ Cost / Latency   │                                    │
└──────────────────┴────────────────────────────────────┘
```

### 2.3 评估方法

```
方法 1: 黄金答案对比(Golden Answer)
   测试集 = {input, expected_output}
   算:exact match / fuzzy match / semantic match

方法 2: LLM-as-a-Judge ★
   用另一个 LLM 给 agent 输出打分
   优势:可以评主观维度(清晰度/完整性/礼貌度)
   工具:DSPy / OpenAI Evals / Promptfoo

方法 3: Human Eval
   人评,精度高但贵
   SWE-bench / Terminal-Bench

方法 4: 程序化校验(Programmatic)
   跑测试 / 编译 / lint
   适合:代码任务(跑通就过,跑不过就失败)
```

## 3. 主流 Benchmark

```
┌──────────────────┬────────────────────────────────────┐
│ Benchmark        │ 测什么                             │
├──────────────────┼─────────────────────────────────────┤
│ SWE-bench        │ 解决真实 GitHub issue              │
│ (Princeton)      │                                    │
├──────────────────┼─────────────────────────────────────┤
│ Terminal-Bench   │ 终端任务(文件操作/编译/部署)    │
├──────────────────┼─────────────────────────────────────┤
│ HumanEval        │ 代码生成(函数级)                 │
├──────────────────┼─────────────────────────────────────┤
│ GAIA             │ 通用 assistant(多步推理)         │
├──────────────────┼─────────────────────────────────────┤
│ τ-bench          │ 多轮工具调用(客服/零售场景)     │
└──────────────────┴─────────────────────────────────────┘
```

## 4. DSPy 框架(提示词自动优化的代表)

### 4.1 DSPy 核心思想

```
传统 prompt 工程:
   人肉改 prompt,跑测试,看效果,再改 — 经验驱动

DSPy "编译" 思想 ★:
   声明式定义:输入输出 + 评分函数
              ↓
   DSPy 自动优化:试 N 种 prompt + few-shot 组合
              ↓
   输出:最优 prompt + few-shot 例子
```

→ **"编译" 把 prompt 工程变成 optimization 问题**。

### 4.2 DSPy 三件套

```
① Module — 声明式 prompt 程序
   class MyAgent(dspy.Module):
       def forward(self, question):
           return self.answer(question=question)

② Metric — 评分函数
   def my_metric(example, prediction, trace=None):
       # 返回 0.0 ~ 1.0
       return 1.0 if prediction.is_correct else 0.0

③ Optimizer — 自动优化器
   teleprompter = dspy.BootstrapFewShot(metric=my_metric)
   compiled_agent = teleprompter.compile(MyAgent(), trainset)
```

### 4.3 DSPy Optimizer 类型

```
┌──────────────────────┬──────────────────────────────────┐
│ Optimizer            │ 做什么                           │
├──────────────────────┼──────────────────────────────────┤
│ BootstrapFewShot     │ 自动选 few-shot 例子             │
│ MIPRO                │ 自动优化 prompt 指令 + 例子      │
│ BootstrapFineTune    │ 微调模型权重(不只是 prompt)    │
│ Ensemble             │ 多个 prompt 集成                 │
└──────────────────────┴──────────────────────────────────┘
```

## 5. 实际怎么做(PuinClaw 示例)

### 5.1 评估 pipeline

```
┌────────────────────────────────────────────────────────┐
│ Step 1: 收集测试集                                     │
│   { goal, success_criteria, expected_actions }         │
│   从历史 /goal 任务里挑 20-50 个有代表性的             │
├────────────────────────────────────────────────────────┤
│ Step 2: 跑 agent + 收集 trace                          │
│   每个 task 跑 N 次(对抗非确定性)                    │
│   记录:turns / cost / tool_calls / final_state        │
├────────────────────────────────────────────────────────┤
│ Step 3: 算指标                                         │
│   任务完成率:success_criteria 满足吗                  │
│   效率:平均 turns / 平均 cost                         │
│   工具使用:误用率                                     │
├────────────────────────────────────────────────────────┤
│ Step 4: LLM-as-Judge 给主观维度打分                    │
│   代码质量 / 文档质量 / 边界处理                       │
├────────────────────────────────────────────────────────┤
│ Step 5: 对比基线                                       │
│   v1 prompt vs v2 prompt,看哪个赢                     │
└────────────────────────────────────────────────────────┘
```

### 5.2 提示词自动优化

```
传统:手工调 system prompt → 跑测试 → 看结果 → 再调
DSPy:把 system prompt 拆成 5-10 个"section"
      ↓
      DSPy 自动尝试不同 section 的组合 + few-shot
      ↓
      算指标,选最优组合
      ↓
      输出 "编译后" 的 system prompt
```

## 6. 面试要点

### 必答清单

```
Q1: 怎么评估 agent 效果?
A: 4 类指标(任务完成率/工具使用/效率/成本) +
   3 种方法(黄金答案/LLM-Judge/程序化校验) +
   对抗非确定性(多次跑取平均)。

Q2: 为什么 agent 评估比传统软件难?
A: 3 个原因 — 输出非确定 / 任务多步复杂 / "好"主观。

Q3: DSPy 是什么?
A: Stanford 的 prompt 编译框架。
   声明式定义输入输出 + 评分函数 →
   自动优化 prompt + few-shot →
   输出"编译后"的最优 prompt。

Q4: 提示词自动优化 vs 手工调,优势?
A: ① 可量化(metric 驱动)
   ② 可重现(同 dataset 可对比)
   ③ 可扩展(自动搜大空间)
   ④ 可累积(优化历史可追溯)

Q5: SWE-bench 是什么?
A: Princeton 提出的真实 GitHub issue benchmark,
   agent 要解决真实 bug 并通过测试。
```

### 加分点

```
• 提到 LLM-as-Judge 的 bias 问题(更偏好同模型输出)
• 提到非确定性 → 必须多次跑取均值
• 提到 in-distribution vs out-of-distribution 测试
• 知道 BootstrapFewShot vs MIPRO 区别
• 知道 AgentBench / WebArena 等其他 benchmark
```

## 7. 关键资料

| 资料 | 内容 |
|---|---|
| [DSPy 官网](https://dspy.ai/) | 框架文档 |
| [DSPy Metrics 文档](https://dspy.ai/getting-started/metrics/) | 评估指标 |
| [SWE-bench](https://www.swebench.com/) | 真实 issue benchmark |
| [Terminal-Bench](https://terminal-bench.com/) | 终端任务 benchmark |
| [LLM-as-Judge 论文](https://arxiv.org/abs/2306.05685) | 用 LLM 评估 LLM |
| [DSPy-HELM](https://github.com/StanfordMIMI/dspy-helm) | Stanford 评估套件 |

## 8. 跟之前学的对照

| 学过的 | 这里的对应 |
|---|---|
| PuinClaw `/goal` 的 success_criteria | 就是评估指标 |
| claw0 s07 Heartbeat | 周期性自检 = 在线评估 |
| pi-mono onPayload 钩子 | 监控每次请求 = 数据收集点 |

_Generated for PuinClaw 面试准备, 2026-06-25_
