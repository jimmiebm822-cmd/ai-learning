---
name: vibe-coding-workflow
description: Transform vague natural-language requirements into runnable code and Hermes skills. A repeatable workflow for vibe coding with Hermes Agent, Claude Code, and Codex.
version: 0.1.0
---

# vibe-coding-workflow v0.1.0

> Vibe coding：用自然语言驱动编码，AI agent 负责实现细节。
> 本 skill 把它从“随便试试”变成可复现的工程流程。

## 触发条件
- 用户说"帮我写个..."、"vibe 一个..."、"快速实现..."
- 用户有一个模糊想法，需要快速变成可运行代码
- 完成一个原型后，判断是否需要封装成 Hermes skill
- 用户使用 Claude Code / Codex 后需要有人 review 或接管

## 工作流（7 步）

### Step 0：Vibe Check — 判断需求清晰度

在动手前，先用 1-2 句话确认：
- 要解决什么问题？
- 输入是什么？输出是什么？
- 成功标准是什么？

如果以上任何一条答不上来 → 进入 **Step 1: Clarify**。否则跳到 **Step 2: Decide execution mode**。

### Step 1：Clarify — 问 ≤5 个关键问题

目标：把模糊需求变成可执行的描述。

问题类型：
1. **范围**：这个功能要做到什么程度？最小可用是什么？
2. **输入/输出**：数据从哪来？结果要什么样？
3. **约束**：技术栈？时间？不能动哪里？
4. **验收**：怎么算完成？有没有测试用例？
5. **复用性**：这是一次性脚本，还是以后会反复用？

> ⚠️ 不要多于 5 个问题。问太多说明需求还太模糊，需要拆任务。

### Step 2：Decide execution mode — 选工具

根据任务性质选择 execution mode：

| 场景 | 工具 | 原因 |
|---|---|---|
| 一次性脚本 / 数据转换 / 小工具 | Hermes terminal 直接写 | 快，不用 context 切换 |
| 已有代码库的新功能 / PR | Claude Code | 理解代码上下文，安全修改 |
| 绿原项目 / 复杂原型 / 多文件 | Codex | 从 0 到 1 最快 |
| 可复用的 workflow / 判断标准 | Hermes skill | 沉淀为可复用能力 |
| 需要多 agent 协作 / 复杂验证 | Hermes + subagents | loop-engineering 模式 |

**决策逻辑**：
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

### Step 3：Plan — 最小可运行原型

不要直接写完整功能。先定义：
- 最小可运行版本（MVP）是什么？
- 第 1 个可以跑通的 test case 是什么？
- 需要哪些文件/模块？

把 plan 用中文或英文简短写在对话开头。复杂任务用 `plan` skill 先写 markdown plan。

### Step 4：Build — 用自然语言驱动

开始 vibe coding：
1. 先写需求描述（不是代码）
2. 让 agent 生成第 1 版
3. 运行 / 测试
4. 根据结果继续用自然语言迭代

**话术模板**：
- "先写一个最简单的版本，只要跑通 [具体 case]"
- "现在的问题是 [错误/期望行为]，请修复"
- "把这个逻辑抽成函数"
- "加一段错误处理，当 [条件] 时 [行为]"

### Step 5：Test — 验证不是可选的

每完成一个改动，必须能回答：
- 这个改动能跑通吗？
- 有没有破坏之前的功能？
- 边界条件处理了吗？

**最低限度**：
- 运行一次主流程
- 看 error / stdout
- 对复杂逻辑加 1-2 个断言

### Step 6：Iterate — 收敛到可用

迭代原则：
- 先跑通，再优雅
- 先解决错误，再优化体验
- 每次只改一个方向（不要同时改 UI、逻辑、错误处理）

如果遇到 agent 陷入循环或偏离目标：
- 回到 plan，确认当前改动是否在主线上
- 缩小范围："先不要管 [X]，只改 [Y]"
- 必要时 spawn 独立 verifier（loop-engineering adversarial）

### Step 7：Package — 判断是否需要 skill

如果满足以下任一条件，封装成 Hermes skill：
- 这个 workflow 以后会反复用
- 其中包含判断标准 / rubric
- 失败模式有教训，需要固化
- 其他人（或其他 session）也需要用

**封装步骤**：
1. 在 `skills/` 下新建目录
2. 写 `SKILL.md`：触发条件、工作流、示例、反模式
3. 加 `references/` 和 `scripts/`（如有必要）
4. 用 `skill_manage create` 或手写到对应目录
5. git commit + push

---

## 工具链

- **Hermes Agent**：orchestrator + reviewer，不直接做重活
- **Claude Code**：已有代码库的新功能 / PR
- **Codex**：绿原项目 / 复杂原型
- **Hermes Studio Web UI**：需要可视化或多轮交互时用
- **loop-engineering skills**：多 agent 协作/验证时用

---

## 常见反模式

| 反模式 | 后果 | 正确做法 |
|---|---|---|
| 需求不清就写代码 | 返工、跑偏 | 先做 vibe check + clarify |
| 一次性 prompt 解决复杂问题 | 输出不能用 | 拆成多步，每步验证 |
| 不测试 | 表面能跑，实际有 bug | 每步至少运行一次 |
| 同时改多个方向 | agent 混乱 | 一次只改一个方向 |
| 不复用成功经验 | 每次都从零开始 | 沉淀为 skill |
| 该写 skill 时继续硬写 | 重复劳动 | 识别 recurring pattern |

---

## 验收清单

- [ ] 需求已经过 vibe check / clarify
- [ ] 选了合适的 execution mode
- [ ] 有最小可运行原型 plan
- [ ] 已经过至少一次运行/测试
- [ ] 已知边界条件和失败模式
- [ ] 判断过是否需要封装成 skill
- [ ] 如果需要，已经提交 git commit

---

## 与 loop-engineering 的关系

| 场景 | 使用 |
|---|---|
| 任务清晰，需要快速实现 | vibe-coding-workflow |
| 任务复杂/关键，需要独立验证 | + loop-engineering-adversarial |
| 任务范围不清，需要多轮探索 | + loop-engineering-patterns |
| 需要把成功经验固化为 workflow | 封装成 skill |

---

## 示例

### 示例 1：一次性数据转换脚本

**User**: "帮我把这些 excel 数据转成 json"

**Workflow**:
1. Clarify：哪几列？输出格式？日期怎么处理？
2. Mode：Hermes terminal 直接写 Python
3. Plan：读取 excel → 选择列 → 输出 json
4. Build：用 pandas / openpyxl 写 10 行代码
5. Test：运行并抽查输出
6. Iterate：处理空值 / 日期格式
7. Package：一次性任务，不 skill

### 示例 2：给现有项目加新功能

**User**: "给日报卡片加一个导出 PDF 的按钮"

**Workflow**:
1. Clarify：按钮位置？导出哪些内容？PDF 样式？
2. Mode：Claude Code（理解现有代码）
3. Plan：改前端 HTML → 加按钮 → 调用生成 PDF 的脚本
4. Build：Claude Code 生成改动
5. Test：在浏览器验证
6. Iterate：调整样式 / 边界情况
7. Package：如果以后常用，封装为 `export-pdf` skill

### 示例 3：沉淀为 skill

**User**："以后每次生成学习文件都要走 outline → md → html → push 四步"

**Workflow**:
1. Clarify：哪些文件需要？固定模板是什么？
2. Mode：Hermes skill
3. Plan：写 `learning-file-generator` / `ai-learning-publish` skill
4. Build：编写 SKILL.md + references
5. Test：用一个真实学习主题走一遍
6. Iterate：根据使用反馈修改
7. Package：git commit + push

---

## 版本历史

- v0.1.0 (2026-06-28): 初始版本
