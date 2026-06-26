# 🤖 AI 学习日报 — 2026-06-25

> 覆盖：arXiv、Hacker News、核心博客（Lilian Weng / Simon Willison / Anthropic）  
> 范围：2026-06-24 ~ 06-25  
> 来源：45+ 次搜索，20 篇 arXiv + 11 条 HN + 5 篇博客

---

## 🔥 今日必读（5 篇，按优先级排序）

### 1. ⭐ Lilian Weng：「Scaling Laws, Carefully」

**链接**：https://lilianweng.github.io/posts/2026-06-24-scaling-laws/

从 Amari (1992) 到 Kaplan → Chinchilla → 2026 前沿，一篇把 scaling laws 讲透。关键洞察：
- 幂律指数是任务域的属性，不是模型架构的属性
- Kaplan vs Chinchilla 的矛盾根源：数据无限 vs 数据受限两种假设
- 小规模实验外推到大模型时的 3 个常见陷阱

**对你的意义**：你正在从「用模型」过渡到「设计 agent 系统」，理解模型能力的成本-性能 tradeoff 是底层基本功。阶段 1 必读。

---

### 2. ⭐ arXiv：「Instruction Bleed」— 多 prompt 模块间的隐性干扰

**链接**：https://arxiv.org/abs/2606.26356

发现了一个致命问题：在 ReAct / multi-tool agent 中，编辑一个 prompt 模块（比如 Planner）会**悄无声息地改变另一个模块（比如 Critic）的行为**——即使两者没有共享变量或依赖。根因是 transformer 的自注意力在共享上下文窗口中不隔离。

**对你的意义**：你的每个 skill 都是多 prompt 模块组成的（system prompt + tool descriptions + 输出格式 + 边界规则）。这篇论文直接告诉你：**模块化 prompt 是一个有漏洞的抽象**，必须做整体测试而不能只测单个模块。

---

### 3. ⭐ DeepSeek Flash：「Code-as-Plan」颠覆 agent 经济学

**链接**：https://www.rtrvr.ai/blog/code-as-plan-deepseek-flash-text-only-browser-agent

架构思路：让模型**一次性生成完整工作流代码**，本地 harness 执行——一个原本需要 80+ 次 API 调用的任务变成 1 次。DeepSeek V4 Flash 便宜、开源、代码能力够用，让这个模式成为现实。

核心金句：*"Developers were being milked for runtime, not intelligence."*

**对你的意义**：你的 agent 如果每个 task 烧几十次模型调用，架构成本就是竞争劣势。护城河从「有模型访问权」变成「harness 设计质量」。

---

### 4. arXiv：67 个模型的 Co-Failure Ceiling — 多模型系统有硬上限

**链接**：https://arxiv.org/abs/2606.27288

在 67 个前沿模型上的实验证明：多模型系统（routing、voting、mixture-of-agents）的准确率永远超不过 **1 - β**，其中 β 是「所有模型同时出错」的概率。只看单个模型准确率会漏掉这个关键指标。

**对你的意义**：如果你将来考虑用多个模型做 routing 或 ensemble，**先测 β**。单模型准确率再高，如果 β 也高，组合系统就是白花钱。

---

### 5. Simon Willison：Prompt Injection 的根因是「角色混淆」

**链接**：https://simonwillison.net/2026/Jun/22/prompt-injection-as-role-confusion/

LLM 遵循的是文本**风格**，不是 `<user>`/`<system>` 这样的角色标签。把注入文本去风格化（destyling），攻击成功率从 61% 降到 10%。结论：在模型真正具备角色感知能力之前，注入防御就是打地鼠。

**对你的意义**：别把 system prompt 当安全边界。你的 agent 在 Telegram 等公开渠道运行时，用户输入可以模仿你的 prompt 风格来逃逸。防御策略是降到风格层。

---

## 📋 其他值得扫一眼的

### Agent 架构 & 安全

| 论文 | 一句话 |
|------|--------|
| [Semantic Early-Stopping](https://arxiv.org/abs/2606.27009) (Jun 25) | 用语义相似度替代 `max_iterations`——简单任务早停省 token，难任务给够空间 |
| [ShareLock: MCP 投毒攻击](https://arxiv.org/abs/2606.27027) (Jun 25) | 首篇针对 Model Context Protocol 的多工具投毒攻击论文。你用 MCP 的话留意 |
| [LLM Agent 的 CoT 训练收益去哪了](https://arxiv.org/abs/2606.26935) (Jun 25) | CoT 提升的是动作选择还是事后解释？直接影响 agent 架构要不要内置思考链 |
| [Agent 配置的 Governance 问题](https://arxiv.org/abs/2606.26924) (Jun 25) | 扫描 10008 个 repo，10.1% 的 agent 配置文件完全重复——版本管理灾难 |

### Agentic Coding

| 来源 | 内容 |
|------|------|
| APIMatic Blog (Jun 24) | 从 Vibe Coding 到 Agentic Engineering 的 3 步法：Structured Prompt → Plan Mode → Review + Walkthrough |
| Anthropic (Jun 16) | Claude Code 在生产环境的经济效益实证研究 |
| GitHub/Orchid (Jun 24) | agent 调试的 record-and-replay 工具，解决可观测性缺口 |

### 生态 & 政策

| 来源 | 内容 |
|------|------|
| Simon Willison (Jun 25) | 德国法院裁定：企业对其 AI agent 的错误承担法律责任。为全球 agent 问责制设先例 |

---

## 🎯 学习路径映射

| 发现 | 对应阶段 | 即时行动 |
|------|---------|---------|
| Scaling Laws (Lilian Weng) | **阶段 1**: LLM 基础 | 今天这篇作为阶段 1 的第 1 篇精读，比原计划的 Lilian Weng agent 文章更优先——先懂模型再懂 agent |
| Instruction Bleed | **阶段 2**: Agent 架构 | 你现有每个 skill 的 prompt 模块都需要整体测试，不是单模块测试 |
| Code-as-Plan | **阶段 3-4**: 实践+评测 | 你的 agent loop 设计应该考虑「生成可执行代码」vs「逐步 tool-calling」两种范式的 tradeoff |
| Co-Failure Ceiling | **阶段 4**: 评测方法论 | 如果你将来用多模型 routing，β 是最重要的指标 |
| Prompt Injection 角色混淆 | **全阶段**: 安全意识 | 每个公开部署的 agent 都需要 destyling 防御 |

---

## 💡 今日核心收获（6 条）

1. **Prompt 模块化是有漏洞的抽象**——只测单个模块不够，必须测整体
2. **Agent 经济学正在被重写**——Code-as-Plan 把 80 次调用压到 1 次
3. **角色标签不是安全边界**——LLM 看的是风格，不是 `<system>` vs `<user>`
4. **Co-Failure Rate（β）是多模型系统的关键指标**——高准确率 ≠ 低共败率
5. **Agentic Engineering > Vibe Coding**——结构化 prompt + plan + review
6. **Agent 配置是治理问题**——集中管理、版本控制，不散落在各个 repo 里

---

## 📊 来源统计

| 来源 | 数量 | 高相关 |
|------|------|--------|
| arXiv cs.AI | 20 篇 | 12 篇 agent 直接相关 |
| arXiv cs.MA | 10 篇 | Instruction Bleed 等 |
| Hacker News | 11 条 | agent 工具 + 经济 + 调试 |
| 博客 | 5 篇 | Scaling Laws / Prompt Injection / Agentic Engineering |
| Anthropic Research | 2 篇 | Agentic Coding / NL Autoencoder |

---

> **关联**：学习资源清单 `docs/agent-learning-resources-v0.0.0.md`  
> **下一篇**：2026-06-26
