# 2026-07-01 — Agent Loop 的下一跳：从自动验证到自主进化

> **路径阶段**：阶段 3 — Vibe Coding 工程化
> **前序依赖**：[loop-engineering v0.3.0](skill:loop-engineering)（三层架构 + 六个模式）
> **本文定位**：不是日报，是深度学习材料——分析当前 loop skill 的能力边界，提出三个进化方向，给出落地路线图。

---

## 目录

1. [引言：loop 不是 endgame](#1-引言loop-不是-endgame)
2. [现状诊断：当前 loop 的三层能力边界](#2-现状诊断当前-loop-的三层能力边界)
3. [外部参考：谁已经在做"自进化 loop"](#3-外部参考谁已经在做自进化-loop)
    - [3.1 Skill Forge v2 — 实验驱动的 skill 自优化](#31-skill-forge-v2)
    - [3.2 Toryo — 质量棘轮 + 信任累积](#32-toryo)
    - [3.3 ForgeGod — 五层记忆 + 策略自进化](#33-forgegod)
    - [3.4 Self-Healing Orchestrators (arXiv) — 失败分类→精准恢复](#34-self-healing-orchestrators)
4. [三个进化方向](#4-三个进化方向)
    - [4.1 方向 A：Meta-Verifier（元验证层）](#41-方向-a元验证层)
    - [4.2 方向 B：Adaptive Rubric（自进化检查表）](#42-方向-b自进化检查表)
    - [4.3 方向 C：Insight Feed（启发推送）](#43-方向-c启发推送)
5. [架构设计：Meta-Verifier 作为起点](#5-架构设计meta-verifier-作为起点)
6. [落地路线图](#6-落地路线图)
7. [差距分析：能做 vs 还不能做](#7-差距分析能做-vs-还不能做)
8. [附录：关键论文与项目索引](#8-附录关键论文与项目索引)

---

## 1. 引言：loop 不是 endgame

当前 loop-engineering skill v0.3.0 已经建立了一个三层架构：

```
loop-engineering (core)
  ├── loop-engineering-adversarial  →  独立 verifier 审查
  └── loop-engineering-patterns     →  Loop Until Done + Tournament
```

这六个模式（Classify-and-act、Fan-out、Adversarial Verification、Generate-and-filter、Tournament、Loop Until Done）解决了一个核心问题：**不要让产出 agent 自己审自己**。

但它的本质是 **reactive**：产出了 → 验证了 → 通过/不通过。它不做三件事：

1. **跨 run 学习** — 这次日报数据算错了，下次还是同一个矿工的数据错，verifier 每次都能发现，但从不提醒"这个问题已经出现 5 次了"
2. **策略自进化** — rubric 是写死的（日报 5 项、良率 4 项），不会根据历史数据自动增减
3. **主动启发人** — verifier 跑完就沉默了，你不会收到"我注意到一个 pattern，你可能想看一下"

本文的目标：提出从 **verify** 到 **observe → learn → coach** 的进化路径。

---

## 2. 现状诊断：当前 loop 的三层能力边界

### 2.1 能力矩阵

| 能力维度 | 当前状态 | 边界 |
|---------|---------|------|
| **单次验证** | ✅ 成熟 — 独立 verifier + rubric 模板 | — |
| **并行竞争** | ✅ 成熟 — Tournament + Parallel Generation + Consistency Check | Token 成本 3-5x |
| **迭代终止** | ✅ 成熟 — Loop Until Done + 3 轮上限 | 固定上限而非自适应 |
| **跨 run 模式识别** | ❌ 不存在 | — |
| **rubric 自进化** | ❌ 不存在 | — |
| **主动启发** | ❌ 不存在 | — |
| **验证 agent 信任管理** | ❌ 不存在 | — |
| **失败分类→精准恢复** | ❌ 不存在 | 只重试，不区分失败类型 |

### 2.2 架构图：当前 loop 流程

<div class="mermaid">
flowchart TD
    A[任务输入] --> B[产出 Agent]
    B --> C[Adversarial Verifier]
    C --> D{通过?}
    D -->|✅ pass| E[交付]
    D -->|❌ fail| F[修正后重验]
    F --> C
    D -->|⚠️ 3轮上限| G[人介入]
    
    style C fill:#fef3c7,stroke:#d97706
    style E fill:#e6f4ea,stroke:#059669
    style G fill:#fef2f2,stroke:#dc2626
</div>

这个流程的问题是：**每次都从零开始**。verifier 不知道"上次这个 check 就没过"，产出 agent 也不知道"这个数据源已经连续错 5 次了"。

---

## 3. 外部参考：谁已经在做"自进化 loop"

### 3.1 Skill Forge v2

> 来源：<https://github.com/GodModeAI2025/skill-forge>（⭐14, MIT License）
> 核心理念：**实验驱动的 skill 自优化**

架构：

```
Hypothesis Agent → Mutator Agent → Scorer Agent → Keep/Revert
     ↑                                              │
     └──────────── 反馈回路 ─────────────────────────┘
```

三个 agent 的分工：

| Agent | 角色 | 输入 | 输出 |
|-------|------|------|------|
| **Hypothesis** | "科学家" — 分析失败，检查覆盖率矩阵 | 评分结果、skill 原文、历史记录 | 可测试的假设 + 修改建议 |
| **Mutator** | "外科医生" — 执行一个聚焦修改 | 假设、目标文件 | 修改后的文件 + 文档 |
| **Scorer** | "法官" — 评估输出质量 | eval 断言、输出 | 归一化质量分（0-1） |

两个关键机制：

**Coverage Matrix（覆盖率矩阵）** — 追踪每个改进类别被探索的次数：
```
| Category       | Experiments | KEEP | REVERT | Best Delta | Saturated |
|----------------|-------------|------|--------|------------|-----------|
| workflow       | 3           | 2    | 1      | +0.16      | no        |
| edge_cases     | 1           | 0    | 0      | ±0.00      | no        |
| formatting     | 0           | -    | -      | -          | untouched |
```

**Exploration-Exploitation Balance**：
```
| Phase  | Rounds | Strategy                                |
|--------|--------|-----------------------------------------|
| Early  | 1-3    | Explore: 优先探索未触及的类别          |
| Mid    | 4-7    | Mixed: 平衡覆盖度与高回报区域          |
| Late   | 8+     | Exploit: 聚焦最佳 delta 的类别         |
```

**对我们 loop 的启发**：不是每次验证都从同一个 rubric 开始。在多次运行后，一个 Hypothesis Agent 分析"哪个 check 项最拖后腿"，Mutator Agent 提议修改 rubric，Scorer Agent 验证修改是否提升了整体质量。

### 3.2 Toryo

> 来源：<https://github.com/JesseRWeigel/toryo>（⭐10, MIT License）
> 核心理念：**质量棘轮 + 信任累积 + 反馈重试**

```
Plan → Research → Execute → Review
                              │
                    Score ≥ threshold → git commit (keep)
                    Score < threshold → git revert → Ralph Loop retry
```

三个关键概念：

| 概念 | 做法 | 对我们 loop 的启发 |
|------|------|-------------------|
| **Quality Ratcheting** | 只允许质量单向上升——坏结果自动回滚，好结果才 commit | verifier 不只看单次 pass/fail，看趋势——"连续 3 次 pass → 这个 agent 可以信任" |
| **Trust-Based Delegation** | agent 通过持续高分积累信任，新人低信任→审查，老人高信任→自动放行 | 给每个 verifier 打分，高信任度的跳过某些 check，低信任度的加强审查 |
| **Ralph Loop** | 失败后不是丢弃，而是把 QA 反馈路由回 agent 重试 | verifier 不只是说"你错了"，而是说"你错在数据源，换备用源重试" |

### 3.3 ForgeGod

> 来源：<https://github.com/waitdeadai/forgegod>（⭐8, Apache 2.0）
> 核心理念：**五层认知记忆 + 策略自进化**

关键差异：ForgeGod 不只在单个任务内 loop，而是在**跨 session 层面积累知识**：

| 能力 | 机制 |
|------|------|
| 五层记忆 | 跨 session 持久化，从每次运行学习 |
| SICA（Self-Improving Cognitive Architecture） | 从每次结果中调整策略 |
| Recon | 执行前先做 web research 了解领域背景 |
| Adversarial Plan Debate | 多个 plan 互相辩论选最优 |

**对我们 loop 的启发**：跨任务的"记忆"——"上次做日报验证时这个矿工的 API 总是超时，这次先换备用数据源"。这不只是单次 loop 内的智能，是跨 session 的学习。

### 3.4 Self-Healing Orchestrators

> 来源：arXiv:2606.01416（Rahul Suresh Babu, Adarsh Agrawal, May 2026）
> 核心理念：**失败分类 → 精准恢复**

核心发现：

| 指标 | 数值 |
|------|------|
| **失败分类成功** | 能精准区分 timeout / stale context / retry loop / unverified intermediates |
| **精准恢复成功率** | 98.8%（vs retry-only 94.5% vs full replanning 93.8%） |
| **静默失败消除** | verifier 加入后 → **0.0%** |
| **单次恢复 vs 重试** | 94.0% vs 85.3% — 知道"为什么失败"比盲目重试有效得多 |

**对我们 loop 的启发**：当前 verifier 只说"没过"，不说"为什么没过"。如果 verifier 能分类失败（数据源问题？计算逻辑问题？格式问题？），产出 agent 就能精准修复，而不是从头重来。

---

## 4. 三个进化方向

### 4.1 方向 A：Meta-Verifier（元验证层）

**目标**：跨 pipeline 的 pattern discovery

**当前**：每个 pipeline 有自己的 verifier（日报一个、良率一个、邮件一个），互相不知道对方在干嘛。

**升级后**：
- 一个独立的 **Meta-Verifier** 定期跑（比如每天一次），不验证单次产出，而是**读过去 N 次 verifier 报告，找模式**
- 输出 insight：*"过去 7 天日报验证中，'计算准确性' 维度 fail 了 5/7 次，但每次都是矿工 f0xxxx 的 API 返回异常。建议：给这个矿工加 fallback 数据源。"*
- 这就是"**自己发现问题**"

**架构**：
```
┌──────────────────────────────────────────┐
│            Meta-Verifier (cron: 每日一次)   │
│                                           │
│  输入:                                     │
│  ├── verification_report_日报_0701.json    │
│  ├── verification_report_日报_0630.json    │
│  ├── verification_report_良率_0701.json    │
│  └── ... (过去 N 次报告)                    │
│                                           │
│  处理:                                     │
│  ├── 失败模式聚合 (按 pipeline / 维度 / 矿工)  │
│  ├── 趋势分析 (is this getting worse?)      │
│  ├── 新失败模式发现 (something new?)         │
│  └── 跨 pipeline 关联 (related failures?)   │
│                                           │
│  输出:                                     │
│  └── meta_insight_report_0701.json         │
│      ├── top 3 anomalies                  │
│      ├── recurring patterns               │
│      ├── suggested actions                │
│      └── trust_score updates for verifiers │
└──────────────────────────────────────────┘
```

**实现难度**：中。核心是一个 cron job + delegate_task。数据源已有（verification_report.json），只需要一个新的 agent 做聚合分析。

### 4.2 方向 B：Adaptive Rubric（自进化检查表）

**目标**：rubric 不再写死，而是根据历史数据自动调整

**当前**：rubric 是写死的——日报 5 项、良率 4 项。

**升级后**：
- Verifier 记录每次 fail 的原因 + 上下文（哪个 check、哪个数据源、什么错误）
- 自动调整规则：
  - 某个 check 项连续 N 次 100% pass → 降级为"跳过"（省 token）
  - 新失败模式出现 3 次以上 → 自动新增 check 项
  - 某 check 项 pass 率持续下降 → 提高权重
- 这就是"**自己学习逻辑**"

**Adaptive Rubric 决策树**：
```
rubric_item.assess():
    if consecutive_pass >= 10:
        → status = "skip" (跳过此检查，省 token)
    elif pass_rate_7d < 0.5:
        → status = "focus" (加严检查，多抽 3 个样本)
    elif is_new_failure_pattern:
        → 自动创建新 rubric_item
    else:
        → status = "normal"
```

**实现难度**：低-中。本质是给 rubric JSON 加一个动态权重系统。

### 4.3 方向 C：Insight Feed（启发推送）

**目标**：verifier 不只是 pass/fail，而是**主动告诉你"你可能想知道"**

**当前**：验证完就结束了，你不会收到任何主动通知。

**升级后**：

- Verifier 跑完后，不只是 pass/fail，而是生成一条 insight
- 格式示例：
  ```
  ⚠️ 日报 anomaly [2026-07-01]
  矿工 f0xxxx 的 power 从 3.2PiB 骤降到 1.8PiB
  与 7 天均值偏离 >2σ
  上次类似事件：2026-06-15（硬件故障，12h 恢复）
  → 建议：检查硬件状态
  ```
- 推送渠道：
  - 飞书 DM（通过 feishu gateway）
  - Obsidian daily note（"⚠️ Agent Alert" 板块）
  - 日报 HTML 侧栏（"今日 anomaly" 角标）

**实现难度**：中。需要一个 insight-generator agent 在 verifier 之后跑。飞书推送已有 gateway，Obsidian 写入已有 skill。

### 4.4 三个方向的协同关系

<div class="mermaid">
flowchart TD
    subgraph "Layer 3: Coach"
        C[Insight Feed] --> U[用户]
    end
    
    subgraph "Layer 2: Learn"
        B1[Adaptive Rubric] --> B2[自动增减 check 项]
        B2 --> B1
    end
    
    subgraph "Layer 1: Observe"
        A1[Meta-Verifier] --> A2[跨 run 模式发现]
        A2 --> A1
    end
    
    A2 -->|发现新失败模式| B1
    B2 -->|显著 pattern| C
    
    style A1 fill:#e6f4ea,stroke:#059669
    style B1 fill:#fef3c7,stroke:#d97706
    style C fill:#e0e7ff,stroke:#4f6ef7
</div>

**重点**：A → B → C 是递进关系。先有 Meta-Verifier 发现模式，模式稳定后升级为 Adaptive Rubric，显著异常推送给用户。不要三个同时搞。

---

## 5. 架构设计：Meta-Verifier 作为起点

### 5.1 为什么选 Meta-Verifier 先做

| 理由 | 说明 |
|------|------|
| **不改现有 pipeline** | 现有日报/良率 verifier 完全不动，Meta-Verifier 是独立的 cron job |
| **数据已有** | verification_report.json 已经每次验证都写了，只是没人读 |
| **即时价值** | 第一天跑就能告诉你"过去 7 天哪个 check 项最拖后腿" |
| **自然延伸** | 模式发现后自然引导到 B（rubric 进化）和 C（主动推送） |

### 5.2 Meta-Verifier 实现方案

**触发**：cron job，每日一次（比如 23:00），跑在 prof_bot profile 下。

**输入**：过去 N 天的 verification_report.json 文件（从各 pipeline 的 report 目录收集）。

**处理 agent**（delegate_task）：

```python
delegate_task(
    goal="""
    分析过去 7 天的 verification 报告，找出：
    1. 重复出现的失败模式（同一 check、同一数据源、同一 pipeline）
    2. 趋势变化（哪些 check 项的 pass 率在上升/下降）
    3. 跨 pipeline 的相关性（日报的错误与良率的错误是否同源）
    4. Top 3 最值得关注的 anomaly
    
    输出格式：meta_insight_report.json
    {
      "period": "2026-06-25 to 2026-07-01",
      "anomalies": [{check, pipeline, severity, pattern, suggestion}],
      "trends": [{check, direction, confidence}],
      "cross_pipeline": [{related_checks, shared_root_cause}],
      "trust_scores": [{verifier_id, score_change, reason}]
    }
    """,
    context="""
    数据文件在 /path/to/verification_reports/
    每个文件格式：{pass, score, issues: [{check, severity, detail}]}
    重点分析：issues 中的 check 字段 → 聚合 → 找模式
    """,
    toolsets=["terminal", "file"]
)
```

### 5.3 文件结构

```
workspace/meta-verifier/
├── meta_insight_report_YYYY-MM-DD.json   # 每日 insight 报告
├── failure_patterns.json                  # 积累的失败模式库
├── trust_scores.json                      # verifier agent 信任分
└── meta-verifier-skill.md                # 新 skill
```

---

## 6. 落地路线图

| 阶段 | 内容 | 预计工作量 | 依赖 |
|------|------|-----------|------|
| **Phase 0** | 确认 verification_report.json 格式统一（回头看日报/良率 pipeline 的现有格式） | 0.5h | — |
| **Phase 1** | 第一个 cron job：每天跑一次 meta-analysis，输出 insight JSON | 2h | Phase 0 |
| **Phase 2** | 验证 insight 质量 — 跑 7 天看模式是否合理 | 自动 + 1h review | Phase 1 |
| **Phase 3** | 基于 insight 结果，决定下一个方向：B（rubric 自进化）或 C（飞书推送） | 1h 决策 | Phase 2 |
| **Phase 4** | 实现选定的下一方向 | 3-5h | Phase 3 |

---

## 7. 差距分析：能做 vs 还不能做

| 能力 | 现在能做？ | 差距在哪？ | 参考来源 |
|------|-----------|-----------|---------|
| 失败模式聚合 | ✅ 可以 — delegate_task + JSON 解析 | 需要统一的 report 格式 | — |
| 趋势分析 | ✅ 可以 — 简单的 7 天滑动窗口 | 阈值设定需要调优 | — |
| 失败分类（区分类型） | ⚠️ 部分 — 需要给 verifier 加分类字段 | 当前 rubric 不记录"为什么失败" | Self-Healing Orchestrators |
| 探索-利用平衡 | ⚠️ 部分 — 可以手动实现 coverage matrix | 需要积累足够历史数据 | Skill Forge |
| 自动创建新 check 项 | ❌ 不能 — 需要人工确认 | 自动改 rubric 的风险太高 | — |
| 信任分管理 | ✅ 可以 — 简单的 pass/fail 计数 + 趋势 | 需要定义"信任分"的计算公式 | Toryo |
| 跨 pipeline 关联 | ⚠️ 部分 — 需要手动映射数据源 | 日报和良率的数据源映射还没做 | — |
| 飞书推送 insight | ✅ 可以 — feishu gateway 已有 | — | — |

---

## 8. 附录：关键论文与项目索引

| 来源 | 类型 | 核心概念 | 链接 |
|------|------|---------|------|
| Skill Forge v2 | GitHub 项目 | Hypothesis→Mutate→Score→Keep/Revert; Coverage Matrix; Exploration-Exploitation | <https://github.com/GodModeAI2025/skill-forge> |
| Toryo | GitHub 项目 | Quality ratcheting; Trust-based delegation; Ralph Loop | <https://github.com/JesseRWeigel/toryo> |
| ForgeGod | GitHub 项目 | 五层认知记忆; SICA; Recon; Adversarial plan debate | <https://github.com/waitdeadai/forgegod> |
| Self-Healing Orchestrators | arXiv:2606.01416 | 失败分类→精准恢复; 98.8% 成功率; 静默失败清零 | <https://arxiv.org/abs/2606.01416> |
| SkillWeaver | arXiv:2606.18051 | 组合式 skill routing; 99% context 压缩 | <https://arxiv.org/abs/2606.18051> |
| Adaptation of Agentic AI | arXiv:2512.16301 | T1(agent-agnostic) vs T2(agent-supervised) 四范式 | <https://arxiv.org/abs/2512.16301> |
| Metacognitive Feedback (RL) | arXiv:2606.32032 | RL with metacognition 提升 LLM 可信度 | <https://arxiv.org/abs/2606.32032> |
| loop-engineering v0.3.0 | 本 repo skill | 三层架构 + 六个模式 + 五步骤 Orange Book | `skill:loop-engineering` |

---

> **下一步**：确认 verification_report.json 的统一格式 → 启动 Phase 1 cron job。
