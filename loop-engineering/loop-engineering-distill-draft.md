# Loop Engineering 蒸馏文档 v0.1.0

> 核心素材：[Anthropic — A harness for every task: dynamic workflows in Claude Code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)（2026-06-02, Thariq Shihipar & Sid Bidasaria, Anthropic）
> 本蒸馏提取 loop engineering 的 why + how + Hermes 适配落地方案。

---

## Part A: Loop Engineering 核心理念

### 定义
Loop engineering = 用**多 agent 编排**替代单体上下文执行，每个 subagent 有独立 context window + 隔离目标，通过结构化的 flow 模式（分类/分拆/对抗/锦标赛/循环）让 agent 在反馈中收敛，对抗三大失败模式。

### 为什么需要 Loop Engineering：三大失败模式

单体 agent 在长任务中必然遇到的退化，这是 loop engineering 要对抗的敌人：

| # | 失败模式 | 英文 | 表现 | 如何发生 |
|---|---------|------|------|---------|
| 1 | **代理懒散** | Agentic laziness | agent 做完部分任务就声称完成（如处理 35/50 项安全审查就交差） | context 太长，agent 丧失"全部完成"的追踪 |
| 2 | **自我偏好偏差** | Self-preferential bias | agent 偏好自己的产出，自我审查时判为无误 | 产出 agent 和验证 agent 是同一个，没有独立视角 |
| 3 | **目标漂移** | Goal drift | 多轮后原始目标逐渐丢失（压缩损失了"别做 X""边界条件"等细节） | 长对话的 compaction/summarization 是有损的 |

> 对应关系：这三个失败模式正是传统单体 agent 的**结构性缺陷**。Loop engineering 通过"用独立 subagent 隔离上下文 + 对抗性验证 + 显式终止条件"来结构性对抗。

### 六个核心工作流模式

| # | 模式 | 英文 | 做法 | 适用场景 |
|---|------|------|------|---------|
| 1 | **分类后行动** | Classify-and-act | 先用一个 classifier agent 判断任务类型，再路由到不同 agent/行为 | 多类型任务入口，需分流 |
| 2 | **分拆后合成** | Fan-out-and-synthesize | 把任务拆成 N 个子任务，各自独立 agent 跑，最后合成 | 大量小步骤、各步骤需干净 context |
| 3 | **对抗性验证** | Adversarial verification | 对每个产出 agent，spawn 另一个独立的 verifier agent 按 rubric 审查 | 需要高质量验证、消除自欺 |
| 4 | **生成后过滤** | Generate-and-filter | 生成一批候选，用 rubric 或 verifier 过滤、去重，只留高质量项 | 头脑风暴、命名、设计方案 |
| 5 | **锦标赛** | Tournament | N 个 agent 竞争同一任务（不同方法），pairwise 评判决出胜者 | 有明确评判标准、需要最优解 |
| 6 | **循环到完成** | Loop until done | 循环 spawn agent，直到停止条件满足（无新发现 / 无错误 / 达标） | 工作量不确定的任务 |

### 什么时候不用

博客原文：*"Workflows are new. They are not needed for every task and may use significantly more tokens. Best for complex, high value tasks."*

- ❌ 常规 coding 任务不需要 5 个 reviewer panel
- ❌ 简单一步完成的任务
- ✅ 长运行、大规模并行、高度结构化、**对抗性任务**

---

## Part B: 博客中的具体用例（选摘）

| 用例 | 用到的模式 | 要点 |
|------|----------|------|
| **Deep Research** | Fan-out + Adversarial | 并行 web 搜索→抓取→对抗验证→合成引用报告 |
| **Deep Verification** | Adversarial | 一个 agent 识别所有事实主张→每个主张 spawn 一个验证 agent→再 spawn 一个 agent 检查验证 agent 的源质量 |
| **Migrations/Refactors** | Fan-out + Adversarial | 每个 fix spawn 一个 agent 在独立 worktree→另一个 agent 对抗审查→merge。Bun 从 Zig 到 Rust 的 migration 就这样做的 |
| **Root-cause Investigation** | Fan-out + Adversarial + Tournament | 不同 agent 从不同证据源（日志/文件/数据）独立产生假设→每个假设面对 verifier 和 refuter 的 panel |
| **Memory/Rule Adherence** | Adversarial + Fan-out | 每条规则 spawn 一个 verifier agent 专门检查，一个 skeptic agent 审查规则本身的合理性。反向：挖掘历史 session 中的纠正→聚类→对抗验证候选规则→蒸馏回 CLAUDE.md |
| **Triaging at Scale** | Classify-and-act + Loop | 分类每个 ticket→去重→行动。建议 quarantine 模式（读取不可信内容的 agent 禁止高权限操作）。配对 /loop 持续运行 |
| **Sorting** | Tournament / Pairwise-comparison | 1000+ 行在单个 prompt 里排序质量下降→用 pairwise comparison agent 做锦标赛排序 |
| **Evals** | Fan-out + Tournament | 并行 spawn agent 跑 evals→spawn comparison agent 按 rubric 打分 |
| **Model/Intelligence Routing** | Classify-and-act | 先 research 任务复杂度→路由到 Sonnet 或 Opus |

---

## Part C: Hermes 适配翻译

### 六大模式 → Hermes 机制映射

| 模式 | Hermes 映射 | 实现方式 | 状态 |
|------|-----------|---------|------|
| **Classify-and-act** | kanban 卡片分类 + profile 路由 | 入口卡片先过 classifier（prof_bot），根据类型分派到 mining-assets/dashboard/hosted-mining | ✅ 基础设施有，classifier 逻辑待显式化 |
| **Fan-out-and-synthesize** | `delegate_task` batch 模式 + 主 agent 合成 | 主 agent 拆任务→delegate_task(tasks=[...])→等返回→去重/交叉验证/合成 | ✅ **已有**（delegate_task 原生支持 batch） |
| **Adversarial verification** | `delegate_task` 双 agent：产出 + 验证 | 对关键产出，spawn 一个独立 verifier subagent（不同 prompt、不同角色）用 rubric 审查。**产出 agent 不审自己的活** | ❌ **缺失**，这是最大 gap |
| **Generate-and-filter** | `delegate_task` 多生成 + 主 agent 过滤 | 并行 spawn N 个生成 agent→主 agent 或独立 filter agent 按 rubric 过滤去重 | ⚠️ 部分有（手动组合） |
| **Tournament** | `delegate_task` N agent 竞争 + pairwise 比较 | N agent 独立解决→pairwise 比较 agent 决出胜者 | ⚠️ 部分有（手动组合） |
| **Loop until done** | kanban 循环卡片 + 显式终止条件 | 卡片设为循环类型，每次跑完检查 stopping condition（无新发现/错误清零/达标）→不满足则回 todo→满足则 done | ⚠️ 终止条件需显式写入 skill |

### 关键 gap 优先级

| 优先级 | gap | 为什么 |
|--------|-----|--------|
| **P0** | Adversarial verification | 博客显示这是质量天花板的关键——破除自我偏好偏差。Hermes 目前无此显式机制 |
| **P1** | Loop until done 的显式终止条件 | 有循环基础设施（kanban/cron）但缺少"何时停止"的标准写入 skill |
| **P2** | Generate-and-filter / Tournament 标准化 | 可以手动组合 delegate_task 实现，但缺少可直接调用的 skill 模板 |

---

## Part D: Skill 落地设计

### skill: `loop-engineering` v0.0.0

**触发条件**：agent 执行复杂/长运行/可拆分任务时（判断标准：3+ 独立环节、对抗性验证需求、未知工作量）

**内容结构**：
1. **背景知识**：三大失败模式 + 六大模式速查表
2. **P0 模式：Adversarial Verification**
   - 触发：产出有质量风险（报表/数据/代码改动的关键交付）
   - 做法：spawn verifier subagent，用独立 prompt + rubric 审查，不告诉 verifier 产出 agent 是谁做的
   - 终止：verifier pass → 交付；verifier fail → 标记修正点 → 产出 agent 修正 → 重新验证
3. **P1 模式：Loop Until Done**
   - 触发：工作量不确定的迭代任务
   - 做法：kanban 卡片类型=loop + stopping_condition 字段（如 `new_findings < 1`、`error_count = 0`、`quality_score >= 0.9`）
   - 终止：满足条件 or 达到 max_iterations → 仍不满足 → ⚠️ 上报人
4. **P2 模式：Generate-and-Filter / Tournament**
   - 模板化 delegate_task batch + 合成分步
5. **Hermes 适配速查表**（Part C 的映射表）
6. **适用边界 + 失败条件**（token 消耗 / 何时不该用）

---

## Part E: 差距分析与下一步

### Daddy 现状 vs Anthropic 前沿（基于此篇博客）

| 维度 | Anthropic 前沿做法 | Daddy 现状 | 差距 |
|------|-------------------|-----------|------|
| 多 agent 编排 | Dynamic workflows JS 脚本 | kanban + delegate_task + 4 profile | ✅ 基础设施对标 |
| 对抗性验证 | 每个产出 spawn 独立 verifier | 无显式机制，靠 agent 自觉或人抽查 | ❌ P0 gap |
| 循环终止条件 | /loop + /goal + stopping condition | kanban 循环卡片，但终止条件未标准化 | ⚠️ P1 gap |
| 模式标准化 | 6 个模式内化在日常使用中 | 模式有（fan-out 已常用），但未写成可复用 skill | ⚠️ P2 gap |
| 模型路由 | classifier 判断复杂度→路由到不同模型 | 4 profile 各有不同模型（已配置），但无自动复杂度判断路由 | ⚠️ 次要 |
| Context 隔离 | subagent 独立 worktree + context | delegate_task 每个子任务独立 session ✅ | ✅ 已对标 |
| 中断恢复 | 恢复 session 后 workflow 继续 | agent session 可恢复 ✅ | ✅ 已对标 |

### 下一步动作序列

1. **立即**：skill_manage 创建 `loop-engineering` v0.0.0 skill（含 Adversarial Verification + Loop Until Done）
2. **立即**：llm-wiki ingest 本蒸馏文档的核心页
3. **验证**：在下次日报/看板/良率报表任务中显式启用 Adversarial Verification 模式
4. **迭代**：使用反馈→skill 升版

---

## 附录：博客原文关键引用

> "The longer Claude works on a complex task in a single context window, the more it becomes susceptible to a few specific failure modes: Agentic laziness, Self-preferential bias, Goal drift."

> "Creating a workflow helps combat these by orchestrating separate Claude subagents with their own context windows and focused, isolated goals."

> "For each spawned agent, run a separate spawned agent to adversarially verify its output against a rubric or criteria."

> "Workflows are not needed for every task and may end up using significantly more tokens. Best for complex, high value tasks."

> "For tasks with an unknown amount of work, loop spawning agents until a stop condition is met instead of a fixed number of passes."
