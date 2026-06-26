---
name: loop-engineering-patterns
description: Agent iteration patterns — Loop Until Done and Tournament. Part of loop-engineering suite. Load when task has unknown scope or needs optimal solution selection.
version: 0.1.0
---

# loop-engineering-patterns v0.1.0

> **加载时机**：工作量不确定的迭代任务（Loop Until Done）或需要最优解的竞争任务（Tournament）。
> **前置依赖**：理解 [loop-engineering](skill:loop-engineering) 的六大模式速查和适用边界。

## Loop Until Done（P1 — 工作量不确定的迭代任务）

### 何时用
工作量未知的任务——bug 排查（有多深）、数据清洗（到无脏数据）、多轮研究（到无新发现）、邮件附件批量下载（不知道多少封）。

### 模板
```
1. DEFINE stopping condition（至少一个）：
   - "new_findings == 0"      # 本轮无新发现
   - "error_count == 0"        # 错误清零
   - "quality_score >= 0.9"    # 达标
2. SET max_iterations（兜底，建议 5-10）
3. LOOP:
   a. 执行本轮任务
   b. EVAL stopping condition
   c. IF 满足 → 标记 done
      ELSE 且 iter < max → 回到 a
      ELSE → ⚠️ 上报人 + 附当前状态
```

### Hermes 实现
- **方式 A（kanban 卡片）**：卡片 type=loop，字段 `stopping_condition` + `max_iterations`。每轮 agent 自行 eval 条件。
- **方式 B（delegate_task 循环）**：主 agent 在循环中 spawn delegate_task，每次检查返回的 `new_findings` / `error_count` → 决定继续/停止。
- **方式 C（cron 循环）**：每周/每日自动触发，每次检查上次产出 → 有新数据则继续，否则停止。适用周期性扫描任务（如邮件检查）。

### 示例：日报累计表数据补全
```
主 agent:
  iter = 0
  loop:
    result = delegate_task(goal="检查累计表最新日期，如滞后>1天则 merge 日报")
    if result.new_files == 0:
      break
    iter += 1
    if iter >= 5:
      alert("⚠️ 累计表补全超过 5 轮")
      break
```

### 反模式
- 无条件循环 = 死循环烧 token
- 用 Loop 做确定性工作（已知要跑 N 次）→ 用 Fan-out 更高效

## Tournament（P2 — 竞争择优）

### 何时用
有明确评判标准、需要最优解的场景：命名/文案/方案优选、代码实现多种对比、配置调优、skill evals。

### 模板
```
1. DEFINE task + rubric（评判标准，如"简洁/准确/可读性"各 1-5 分）
2. SPAWN N 个 agent，各自独立完成同一任务（不同方法）
3. RUN pairwise comparison：comparison agent 按 rubric 逐对评判
4. 晋级制或全量排序：
   - 晋级：每轮淘汰一半，直到剩 1 个
   - 全量：所有 agent 产出全量 pairwise 比较，排总序
5. RETURN 胜者 + runner-up + 评判理由
```

### Hermes 实现
```
# Step 1: 并行生成
results = delegate_task(tasks=[
  {goal: "从简洁角度设计 CLI 工具名"},
  {goal: "从行业惯例角度设计 CLI 工具名"},
  {goal: "从品牌记忆角度设计 CLI 工具名"},
])

# Step 2: pairwise 评判
judge = delegate_task(
  goal="按 rubric(简洁/准确/可读 各 1-5) pairwise 比较 3 个候选名，给出排名和理由",
  context=f"候选 A: {results[0]}\n候选 B: {results[1]}\n候选 C: {results[2]}"
)
```

### 反模式
- 任务本身没有客观评判标准（纯主观审美 = 无意义锦标赛）
- N 太少（N<3 无竞争意义，直接用 Fan-out+Adversarial 即可）
- 评判 agent 也是产出 agent 之一（自欺重演）→ 评判必须独立

## ⚠️ Tournament 和 Adversarial 不要混用
Tournament 已经有多 agent 竞争 + pairwise 评判，内部自带验证。**不要在 Tournament 外层再加 Adversarial Verification**——双层验证无增益且 double token。

## Fan-out-and-Synthesize（已有基础设施）

大量可并行的独立子任务——delegate_task(tasks=[...]) batch 模式原生支持。关键：合成阶段要**去重 + 交叉验证 + 补遗漏**，不是逐份粘贴。

## 参考
- [loop-engineering](skill:loop-engineering) — 核心概念 + 速查表 + 适用边界
- [loop-engineering-adversarial](skill:loop-engineering-adversarial) — 对抗性验证模式