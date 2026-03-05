# Brainstorm Evolution Reference

完整的 Phase 4 进化机制文档。主 SKILL.md 只保留触发条件摘要，详细逻辑在此。

---

## Phase 4 — 进化（Evolution）

参考 OMO 的 Wisdom Accumulation 机制，Brainstorm 支持基于使用历史的 Preset 自动进化。

### 4.1 Session 反馈收集

每次讨论结束后，主动询问用户反馈：

```
讨论已完成。为了帮助我优化这套角色配置，请简单反馈：

1. 哪些角色的观点最有价值？
2. 哪些角色的输出不太有用或重复？
3. 是否缺少某个视角？
4. 整体评分（1-5）？

（回复"跳过"可不保存反馈）
```

反馈保存到 `sessions/{preset-id}/{date}-feedback.json`：

```json
{
  "session_id": "{date}-001",
  "preset_id": "{preset-slug}",
  "timestamp": "...",
  "feedback": {
    "useful_agents": ["agent-id-1", "agent-id-2"],
    "weak_agents": ["agent-id-3"],
    "weak_reasons": {
      "agent-id-3": "具体原因（如：内容与另一角色重复、输出不够具体）"
    },
    "missing_perspectives": ["缺失的视角名称"],
    "adopted_insights": ["实际采纳的具体建议"],
    "overall_quality": 4
  }
}
```

### 4.2 Evolution Triggers

当满足以下条件时，触发 Preset 进化建议：

| 触发条件 | 检查逻辑 |
|----------|---------|
| **使用次数** | 同一 Preset 使用 ≥ 3 次 |
| **低分角色** | 某角色在 2+ 次反馈中被标记为 `weak_agents` |
| **缺失视角** | 同一视角在 2+ 次反馈中被提及 `missing_perspectives` |
| **低整体评分** | 平均 `overall_quality` < 3.0 |

### 4.3 Evolution Actions

基于反馈数据，系统可建议以下进化操作：

| 反馈模式 | 进化操作 |
|----------|---------|
| 角色输出重复 | 合并角色职责，或删除其中一个 |
| 角色输出太发散 | 优化 system_prompt，增加输出约束 |
| 角色输出太保守 | 优化 system_prompt，增加开放性引导 |
| 缺失视角 | 生成新角色并添加到 agents 数组 |
| 角色多次无用 | 标记为 `deprecated`，下次讨论时跳过 |
| Prompt 不够精准 | 基于 `adopted_insights` 优化 system_prompt |

**进化流程**：
```
收集到 3+ 次反馈
      │
      ▼
分析反馈模式
      │
      ▼
生成进化建议（展示给用户）
      │
      ├─ 用户确认 → 更新 Preset JSON（更新 updated_at + evolution_history）
      │
      └─ 用户拒绝 → 保留原配置
```

### 4.4 Evolution History

Preset 中新增 `evolution_history` 字段，记录每次进化：

```json
{
  "evolution_history": [
    {
      "date": "2026-XX-XX",
      "trigger": "weak_agent_threshold",
      "changes": [
        {
          "type": "remove_agent",
          "agent_id": "被移除的角色 id",
          "reason": "2次反馈标记为重复/无用"
        },
        {
          "type": "add_agent",
          "agent_id": "新增的角色 id",
          "reason": "2次反馈要求增加该视角"
        }
      ],
      "before_score": 3.2,
      "after_score": null
    }
  ]
}
```

---

## Cross-Preset Wisdom

不同 Preset 之间可以共享学到的模式。保存在 `presets/_wisdom.json`。

三个核心区块：

| 区块 | 用途 | Phase 1 使用 |
|------|------|-------------|
| `workflow_patterns` | 工作流层面的最佳实践 | 执行流程时参考 |
| `effective_patterns` | 各类角色的推荐配置 | 生成新角色时参考 |
| `anti_patterns` | 已知的错误配置 | 生成/校验时自动检测 |

详细格式见实际 `_wisdom.json` 文件。

**Wisdom 使用时机**：
- Phase 1 生成新角色时，参考 `effective_patterns` 选择配置
- 检测到与 `anti_patterns` 匹配时，自动修正
