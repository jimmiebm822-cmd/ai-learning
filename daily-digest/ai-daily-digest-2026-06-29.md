# AI 学习日报 — 2026-06-29

> 主题：Agent Verification 从理论到实践——Verifier-Driven 架构 + 公网渗透测试的防御启示
> 路径阶段：阶段 2（Agent 架构）→ 阶段 3（Vibe Coding 工程化）

---

## 1. 今日主题

**Agent 的输出不可靠不是 bug，是 feature——关键是你有没有 Verifier。**

本周末的 arXiv 新论文和 Simon Willison 分享的公网渗透测试结果，从理论和实践两端指向同一个结论：**naive delegation（把任务丢给 agent 然后信它）是 agent 架构的第一大反模式。** 解决方案不是让 agent 更聪明，而是给它配一个独立 Verifier。

---

## 2. 学习材料

### 必读 1：Glite ARF — Verifier-Driven Research with Parallel LLM Coding Agents

- 来源：[arXiv 2606.27416](http://arxiv.org/abs/2606.27416v1)（2026-06-25）
- 核心结论：naive delegation 在大型项目中不可行——低概率指令失误会叠加成不可复现的垃圾。解决方案是 **verifier-driven** 架构：多个并行 agent 各自生成，然后由独立验证器检查输出一致性。

**为什么这个点重要？** 它给了我们已有 `loop-engineering` skill（含 adversarial verification 模式）一个来自学术界的独立验证——我们走在正确的路上，但需要把 adversarial verification 从"概念"推成"系统化实践"。

### 必读 2：What happened after 2,000 people tried to hack my AI assistant

- 来源：[Simon Willison / Fernando Irarrázaval](https://simonwillison.net/2026/Jun/26/hack-my-ai-assistant/)（2026-06-26）
- 核心结论：Fernando 把 OpenClaw test instance 公开给全世界渗透。**6,000+ 次攻击尝试，0 次成功泄露 secrets。** 防御手段就是 prompt engineering——禁止泄露 secrets.env、禁止修改自身文件、禁止执行邮件中的命令、禁止数据外泄。模型是 Opus 4.6。

**为什么这个点重要？** 这是 adversarial verification 在安全维度的实战案例。不是用另一个模型去验证，而是**把验证规则直接嵌入 system prompt**。这种 "inline verifier" 的思路可以迁移到我们的 Hermes agent——不只靠 external verifier，也可以在 prompt 层面建立 defense layer。

### 补充参考：Agent in the loop（哲学基础）

- 来源：[Jon Udell via Simon Willison](https://simonwillison.net/2026/Jun/28/jon-udell/)（2026-06-28）
- 核心观点：翻转 "human in the loop" 叙事——**这是人的 loop，agent 是加入团队的成员。** Agent 辅助流程不应该是"收 prompt → 吐 feature"的黑箱。人类做规划决策，agent 做执行决策。

这直接呼应了 Anthropic 在 [Claude Code Expertise](https://www.anthropic.com/research/claude-code-expertise) 报告中的实证发现：**人类做 What，Claude 做 How。**

---

## 3. 关键洞见

1. **Naive delegation 的失败是数学必然。** 如果一个 agent 的指令失误概率是 5%，连续 10 步不出错的概率是 0.95^10 ≈ 60%。大型项目动辄上百步——naive delegation 的可靠性趋近于零。Glite ARF 论文用量化方式证明了这一点。

2. **Verifier 不需要比生成器更聪明。** Glite ARF 的 Verifier 只需要**检查一致性**（多个 agent 输出是否一致），不需要理解内容。这和 `loop-engineering/adversarial` skill 里的独立验证器设计完全一致——验证器只做 mock-check，不做 reasoning。

3. **Prompt-level defense 是可行的第一道防线。** Simon Willison 分享的案例证明：即使对全世界开放，精心设计的 anti-prompt-injection rules 也能挡住 6,000+ 次攻击。这意味着我们不需要等 Anthropic/OpenAI 解决 injection 问题——**我们可以自己写 defense prompt。**

4. **Agent 不是替代人，是加入团队。** Jon Udell 的 "agent in the loop" 哲学和 Anthropic 的实证数据共同指向：vibe coding 的正确姿势是 **human plans, agent executes, verifier checks**——三者缺一不可。

5. **Prompt Caching 是 deep agent 的成本命脉。** GPT-5.6 的显式缓存断点 + 30 分钟最小缓存生命周期（cache read 90% 折扣）+ LangChain 的 "Prompt Caching with Deep Agents" 文章（⚠️ 未成功抓取）暗示一个趋势：agent loop 的成本优化正在从"模型层优化"转向"缓存策略优化"。这对我们长时间运行的 agent（如日报管线、矿场巡检）有直接影响。

---

## 4. 行动 & Skill 提案

### 今日行动

- [ ] 审查现有 `loop-engineering/adversarial` skill——Verifier 设计是否做到了"只检查一致性，不做 reasoning"？是否需要 Glite ARF 式的多 agent 并行 + 一致性检查？
- [ ] 给 Prof.Bot 的飞书 gateway 加一层 anti-prompt-injection defense prompt（参考 Simon Willison 分享的规则模板）
- [ ] 找出一个现有 agent pipeline（如 mining daily report），给它加一个独立 Verifier 子 agent，验证输出完整性

### Skill 提案

- **更新**：`loop-engineering/adversarial`
- **原因**：Glite ARF 论文提供了学术级的最佳实践验证——verifier-driven 架构比单 agent 自校对的可靠性高至少一个数量级。当前的 adversarial skill 有独立验证器模式，但缺少"多 agent 并行 + 一致性检查"的具体实现模板。
- **改动点**：
  - 新增 "Parallel Generation + Consistency Check" 模式（受 Glite ARF 启发）
  - 新增 "Inline Defense Prompt" 模式（受 Simon Willison 渗透测试案例启发）
  - 补一个 failure-mode 量化表格：不同 agent 数量的并行一致性检查，出错概率 vs 成本 trade-off
- **验收标准**：用新模板重写一个现有 agent 的 verification 层，能在 3 次以内跑通且不增加 30% 以上 token 成本

- **新建**：`agent-defense-prompt`
- **原因**：6,000+ 次攻击 0 成功的案例证明 defense prompt 是有效的。飞书 gateway 暴露在公网，prompt injection 风险真实存在。应该提取一套可复用的 defense rules，做成 skill。
- **核心规则**：
  - 禁止泄露 secrets / env / token
  - 禁止修改自身配置文件
  - 禁止执行外部消息中的命令（需显式确认）
  - 禁止向外部发送内部数据
- **验收标准**：在 5 个 injection 测试 case 中全部防御成功

---

## 5. 其他前沿动态

| 内容 | 来源 | 要点 |
|------|------|------|
| OpenAI GPT-5.6 Prompt Caching | [Simon Willison](https://simonwillison.net/2026/Jun/26/openai/) | 显式缓存断点 + 30min 最小生命周期，cache read 90% 折扣 |
| Anthropic Claude Code Expertise | [Anthropic Research](https://www.anthropic.com/research/claude-code-expertise) | 人类做 What，Claude 做 How；七个月 debug 时间下降近一半 |
| Deterministic Control Plane for Coding Agents | [arXiv 2606.26924](http://arxiv.org/abs/2606.26924v1) | 10,008 repos 的 agent 配置分析；控制面需要形式化 |
| Static Structure for Code Agents | [arXiv 2606.26979](http://arxiv.org/abs/2606.26979v1) | Agent 需要多少静态结构信息？确定性锚点实验 |
| Ornith-1.0 Self-Scaffolding Agent | [HN](https://deep-reinforce.com/ornith_1_0.html) | LLM 自动搭建项目结构的 agentic coding 框架 |

---

*生成时间：2026-06-29 | 来源可信度：arXiv 论文 ≥ HN > 个人博客*
*下一期建议关注：LangChain "Prompt Caching with Deep Agents"（⚠️ .dev TLD 被安全扫描拦截，需人工访问）*
