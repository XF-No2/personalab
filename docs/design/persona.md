# 角色人格机制详细设计

> **需求追溯**：解决 [REQUIREMENTS.md](../REQUIREMENTS.md) "需求2：人格的丰富性与成长"
>
> **架构上下文**：见 [DESIGN_OVERVIEW.md](../DESIGN_OVERVIEW.md) 第6章

**文档版本**: v1.0
**最后更新**: 2025-10-16

---

## 模块职责

管理角色人格状态，支持角色人格的初始化和成长演化。

---

## 核心设计：两层人格

### 设计理念

将角色人格分为两层：

1. **base_persona（初始人格）**：角色的底色，永不改变
2. **evolved_persona（成长人格）**：随对话经历动态演化，由用户手动触发更新

**优势**：
- 保留角色的核心特质，避免人格崩坏
- 支持角色成长，增强沉浸感
- 纯定性描述，符合人类理解习惯

---

## 数据结构

### character_snapshot.json 结构

格式定义：见 [data_structure.md](data_structure.md#character_snapshotjson角色快照)

```json
{
  "source_preset": "alserqi",
  "snapshot_created_at": "2025-10-16T10:00:00Z",
  "base_persona": "角色的初始人格，永不改变",
  "evolved_persona": "角色经历成长后的状态，手动更新"
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `source_preset` | string | 来源角色预设ID |
| `snapshot_created_at` | string | 快照创建时间 |
| `base_persona` | string | 角色的初始人格，永不改变。包含：原型、核心目标、核心特质、背景故事 |
| `evolved_persona` | string | 角色经历成长后的状态。包含：信念变化、行为模式、关系深度、当前情绪等。**纯定性描述**，不使用量化数值 |

### 初始化规则

1. 创建对话时，从角色预设（`presets/characters/{character_id}.json`）复制 `base_persona`
2. `evolved_persona` 初始为空字符串（角色尚未成长）
3. 用户手动触发"更新记忆"后，`evolved_persona` 才有内容

### 示例

```json
{
  "source_preset": "alserqi",
  "snapshot_created_at": "2025-10-16T10:00:00Z",
  "base_persona": "Alserqi，废土黑帮老大，曾经掌控北区，被心腹背叛后失去一切。核心目标是复仇，但内心深处渴望重建信任。性格坚毅、多疑、重情义。",
  "evolved_persona": "经历背叛后变得极度多疑，不再轻易相信他人。但在与玩家的互动中，开始展现出脆弱的一面，愿意分享内心的痛苦。对复仇的执念逐渐转变为对正义的追求。"
}
```

---

## Prompt组装策略

### 在Prompt中的位置

**头部0-4K区域**（高权重）：
- 系统指令
- **角色人格（base_persona + evolved_persona）**
- 剧情状态（如果导演模块开启）
- 故事大纲（JSON格式）

详见：[DESIGN_OVERVIEW.md](../DESIGN_OVERVIEW.md) 第3.5章"Prompt组装策略"

### Prompt格式

```
---CHARACTER_PERSONA---
## Base Identity (Immutable Core) ##
{base_persona}

## Evolved State (Growth Through Experience) ##
{evolved_persona}

## Recent Context ##
{当前会话的全部对话}
---
```

**设计说明**：
- Base Identity 标记为"Immutable Core"（不可变核心），提示AI这是角色的底色
- Evolved State 标记为"Growth Through Experience"（经验成长），提示AI这是可变的状态
- Recent Context 提供当前会话的完整对话历史，帮助AI理解当前剧情

---

## 更新记忆流程

### 触发方式

用户点击"🧠 更新记忆"按钮（手动触发）

### 流程

```
用户点击"🧠 更新记忆"
  ↓
[1] 加载当前对话的角色快照
  - 读取 character_snapshot.json
  ↓
[2] 全量读取当前对话历史
  - 读取 conversation.jsonl
  ↓
[3] 调用 LLM 更新 evolved_persona
  - 输入: base_persona + old_evolved_persona + 对话历史
  - 输出: new_evolved_persona
  ↓
[4] 保存新的 evolved_persona 到 character_snapshot.json
  ↓
[5] 返回前端
```

### 与汇总功能的区别

| 功能 | 更新记忆 | 汇总功能 |
|------|---------|---------|
| 目的 | 更新角色人格状态 | 总结对话内容，延续剧情 |
| 修改的数据 | `character_snapshot.json` | Chroma（写入摘要），生成摘要预设，创建新对话 |
| 触发方式 | 用户手动触发 | 用户手动触发 |
| LLM调用 | 1次（更新evolved_persona） | 1次（生成摘要） |

**关键区别**：
- 更新记忆：只修改角色人格状态，不生成摘要，不写入Chroma
- 汇总功能：总结对话内容，生成摘要写入Chroma，创建新对话

---

## LLM Prompt设计

### 更新记忆的Prompt

```
你是一个角色人格分析专家。请根据以下信息，更新角色的成长人格状态。

## 角色初始人格（不可变）
{base_persona}

## 当前成长人格（可更新）
{old_evolved_persona}

## 对话历史
{conversation_history}

## 任务
请分析对话历史中角色的经历和变化，更新角色的成长人格状态。
要求：
1. 保留角色的核心特质（由初始人格定义）
2. 描述角色的信念变化、行为模式、关系深度、当前情绪等
3. 使用自然语言描述，不使用量化数值（如priority、intensity、level等）
4. 突出角色的成长轨迹

输出格式：纯文本，直接输出新的成长人格描述
```

---

## 与其他模块的交互

### 依赖：对话文件架构

- 每个对话有独立的 `character_snapshot.json`
- 同一角色预设创建的不同对话，人格状态完全隔离

### 依赖：Prompt组装模块

- 角色人格放入Prompt头部4K（高权重区域）
- 确保AI始终记得角色人格

### 依赖：角色预设

- 创建对话时，从角色预设复制 `base_persona`
- 角色预设格式：`data/presets/characters/{character_id}.json`

---

## 扩展性

未来可扩展的功能：

1. **自动分析**：每次对话后，后台分析角色变化趋势，提示用户是否需要更新记忆
2. **版本历史**：记录 evolved_persona 的历史版本，支持回退
3. **多维度人格**：将 evolved_persona 拆分为多个维度（情感、信念、行为模式等）
4. **人格可视化**：前端展示角色人格的变化曲线

---

**文档版本**: v2.0
**最后更新**: 2025-10-16

---

## 变更记录

### v2.0 (2025-10-16)
- 🔄 **架构适配**：从"会话实例"改为"对话文件 + 预设引用"
- 📝 **术语更新**：character_state.json → character_snapshot.json
- 📝 **流程更新**：适配新的数据结构和文件路径

### v1.0 (2025-10-16)
- 初始版本
