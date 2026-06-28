# AI 学习路径 v0.1.0

> 目标：成为 vibe coding + agentic AI 专家，把每日学习蒸馏成可复用的 Hermes skills，持续提升 Prof.Bot 性能。

---

## 路径总览

| 阶段 | 主题 | 目标 | 已落地 / 待落地 | 输出形式 |
|---|---|---|---|---|
| **1. 基础机制** | LLM/Agent 内核 | 理解 agent 为什么会失败，feedback 如何驱动 agent | ✅ Karpathy minimalism (microgpt)；⚠️ RL/RLVR 待补 | 概念卡 + skill |
| **2. Agent 架构** | 多 agent 协作与验证 | 掌握 loop、subagent、adversarial verification、convergence | ✅ loop-engineering core/adversarial/patterns | skill 组 |
| **3. Vibe Coding 工程化** | 自然语言 → 可运行代码/skill | 建立稳定、可复现的 vibe coding 工作流 | ❌ Obsidian 笔记为空，待建 | workflow + skill |
| **4. 生产级能力** | 安全 / 成本 / 延迟 / fallback | 让 agent 能稳定跑在个人业务上 | ⚠️ 只有零散经验，待系统化 | skill 组 + rubric |
| **5. 领域 Agent (Personal AI OS)** | 矿场 / 日报 / 飞书等 | 把个人工作流封装成自治 agent | 🔄 持续迭代 | domain skills |

---

## 阶段详情

### 阶段 1：基础机制

**关键问题**
- LLM 是怎么“学会”的？训练 loss 和 agent feedback 有什么对应关系？
- 没有 ground-truth 时，如何用 RL 改进模型/ agent？
- 为什么 Karpathy 说 "feedback is the agent's training data"？

**核心材料**
- [Karpathy — microgpt](https://karpathy.github.io/2026/02/12/microgpt/)（已完成）
- RL without ground-truth（arXiv 2606.27377，日报 0627 已覆盖）
- 可选：DeepSeek-V3/V4 训练 post / OpenAI 模型经济学

**输出要求**
- 每个概念必须对应一条 Hermes 使用原则
- 例如："feedback = training data" → "每次 subagent 调用都要设计可验证的 reward signal"

### 阶段 2：Agent 架构

**关键问题**
- 单体 agent 的三大失败模式是什么？
- Adversarial verification、Loop Until Done、Tournament 怎么在 Hermes 里落地？
- 怎么防止 multi-agent 死循环和成本失控？

**核心材料**
- Anthropic — Dynamic Workflows in Claude Code（已蒸馏）
- 每日实践：给现有 Hermes skills 加 adversarial verifier

**输出要求**
- 每个模式必须有 Hermes skill 或 workflow 模板
- 必须有明确的 "何时使用" 决策树

### 阶段 3：Vibe Coding 工程化

**关键问题**
- 如何把自然语言需求稳定转化为可运行代码？
- prompt → plan → code → test → iterate 的闭环怎么做？
- 什么情况下应该写 skill，什么情况下应该直接 terminal？

**核心材料**
- Hermes `skill_authoring` 文档
- Claude Code / Codex 最佳实践
- 自己的翻车记录（Obsidian vibe-coding实践.md）

**输出要求**
- 建立 `vibe-coding-workflow` skill：需求澄清 → 最小可运行原型 → 测试 → 封装 skill
- 所有新功能必须能说出 "为什么不用一次 prompt 解决"

### 阶段 4：生产级能力

**关键问题**
- 公开渠道（飞书）的 prompt injection 怎么防？
- Multi-agent 成本怎么 cap？延迟怎么降？
- 模型/ provider 不稳定时怎么 fallback？

**核心材料**
- Agent 安全案例（Simon Willison hackmyclaw 等）
- Speculative decoding / 推理优化
- Provider 经济学与选型

**输出要求**
- `agent-safety-checklist` skill
- `provider-fallback` workflow
- 每个 agent 必须有 budget cap + stopping condition

### 阶段 5：领域 Agent (Personal AI OS)

**关键问题**
- 如何把矿场、日报、飞书等工作流抽象成 agent？
- 这些领域 agent 的输入/输出/失败模式是什么？
- 如何让 agent 持续从每日数据中学习？

**核心材料**
- 现有业务 pipeline（mining daily report、hardware-pm 等）
- 领域知识（LLM Wiki / Obsidian）

**输出要求**
- 每个业务 pipeline 对应一个 domain skill
- 每月回顾一次 skill 效果，决定升级/废弃

---

## 每日学习规则

1. **一篇为主，最多两篇**：不再堆 6 篇新闻式摘要；选择 1-2 个最相关、最深刻的点深入
2. **必须映射到阶段**：每篇材料标注属于哪个阶段
3. **必须有 actionable takeaway**：读完今天的内容，我明天使用 AI 时会做什么不同？
4. **必须有 skill 改进提案**：这个学习点应该更新哪个 skill，或新建什么 skill？
5. **每周回顾一次**：本周学习是否围绕当前阶段主线？是否需要调整路径？

---

## 当前阶段判断

- **主线阶段**：2（Agent 架构）→ 3（Vibe Coding 工程化）
- **阶段 1 已足够**，阶段 4/5 按需补强
- **最大缺口**：阶段 3 没有系统化，导致学了 loop engineering 但 vibe coding 效率没提升

---

## 下一步行动

1. 把本路径写入 `ai-learning/README.md`
2. 重写 daily digest 模板，按“1 主题 + 1 takeaway + 1 skill 提案”输出
3. 补完阶段 3：`vibe-coding-workflow` skill 初稿
4. 0627 日报的 6 条内容拆散：RL without ground-truth 单独成文并关联 loop-engineering skill；其他新闻只留 frontier watch
