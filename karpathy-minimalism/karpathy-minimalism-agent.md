# Karpathy 极简主义 — Agent Architecture 哲学

> 基于 Andrej Karpathy 的 [microgpt](https://karpathy.github.io/2026/02/12/microgpt/)（2026-02-12）、LLM Wiki pattern 和公开言论蒸馏。
> 学习目标：把 Karpathy 的极简哲学映射到 agent design，深化 loop engineering 的理论根基。

---

## 提纲

1. microgpt：200 行的完整 GPT
2. 极简主义的三个原则
3. 从 microgpt 到 agent architecture
4. Karpathy 哲学 × Loop Engineering 交叉点
5. 对你工作的启示
6. 附录

---

## 1. microgpt：200 行的完整 GPT

> *"This script is the culmination of multiple projects (micrograd, makemore, nanogpt) and a decade-long obsession to simplify LLMs to their bare essentials, and I think it is beautiful."*

Karpathy 用 **200 行纯 Python、零依赖** 实现了一个完整的 GPT：
- 数据集（names.txt，32,000 个名字）
- Tokenizer（character-level）
- Autograd 引擎（micrograd）
- GPT-2-like 神经网络架构
- Adam 优化器
- **Training loop**
- **Inference loop**

**核心金句**：*"This file contains the full algorithmic content of what is needed. Everything else is just efficiency."*

### 200 行里有什么

| 组件 | 行数 | 对应 agent 概念 |
|------|:--:|------|
| Dataset + Tokenizer | ~30 | Agent 的输入源 + 解析 |
| Neural Net Architecture | ~50 | Agent 的推理引擎 |
| Autograd Engine | ~40 | Agent 的反馈信号 |
| Training Loop | ~30 | Agent 的**闭环迭代** |
| Inference Loop | ~20 | Agent 的**执行产出** |
| 其余（Adam/Opt） | ~30 | Agent 的优化策略 |

**总共 200 行。一个完整的 learning system。**

---

## 2. 极简主义的三个原则

从 microgpt 和 Karpathy 的其他项目（micrograd, makemore, nanogpt, llm.c）提炼：

### 原则 1：核心算法极简

> 去掉所有优化、并行、缓存、分布式——核心还能剩什么？

microgpt 证明了：一个完整的学习系统核心只需要 200 行。同样的原则适用于 agent design：

- **Agent 的核心 loop 应该能在 ~50 行里描述**（plan → execute → observe → verify）
- Adversarial Verification 的核心只需要 spawn 两个 agent
- "复杂"是工程优化，不是算法必要

### 原则 2：从数据中学

> microgpt 不需要手工规则来生成名字。它从 32,000 个名字里学。

Agent 同理：
- Agent 不需要手工写的"验证规则"——它从 **verification feedback** 中学
- Closed-loop 的 feedback 就是 agent 的 "训练数据"
- 和 ComPilot 论文的发现一致：feedback 闭环比 open-loop 好 23-40%

### 原则 3：看原始输出

> Karpathy 反复强调：直接看 loss curve、看生成的样本、看 attention pattern。不要只看汇总指标。

Agent 同理：
- **不要只看 verification_report.json 的 pass/fail**
- 要看 verifier 的具体 issues、看 agent 的 reasoning trace
- Cognitive surrender 的解药就是"看原始输出"

---

## 3. 从 microgpt 到 agent architecture

### Training Loop → Agent Iteration Loop

```
microgpt training loop:           Agent iteration loop:
─────────────────────────         ─────────────────────
for step in range(num_steps):     for iter in range(max_iterations):
    # forward pass                    # 执行任务
    logits = model(x)                 result = delegate_task(...)
    loss = cross_entropy(logits, y)   # 验证产出
    # backward pass                   issues = verify(result, rubric)
    model.zero_grad()                 # 收集反馈
    loss.backward()                   feedback = extract_lessons(issues)
    # update                          # 更新策略
    optimizer.step()                  strategy.update(feedback)
    # evaluate                        # 检查条件
    if step % eval_interval == 0:     if stopping_condition_met():
        print(f"step {step},              break
              loss {loss.item()}")
```

**同构**：forward pass = agent 执行，backward pass = 对抗验证，update = 修正策略，evaluate = 检查停止条件。

### Inference Loop → Agent Execution Loop

```
microgpt inference:               Agent execution:
───────────────────               ───────────────
context = [token]                  context = [user_prompt]
while len(output) < max_tokens:    while not done:
    logits = model(context)            plan = think(context)
    probs = softmax(logits[-1])        action = decide(plan)
    next_token = sample(probs)         result = execute(action)
    context.append(next_token)         context.append(result)
```

**同构**：两者都是 autoregressive loop——每一步基于累积的 context 决定下一步。

---

## 4. Karpathy 哲学 × Loop Engineering 交叉点

| 维度 | Karpathy 哲学 | Loop Engineering | 交叉 |
|------|-------------|-----------------|------|
| 复杂度 | 核心 200 行，其他是效率 | core ~120 行，模板放 references | ⚡ 同构 |
| 学习机制 | Data → Model → Loss → Backprop | 执行 → 验证 → 反馈 → 修正 | ⚡ 同构 |
| 极简验证 | "看原始输出，不看汇总" | "看 verifier 的具体 issues" | ⚡ 同构 |
| 数据集 | 数据越多模型越好 | 反馈越多 agent 越好 | ⚡ 同构 |
| 规模 | microgpt → nanogpt → GPT-4 | 单 agent → 单 pipeline → 多 pipeline | 同样可 scale |

### 核心共鸣

> **"This file contains the full algorithmic content of what is needed."**

Loop engineering 的核心算法是什么？
1. **Plan**（决定做什么）
2. **Execute**（去做）
3. **Verify**（独立 agent 验证，不自己审自己）
4. **Feedback**（从验证中学，修正）
5. **Repeat**（直到停止条件满足）

这就是 5 行。其他都是效率——Fan-out、Tournament、rubric 模板都是优化，不是核心。

---

## 5. 对你工作的启示

### 5.1 不要过度设计

microgpt 证明了你可以用 200 行纯 Python 做一个完整 GPT。你的 agent loop 不应该需要 5 个 skill 文件才跑起来。

**现在回头看 loop-engineering 的三层拆分（core + adversarial + patterns）——三个文件、合计 ~260 行。刚好是 Karpathy 尺度。**

### 5.2 Feedback 就是 agent 的训练数据

Karpathy 说 GPT 从数据集中学。Agent 从 verification feedback 中学。

- 每次 verifier 抓到问题 → 就是一条训练样本
- 积累足够多的 verification_report.json → agent 的"dataset"
- 未来的 agent 可以从这些 feedback 中自我改进（不用你动手）

### 5.3 极简验证原则

Karpathy 说"看原始输出"。

对 agent 验证的意义：
- 不要只看 `verification_report.pass = true/false`
- **要看 verifier 写的具体 issues**
- 一条 issue 比一个 pass/fail 值 100 倍

### 5.4 等待 xurl 后的补充

Karpathy 最近（2025-2026）在 X 上发表了大量关于 agent architecture 的 thread。包括：
- Agent 作为 LLM OS 的进程
- Agent 的 cognitive architecture
- Reward modeling for agent verification

待 xurl 配置后可系统抓取这些内容，作为本文件的续篇。

---

## 6. 附录

### 6.1 microgpt 源码
- Gist: https://gist.github.com/karpathy/...
- 在线: https://karpathy.ai/microgpt.html

### 6.2 Karpathy 关键项目（极简主义系列）

| 项目 | 行数 | 内容 | 年份 |
|------|:--:|------|------|
| micrograd | ~150 | Autograd 引擎 | 2020 |
| makemore | ~300 | Character-level LM | 2022 |
| nanogpt | ~500 | GPT-2 复现 | 2023 |
| llm.c | ~1000 | LLM 训练 | 2024 |
| **microgpt** | **~200** | **完整 GPT（极简）** | **2026** |

每个项目都在说同一件事：**核心算法极简，其他是效率。**

### 6.3 与本系列其他学习文件的关系
- `loop-engineering/` — Anthropic 六大模式 + 五步骤框架
- 本文 — Karpathy 极简哲学 = loop engineering 的理论根基
- 下一站（待 xurl）— Karpathy 的 agent architecture X threads

---

> *"I cannot simplify this any further."* — Andrej Karpathy, microgpt, 2026
