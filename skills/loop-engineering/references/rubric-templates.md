# Pipeline Rubric 模板库

> 以下是为不同 pipeline 定制的 Adversarial Verification rubric。每个模板都是 [loop-engineering-adversarial](skill:loop-engineering-adversarial) 的实例化。
> 使用时：复制对应模板 → 替换 {占位符} → 嵌入目标 skill 的 Step N。

## 日报 rubric（daily-card-update 使用）

```markdown
1. 数据源完整性：card_context.json 的 date 字段是否匹配预期日期
2. 计算准确性：抽样 3 条指标（现金流/在线率/FOO2），反算校验公式
3. 边界值：托管台数 delta 是否合理（单日波动>5K台需标注）
4. 逻辑一致性：banner_text 是否与指标变化方向一致（效率↑但banner=红色 → 异常）
5. 归因质量：foo2_note/cashflow_note 是否引用了具体数据值（≥2 个数据点）
```

## 良率 rubric（yield-report-base 使用）

```markdown
1. 公式正确性：6 站直通率公式是否与 Excel 源一致（TR1→FVI→TR2→OS→FQC→OQC）
2. 异常值检测：单站 yield <90% 或 >99.5% 的批次是否标注并查证
3. 趋势一致性：本周良率变化是否与制造端工况（机台/物料/人员变动）方向一致
4. 历史对比：本月 YTD 累计良率与上月同期偏差是否在 ±3pp 内
5. 数据粒度：是否按 S19/S21/S23 系列分列，避免混算
```

## 邮件总结 rubric（mail-summary 使用）

```markdown
1. 发件人覆盖率：是否漏掉关键发件人（按邮件量 top 10 逐一核对）
2. 摘要准确度：随机抽 3 封原文，核对摘要是否忠实于原文，无幻觉
3. 待办事项完整性：
   - 原文中明确标注 TODO/action/fyi/请回复 的条目是否全部提取
   - 优先级排序是否合理（urgent > important > fyi）
4. 时间范围：确认覆盖了目标时间段内的全部邮件，无时间窗口偏移
5. 去重：同一 thread 的多封邮件是否合并展示，避免重复计数
```

## 嵌入式 vs 独立 cron 验证决策表

| 维度 | 嵌入式验证（skill 内 Step N） | 独立 cron 验证 |
|------|------------------------------|----------------|
| 触发方式 | pipeline 运行时自动触发 | cron 定时触发 |
| 适用场景 | 前置质量关卡 | 周期性巡检 |
| 时效性 | 即时 | 延迟 |
| 失败处理 | 修正→重验→上报 | 告警→人介入 |

**决策逻辑**：前置条件→嵌入式；质量监控→cron；两者可共存。

## 推广状态追踪

| Pipeline | 嵌入状态 | version |
|----------|:--------:|:-------:|
| daily-card-update (日报) | ✅ 已嵌入 Step 4 | v0.1.0 |
| yield-report-base (良率) | ⬜ 待嵌入 | — |
| mail-summary (邮件) | ⬜ 待嵌入 | — |
| weekly-report-analyzer | ⬜ 待嵌入 | — |
