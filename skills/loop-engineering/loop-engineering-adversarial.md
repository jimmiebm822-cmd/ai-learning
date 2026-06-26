---
name: loop-engineering-adversarial
description: Adversarial Verification pattern — independent verifier subagent reviews output against rubric. Part of loop-engineering suite. Load when output quality is critical.
version: 0.1.0
---

# loop-engineering-adversarial v0.1.0

> **加载时机**：当 agent 产出有质量风险且不能由产出 agent 自己审时——日报数据、报表统计、良率计算、代码改动关键路径。
> **前置依赖**：理解 [loop-engineering](skill:loop-engineering) 的三大失败模式和 self-preferential bias 概念。

## 触发条件

Agent 在以下场景应加载本 skill（与 loop-engineering core 互补）：
- 关键产出（报表/数据/代码改动）需要自我验证但**不能由产出 agent 自己审**
- 需要可复用的验证 rubric 模板

## Adversarial Verification（P0 — Hermes 核心缺口）

### 何时用
任何产出有质量风险且**不能由产出 agent 自己审**的场景——日报数据、报表统计、良率计算、代码改动关键路径。

### 步骤
1. **产出版 agent** 生成交付物，标注 `⚠️ 待验证`
2. **验证版 agent**（独立 spawn，不同 prompt）按 rubric 逐项审查：
   - rubric 来自原任务需求或 checklist
   - **不告诉验证 agent 产出 agent 的 prompt/身份**（防自欺）
3. 验证结果：
   - ✅ pass at %score% → 交付
   - ❌ fail, N issues → 产出版修正 → 重新验证（最多 3 轮）
   - ⚠️ inconclusive → 上报人介入

### 验证 rubric 骨架模板
```markdown
## 验证检查项
1. 数据源完整性：{检查项}
2. 计算准确性：{抽样验证 N 条}
3. 边界值：{检查极值/空值/异常值}
4. 逻辑一致性：{前后数据自洽}
5. {其他任务特定检查项}
```

> 各 pipeline 的实例化 rubric 见 [rubric-templates](skill-ref:loop-engineering/rubric-templates) — 含日报/良率/邮件三套完整模板。

### Hermes 实现
- **产出版**：delegate_task(goal="完成日报数据抓取+计算", toolsets=["terminal","file","web"])
- **验证版**：delegate_task(goal="按 rubric 逐项审查 {产出文件}", context="独立验证，不与产出 agent 共享 prompt", toolsets=["terminal","file"])
- 验证 agent 的 prompt 必须**不引用**产出 agent 的 prompt 或 reasoning
- 验证结果写入 `verification_report.json`：`{ pass: bool, score: 0-5, issues: [...], checked_at: ISO }`

### 嵌入式 vs 独立 cron 验证 — 决策表

| 维度 | 嵌入式验证（skill 内 Step N） | 独立 cron 验证 |
|------|------------------------------|----------------|
| 触发方式 | pipeline 运行时自动触发 | cron 定时触发（独立于 pipeline） |
| 适用场景 | 每次产出都需要验证的质量关卡 | 周期性巡检已有产出 |
| 时效性 | 即时 | 延迟 |
| 失败处理 | 修正→重验（最多 3 轮）→ 上报 | 告警通知 → 人介入 |

**决策逻辑**：前置条件→嵌入式；质量监控→cron；两者可共存。

## 参考
- [loop-engineering](skill:loop-engineering) — 核心概念（三大失败模式 + 五步骤 + 速查表）
- [loop-engineering-patterns](skill:loop-engineering-patterns) — Loop Until Done + Tournament
- [rubric-templates](skill-ref:loop-engineering/rubric-templates) — 日报/良率/邮件实例化 rubric