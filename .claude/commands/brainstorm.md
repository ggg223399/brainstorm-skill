---
description: "Dynamic multi-agent discussion with auto-generated expert roles, preset reuse, and evolution (V4)"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent, AskUserQuestion
---

# Brainstorm V4 — Dynamic Multi-Agent Discussion

话题自适应的多角色 AI 讨论系统。根据话题动态生成专家角色，支持 preset 复用。

**核心原则**：所有内容生成 delegate 给 sub-agent，主线程只做编排、用户沟通和审校。

## 完整工作流

Phase 0 (Preset查找) → Phase 1 (角色设计) → Phase 2 (上下文收集) → Phase 3 (执行讨论) → Phase 4 (迭代/进化)

---

## Phase 0 — Preset 查找

在 `Apps/brainstorm/presets/` 下查找匹配的 JSON 文件（`{topic-slug}.json`）。有就用，没有就进 Phase 1。

## Phase 1 — 角色设计

### 1.1 话题分析

用一个 Agent 分析话题，输出所需专业视角（3-5 个）、核心职责。要求角色间有张力，至少含一个质疑型和一个实践型角色。

### 1.2 生成配置

每个 agent 的字段：

```json
{
  "id": "unique-slug",
  "name": "角色名",
  "responsibility": "一句话职责描述",
  "thinking_style": "analytical | divergent | critical | practical | empathetic",
  "system_prompt": "完整的 system prompt",
  "subagent_type": "oracle | metis",
  "order": 1
}
```

**subagent_type 选择**（按输出类型）：

| 输出类型 | thinking_style | subagent_type |
|---------|---------------|---------------|
| 结构化方案（表格/计划） | `analytical` | `oracle` |
| 创意点子 | `divergent` | `oracle` |
| 问题/风险清单 | `critical` | `oracle` |
| 执行指南/步骤 | `practical` | `metis` |
| 用户故事/感受 | `empathetic` | `oracle` 或 `metis` |

> 批判型角色统一用 `oracle`，不用 `momus`。

### 1.3 执行结构设计

讨论的执行由两个正交参数控制：

- **`order`（阶段分组）**：相同 order 的 agent 在同一阶段并行执行，不同 order 的阶段顺序执行，后续阶段能看到前序阶段的全部输出
- **`rounds`（讨论轮数）**：1 轮 = 单次发言；2-3 轮 = 每轮所有 agent 看到上轮输出后重新回应

两个参数组合自然涌现多种行为：

| order 分布 | rounds | 效果 |
|-----------|--------|------|
| 全相同 | 1 | **并行**：独立视角收集 |
| 多组 | 1 | **分阶段**：出方案→审查→修订 |
| 全相同 | 2-3 | **辩论**：多轮交锋收敛共识 |
| 多组 | 2-3 | **分阶段辩论**：先分阶段，再整体多轮 |

**设计建议**：
- 各角色视角独立、无依赖 → 全同 order + 1 轮
- 有"出方案→审方案"链条 → 建设型 order=1，审查型 order=2
- 存在对立观点需要交锋 → rounds ≥ 2

### 1.4 用户确认

展示角色配置 + 执行结构（order 分组、rounds），询问是否调整。确认后保存到 `presets/{topic-slug}.json`。

Preset schema 见 `Apps/brainstorm/schema.json`。

---

## Phase 2 — 用户上下文收集

**在执行讨论前，收集用户的个人上下文**（除非用户明确说"不需要个性化"）。

用 AskUserQuestion 一次性收集。根据话题和角色需求动态构造问题，必问项：
- 当前情况 / 背景
- 核心目标 + 优先级
- 已知约束

收集到的信息追加到每个 agent 的 prompt 末尾。

---

## Phase 3 — 执行讨论

### 3.1 分阶段执行

按 `order` 分组，阶段间顺序执行：

1. 将 agents 按 order 值分组
2. 从最小 order 开始，同组 agent 全部并行启动（`run_in_background=true`）
3. 等待本阶段全部完成，收集输出
4. 将输出累积，作为下一阶段 agent 的上下文注入（"前序阶段的输出"）
5. 重复直到所有阶段完成

如果只有一组（全同 order），则全员并行，等同于传统并行模式。

### 3.2 多轮处理（rounds > 1 时）

Round 1 按 3.1 执行。Round 2+ 中每个 agent 看到上一轮所有输出，并行给出回应。

每轮注入的指令：
- **Round 2**："回应其他角色的观点：认同的部分、分歧点、需要修正的地方。如果其他角色的论点改变了你的判断，明确说明。"
- **Round 3**（如有）："最终轮。给出最终立场，重点回答：你的观点是否有变化？关键分歧是否已收敛？"

### 3.3 综合输出

讨论完成后，做冲突检测 → 综合报告。

**冲突检测**：提取各角色的关键数据点（数值、时间、顺序），对比不一致。消解优先级：
1. 批判型角色的修正 → 优先采纳
2. 安全/风险相关 → 取保守值
3. 多轮讨论已收敛的 → 直接采纳共识
4. 纯偏好差异 → 标注两种选项 + 推荐

> 分阶段模式下，后序阶段已回应了前序输出，冲突通常已部分消解，重点检查遗留分歧。

**报告分两档**：

**轻量输出**（快速话题、角色 ≤ 3）：直接输出要点 + 行动建议，不需要完整报告格式。

**完整报告**（深度话题、角色 ≥ 4 或用户要求）：delegate 给一个 Agent 生成，包含：
1. 参与角色总览（表格）
2. 详细讨论（分阶段/分轮次展示）
3. 共识点
4. 分歧与决策（表格：话题 / 各方立场 / 采纳决策）
5. 行动建议

---

## Phase 4 — 迭代与进化

- 用户对角色不满意 → 更新 preset → 重新讨论
- 讨论结束后询问评分（1-5）和角色反馈（哪些有用/哪些弱/缺什么视角）
- 进化机制详见 `EVOLUTION-REFERENCE.md`

---

## Preset 管理

| 用户说 | 系统做 |
|--------|--------|
| "帮我头脑风暴 XX" | Phase 0 检查 preset → 有就用，没有就 Phase 1 |
| "我觉得角色不太对" | 更新 preset JSON |
| "看看我有哪些 preset" | 列出 presets/ 目录 |

**向后兼容**：旧 preset 使用 `discussion_config: { parallel: true, rounds: 1 }` 格式，等价于全同 order + rounds=1。遇到旧格式自动映射。
