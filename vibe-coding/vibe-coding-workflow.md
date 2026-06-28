# Vibe Coding 工程化

> 目标：把 "用自然语言驱动编码" 从玄学变成可复现的工程流程，并沉淀为 Hermes skill。

---

## 一句话定义

**Vibe coding** = 用自然语言描述需求，让 AI agent 负责实现细节，人类只负责验收和方向判断。

但 "vibe" 不等于 "随便"。要工程化，需要一套决策流程、一套工具链、一套验收标准。

---

## 为什么现在需要工程化

| 现状 | 问题 |
|---|---|
| 想法很多，落地靠运气 | 没有固定流程，每次从零开始 |
| 工具很多，不知道用哪个 | Hermes / Claude Code / Codex 混用，选错效率低 |
| 原型能跑，但不可持续 | 不复用成功经验，每次都重新 prompt |
| 复杂任务容易跑偏 | 缺少计划、验证、收敛的步骤 |

工程化的目标：让 vibe coding 的输出**稳定、可复现、可沉淀**。

---

## 核心工作流

```
Vibe Check → Clarify → Decide Mode → Plan → Build → Test → Iterate → Package
```

| 步骤 | 目的 | 关键问题 |
|---|---|---|
| **Vibe Check** | 判断需求是否够清晰 | 要做什么？输入输出？成功标准？ |
| **Clarify** | 把模糊需求变成可执行描述 | 问 ≤5 个关键问题 |
| **Decide Mode** | 选对工具 | Hermes / Claude Code / Codex / Skill? |
| **Plan** | 定义最小可运行原型 | MVP 是什么？第一个 test case？ |
| **Build** | 自然语言驱动实现 | 先跑通，再优化 |
| **Test** | 验证改动 | 能跑吗？有破坏吗？边界处理了吗？ |
| **Iterate** | 收敛到可用 | 一次只改一个方向 |
| **Package** | 沉淀为 skill | 这个 workflow 以后会反复用吗？ |

---

## 工具选择决策树

```
是否是 recurring pattern?
├── 是 → 写 Hermes skill
└── 否
    ├── 是否已有代码库?
    │   ├── 是 → Claude Code
    │   └── 否
    │       ├── 复杂 / 多文件? → Codex
    │       └── 简单 / 一次性? → Hermes terminal
    └── 是否需要多个 agent 协作验证? → subagents
```

| 工具 | 最佳场景 | 劣势 |
|---|---|---|
| Hermes terminal | 一次性脚本、小工具、快速验证 | 复杂上下文容易丢 |
| Claude Code | 已有代码库的新功能 / PR | 从零创建项目慢 |
| Codex | 绿原项目、复杂原型、多文件 | 可能生成过度复杂的结构 |
| Hermes skill | 可复用 workflow、判断标准 | 前期需要设计和测试 |
| Subagents | 复杂验证、多步骤协作 | 成本高，需要编排 |

---

## 关键原则

1. **先跑通，再优雅**
   - 不要一开始就追求完美架构
   - 先有一个能工作的最小版本

2. **一次只改一个方向**
   - 不要同时改 UI、逻辑、错误处理
   - 每个改动都要能独立验证

3. **自然语言是手段，不是目的**
   - 用自然语言是为了快，不是为了逃避清晰思考
   - 复杂任务必须先有 plan

4. **可复用的都要沉淀为 skill**
   - 如果一个 workflow 出现第二次，就应该 skill 化
   - 技能是 vibe coding 的复利

---

## 与 loop-engineering 的关系

| 场景 | 组合使用 |
|---|---|
| 任务清晰，需要快速实现 | `vibe-coding-workflow` |
| 输出质量关键，需要独立验证 | `vibe-coding-workflow` + `loop-engineering-adversarial` |
| 任务范围不清，需要探索 | `vibe-coding-workflow` + `loop-engineering-patterns` |
| 多 agent 协作 | `vibe-coding-workflow` + subagents |

---

## 已沉淀的 skill

- `vibe-coding-workflow`：本工作流

---

## 下一步

1. 用本 workflow 重做/优化一个现有小工具，验证流程
2. 收集 3-5 个真实案例，提炼更多反模式和决策规则
3. 和 `loop-engineering`、`ai-daily-learning` 联动，形成完整的 agentic workflow 生态
