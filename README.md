# AI Learning Notes

Daddy's personal AI learning repository — distilled knowledge, patterns, and skills from frontier AI research and practice.

## Contents

### Loop Engineering (June 2026)
- **Source**: [Anthropic — A harness for every task: dynamic workflows in Claude Code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)
- **Learning file**: [loop-engineering/loop-engineering-learning.html](loop-engineering/loop-engineering-learning.html) (Claude-style formatted)
- **Distillation doc**: [loop-engineering/loop-engineering-distill-draft.md](loop-engineering/loop-engineering-distill-draft.md)
- **Key takeaway**: Never let a single agent verify its own output. Use adversarial verification — spawn an independent verifier subagent.

## Structure

```
ai-learning/
├── README.md
├── loop-engineering/
│   ├── loop-engineering-learning.html
│   ├── loop-engineering-learning.md
│   └── loop-engineering-distill-draft.md
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
