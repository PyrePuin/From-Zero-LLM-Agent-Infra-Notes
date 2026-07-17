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
  - benchmark
  - llm-as-judge
  - dspy
aliases:
  - Agent 评估
  - Eval
  - 评估框架
---

# 02 - Agent 评估

> [!note]
> 主体内容遵循 Anthropic 工程博客《Demystifying evals for AI agents》。
> 这是面试里**最容易被低估、却最能拉开档次**的一块——大部分候选人
> 只会说"我调了 prompt 效果变好了"。你能讲清楚怎么量化评估 + 怎么
> 应对非确定性 + 知道行业 benchmark 现状,直接展现工程深度。

## 1. 评估的必要性(Why)

```
没有评估体系的团队 → 陷在 reactive loop 里:

   ┌──────────────────────────────────────────┐
   │  用户报"agent 跑得不对"                 │
   │       ↓                                  │
   │  工程师猜测原因,改 prompt               │
   │       ↓                                  │
   │  上线 → 新版本引入新 bug                │
   │       ↓                                  │
   │  另一个用户报错 → 循环                  │
   └──────────────────────────────────────────┘

有评估体系的团队 → 主动改进:

   ┌──────────────────────────────────────────┐
   │  golden dataset + 自动 scorer           │
   │       ↓                                  │
   │  改动(prompt / model / tool)触发 CI    │
   │       ↓                                  │
   │  分数回归 → 阻止 merge                  │
   │       ↓                                  │
   │  分数上升 → 系统性进步                  │
   └──────────────────────────────────────────┘

Anthropic 的观点:
   • Agent 是系统,不是 prompt
   • 系统必须有反馈回路(feedback loop)
   • 评估是反馈回路的核心
   • 没评估 = 飞盲飞行(flying blind)
```

## 2. Agent eval 的特点(What)

### 2.1 Agent ≠ Model(根本区别)

```
┌────────────────┬────────────────────────────────────┐
│ Model eval     │ Agent eval                         │
├────────────────┼────────────────────────────────────┤
│ prompt → out   │ task → plan → tools → state → out  │
│ 单点评分        │ 轨迹 + 终态评分                    │
│ 输出文本        │ 副作用(文件/PR/数据库)           │
│ 一次调用        │ 多轮 + 重试 + 恢复                 │
│ 确定性高        │ 不确定性是核心特征                 │
│ 单一指标        │ 任务完成率 + 效率 + 成本 + 质量    │
└────────────────┴────────────────────────────────────┘

→ 不能用单次 output pass/fail 评判 agent
→ 必须评估"轨迹 + 终态"两层
```

### 2.2 Anthropic 的 4 种 Agent 类型

```
┌─────────────────────┬──────────────────────────────────┐
│ Agent 类型          │ 评估侧重                         │
├─────────────────────┼──────────────────────────────────┤
│ ① Coding Agent      │ 单元测试 / 编译 / SWE-bench      │
│   (编码)           │ 可用 rule-based 评分             │
├─────────────────────┼──────────────────────────────────┤
│ ② Conversational    │ 多轮对话质量 / 工具使用 /        │
│   Tool-Use          │ 策略遵守(τ-bench)              │
├─────────────────────┼──────────────────────────────────┤
│ ③ Research Agent    │ 答案准确性 + 推理链完整性 +      │
│   (研究)           │ 专家评估(BrowseComp / GAIA)    │
├─────────────────────┼──────────────────────────────────┤
│ ④ Computer Use      │ 桌面/移动 GUI 任务完成           │
│   Agent             │ (OSWorld / AndroidWorld)         │
└─────────────────────┴──────────────────────────────────┘

不同类型 → 不同 benchmark + 不同 scorer
```

## 3. 怎么做(How)

### 3.1 核心概念(必背定义)

```
┌──────────────────┬──────────────────────────────────────┐
│ Term             │ 定义                                 │
├──────────────────┼──────────────────────────────────────┤
│ Task             │ 一个评估单元(有明确成功条件)      │
│ Trial            │ 同 task 跑一次(可能多次)          │
│ Transcript       │ 一次 trial 的完整轨迹(消息+工具)  │
│ Outcome          │ 终态(数据库/文件/PR/答案)         │
│ Scorer           │ 把 (task, outcome) 映射成 0~1 分    │
│ Test Harness     │ 跑 trial + 收 trace 的运行框架      │
│ Eval Suite       │ task + scorer + harness 的集合      │
└──────────────────┴──────────────────────────────────────┘
```

### 3.2 Anthropic 8 步路线图

```
① 先定义任务         — 任务必须有明确成功条件
                       "改 bug" 不行,"测试通过" 行

② 从失败开始         — golden set 优先收集已知失败 case
                       已知能过的没诊断价值

③ 多样性覆盖         — 任务类型/长度/工具组合均衡
                       避免偏斜

④ 跑多 trial         — 同任务 k=1, k=5, k=10 都跑
                       非确定性 → 单次不可信

⑤ Scorer 选择        — 优先级:rule > LLM > human
                       能用代码判就不用模型评

⑥ 不测中间步骤       — 步骤失败 ≠ 任务失败
                       步骤评估只用于诊断,不进入 pass/fail

⑦ 复杂任务给部分分   — 多组件任务,正确步骤给部分功劳
                       避免 0/1 二元评分

⑧ 周期性 + 回归      — capability 周期评估
                       regression 每次 PR 跑
```

### 3.3 两套核心指标:pass@k vs pass^k ★

```
┌─────────────────────────────────────────────────────────────┐
│ pass@k (覆盖率 / best-case)                                │
│   定义:在 k 次独立采样中,至少 1 次成功的概率            │
│   含义:模型的上限能力                                     │
│   公式:1 - C(n-c, k) / C(n, k)   (无偏估计)              │
│   问题:k 越大分数越高,容易误导                          │
├─────────────────────────────────────────────────────────────┤
│ pass^k (可靠性 / worst-case)                               │
│   定义:连续 k 次运行全部成功的概率                       │
│   含义:模型的稳定可靠性                                   │
│   公式:p^k(若单次成功率 p)                              │
│   特点:对失败零容忍 — 一次失败即拉低分数                │
└─────────────────────────────────────────────────────────────┘

实用建议:
   • 谈"能力上限"     → pass@5 / pass@10
   • 谈"生产可用"     → pass^5 / pass^10(更严苛)
   • 必须同时报 k + temperature + 样本数,否则不可比
```

### 3.4 三类评分器(Scorer)

```
┌──────────────────┬──────────┬──────────┬────────────────────┐
│ 类型             │ 准确度   │ 成本     │ 适用场景          │
├──────────────────┼──────────┼──────────┼────────────────────┤
│ Rule-based       │ ★★★★★   │ $        │ 有明确答案的任务  │
│ (代码/正则/状态) │          │          │ 单元测试/JSON     │
├──────────────────┼──────────┼──────────┼────────────────────┤
│ LLM-as-Judge     │ ★★★     │ $$       │ 主观/开放任务     │
│ (模型评分)       │          │          │ 研究/对话质量     │
├──────────────────┼──────────┼──────────┼────────────────────┤
│ Human Eval       │ ★★★★★   │ $$$$$    │ 金标 / 校准       │
│                  │ (金标)   │          │                   │
└──────────────────┴──────────┴──────────┴────────────────────┘

Anthropic 原则:能 rule 就不 LLM,能 LLM 就不 human
```

### 3.5 LLM-as-Judge 的四大偏见(必背)

```
① Position Bias    — A-vs-B 时偏向先/后位置
                     缓解:必须双向跑(A vs B + B vs A)

② Self-Preference   — 模型偏好自己生成的答案
                     缓解:生成与评判用不同模型家族

③ Verbosity Bias   — 偏向更长的回答

④ Authority Bias   — 偏向权威语气/格式

→ 实操:LLM 评分永远要配合 human spot-check 校准
```

### 3.6 不测步骤 vs 部分功劳(易混)

```
Anthropic 的两个看似矛盾的原则:

原则 1:不要把"步骤"作为评估目标
        step 失败 ≠ task 失败
        agent 经常走弯路但最终成功

原则 2:复杂任务给部分功劳
        task 有 5 个组件,做对 3 个 → 0.6 分
        不是 0/1 二元

区别:
   部分功劳是按"任务的子组件"打分(每个组件本身有成功条件)
   不是按"agent 的步骤"打分

   错误做法:step 1 ok + step 2 fail + step 3 ok = 2/3 分
   正确做法:subtask A ok + subtask B fail + subtask C ok = 2/3 分
```

### 3.7 Capability vs Regression(两种评估目的)

```
┌──────────────────┬─────────────────────────────────────────┐
│ Capability Eval  │ "模型能做什么"(绝对水平)              │
│                  │  • benchmark 跑分                       │
│                  │  • 任务覆盖率                          │
│                  │  • 用于选型 / 公关                     │
├──────────────────┼─────────────────────────────────────────┤
│ Regression Eval  │ "改动后是否变差"(相对水平)            │
│                  │  • CI/CD 集成                          │
│                  │  • golden dataset 对比                 │
│                  │  • prompt 改动 / model swap 后必跑     │
│                  │  • 用于工程保护                       │
└──────────────────┴─────────────────────────────────────────┘

实操模式:
   prod model + golden set (200-500 task)
       ↓
   prompt / model 改动触发 CI
       ↓
   跑 golden,对比新旧分数
       ↓
   总分下降 > 3% → 阻止 PR
   单类下降 > 10% → 阻止 PR
```

## 4. 实际可用的 eval 框架(2026 现状)

### 4.1 框架对比

```
┌─────────────────┬─────────────────────────────────────────┐
│ 框架            │ 定位                                   │
├─────────────────┼─────────────────────────────────────────┤
│ DeepEval v4     │ Pytest 风格,工程师最易上手            │
│ (开源)         │ from deepeval import assert_test      │
│                 │ 写法跟写 pytest 一样                   │
├─────────────────┼─────────────────────────────────────────┤
│ Inspect AI      │ UK AISI 出品,科研级严谨               │
│ (开源)         │ 英国政府 AI 安全研究所用              │
├─────────────────┼─────────────────────────────────────────┤
│ Harbor          │ Anthropic Terminal-Bench 同款 task fmt │
│ (开源)         │ 适合 agent task 标准化描述            │
├─────────────────┼─────────────────────────────────────────┤
│ DSPy            │ Stanford,"编程而非 prompt"          │
│ (开源)         │ 不只是评估,核心是自动优化            │
│                 │ LM Assertions + Optimizer              │
├─────────────────┼─────────────────────────────────────────┤
│ LM Eval Harness │ EleutherAI,模型层综合评测经典        │
│ (开源)         │ 几百个基准开箱即用                    │
├─────────────────┼─────────────────────────────────────────┤
│ LangSmith       │ 商业,trace + dataset + scorer 一条龙 │
│ Braintrust      │ 商业,task/dataset/scorer/evaluator   │
│ MLflow          │ Databricks,跟 ML 生态集成好         │
│ Arize Phoenix   │ 开源 + 商业,可观测性强              │
├─────────────────┼─────────────────────────────────────────┤
│ OpenAI Evals    │ ⚠ 2026-11-30 平台停服                │
│                 │ 开源代码仍可自托管                    │
└─────────────────┴─────────────────────────────────────────┘
```

### 4.2 DSPy:提示词自动优化代表

```
传统 prompt 工程:
   人肉改 prompt,跑测试,看效果,再改 — 经验驱动

DSPy "编译" 思想 ★:
   声明式定义:输入输出 + 评分函数
              ↓
   DSPy 自动优化:试 N 种 prompt + few-shot 组合
              ↓
   输出:最优 prompt + few-shot 例子

→ "编译" 把 prompt 工程变成 optimization 问题
```

DSPy 三件套:

```
① Module — 声明式 prompt 程序
   class MyAgent(dspy.Module):
       def forward(self, question):
           return self.answer(question=question)

② Metric — 评分函数
   def my_metric(example, prediction, trace=None):
       return 1.0 if prediction.is_correct else 0.0

③ Optimizer — 自动优化器
   teleprompter = dspy.BootstrapFewShot(metric=my_metric)
   compiled = teleprompter.compile(MyAgent(), trainset)
```

DSPy Optimizer 类型(早期叫 teleprompter):

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

DSPy 还有一个独特特性:**LM Assertions**

```
with dspy.Assert(max_tokens < 100):
    # 强制约束:如果输出超 100 token,自动重试
    ...
```

类似 Python assert,但失败时让 LLM 自动重试而非崩溃。

### 4.3 DeepEval 最小示例

```python
from deepeval import assert_test
from deepeval.metrics import TaskCompletionMetric
from deepeval.test_case import LLMTestCase

def test_my_agent():
    case = LLMTestCase(input="...", actual_output="...")
    metric = TaskCompletionMetric(threshold=0.7)
    assert_test(case, [metric])
```

写法跟 pytest 完全一致,门槛最低。

## 5. 主流 Benchmark 现状(2026)

### 5.1 编码 Agent

```
┌──────────────────────┬─────────────┬──────────────────────────┐
│ Benchmark            │ 状态         │ 特点                    │
├──────────────────────┼─────────────┼──────────────────────────┤
│ SWE-bench Verified   │ ⚠ 已饱和    │ 2026-02 退役,顶部 >90% │
│ (OpenAI)            │              │ human-filtered GitHub    │
├──────────────────────┼─────────────┼──────────────────────────┤
│ SWE-bench Pro        │ ★ 活跃      │ Scale AI,1,865 task    │
│                      │              │ 顶部 ~50%               │
├──────────────────────┼─────────────┼──────────────────────────┤
│ Terminal-Bench       │ ★ 活跃      │ Harbor task 格式        │
│ (Harbor)             │              │ 端到端 CLI 任务         │
├──────────────────────┼─────────────┼──────────────────────────┤
│ Aider Polyglot       │ ★ 活跃      │ 6 语言 225 题           │
│                      │              │ 测代码编辑能力          │
├──────────────────────┼─────────────┼──────────────────────────┤
│ LiveCodeBench        │ ★ 活跃      │ 滚动新题,抗污染       │
└──────────────────────┴─────────────┴──────────────────────────┘
```

### 5.2 Web 搜索 / 浏览

```
┌──────────────────────┬─────────────┬──────────────────────────┐
│ Benchmark            │ 状态         │ 特点                    │
├──────────────────────┼─────────────┼──────────────────────────┤
│ BrowseComp (OpenAI)  │ ★ 极难      │ 1,266 题,基线 ~1.9%   │
│                      │              │ 98% agent 失败          │
├──────────────────────┼─────────────┼──────────────────────────┤
│ GAIA (DeepMind)      │ ⚠ 接近饱和  │ 2026 达 ~74.5%          │
│                      │              │ 通用助手真实任务        │
├──────────────────────┼─────────────┼──────────────────────────┤
│ HLE (Scale AI)       │ ⚠ 接近饱和  │ Humanity's Last Exam    │
│                      │              │ 专家级学术多模态        │
├──────────────────────┼─────────────┼──────────────────────────┤
│ WebArena             │ ⚠ 接近饱和  │ 812 task,自托管       │
└──────────────────────┴─────────────┴──────────────────────────┘
```

### 5.3 Computer Use(桌面 GUI)

```
┌──────────────────────┬─────────────┬──────────────────────────┐
│ Benchmark            │ 状态         │ 特点                    │
├──────────────────────┼─────────────┼──────────────────────────┤
│ OSWorld              │ ★ 主基准    │ 369 task,跨平台        │
│ (xlang-ai)          │              │ 顶部 ~38% (Operator)    │
├──────────────────────┼─────────────┼──────────────────────────┤
│ OSWorld-Verified     │ ★ 活跃      │ 剔除原版不可行任务      │
├──────────────────────┼─────────────┼──────────────────────────┤
│ WindowsAgentArena    │ ★ 活跃      │ 微软出品,Windows 专属 │
│ (Microsoft)          │              │ 150+ task               │
└──────────────────────┴─────────────┴──────────────────────────┘

观察:即使最强 agent 也 < 40% 成功率 → 差距最大的领域
```

### 5.4 移动 GUI

```
┌──────────────────────┬─────────────┬──────────────────────────┐
│ Benchmark            │ 状态         │ 特点                    │
├──────────────────────┼─────────────┼──────────────────────────┤
│ AndroidWorld         │ ★ 主基准    │ Google,116 task        │
│ (Google)             │              │ 顶部 91.4% (Droidrun)   │
├──────────────────────┼─────────────┼──────────────────────────┤
│ MobileWorld          │ ★ 新生      │ 通义千问,201 task      │
│ (Tongyi-MAI)        │              │ 比 AndroidWorld 更难    │
└──────────────────────┴─────────────┴──────────────────────────┘
```

### 5.5 工具调用 / 对话

```
┌──────────────────────┬─────────────┬──────────────────────────┐
│ Benchmark            │ 状态         │ 特点                    │
├──────────────────────┼─────────────┼──────────────────────────┤
│ τ-bench (Sierra)     │ ★ 主基准    │ 165 task,3 域          │
│                      │              │ 工具 + 策略遵守         │
├──────────────────────┼─────────────┼──────────────────────────┤
│ τ²-bench             │ ★ 活跃      │ dual-control:agent 还  │
│                      │              │ 要引导用户操作          │
├──────────────────────┼─────────────┼──────────────────────────┤
│ BFCL v4              │ ★ 活跃      │ Berkeley 函数调用排行榜 │
│ (UC Berkeley)        │              │ 2,000+ 测试用例         │
└──────────────────────┴─────────────┴──────────────────────────┘
```

### 5.6 饱和与污染(行业难题)

```
┌──────────────────────────────────────────────────────────┐
│ Saturation(饱和)                                       │
│   顶部模型分数逼近上限 → 区分度消失                     │
│   现状:SWE-bench Verified 2026-02 退役                 │
│         MMLU / HumanEval / GPQA 早已饱和                │
├──────────────────────────────────────────────────────────┤
│ Contamination(污染)                                   │
│   训练数据泄漏到测试集 → 分数虚高                       │
│   《The SWE-Bench Illusion》arXiv 2506.12286 发现       │
│   LLM 可能是"记住"而非"解决"                          │
├──────────────────────────────────────────────────────────┤
│ 应对策略                                                │
│   ① Living benchmark(SWE-bench Live / LiveBrowseComp) │
│   ② Procedural generation(程序合成任务)              │
│   ③ Multi-agent games(Agent Island — 环境对抗)       │
│   ④ Hold-out 持续更新                                 │
└──────────────────────────────────────────────────────────┘
```

## 6. 面试要点

### 必答清单

```
Q1: 怎么评估 agent 效果?
A: ① 先建 golden dataset(从失败 case 开始)
   ② 跑多 trial(对抗非确定性,pass@k + pass^k)
   ③ 选 scorer(rule > LLM > human)
   ④ 不测中间步骤,但复杂任务给部分功劳
   ⑤ capability 周期跑 + regression 每次 PR

Q2: 为什么 agent 评估比传统软件/LLM 难?
A: 3 个根本不同 —
   ① agent 是系统不是 prompt → 评轨迹 + 终态
   ② 有副作用(文件/PR/数据库)→ 状态评判
   ③ 不确定性是核心特征 → 必须多次跑

Q3: pass@k 和 pass^k 区别?
A: pass@k = k 次至少 1 次成功(覆盖率/上限)
   pass^k = k 次全部成功(可靠性/稳定)
   前者谈能力,后者谈生产可用。

Q4: LLM-as-Judge 的偏见?怎么缓解?
A: 4 大偏见 — position / self-preference / verbosity / authority
   缓解:① 双向跑(A vs B + B vs A)
         ② 生成和评判用不同模型家族
         ③ 配合 human spot-check 校准

Q5: 怎么避免 agent 损坏代码库(回归保护)?
A: ① CI 集成 golden dataset
   ② prompt / model 改动触发评估
   ③ 总分下降 > 3% 阻止 merge
   ④ 单类下降 > 10% 阻止 merge

Q6: SWE-bench 现在还能用吗?
A: SWE-bench Verified 2026-02 退役(已饱和,顶部 >90%)。
   接棒者:SWE-bench Pro(顶部 ~50%) + SWE-bench Live(抗污染)。
```

### 加分点

```
• 提到 Anthropic"不测步骤,但给部分功劳"的微妙区分
• 知道 SWE-bench Verified 已退役 → 显示你跟进行业
• 提到 OpenAI Evals 2026-11 停服
• 知道 LLM-as-Judge 的 self-preference 偏见(同模型生成+评判)
• 知道 Harbor 是 Anthropic Terminal-Bench 的 task 格式
• 能讲 capability vs regression 区别
```

## 7. 关键资料

| 资料 | 内容 |
|---|---|
| [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) | Anthropic 原文(本笔记主线) |
| [DSPy 官网](https://dspy.ai/) | 提示词自动优化框架 |
| [DSPy Assertions](https://dspy.ai/learn/programming/7-assertions/) | LM 强制约束 |
| [SWE-bench](https://www.swebench.com/) | 编码 agent 主基准 |
| [SWE-bench Pro](https://scale.com/blog/swe-bench-pro) | 接棒 Verified |
| [τ-bench (Sierra)](https://github.com/sierra-research/tau-bench) | 工具调用对话基准 |
| [BrowseComp (OpenAI)](https://openai.com/index/browsecomp/) | 极难浏览基准 |
| [OSWorld](https://os-world.github.io/) | 桌面 GUI 基准 |
| [AndroidWorld](https://github.com/google-research/android_world) | 移动 GUI 基准 |
| [DeepEval](https://github.com/confident-ai/deepeval) | Pytest 风格 eval 框架 |
| [Inspect AI](https://github.com/UKGovernmentBEIS/inspect_ai) | UK AISI 出品 |
| [Harbor](https://github.com/harbor-framework/harbor) | Terminal-Bench task 格式 |
| [LLM-as-Judge 论文](https://arxiv.org/abs/2306.05685) | 用 LLM 评估 LLM |
| [The SWE-Bench Illusion](https://arxiv.org/html/2506.12286v3) | 污染问题分析 |
| [pass@k vs pass^k](https://hippocampus-garden.com/pass_k/) | 数学定义对比 |

_Generated for PuinClaw 面试准备, 2026-06-25_
