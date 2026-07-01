# AI 学习日报 — 2026-07-01

> 主题：Agentic Coding 的实证数据——领域知识，而非编程技能，决定 agent 协作成败
> 路径阶段：阶段 2（Agent 架构）→ 阶段 3（Vibe Coding 工程化）

## 1. 今日主题

**你需要的是"懂问题的人"，不是"会写代码的人"**

Anthropic 发布了 Claude Code 使用行为的大规模实证研究——基于 2025.10-2026.04 间 ~40 万次交互会话、~23.5 万用户的数据。结论颠覆直觉：决定 agentic coding 成功率的不是编程水平，而是**领域专业度**（domain expertise）。这对 Daddy 目前从阶段 2（Agent 架构）走向阶段 3（Vibe Coding 工程化）尤为关键——我们的核心任务不是写更好的代码，而是**写更好的 prompt/plan/spec**，让 agent 去执行。

## 2. 学习材料

### 必读：Agentic Coding and Persistent Returns to Expertise
- 来源：<https://www.anthropic.com/research/claude-code-expertise>
- 作者/机构：Anthropic Economic Research Team (Jun 16, 2026)
- 核心结论：
  1. **分工模式已稳定**：人类做 70% 的 planning 决策（做什么/什么算完成），agent 做 80% 的 execution 决策（怎么改代码/用什么语言/跑什么命令）
  2. **专业度决定产出量**：expert 用户每次 prompt 触发 ~12 个 action + 3,200 字输出，novice 仅 ~5 action + 600 字——差距 2-5 倍
  3. **成功率梯度**：novice verified success 15%，intermediate 28%，expert 33%——最大提升在 novice→intermediate
  4. **容错能力差异更大**：trouble session 中的 verified success：novice 4% vs expert 15%
  5. **debug 占比从 33% 降到 19%**（7 个月内），operating/writing/analysis 翻倍——agent 越来越能做"端到端"任务

### 可选：Scaling Laws, Carefully
- 来源：<https://lilianweng.github.io/posts/2026-06-24-scaling-laws/>
- 作者：Lilian Weng (Jun 24, 2026)
- 核心结论：深入梳理了从 Amari (1992) 到 Chinchilla (2022) 的 scaling laws 演进，关键 insight——**预测 loss 可以用小模型外推大模型**，数据量需要按 N^(α/β) 的比例跟随模型规模增长。对 agent 设计的启发：验证性实验（小规模 loop）的结果能外推到大系统的行为。

## 3. 关键洞见

1. **"spec 质量 > 代码能力"被实证化**：Anthropic 数据直接量化了这一点。Expert 用户不是更会写代码，而是更清楚"我想要什么结果"、"什么算成功"、"边界条件是什么"。这正是 vibe coding workflow 需要工程化的核心——spec-first, not code-first。

2. **Novice→Intermediate 是最大杠杆**：大部分成功率的提升来自从"新手"到"中级"的跳跃。这意味着我们不需要成为领域顶级专家才能用好 agent——**有基本的领域概念 + 清晰的验收标准就够了**。

3. **容错是 agent 的天花板**：Trouble session 中 novice 只有 4% 能恢复成功，expert 有 15%。expert 不是不犯错，而是能**识别错误 + 给 agent 正确的纠偏指令**。这对应我们的 loop-engineering skill——adversarial verifier 的价值就在此。

4. **Agent 正在从"写代码工具"变成"端到端工作引擎"**：debug 占比降了近一半，operating/writing/analysis 翻倍。这与 Daddy 用 Hermes 的轨迹一致——从最初写代码，到管理多 pipeline（日报/看板/矿场）。

5. **Scaling Laws 的元教训**：小规模实验可外推。我们在 Hermes 里做的 loop-engineering 验证、daily digest 闭环、subagent 协作——这些"小实验"的结果可以用来预测更大系统的行为模式。

## 4. 行动 & Skill 提案

### 今日行动
- [ ] 审视现有 `vibe-coding-workflow` skill 初稿，加入"领域知识前置检查"步骤（参考 Anthropic 的 expertise classifier 三信号：指令精确度/验证请求/纠错方向）
- [ ] 在 daily 使用 Hermes 时，刻意练习"spec-first"——先写清楚验收标准再让 agent 动手

### Skill 提案
- **更新**：`vibe-coding-workflow`
- **原因**：Anthropic 数据提供了可操作的 spec 质量标准——expertise 的三个信号（指令精确度、验证需求、纠错方向）可直接内化为 skill 步骤
- **改动点**：
  - 在"需求澄清"阶段加入三个 check：①你是否能说出验收标准？②你是否知道什么算失败？③你能识别 agent 的哪个输出是错的吗？
  - 加入"expertise ladder"概念：首次接触新领域时预期 novice，第二次 intermediate，第三次后才能达到 expert 级输出
- **验收标准**：用该 skill 完成一个跨领域任务（如矿场日报+HTML 生成），对比未用 skill 时的成功率差异

## 5. 其他前沿动态

- **Anthropic "Economic Index: Cadences"** (Jun 26) — 首次按小时采样分析 Claude 使用节奏 <https://www.anthropic.com/research/economic-index-june-2026-report>
- **Anthropic "Project Fetch: Phase two"** (Jun 18) — Frontier Red Team 安全评估第二阶段 <https://www.anthropic.com/research/project-fetch-phase-two>
- **Anthropic "Teaching Claude why"** (May 8) — 减少 agentic misalignment 的新方法（将"为什么"作为训练信号） <https://www.anthropic.com/research/teaching-claude-why>
