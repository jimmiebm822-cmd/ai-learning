# Loop Engineering 学习文件

> 基于 Anthropic Claude Code Dynamic Workflows（2026-06-02）蒸馏 + Hermes 适配。
> 读完这篇，你会理解：这篇文章在讲什么、你和前沿差在哪、已经蒸馏出什么可用的东西。

---

## 提纲

1. [这篇文章在讲什么](#1-这篇文章在讲什么)
   - 1.1 一句话总结
   - 1.2 核心主张
   - 1.3 关键概念速查
2. [为什么要搞这个——三大失败模式](#2-为什么要搞这个三大失败模式)
   - 2.1 Agentic Laziness
   - 2.2 Self-preferential Bias
   - 2.3 Goal Drift
   - 2.4 三者关系
3. [怎么搞——六大工作流模式](#3-怎么搞六大工作流模式)
   - 3.1 模式全景
   - 3.2 ⭐ Adversarial Verification
   - 3.3 Fan-out-and-Synthesize
   - 3.4 Loop Until Done
   - 3.5 Tournament
   - 3.6 其他模式
   - 3.7 模式决策树
4. [你和 Anthropic 的差距](#4-你和-anthropic-的差距)
   - 4.1 现有基础设施
   - 4.2 差距矩阵
   - 4.3 核心缺口
5. [蒸馏出的 Skills](#5-蒸馏出的-skills)
   - 5.1 loop-engineering skill v0.0.0
   - 5.2 各模式 Hermes 落地
   - 5.3 使用指南
6. [落地路线图](#6-落地路线图)
   - 6.1 今天一个改动
   - 6.2 本周验证
   - 6.3 迭代方向
7. [附录](#7-附录)

---

## 1. 这篇文章在讲什么

### 1.1 一句话总结

> **别再让一个 agent 从头跑到尾了。** 把任务拆给多个独立 subagent，各自有干净的 context window，用结构化模式（对抗验证/分拆合成/锦标赛/循环）保证质量和收敛。

### 1.2 核心主张

Anthropic 发现了一个硬问题：**单体 agent 在长任务中必然退化**。不管 prompt 写得多好，agent 跑着跑着就会懒、会自欺、会忘记原始目标。

解决方案不是写更好的 prompt，而是**换架构**——从"一个 agent 扛全部"变成"多个 agent 协同，每个只扛一小块，互相检查"。

这就是 Dynamic Workflows：Claude 自己写 JS 脚本来编排 subagent。

### 1.3 关键概念速查

| 概念 | 一句话 |
|------|--------|
| Harness（挽具） | agent 的执行框架——决定它怎么跑循环、怎么 spawn 子任务 |
| Dynamic workflow | agent 在运行时自己写的 harness，不是预设的固定流程 |
| Subagent | 独立 context window 的子 agent，执行单一任务 |
| Adversarial verification | 用一个独立 agent 审查另一个 agent 的产出 |
| Context isolation | 每个 subagent 有自己的 context，互不污染 |

---

## 2. 为什么要搞这个——三大失败模式

这三个问题是单体 agent 的**结构性缺陷**，不是"prompt 没写好"能解决的。

### 2.1 Agentic Laziness（代理懒散）

> *"addressing 35 of the 50 items in a security review and declares the job done."*

Agent 在长任务中会**提前交差**——做了一部分就说做完了。因为 context 太长，它失去了"还有哪些没做"的追踪。

**根因**：单体 context window 无法可靠追踪大规模任务完成度。

**解决**：把 50 项拆成 50 个独立 subagent，每个只做一项 + 验证一项。没有"做一半交差"的空间。

### 2.2 Self-preferential Bias（自我偏好偏差）

> *"Claude's tendency to prefer its own results or findings, especially when asked to verify or judge them against a rubric."*

**自己检查自己的产出**——判自己的结果是无误的。这是"自审"的结构性缺陷。

**根因**：产出 agent 和验证 agent 是同一个，没有独立视角。

**解决**：spawn 一个独立的 verifier subagent，不和产出 agent 共享 prompt/身份。verifier 只知道"按这个 rubric 审查这个结果"，不知道是谁做的。

### 2.3 Goal Drift（目标漂移）

> *"The gradual loss of fidelity to the original objective across many turns, especially after compaction."*

多轮对话后，原始目标被压缩丢失。每次 summarization 是有损的，"别做 X""边界条件"这些细节优先被吞掉。

**根因**：长上下文压缩是 lossy compression。

**解决**：break 成多个短上下文 subagent，每个在"新鲜的" context 里从明确的 prompt 开始。原始目标不经过压缩。

### 2.4 三者关系

```
            单体 agent 长任务
                  │
     ┌────────────┼────────────┐
     ▼            ▼            ▼
  Laziness      Bias        Drift
  (做不完)    (自欺)    (忘记目标)
     │            │            │
     └────────────┼────────────┘
                  ▼
        三大失败模式叠加 = 质量崩塌
                  │
                  ▼
        换架构：多 agent + 独立 context
```

> **一句话**：一个 agent 跑久了必然懒 + 自欺 + 忘目标。每个 subagent 只跑一小段，这些问题自然消失。

---

## 3. 怎么搞——六大工作流模式

### 3.1 模式全景

```
                任务入口
                   │
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
Classify      Fan-out        Tournament
(分类路由)    (分拆并行)      (竞争选优)
    │              │              │
    ▼              ▼              ▼
 路由到        N agent        N agent
 对应 agent    并行执行       竞争同任务
    │              │              │
    └──────────────┼──────────────┘
                   ▼
          Adversarial Verification
          (独立 agent 审查产出)
                   │
          ┌────────┼────────┐
          ▼                 ▼
      pass → 交付      fail → 修正回环
                            │
                     Loop Until Done
                     (循环直到条件满足)
```

| # | 模式 | 一句话 | Hermes 有没有 |
|---|------|--------|-------------|
| 1 | Classify-and-act | 先分类，再决定谁做 | ⚠️ 手动做 |
| 2 | Fan-out-and-synthesize | 拆 N 份，并行跑，最后合 | ✅ 有（delegate_task batch） |
| 3 | **Adversarial Verification** | 另一个 agent 来审你的活 | ❌ **没有** |
| 4 | Generate-and-filter | 批量生成 → 过滤去重 | ⚠️ 手动组合 |
| 5 | Tournament | N 个 agent 竞争，pairwise 评判 | ⚠️ 手动组合 |
| 6 | Loop until done | 循环跑，直到停止条件满足 | ⚠️ 循环有，停止条件没标准化 |

### 3.2 ⭐ Adversarial Verification（对抗性验证）— P0 缺口

这是整个文章里**最有价值的单个模式**，也是你当前最大的缺口。

**怎么做**：

```
    ┌─────────────────┐
    │ 产出版 agent     │  生成日报/报表/代码改动
    │ "完成日报数据     │
    │  抓取+计算"      │
    └────────┬────────┘
             │ 产出文件
             ▼
    ┌─────────────────┐
    │ 验证版 agent     │  独立 spawn，不同 prompt
    │ "按 rubric 逐项  │  ★ 不知道产出 agent 的 prompt/身份
    │  审查这个文件"    │
    └────────┬────────┘
             │
      ┌──────┼──────┐
      ▼      ▼      ▼
    pass   fail  inconclusive
    (交付) (修正) (上报人)
           │
     最多 3 轮
```

**验证 rubric 模板**：

```
1. 数据源完整性：{检查项}
2. 计算准确性：{抽样 N 条验证}
3. 边界值：{极值/空值/异常值}
4. 逻辑一致性：{前后数据自洽}
5. {任务特定检查项}
```

### 3.3 Fan-out-and-Synthesize（分拆合成）— 已有

```
      主 agent：拆任务
           │
    ┌──────┼──────┐
    ▼      ▼      ▼
  Sub A  Sub B  Sub C  并行执行，独立 context
    │      │      │
    └──────┼──────┘
           ▼
      主 agent：合成
      (去重 + 交叉验证 + 补遗漏)
```

Hermes 的 `delegate_task(tasks=[...])` 原生支持。你已经在用了。

### 3.4 Loop Until Done（循环到完成）— P1

```
        开始
         │
         ▼
    ┌─────────┐     不满足     ┌──────────────┐
    │ 执行    │──────────────▶│ 检查条件      │
    └─────────┘               │ - 无新发现？  │
         ▲                    │ - 错误清零？  │
         │                    │ - 质量达标？  │
         │    满足            └──────┬───────┘
         │          ┌───────────────┘
         │          ▼
         └──── 继续 / 达标
                        │
                        ▼
                      完成
                 (或 max_iterations → ⚠️ 上报)
```

### 3.5 Tournament（锦标赛）

多个 agent 竞争同一任务，用不同方法。pairwise comparison agent 评判，逐轮淘汰直到胜者。适合有明确评判标准、需要最优解的场景。

### 3.6 其他模式

- **Classify-and-act**：分类 agent → 路由到对应处理 agent 或模型
- **Generate-and-filter**：批量生成 → rubric 过滤 → 去重 → 只留高质量项

### 3.7 模式决策树

```
你的任务：
     │
     ├─ 不知道分给谁？ ──→ Classify-and-act
     │
     ├─ 大量可并行小任务？ ──→ Fan-out-and-synthesize
     │    │
     │    └─ 产出质量关键？ ──→ + Adversarial Verification
     │
     ├─ 需最优解、有评判标准？ ──→ Tournament
     │
     ├─ 工作量不确定？ ──→ Loop Until Done
     │
     └─ 简单一步任务？ ──→ 别用 workflow，浪费 token
```

---

---

## 3.8 上层架构：五步骤框架（Orange Book）

> 来源：HuaShu Orange Book，基于 Anthropic 工程师公开言论。Loop engineering 是第四层（prompt → context → harness → **loop**）。

| # | 步骤 | 英文 | 核心问题 | 对应六大模式 |
|---|------|------|---------|------------|
| 1 | 发现 | Discovery | 有什么需要做的？ | Classify-and-act |
| 2 | 交接 | Handoff | 谁来做？ | Fan-out / Tournament |
| 3 | 验证 | Verification | 做对了吗？ | Adversarial Verification |
| 4 | 持久化 | Persistence | 结果存哪？ | — |
| 5 | 调度 | Scheduling | 下次什么时候跑？ | Loop Until Done |

**五步骤是上层流程骨架，六大模式是实现各步骤的具体方法。**

## 3.9 四种隐性成本（Orange Book）

Loop engineering **不是免费的**。设计 loop 前必须评估：

| # | 成本 | 英文 | 表现 | 怎么防 |
|---|------|------|------|--------|
| 1 | 验证债 | Verification debt | 未处理的验证失败堆积 | 连续 fail≥3 强制人介入 |
| 2 | 理解腐烂 | Comprehension rot | agent 长期跑后偏离原意图 | N 轮后 reset context |
| 3 | 认知投降 | Cognitive surrender | 人过度信任自动输出 | Adversarial Verification 本身就是解 |
| 4 | Token 爆炸 | Token blowout | 嵌套 loop 烧 token 指数增 | 简单任务不用 loop |

> **核心金句**：*"Loops make generation nearly free and leave judgment as the scarce resource."*

---

## 4. 你和 Anthropic 的差距

### 4.1 现有基础设施

你已经有了一套相当完整的多 agent 系统：

| 组件 | 作用 | 对标 |
|------|------|------|
| 4 个 Hermes profile | 按领域分 agent（矿场/看板/代托/主控） | ≈ Claude Code 的多个 project context |
| kanban 编排 | 卡片流转驱动 agent 执行 | ≈ workflow 的编排层 |
| delegate_task batch | 并行 spawn subagent | ≈ Fan-out-and-synthesize |
| skill 系统 | 可复用操作模板 | ≈ Claude Code skills |
| cron + 通知式串行 | 定时 + 跨 profile 联动 | ≈ Claude Code /loop |

### 4.2 差距矩阵

```
                    你的现状          Anthropic 前沿         差距
                    ─────────        ──────────────         ────
多 agent 编排      ████████░░ 80%    ██████████ 100%       小
Fan-out 并行       █████████░ 90%    ██████████ 100%       微
对抗性验证          ░░░░░░░░░░  0%    ██████████ 100%       大 ⚠️
循环终止标准化      ████░░░░░░ 40%    ██████████ 100%       中
模式标准化(skill)   ██░░░░░░░░ 20%    █████████░ 90%        中
模型智能路由        ░░░░░░░░░░  0%    ████████░░ 80%        中(次要)
Context 隔离       ████████░░ 80%    ██████████ 100%       小
中断恢复           █████████░ 90%    ██████████ 100%       微
```

### 4.3 核心缺口

**最大的 gap：没有"独立验证"机制。**

你现在 agent 做完日报、报表、代码改动——谁来审查？agent 自己。这是 self-preferential bias 的直接体现。

Anthropic 的答案是：**永远不要让产出 agent 验证自己。** spawn 一个独立的 verifier，用不同 prompt，不看产出 agent 的 reasoning。

要做到这一点，在 Hermes 里只需要：
```
delegate_task(goal="完成日报")
→ delegate_task(goal="按 rubric 审查日报", context="你是独立验证者，不共享产出 agent 的 prompt")
```

---

## 5. 蒸馏出的 Skills

### 5.1 loop-engineering skill v0.2.1

已创建，路径：`~/.hermes/profiles/prof_bot/skills/devops/loop-engineering/SKILL.md`

**版本演进**：

| 版本 | 新增 |
|:--:|------|
| v0.0.0 | 创立，三大失败模式 + 六大模式速查 + Adversarial Verification |
| v0.1.0 | Loop Until Done 可执行模板 + Tournament（含 pairwise 评判代码） |
| v0.2.0 | 推广模式：通用嵌入法 + 日报/良率/邮件三套 rubric 模板 + 决策表 |
| v0.2.1 | 五步骤框架 + 四种隐性成本（Orange Book） |

**当前全貌**：
- 三大失败模式 + 四种隐性成本（知道在对抗什么 + 花什么代价）
- 五步骤框架 + 六大模式（道 + 术）
- 推广模式：如何嵌入到日报/良率/邮件等 pipeline
- 适用边界 + 失败条件 + 反模式（何时不该用）

### 5.2 各模式 Hermes 落地方式

| 模式 | Hermes 实现 | 一条命令 |
|------|-----------|---------|
| Adversarial Verification | delegate_task(做) + delegate_task(审) | spawn 两个独立 subagent |
| Fan-out | delegate_task(tasks=[...]) batch | 原生支持 |
| Loop Until Done | kanban 循环卡片 + stopping_condition 字段 | 卡片 type=loop + 每轮 eval |
| Tournament | delegate_task N agent + pairwise compare | 手动组合，暂无模板 |
| Classify-and-act | prof_bot 先分类，再路由到对应 profile | 手动分派，暂无自动 |

### 5.3 使用指南

在下次关键任务（日报、报表、良率计算）时：

1. 我开始执行任务前，先判断：产出有质量风险吗？
2. 如果有 → 加载 `loop-engineering` skill → 启用 Adversarial Verification
3. 产出版做完 → spawn 验证版 → 按 rubric 审查 → pass 才交付

---

## 6. 落地路线图

### 6.1 今天一个改动

在**下一次日报生成任务**中，加一行：
> "做完日报后，spawn 一个独立 verifier agent 按以下 rubric 审查：数据源完整性 / 计算准确性抽样 5 条 / 边界值 / 逻辑一致性"

这是 P0 对抗性验证的第一次实战。

### 6.2 本周验证

- 跑 3-5 次带对抗验证的日报/报表任务
- 记录 verifier 抓到的问题数量 vs 传统自审
- 评估 token 成本（验证版确实会多烧 token）

### 6.3 迭代方向

| 优先级 | 动作 | 状态 |
|--------|------|:--:|
| P0 | 对抗性验证常态化（daily-card-update v0.1.0 Step 4） | ✅ 已嵌入 |
| P0 | 推广模式（通用嵌入法 + rubric 模板） | ✅ v0.2.0 |
| P1 | Loop Until Done stopping condition 标准化 | ⏳ 模板已写，待实战 |
| P2 | Tournament / Generate-filter 模板化 | ⏳ 模板已写，待实战 |
| P3 | 模型智能路由 | 待做 |

### 7. 附录

### 7.1 原文章链接
- [Anthropic: A harness for every task — dynamic workflows in Claude Code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)（2026-06-02）
- **Orange Book**: Loop Engineering: The Anthropic Playbook（HuaShu 整理，基于 Addy Osmani / Peter Steinberger 公开言论）— 五步骤框架 + 四种隐性成本

### 7.2 跨域验证
- [ComPilot: Agentic Auto-Scheduling (arXiv 2511.00592)](https://arxiv.org/abs/2511.00592) — closed-loop 比 open-loop 好 23-40%；agent premature stopping；multi-run K=5 最优边际收益。编译器领域的 loop 实验为 agent orchestration loop 提供了独立量化验证。

### 7.3 LLM Wiki 页面
- `LLM Wiki/concepts/loop-engineering.md`
- `LLM Wiki/concepts/agent-orchestration.md`（已更新，新增模式 3）
- `LLM Wiki/raw/articles/anthropic-dynamic-workflows-2026.md`

### 7.3 Skill 路径
- `~/.hermes/profiles/prof_bot/skills/devops/loop-engineering/SKILL.md`
- `workspace/loop-engineering-distill-draft.md`（完整蒸馏文档）

---

> *"Creating a workflow helps combat these by orchestrating separate Claude subagents with their own context windows and focused, isolated goals."*
> — Anthropic, June 2026
