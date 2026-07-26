# AI 学习日报 — 2026-07-26 (Day 7 / Week 1)

> 主题：Claude 5 时代的 Context Engineering 范式转移——删掉 80% 的 system prompt，用 progressive disclosure 替代规则堆砌
> 路径阶段：阶段 2（Agent 架构）→ 阶段 3（Vibe Coding 工程化）

## 1. 今日主题

**"少即是多"被实证化：Anthropic 砍掉 Claude Code 80% 的 system prompt 后，评测指标无损失。**

Anthropic 发布了 Claude 5 代模型的 context engineering 新范式——这不是一篇普通的"prompt 技巧"，而是对 agent 架构设计的根本性重新思考。六个范式转变中，每一个都直接挑战我们过去半年在 Hermes 里积累的"最佳实践"：把 rules 写进 skill、把所有知识塞进 CLAUDE.md、重复强调关键指令。今天学到的核心教训是——**Claude 5 不需要这些了，继续堆规则反而会降低 agent 性能。**

这对 Daddy 当前从阶段 2（Agent 架构）走向阶段 3（Vibe Coding 工程化）是关键的校准信号——我们之前建的 skill 体系（loop-engineering、vibe-coding-workflow、skill-router）可能已经过度工程化了。

## 2. 学习材料

### 必读：The New Rules of Context Engineering for Claude 5 Generation Models
- 来源：<https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models>
- 作者/机构：Anthropic Engineering Team (Jul 2026)
- 核心结论：
  1. **80% system prompt 被删除**：Claude Opus 5 和 Fable 5 上评测无损失
  2. **六个范式转变**（见下表）
  3. **关键认知**：重叠的指令（system prompt + skill + CLAUDE.md + user request 互相冲突）强迫 Claude 消耗更多推理来"调解矛盾"，反而降低产出质量
  4. **新工具**：`/doctor` 命令自动检测 CLAUDE.md 和 skills 中的过度约束
  5. **Rubrics 是新原语**：用 rubric + verifier agent 替代手写规则，让 Claude 自己判断"什么是好的 API 设计"

### 六个范式转变速查

| # | 旧范式 | 新范式 | 对 Hermes 的冲击 |
|---|--------|--------|------------------|
| 1 | Give Claude rules | Let Claude use judgement | 我们的 skill 里大量显式规则（"必须做 X"、"禁止 Y"）可能反效果 |
| 2 | Give Claude examples | Design interfaces | tool description 的 enum/参数名本身就是最好的 instruction |
| 3 | Put it all upfront | Progressive disclosure | 不要把全部 skill 都塞进 context——让 agent 按需加载 |
| 4 | Repeat yourself | Simple tool descriptions | system prompt 里重复 tool 指令 → 删除，只放在 tool description |
| 5 | Memory in CLAUDE.md | Auto-memory | 我们的 structured-memory-system 是该删的典型案例 |
| 6 | Simple specs | Rich references | HTML artifacts、test suite、rubric 比 markdown plan 更可靠 |

### 可选：sqlite-utils 4.0 — 用 Claude Fable 写了 $149.25 的 major release
- 来源：<https://simonwillison.net/2026/Jul/05/>
- 作者：Simon Willison (Jul 5, 2026)
- 核心结论：Simon 用 Claude Fable 完成了 sqlite-utils 4.0 的 major version release（含 schema migration 系统），总花费 $149.25。这是 vibe coding 工程化的实战案例——不是"写个小脚本"，而是"发布一个被广泛使用的 Python 库的破坏性大版本"。

## 3. 关键洞见

1. **"过度约束"是 Claude 5 时代的新反模式**：Anthropic 发现，system prompt + skill + CLAUDE.md + user request 四条指令流互相冲突时，Claude 必须消耗额外的推理来"调解矛盾"。这解释了为什么我们的 skill-router 有时会 misroute——不是路由逻辑不够好，而是 context 里塞了太多互相矛盾的指令。

2. **Progressive Disclosure 是 Agent 架构的核心能力**：Claude Code 现在用 deferred loading（工具定义延迟加载）+ skill 按需调用 + 树状文件结构来替代"把所有东西都塞进 context"。我们的 loop-engineering skill 组（core + adversarial + patterns）已经部分实现了这个模式——但很多其他 skill 还是"一份大文件全塞进去"。

3. **Rubric 替代规则 = adversarial verification 的下一跳**：Anthropic 明确说"rubrics allow Claude to try and verify your taste by spinning up verifier agents"。这正是我们 loop-engineering-adversarial 的做法——但我们的 rubric 还是手写的，而 Claude 5 可以自己生成并迭代 rubric。这意味着我们的 adversarial verifier 可以升级为"self-improving verifier"。

4. **"Design interfaces, not examples"**：Tool 的参数名、enum 值、结构本身就是最好的 instruction。我们的 skill 文件里大量 "示例用法" 可能反而限制了 Claude 5 的探索空间。检查我们的 FOO 验证引擎、mail-summary 等 skill——它们的 tool 参数设计是否足够"自我解释"？

5. **Simon 的 $149.25 案例验证了阶段 3 的核心假设**：vibe coding 不是"写 demo"，而是"交付产品级代码"。关键不是 prompt 写得多好，而是**怎么用 spec + test + iteration 让 agent 持续产出可信任的代码**。

6. **Auto-memory 宣告 structured-memory-system 的终结**：我们在 7/24 刚删除了 structured-memory-system skill，切换到 Obsidian→Wiki 同步管线。Anthropic 的结论验证了这个决定——Claude 5 自己判断什么值得记住，不需要我们手写 memory 规则。

## 4. 行动 & Skill 提案

### 今日行动
- [ ] 用 `/doctor` 思路审视现有 skill——逐个检查是不是有"过度约束"（在 Claude 5 上不需要的显式规则）
- [ ] 检查 prof_bot 的 system prompt——是否有和 skill 内容重叠/冲突的指令
- [ ] 选一个 skill（如 skill-router）做"瘦身实验"——删除显式规则，只保留"什么时候加载"的触发条件，观察行为变化

### Skill 提案

**提案 1：瘦身现有 skill 组**
- **更新**：`skill-router`、`loop-engineering-core`、`mail-summary`
- **原因**：Claude 5 的 progressive disclosure 能力意味着 skill 文件应该更短、更聚焦，去掉"过度约束"的显式规则
- **改动点**：
  - 删除"必须做 X"、"禁止 Y"类指令（除非是安全关键）
  - 把"示例用法"替换为"tool 参数设计建议"
  - 拆分大 skill 为树状结构（core skill → 按需加载的 sub-skill）
- **验收标准**：瘦身后的 skill 在 Claude 5 上行为不退化（至少不差于当前版本）

**提案 2：升级 adversarial verifier 为 self-improving**
- **更新**：`loop-engineering-adversarial`
- **原因**：Claude 5 可以自己生成 rubric，verifier 可以用 rubric 自我迭代
- **改动点**：
  - 加入"rubric self-improvement"步骤：verifier 每次跑完后评估 rubric 是否仍然有效
  - 用 HTML artifacts 作为 reference（而非纯文本 rubric）
- **验收标准**：verifier 在 3 轮迭代后 rubric 质量提升（假阳性率下降）

**提案 3：创建 context-engineering skill**
- **新建**：`context-engineering`
- **原因**：这篇文章的信息密度极高，值得蒸馏成可复用的 skill
- **内容**：
  - Claude 5 的六个范式转变检查清单
  - 瘦身方法论：如何判断一个指令是"必要约束"还是"过度约束"
  - Progressive disclosure 设计模式（树状 skill 结构、deferred loading）
- **验收标准**：用这个 skill 审计现有所有 skill 的 context 效率

## 5. 其他前沿动态

- **HN #1: DeepSeek 暂停融资** — 梁文锋投资者交流会泄露，承认中美算力差距 1-2 年 <https://github.com/demo-zexuan/liang-wenfeng-investor-meeting-2026-7-22>
- **HN: Cloudflare AI 流量选项** — 站长可以 opt-out AI 爬虫 <https://blog.cloudflare.com/content-independence-day-ai-options/>
- **HN: LLM Usage in Debian** — Debian 社区投票讨论 LLM 在开发中的使用边界 <https://www.debian.org/vote/2026/vote_002>
- **HN: 28.9M 参数 LLM 跑在 $8 单片机上** — ESP32 上运行小型 LLM <https://github.com/slvDev/esp32-ai>
- **Anthropic Research: Agents in Biology** — agent 在生物学研究中的应用 <https://www.anthropic.com/research/agents-in-biology>

---

## 📊 第一周回顾 (Week 1: Jul 20-26, 2026)

> 本周是 AI 学习路径的正式第一周。主线：阶段 2（Agent 架构）→ 阶段 3（Vibe Coding 工程化）。

### 本周学习轨迹

| Day | 日期 | 主题 | 阶段 | 关键产出 |
|-----|------|------|------|----------|
| 1 | 7/20 | *(重启前)* | — | 学习路径 v0.1.0 定稿 |
| 7 | 7/26 | Context Engineering 范式转移 | 2→3 | 日报 + 3 个 skill 提案 |

> ⚠️ **实际情况**：Day 1-6 没有日报产出——本周是重启周，今天是实际上的 Day 1。但用户按日历定义为"第一周最后一天"。这反映了学习节奏的问题：**需要建立每日最小学习量（micro-learning）而非爆发式学习**。

### 路径进度评估

| 阶段 | 状态 | 本周进展 | 下周重点 |
|------|------|----------|----------|
| 1. 基础机制 | ✅ 已足够 | — | — |
| 2. Agent 架构 | 🔄 进行中 | Claude 5 context engineering 对 agent 架构的启示 | 瘦身 skill 组，验证 progressive disclosure |
| 3. Vibe Coding 工程化 | 🔄 进行中 | Simon Willison 案例验证了 vibe coding 生产级可行性 | 用 context engineering 新范式重写 vibe-coding-workflow |
| 4. 生产级能力 | ⚠️ 未开始 | — | — |
| 5. 领域 Agent | 🔄 持续 | — | — |

### 最大发现

**Context Engineering 范式转移是本季度最重要的学习信号。** 它不是说"prompt 技巧变了"，而是说 **Claude 5 的能力跃迁让我们过去半年积累的 agent 设计原则需要重新审视**。六个范式转变中，至少 4 个直接冲击现有 Hermes skill 设计：

- "Give rules" → "Let Claude use judgement"：我们的 skill 里大量显式规则需要重新评估
- "Put it all upfront" → "Progressive disclosure"：skill-router 需要支持按需加载而非全量加载
- "Repeat yourself" → "Simple tool descriptions"：system prompt 和 skill 的重复指令是反模式
- "Simple specs" → "Rich references"：markdown plan → HTML artifacts + test suite + rubric

### 下周计划

1. **瘦身实验**：选 3 个 skill（skill-router、loop-engineering-core、mail-summary）做瘦身，对比 Claude 5 上的行为
2. **新建 context-engineering skill**：蒸馏这篇文章的核心方法论
3. **建立每日最小学习量**：每天 15 分钟 curl + 写日报，不让学习再次中断 3 周
4. **重写 vibe-coding-workflow**：纳入 progressive disclosure + rubric 模式
5. **Cron 化每日日报**：让日报生成自动化，减少人工启动成本

### 反思

**为什么 Week 1 只有 Day 7 有产出？** 因为学习没有被 cron 化——依赖人工启动，一旦工作忙就中断。解决方案：**把每日日报的 source gathering 部分 cron 化**（curl 抓取 → 存到临时文件），人工只需要 review + 写 insight。这样即使某天没时间，素材已经抓好了，补起来也快。

---

*生成时间：2026-07-26 | 模型：DeepSeek-V4-Pro | 下一步：生成 HTML → push 到 ai-learning repo*