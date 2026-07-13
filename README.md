# AI Learning Notes

Daddy's personal AI learning repository — distilled knowledge, patterns, and skills from frontier AI research and practice.

## AI 学习路径

[ai-learning-path.md](ai-learning-path.md) 是整体学习路径，分 5 个阶段，明确每个阶段的目标、材料、输出形式和当前状态。每日学习内容应优先映射到当前主攻阶段。

- 当前主线：**阶段 2（Agent 架构）→ 阶段 3（Vibe Coding 工程化）**
- 最大缺口：4 个 CRITICAL gap — Context Engineering、Tool Design & MCP、Agent Evaluation、Harness Engineering（详见 [Agent Engineer Roadmap](agent-engineer-roadmap-2026-07-12.html)）

## Contents

### Loop Engineering (June 2026)
- **Source**: [Anthropic — A harness for every task: dynamic workflows in Claude Code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)
- **Learning file**: [loop-engineering/loop-engineering-learning.html](loop-engineering/loop-engineering-learning.html) (Claude-style formatted)
- **Distillation doc**: [loop-engineering/loop-engineering-distill-draft.md](loop-engineering/loop-engineering-distill-draft.md)
- **Key takeaway**: Never let a single agent verify its own output. Use adversarial verification — spawn an independent verifier subagent.

### Vibe Coding (June 2026)
- **Source**: Own practice + loop-engineering integration
- **Learning file**: [vibe-coding/vibe-coding-workflow.md](vibe-coding/vibe-coding-workflow.md)
- **Key takeaway**: Vibe coding needs a workflow — Vibe Check → Clarify → Decide Mode → Plan → Build → Test → Iterate → Package.

### Agent Engineer Roadmap (July 2026) ⭐ NEW
- **Learning file**: [agent-engineer-roadmap-2026-07-12.html](agent-engineer-roadmap-2026-07-12.html) (Claude-style, 72KB, self-contained)
- **Data sources**: Anthropic Engineering/Research · OpenAI · Google AI · arXiv (36 papers) · Simon Willison · Lilian Weng · swyx · GitHub API
- **Gap diagnosis**: 4 CRITICAL gaps identified — Context Engineering, Tool Design & MCP, Agent Evaluation, Harness Engineering
- **Flight study pack**: P0 (~5.5h) + P1 (~3.75h) = ~9h of curated reading
- **Quality**: Adversarial verification 4.5/5, all issues fixed
- **Key takeaway**: Agent ≠ smarter LLM, but LLM in a loop with tools. Context engineering > prompt engineering. "An optimizer that grades itself learns to game the metric."

### Agent Memory System (July 2026) ⭐ NEW
- **Deployment guide**: [agent-memory-system/agent-memory-system-deployment.md](agent-memory-system/agent-memory-system-deployment.md) + [HTML](agent-memory-system/agent-memory-system-deployment.html)
- **What**: Obsidian + LLM Wiki long-term memory system for AI agents — "compile once, continuously accumulate"
- **Architecture**: Three-layer storage (Agent runtime / Knowledge Base / Workspace) + three cron pipeline (22:00 memory index → 12:00 Wiki sync → 23:00 Meta-Verifier)
- **Key insight**: LLM Wiki Pattern (Karpathy) > RAG for agent memory — pre-compiled knowledge with cross-references, contradiction tracking, and progressive disclosure

## Structure

```
ai-learning/
├── README.md
├── ai-learning-path.md
├── agent-engineer-roadmap-2026-07-12.html   ⭐ Agent Engineer 知识体系+gap诊断+飞行学习包
├── agent-memory-system/                      ⭐ Obsidian + LLM Wiki 记忆系统部署指南
│   ├── agent-memory-system-deployment.md
│   └── agent-memory-system-deployment.html
├── loop-engineering/
│   ├── loop-engineering-learning.html
│   ├── loop-engineering-learning.md
│   └── loop-engineering-distill-draft.md
├── karpathy-minimalism/
│   ├── karpathy-minimalism-agent.html
│   └── karpathy-minimalism-agent.md
├── vibe-coding/
│   └── vibe-coding-workflow.md
└── (more topics coming...)
```

## Related

- [Hermes Agent](https://github.com/nousresearch/hermes-agent)
- [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)

---

*"Creating a workflow helps combat agent failure modes by orchestrating separate subagents with their own context windows and focused, isolated goals."* — Anthropic, June 2026

### Karpathy Minimalism (June 2026)
- **Source**: [microgpt](https://karpathy.github.io/2026/02/12/microgpt/) — 200 lines of pure Python = a complete GPT
- **Learning file**: [karpathy-minimalism/karpathy-minimalism-agent.md](karpathy-minimalism/karpathy-minimalism-agent.md)
- **Key takeaway**: Core algorithm is minimal — training loop ≈ agent iteration loop. Feedback is the agent's training data.

## Skills

Synced from Hermes prof_bot profile. These are the executable agent skills, not just documentation.

| Skill | File | Purpose |
|-------|------|---------|
| loop-engineering (core) | [skills/loop-engineering/loop-engineering-core.md](skills/loop-engineering/loop-engineering-core.md) | 三大失败模式 + 五步骤 + 四种成本 + 速查 |
| loop-engineering-adversarial | [skills/loop-engineering/loop-engineering-adversarial.md](skills/loop-engineering/loop-engineering-adversarial.md) | 对抗性验证完整模板 |
| loop-engineering-patterns | [skills/loop-engineering/loop-engineering-patterns.md](skills/loop-engineering/loop-engineering-patterns.md) | Loop Until Done + Tournament |
| rubric templates | [skills/loop-engineering/references/rubric-templates.md](skills/loop-engineering/references/rubric-templates.md) | 日报/良率/邮件实例化rubric |
