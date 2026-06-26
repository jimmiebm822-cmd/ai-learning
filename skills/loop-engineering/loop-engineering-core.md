---
name: loop-engineering
description: Agent loop engineering — core concepts, failure modes, cost model, and pattern index. Load first; then load loop-engineering-adversarial or loop-engineering-patterns for specific execution templates.
version: 0.3.0
---

# loop-engineering v0.3.0

> **重构说明 (v0.3.0)**：本 skill 拆分为三层 — core（本文件，概念+速查）、[loop-engineering-adversarial](skill:loop-engineering-adversarial)（对抗验证）、[loop-engineering-patterns](skill:loop-engineering-patterns)（循环+锦标赛）。Rubric 模板库移至 [references/rubric-templates.md](skill-ref:loop-engineering/rubric-templates)。

## 触发条件
Agent 在以下场景应先加载本 skill（快速扫概念 + 速查），再按需加载子 skill：
- 关键产出需要自我验证但**不能由产出 agent 自己审** → 加 [loop-engineering-adversarial](skill:loop-engineering-adversarial)
- 工作量不确定的迭代任务 → 加 [loop-engineering-patterns](skill:loop-engineering-patterns)
- 需要最优解的竞争任务 → 加 [loop-engineering-patterns](skill:loop-engineering-patterns)

不适用：简单一步任务、开放式创意、实时交互。

## 背景：三大失败模式
单体 agent 在长任务中必然退化：
1. **Agentic laziness** — 做一半就交差
2. **Self-preferential bias** — 自己审自己的产出判为无误
3. **Goal drift** — 长上下文压缩吃掉边界条件

解决方案：用独立 subagent 隔离 context + 对抗性验证 + 显式终止条件。

## 上层架构：五步骤框架（Orange Book）

> 来源：HuaShu Orange Book，基于 Anthropic 工程师公开言论。confidence: medium。

Loop engineering 在 harness 之上构成**第四层**（prompt → context → harness → **loop**）：

| # | 步骤 | 英文 | 核心问题 | 对应模式 |
|---|------|------|---------|---------|
| 1 | 发现 | Discovery | 有什么需要做的？ | Classify-and-act |
| 2 | 交接 | Handoff | 谁来做？ | Fan-out / Tournament |
| 3 | 验证 | Verification | 做对了吗？ | Adversarial Verification |
| 4 | 持久化 | Persistence | 结果存哪？ | — |
| 5 | 调度 | Scheduling | 下次什么时候跑？ | Loop Until Done |

## 四种隐性成本（Orange Book）

| # | 成本 | 英文 | 表现 | 怎么防 |
|---|------|------|------|--------|
| 1 | 验证债 | Verification debt | 未处理的验证失败堆积 | 连续 fail≥3 强制人介入 |
| 2 | 理解腐烂 | Comprehension rot | agent 长期跑后偏离原意图 | N 轮后 reset context |
| 3 | 认知投降 | Cognitive surrender | 人过度信任自动输出 | Adversarial Verification 本身就是解 |
| 4 | Token 爆炸 | Token blowout | 嵌套 loop 烧 token 指数增 | 简单任务不用 loop |

核心金句：**"Loops make generation nearly free and leave judgment as the scarce resource."**

## 六大模式速查

| 模式 | 做法 | Hermes 实现 | 执行模板 |
|------|------|-----------|---------|
| Classify-and-act | 分类→路由 | kanban 分类卡片 + profile 分派 | — |
| Fan-out-and-synthesize | 拆 N 份→并行→合成 | delegate_task batch | 原生支持 |
| **Adversarial Verification** ⭐ | 产出→独立验证→修正 | delegate_task(做) + delegate_task(审) | [adversarial](skill:loop-engineering-adversarial) |
| Generate-and-filter | 批量生成→过滤去重 | delegate_task 多生成 + 过滤 | — |
| Tournament | N 竞争→pairwise 评判 | delegate_task 并行 + pairwise | [patterns](skill:loop-engineering-patterns) |
| Loop until done | 循环→检查终止条件 | kanban 循环 + stopping_condition | [patterns](skill:loop-engineering-patterns) |

## 适用边界
- ✅ 有可验证交付物的任务
- ✅ 长运行（>5 步）或多并行（>2 个子任务）
- ❌ 简单一步完成（token waste）
- ❌ 开放式创意（无可验证 rubric）
- ❌ 强实时交互（loop latency 不可接受）

## 失败条件
- 验证 agent 偷懒（走过场审查）→ rubric 项必须具体可执行
- 对抗验证死循环（产出版和验证版持续拉锯）→ 3 轮上限 + 人介入
- Token 浪费（过度用 loop 做小事）→ 触发前评估复杂度

## 参考
- [loop-engineering-adversarial](skill:loop-engineering-adversarial) — 对抗性验证完整模板
- [loop-engineering-patterns](skill:loop-engineering-patterns) — Loop Until Done + Tournament 模板
- [rubric-templates](skill-ref:loop-engineering/rubric-templates) — 日报/良率/邮件实例化 rubric
- [learning-file-production](skill-ref:loop-engineering/learning-file-production) — 如何从研究材料生产学习文件（提纲→md→html→GitHub 发布）
- 蒸馏文档：workspace/loop-engineering-distill-draft.md
- Anthropic blog: https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code