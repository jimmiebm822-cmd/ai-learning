# Agent Memory System: Obsidian + LLM Wiki 部署指南

> **目标**：为 AI Agent 构建一个"编译一次、持续增值"的长期记忆系统。本文档剥离了所有业务信息，只保留可复现的架构和部署步骤。
>
> **适用读者**：正在使用 Hermes Agent / Claude Code / 类似 AI Agent 框架，希望给 Agent 装上"跨会话记忆"的开发者。
>
> **版本**：v1.0.0 · 2026-07-13

---

## 1. 为什么需要这个系统

LLM Agent 的核心限制：**context window 是有限的**。每次会话开始，Agent 没有上一次的任何记忆。传统解决方案是 RAG — 但 RAG 在每次查询时从原始文档重新发现知识，不累积结构、不标记矛盾、不建交叉引用。

**LLM Wiki Pattern**（来自 Karpathy）反过来：**编译一次**，编译出的 wiki 持续维护、累积结构、累积交叉引用、累积已标记的矛盾。

```
raw 源 → [LLM 编译] → interlinked .md wiki → [LLM Q&A / lint] → 持续增值
```

| 维度 | RAG | LLM Wiki |
|------|-----|----------|
| 知识组织 | 查询时重组 | 预先编译，持续累积 |
| 交叉引用 | 查询时发现 | 已内建 `[[wikilinks]]` |
| 矛盾处理 | 可能重复呈现矛盾 | 已标记，供 review |
| 综合质量 | 每次重新综合 | 反映已 ingest 的全部内容 |

---

## 2. 系统架构总览

### 三层存储架构

```
┌─────────────────────────────────────────────────┐
│  Layer 1: Agent Runtime (~/.hermes/)            │
│  ├── skills/          ← 可执行知识（Agent 调用）  │
│  ├── cron/             ← 定时自动化              │
│  ├── profiles/         ← 多角色隔离              │
│  └── workspace/        ← 临时产出               │
├─────────────────────────────────────────────────┤
│  Layer 2: Knowledge Base (Desktop/)              │
│  ├── LLM Wiki/         ← 编译后的知识库          │
│  │   ├── SCHEMA.md     ← 结构约定               │
│  │   ├── index.md      ← 内容目录               │
│  │   ├── log.md        ← 操作日志               │
│  │   ├── concepts/      ← 概念页                │
│  │   ├── entities/      ← 实体页                │
│  │   ├── comparisons/   ← 对比页                │
│  │   ├── queries/       ← 查询结果页            │
│  │   └── raw/           ← 不可变原始素材         │
│  └── Obsidian Vault/    ← 每日记忆索引           │
│      └── 每日日志/      ← YYYY-MM-DD.md         │
├─────────────────────────────────────────────────┤
│  Layer 3: Workspace (~/workspace/)              │
│  └── 临时草稿、实验性产出（不持久化）              │
└─────────────────────────────────────────────────┘
```

**核心原则**：
- L1 是工具箱 — Agent 运行时、skills、cron 配置
- L2 是权威数据源 — 知识库本体，Desktop 上方便人直接查看
- L3 是草稿本 — 临时产出，不作为知识源

### 记忆管线（3 个 Cron Job）

```
  22:00                    12:00                    23:00
  ┌──────────┐            ┌──────────────┐        ┌──────────────┐
  │ 记忆索引  │ ────────→  │ Wiki 模块化  │ ────→  │ Meta-Verifier │
  │ Scan 对话 │  每日日志  │ 同步         │ Wiki   │ 跨管线元验证  │
  │ 提取记忆  │  传递      │ 拆解注入     │ 更新   │ 失败模式聚合  │
  └──────────┘            └──────────────┘        └──────────────┘
     Obsidian                  LLM Wiki              信任分更新
    每日日志.md              concepts/*.md          reports/*.json
```

---

## 3. LLM Wiki 部署

### 3.1 目录结构

```
LLM Wiki/
├── SCHEMA.md           ← 结构约定 + tag taxonomy（Agent 每次会话先读）
├── index.md            ← 内容目录（每页附一行摘要）
├── log.md              ← 操作日志（每个写操作追加一行）
├── _meta/              ← 元数据（lint 报告等）
├── concepts/           ← 概念页（方法论、模式、框架）
├── entities/           ← 实体页（工具、平台、模型）
├── comparisons/        ← 对比页（A vs B 选型）
├── queries/            ← 查询结果页（复杂问题的编译答案）
└── raw/                ← 不可变原始素材
    ├── articles/       ← 文章原文
    ├── papers/         ← 论文原文
    ├── transcripts/    ← 对话/播客 transcript
    └── assets/         ← 图片等
```

### 3.2 SCHEMA.md — 结构约定

SCHEMA.md 是整个系统的"宪法"。Agent 在每次会话开始必须先读这个文件。它定义：

**文件命名规则**：
- 英文、小写、连字符，无空格（如 `agent-orchestration.md`）
- 中文概念用英文 slug，正文可中文

**页面长度限制**：
- 每页上限 **60 行**（确保可独立加载、精准路由）
- 超过 60 行 → 按子主题"有机拆解"为子页面（不是机械切割）
- 子页面在父页面的 frontmatter 中用 `sub_pages` 列出

**YAML Frontmatter 格式**：

```yaml
---
title: Page Title
created: YYYY-MM-DD
updated: YYYY-MM-DD
type: entity | concept | comparison | query | summary
tags: [from taxonomy]
sources: [raw/articles/source-name.md]
# 子页面集合的父页面使用：
sub_pages: [sub-slug-1, sub-slug-2]
# 可选质量信号：
confidence: high | medium | low
contested: true
contradictions: [other-page-slug]
---

# 页面标题

> 一句话摘要（做什么用的）

## 正文内容

每页至少 2 个 [[wikilinks]] 出站链接。

^[raw/articles/source-file.md]  ← provenance 标记
```

**raw/ 子目录的 Frontmatter**：

```yaml
---
source_url: https://example.com/article
ingested: YYYY-MM-DD
sha256: <hex digest of the raw content below the frontmatter>
---
```

`sha256` 让未来重新 ingest 同一 URL 时检测内容漂移（相同→跳过，不同→标记）。

### 3.3 Tag Taxonomy（必须先定义再用）

SCHEMA.md 中定义的 tag 表是**封闭的** — 页面上的每个 tag 必须出现在 taxonomy 表中。新增 tag 先加到 SCHEMA.md 再用。

建议的分类维度（根据你的领域调整）：

```
Models:       model, llm, inference, context-engineering
Platforms:    hermes, obsidian, wsl, claude-code, codex
Practices:    vibe-coding, agent-orchestration, debugging, automation, cron
Domain:       (你的业务领域 tag)
```

### 3.4 核心工作流

**Ingest（知识摄入）**：
1. 原始素材放入 `raw/articles/` 或 `raw/papers/`
2. Agent 读 SCHEMA.md → 读 index.md → 读 log.md（最近 30 行）
3. Agent 提炼知识到 wiki 页面，建 `[[wikilinks]]` 交叉引用
4. 更新 `index.md` 添加新页面条目
5. 追加 `log.md` 记录操作

**Q&A（知识查询）**：
1. Agent 读 SCHEMA.md → 读 index.md 定位相关页面
2. 读具体页面回答问题
3. 有价值的答案"filing"回 wiki（更新已有页面或创建新页面）

**Lint（健康检查）**：
- 找 orphan 页（无入站链接）
- 找断链（`[[wikilink]]` 指向不存在的页面）
- 找 frontmatter 缺失的页面
- 找 stale 内容（`updated` 日期过旧）
- 找矛盾标记（`contested: true`）

---

## 4. Obsidian 每日日志部署

### 4.1 日志格式

每日日志文件命名：`YYYY-MM-DD.md`，存放在 Obsidian Vault 的 `每日日志/` 目录。

```yaml
---
created: 2026-07-12
tags: [daily-log, memory-index]
type: daily
---

# 2026-07-12 记忆索引

> 本日志仅包含具有长期价值的记忆。事务性内容已过滤。

## 技术决策

- **决策标题**：决策内容描述。涉及 [[related-wiki-page]]。

## 方法论突破

- **方法论标题**：方法论描述。

## 知识积累

- **知识标题**：知识描述。详见 [[related-wiki-page]]。

---
> 📚 待 wiki-sync 拆解：
> - [[page-name]] → 拆解方向说明
```

### 4.2 内容过滤规则

**❌ 直接丢弃的内容**：
- Cron 运行元数据（跑了多少次、有无新邮件、运行时间）
- 日常事务性操作（启动了某个服务、打开了某个 session）
- 邮件摘要内容（属于邮件系统，不属于长期记忆）
- 纯执行性操作没有长期价值的

**✅ 需要捕捉的内容**：

| 类别 | 判断标准 | 示例 |
|------|---------|------|
| **技术决策** | 涉及架构选择、方向判断、trade-off 分析 | "决定用模块化分解而非单体架构" |
| **方法论突破** | 新发现的方法、模式、改进 | "三路并行搜索+综合产出工作流" |
| **系统异常** | 涉及系统性问题（非偶发） | "跨 profile 协作发现隔离问题" |
| **知识积累** | 新学到的、可复用的知识 | "Context Engineering 是 prompt eng 的范式迁移" |

### 4.3 设计原则

1. **不是流水账** — 目标是产出简洁但有深度的记忆索引，供明天的 wiki-sync 拆解和路由
2. **每条必须可路由** — 每条记忆要能对应到一个 wiki 页面（已有或待创建）
3. **底部待拆解区** — 日志末尾标注"待 wiki-sync 拆解"的清单，指导下一步同步

---

## 5. 记忆管线部署（3 个 Cron Job）

### 5.1 Cron 1: 记忆索引（每日 22:00）

**目标**：扫描今天的对话，提取具有长期价值的内容，写入 Obsidian 每日日志。

**Cron 配置**：
```json
{
  "id": "memory-index-cron",
  "name": "Obsidian 记忆索引",
  "schedule": { "kind": "cron", "expr": "0 22 * * *" },
  "skills": ["obsidian"],
  "enabled_toolsets": ["terminal", "file", "session_search"]
}
```

**Agent Prompt 核心**：
```
你是长期记忆 curator。扫描今天的对话，提取具有长期价值的内容，写入 Obsidian 每日日志。
这不是流水账。目标是产出简洁但有深度的记忆索引，供明天的 wiki-sync 管线逐条拆解和路由。

## 内容过滤规则

### ❌ 直接丢弃：
- cron 运行元数据
- 纯事务性操作
- 邮件摘要内容

### ✅ 需要捕捉：
| 类别 | 判断标准 |
|------|---------|
| 技术决策 | 涉及架构选择、方向判断、trade-off |
| 方法论突破 | 新发现的方法、模式、改进 |
| 系统异常 | 涉及系统性问题（非偶发） |
| 知识积累 | 新学到的、可复用的知识 |

## 输出格式
写入 /path/to/Obsidian Vault/每日日志/{今天日期}.md
按 4 个分区组织：技术决策 / 方法论突破 / 系统异常 / 知识积累
末尾加"待 wiki-sync 拆解"清单。
```

### 5.2 Cron 2: Wiki 模块化同步（每日 12:00）

**目标**：读昨天的每日日志，逐条拆解，增量注入 Wiki 已有页面（不创建"日记"页）。

**Cron 配置**：
```json
{
  "id": "wiki-sync-cron",
  "name": "Obsidian→Wiki 模块化同步",
  "schedule": { "kind": "cron", "expr": "0 12 * * *" },
  "skills": ["obsidian-wiki-sync", "obsidian", "llm-wiki"]
}
```

**Agent Prompt 核心**：
```
执行 Obsidian 记忆索引 → LLM Wiki 模块化同步。

## 核心原则
**不创建"日记"页。不整篇搬运。** 把昨天的记忆索引逐条拆解，增量注入 Wiki 已有页面。

## Step 1: 定位源文件
读 /path/to/Obsidian Vault/每日日志/{昨天日期}.md

## Step 2: 读取 Wiki 现状
读以下文件作为路由上下文：
- /path/to/LLM Wiki/index.md — 已有页面目录
- /path/to/LLM Wiki/SCHEMA.md — 约定和规则
- /path/to/LLM Wiki/log.md（最近 30 行）— 近期活动

## Step 3: 模块化拆解
逐条读取记忆索引中的每个项目。

对每条内容执行判断：
| 内容特征 | 动作 |
|---------|------|
| 涉及已有 concept 页的主题 | 搜索 index.md 匹配页面 → 增量更新 |
| 涉及已有 entity 页的主题 | 同上 → 增量更新 |
| 全新概念且符合创建阈值 | 按 SCHEMA.md 规范创建新页面 |
| 纯事务性/过时信息 | 跳过 |

## Step 4: 更新索引和日志
- 更新 index.md 添加新页面条目
- 追加 log.md 记录所有操作
- 每个更新过的页面更新 updated 日期
```

### 5.3 Cron 3: Meta-Verifier（每日 23:00）

**目标**：跨管线的元验证 — 聚合失败模式、趋势分析、信任分更新。

**Cron 配置**：
```json
{
  "id": "meta-verifier-cron",
  "name": "Meta-Verifier (每日 23:00)",
  "schedule": { "kind": "cron", "expr": "0 23 * * *" },
  "skills": ["meta-verifier"],
  "enabled_toolsets": ["terminal", "file", "skills"]
}
```

**Agent Prompt 核心**：
```
你是 Meta-Verifier — 跨管线的元验证 agent。

## Step 1: 收集验证报告
从 reports/ 目录读取所有 verification_report_*.json。筛选最近 7 天。

## Step 2: 失败模式聚合
按四个维度聚合：failure_category / check / data_source_ref / pipeline

## Step 3: 趋势分析
对比 7 日均值 vs 前 7 日均值，判断趋势：improving / stable / worsening

## Step 4: 跨管线关联
检查不同管线的 issues 是否共享 data_source_ref

## Step 5: 新失败模式发现
对比已知 failure_patterns.json，发现新出现的 check+failure_category 组合

## Step 6: 信任分更新
读 trust_scores.json，按规则更新每个 verifier_id 的信任分

## Step 7: 生成 insight
输出 top 3 anomalies（按 severity 排序）
```

---

## 6. 关键设计原则

### 6.1 编译一次，持续增值
传统 RAG 在每次查询时从原始文档重新发现知识。LLM Wiki Pattern 反过来：编译一次，持续维护。优势在于交叉引用和矛盾标记是累积的。

### 6.2 模块化分解（不整篇搬运）
每日日志的每条记忆，按主题拆解到对应的 wiki 页面。**不创建"日记"页，不整篇搬运**。这确保每个 wiki 页面聚焦一个主题，可独立加载。

### 6.3 增量注入（更新已有页面）
新知识优先注入已有页面（增量更新 `updated` 日期），只在"全新概念且符合创建阈值"时才创建新页面。这防止 wiki 膨胀。

### 6.4 Progressive Disclosure（渐进式加载）
- Agent 会话开始：只加载 SCHEMA.md + index.md（几十行）
- 定位到相关页面后：加载具体页面（≤60 行）
- 需要原始素材时：从 raw/ 按需加载
- 这与 Anthropic 的 Context Engineering 原则完全一致

### 6.5 Provenance Tracking
每个 wiki 页面在 frontmatter 中标注 `sources`，页面末尾用 `^[raw/articles/source.md]` 标注来源。raw/ 中的文件计算 `sha256`，让未来重新 ingest 时检测内容漂移。

### 6.6 闭环验证
22:00 记忆索引 → 12:00 Wiki 同步 → 23:00 Meta-Verifier。三个 cron 形成闭环：产出 → 消化 → 验证。Meta-Verifier 确保管线质量持续提升。

---

## 7. 部署步骤

### Step 1: 创建 LLM Wiki 目录结构
```bash
mkdir -p ~/Desktop/"LLM Wiki"/{concepts,entities,comparisons,queries,raw/{articles,papers,transcripts,assets},_meta}
```

### Step 2: 编写 SCHEMA.md
参考第 3.2 节的模板。关键：定义你的 tag taxonomy、页面长度限制（建议 60 行）、frontmatter 格式。

### Step 3: 初始化 index.md 和 log.md
```bash
echo "# Wiki Index\n\n> 内容目录。Last updated: $(date +%Y-%m-%d)\n" > ~/Desktop/"LLM Wiki"/index.md
echo "# Wiki Log\n\n> 操作日志。\n" > ~/Desktop/"LLM Wiki"/log.md
```

### Step 4: 创建 Obsidian 每日日志目录
```bash
mkdir -p ~/Desktop/"Obsidian Vault"/每日日志/
```

### Step 5: 用 Obsidian 打开 LLM Wiki 作为独立 Vault
- 打开 Obsidian → Open folder as vault → 选择 `LLM Wiki/`
- 这让你能在 Obsidian 中可视化 wiki 的 wikilinks 和 graph view

### Step 6: 部署 3 个 Cron Job
参考第 5 节的配置。关键是：
- **22:00 记忆索引**：需要 `session_search` 工具来扫描当天对话
- **12:00 Wiki 同步**：需要 `obsidian-wiki-sync` skill（或等效的同步逻辑）
- **23:00 Meta-Verifier**：可选但推荐 — 确保管线质量

### Step 7: 验证管线
1. 手动触发 22:00 cron → 检查每日日志是否生成
2. 手动触发 12:00 cron → 检查 Wiki 页面是否更新
3. 检查 index.md 和 log.md 是否正确更新

---

## 8. Pitfalls（踩坑记录）

1. **不要创建"日记"页** — 每日日志是 Obsidian 侧的临时索引，不是 Wiki 页面。Wiki 同步时必须拆解到已有主题页面。
2. **60 行限制是硬性的** — 超过就拆子页面。这确保 progressive disclosure 有效。
3. **Tag taxonomy 是封闭的** — 新 tag 先加到 SCHEMA.md 再用。否则 tag 会失控。
4. **raw/ 是只读的** — Agent 只读不写 raw/。原始素材不可变。
5. **每日日志不是流水账** — 严格过滤事务性内容。只保留有长期价值的记忆。
6. **Meta-Verifier 不是可选的** — 没有它，管线会悄悄退化（失败模式累积而不被发现）。
7. **路径用 Desktop 而非 ~/** — 知识库放在 Desktop 确保人也能直接查看，不只是 Agent 的黑盒。
8. **Obsidian 作为 Wiki 的 IDE** — 用 Obsidian 打开 LLM Wiki 目录，利用 graph view 可视化交叉引用。

---

## 9. 与 Anthropic Context Engineering 的对应

这个系统的设计原则与 Anthropic 2025-09 提出的 Context Engineering 高度一致：

| Anthropic 原则 | 本系统实现 |
|---------------|-----------|
| Just-in-time context | Agent 只加载 SCHEMA + index，按需加载具体页面 |
| Context as finite resource | 60 行限制 + 模块化拆解 = 精准 token 控制 |
| Progressive disclosure | 三级加载：metadata → SKILL.md → bundled files |
| Mirrors human cognition | 人用文件系统/收件箱/书签；Agent 用 Wiki/Obsidian/cron |

---

*"Creating a workflow helps combat agent failure modes by orchestrating separate subagents with their own context windows and focused, isolated goals." — Anthropic*

*"The essence of search is compression." — Anthropic Multi-Agent Research System*

*"An optimizer that grades itself learns to game the metric." — Google Agent Quality Flywheel*
