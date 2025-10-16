# 数据存储结构

> **上下文**：本文档定义PersonaLab的所有数据结构格式。
> 概要架构见 [DESIGN_OVERVIEW.md](../DESIGN_OVERVIEW.md) 第2章

**文档版本**: v2.0
**最后更新**: 2025-10-16

---

## 架构概览

PersonaLab 采用**对话文件中心 + 预设引用**架构：

- **对话文件**（Conversation）：核心数据，存储真实对话历史和状态
- **预设库**（Presets）：可复用的 Prompt 模块（角色、背景、摘要等）
- **创建对话时**：从预设库复制快照，对话拥有独立的演化状态

---

## 目录结构

```
data/
├── config.json              # 全局配置文件
│
├── presets/                 # 预设库（可复用模块）
│   ├── characters/
│   │   ├── alserqi.json
│   │   └── bob.json
│   ├── backgrounds/
│   │   ├── wasteland.json
│   │   └── cyberpunk.json
│   └── summaries/
│       └── conv_001_summary.json
│
├── conversations/           # 对话文件（核心）
│   ├── conv_001/
│   │   ├── conversation.jsonl      # 对话记录
│   │   ├── metadata.json           # 元数据（引用的预设）
│   │   ├── character_snapshot.json # 角色快照（会成长）
│   │   └── plot_state.json         # 剧情进度（可选）
│   └── conv_002/
│       ├── conversation.jsonl
│       ├── metadata.json
│       └── character_snapshot.json
│
├── event_library/           # 事件库（RAG向量数据库）
│   └── chroma_db/
│
└── prompt_templates/        # Prompt 模板（可配置）
```

---

## 对话文件格式

### conversation.jsonl（对话记录）

**首次对话示例**：
```jsonl
{"role":"user","content":"你好","turn":1,"timestamp":"2025-10-16T10:01:00Z"}
{"role":"assistant","content":"你好，有什么事吗？","turn":1,"timestamp":"2025-10-16T10:01:05Z"}
{"role":"user","content":"你还记得我之前说的话吗？","turn":2,"timestamp":"2025-10-16T10:02:00Z"}
{"role":"assistant","content":"...","turn":2,"timestamp":"2025-10-16T10:02:05Z"}
```

**汇总后的新对话示例**：
```jsonl
{"type":"summary","content":"情报交换与信任建立。用户告知 Alserqi 北区据点有背叛者的情报..."}
{"type":"summary","content":"制定复仇计划。Alserqi 决定前往调查，制定了详细的潜入计划..."}
{"role":"user","content":"你确定这个计划可行吗？","turn":1,"timestamp":"2025-10-16T11:00:30Z"}
{"role":"assistant","content":"当然，我已经考虑了所有可能的风险...","turn":1,"timestamp":"2025-10-16T11:00:35Z"}
```

**消息类型定义**：

| 类型 | 字段 | 说明 |
|------|------|------|
| `type:"summary"` | `content` | 历史剧情摘要（汇总后新对话才有） |
| `role:"user"` | `content`, `turn`, `timestamp` | 用户消息 |
| `role:"assistant"` | `content`, `turn`, `timestamp` | AI回复 |

**字段说明**：
- `turn`: 轮次编号（一轮 = 一条 user + 一条 assistant），从 1 开始
- `timestamp`: 消息时间戳（ISO 8601格式）
- `content`: 消息内容（纯文本）

**说明**：
- 汇总后新对话的 `turn` 从 1 重新开始编号
- 消息内容是纯净的对话，不包含任何元信息注释

---

### metadata.json（元数据）

```json
{
  "conversation_id": "conv_001",
  "created_at": "2025-10-16T10:00:00Z",
  "presets": {
    "character": "alserqi",
    "background": "wasteland",
    "summaries": []
  },
  "rag_scope": {
    "mode": "self_only",
    "related_conversations": []
  }
}
```

**字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `conversation_id` | string | 对话唯一标识符 |
| `created_at` | string | 创建时间（ISO 8601格式） |
| `presets` | object | 当前使用的预设引用 |
| `presets.character` | string | 角色预设ID |
| `presets.background` | string \| null | 背景预设ID（null表示不使用背景） |
| `presets.summaries` | array | 引用的摘要预设ID列表 |
| `rag_scope` | object | RAG检索范围配置（预留扩展） |
| `rag_scope.mode` | string | 检索模式（V1.0固定为"self_only"） |
| `rag_scope.related_conversations` | array | 关联的对话ID列表（预留扩展） |

**rag_scope 扩展说明**：
- **V1.0**：`mode = "self_only"`，只检索当前对话的历史事件
- **V2.0+**：可扩展为基于标签的语义检索（字段预留，逻辑待定）

---

### character_snapshot.json（角色快照）

```json
{
  "source_preset": "alserqi",
  "snapshot_created_at": "2025-10-16T10:00:00Z",
  "base_persona": "Alserqi，废土黑帮老大，曾经掌控北区，被心腹背叛后失去一切。核心目标是复仇，但内心深处渴望重建信任。性格坚毅、多疑、重情义。",
  "evolved_persona": ""
}
```

**字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `source_preset` | string | 来源角色预设ID |
| `snapshot_created_at` | string | 快照创建时间 |
| `base_persona` | string | 初始人格（从预设复制，不再变化） |
| `evolved_persona` | string | 成长人格（随对话演化，用户手动触发更新） |

**初始化规则**：
- 创建对话时，从 `presets/characters/{character_id}.json` 复制 `base_persona`
- `evolved_persona` 初始为空字符串
- 用户手动触发"更新记忆"后，`evolved_persona` 才有内容

**更新机制**：
- `base_persona` 永不改变
- `evolved_persona` 由用户手动触发更新（"🧠 更新记忆"按钮）
- 纯定性描述，不使用量化数值

---

### plot_state.json（剧情进度，可选）

```json
{
  "enabled": true,
  "current_plot_index": 3,
  "current_status": "in_progress",
  "no_update_count": 2
}
```

**字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `enabled` | boolean | 是否启用剧情大纲控制（由背景预设决定） |
| `current_plot_index` | number | 当前剧情点索引（1-based） |
| `current_status` | string | 当前剧情点状态：`completed`, `in_progress`, `pending` |
| `no_update_count` | number | 连续多少次AI回复没有输出`[PROGRESS]`标记（用于触发容错处理） |

**说明**：
- 只有使用了包含 `story_outline` 的背景预设时，才会生成此文件
- 如果对话中途切换背景，`plot_state.json` 可能失效（用户自行承担后果）

---

## 预设库格式

### 角色预设（presets/characters/*.json）

```json
{
  "character_id": "alserqi",
  "name": "Alserqi",
  "description": "废土黑帮老大，被背叛后寻求复仇",
  "avatar": "path/to/avatar.png",
  "base_persona": "Alserqi，废土黑帮老大，曾经掌控北区，被心腹背叛后失去一切。核心目标是复仇，但内心深处渴望重建信任。性格坚毅、多疑、重情义。",
  "created_at": "2025-10-10T10:00:00Z",
  "updated_at": "2025-10-15T15:30:00Z"
}
```

**字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `character_id` | string | 角色唯一标识符 |
| `name` | string | 角色名称 |
| `description` | string | 简短描述（用于UI展示） |
| `avatar` | string | 头像路径（可选） |
| `base_persona` | string | 初始人格定义（完整描述） |
| `created_at` | string | 创建时间 |
| `updated_at` | string | 最后修改时间 |

**说明**：
- 预设可以自由修改/迭代
- 修改预设不会影响已创建的对话（对话使用的是快照）

---

### 背景预设（presets/backgrounds/*.json）

```json
{
  "background_id": "wasteland",
  "name": "废土复仇记",
  "description": "核战后的废土世界，资源匮乏，强者为王",
  "world_setting": "核战后的废土世界，资源匮乏，强者为王。幸存者聚集在几个主要据点，帮派之间争夺控制权。你是一个被背叛的前帮派老大，现在要复仇。",
  "story_outline": [
    {"index": 1, "content": "发现背叛者的线索"},
    {"index": 2, "content": "潜入敌人据点"},
    {"index": 3, "content": "与仇人对峙"},
    {"index": 4, "content": "做出关键选择（杀/放/合作）"},
    {"index": 5, "content": "应对选择的后果"}
  ],
  "created_at": "2025-10-10T10:00:00Z",
  "updated_at": "2025-10-15T15:30:00Z"
}
```

**字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `background_id` | string | 背景唯一标识符 |
| `name` | string | 背景名称 |
| `description` | string | 简短描述（用于UI展示） |
| `world_setting` | string | 世界设定（完整描述） |
| `story_outline` | array | 故事主线大纲（可选，为空则不启用导演模块） |
| `created_at` | string | 创建时间 |
| `updated_at` | string | 最后修改时间 |

**story_outline 数组元素**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `index` | number | 大纲点索引（1-based，顺序编号） |
| `content` | string | 大纲点内容描述 |

**说明**：
- `story_outline` 为空数组或省略时，不启用导演模块
- 大纲点数量灵活，由用户自由编写

---

### 摘要预设（presets/summaries/*.json）

```json
{
  "summary_id": "conv_001_summary",
  "source_conversation": "conv_001",
  "created_at": "2025-10-16T12:00:00Z",
  "summaries": [
    {
      "index": 1,
      "content": "情报交换与信任建立。用户告知 Alserqi 北区据点有背叛者的情报..."
    },
    {
      "index": 2,
      "content": "制定复仇计划。Alserqi 决定前往调查，制定了详细的潜入计划..."
    }
  ]
}
```

**字段说明**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `summary_id` | string | 摘要唯一标识符 |
| `source_conversation` | string | 来源对话ID |
| `created_at` | string | 生成时间 |
| `summaries` | array | 情节点摘要列表 |

**说明**：
- 由汇总功能自动生成
- 可以被其他对话引用（作为历史经验）

---

## Chroma向量库格式

### Summaries Collection（情节点摘要）

**Document结构**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | `summary_{conversation_id}_{index}` |
| `content` | string | 情节点摘要文本 |
| `metadata` | object | 元数据（见下表） |

**Metadata字段**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `conversation_id` | string | 所属对话ID |
| `character_preset` | string | 角色预设ID |
| `background_preset` | string | 背景预设ID |
| `related_plot_id` | string | 关联的详细素材ID（预留扩展） |

**说明**：
- V1.0 RAG检索基于 `conversation_id` 过滤（只检索当前对话）
- V2.0+ 可基于 `character_preset` + `background_preset` 进行语义检索（预留扩展）

### Plots Collection（详细剧情素材，预留扩展）

**说明**：
- V1.0 暂不实现深度搜索，Plots Collection 预留但不使用
- 数据结构与 Summaries Collection 类似，但 `content` 字段存储详细剧情素材

---

## 配置文件格式（config.json）

```json
{
  "features": {
    "director_plot_control": {
      "enabled": true,
      "rag_fallback_threshold": 3
    }
  },
  "summary": {
    "order": "summary_first",
    "last_n_turns": 5
  },
  "conversation_file": {
    "load_all": true,
    "max_tokens": 100000
  }
}
```

### 字段说明

#### features.director_plot_control

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enabled` | boolean | `true` | 剧情大纲控制功能开关 |
| `rag_fallback_threshold` | number | `3` | 触发容错处理的阈值（连续多少次未更新） |

#### summary

| 字段 | 类型 | 可选值 | 说明 |
|------|------|--------|------|
| `order` | string | `summary_first`, `last_n_first` | 汇总后新对话的初始内容顺序 |
| `last_n_turns` | number | - | 汇总后新对话保留多少轮对话 |

#### conversation_file

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `load_all` | boolean | `true` | 是否全量读取对话文件 |
| `max_tokens` | number | `100000` | 对话文件的最大token数（预留） |

---

## 变更记录

### v2.0 (2025-10-16)
- 🔄 **架构重构**：从"会话实例"改为"对话文件 + 预设引用"
- ❌ **删除**：instance_state.json, character_state.json（改为 character_snapshot.json）
- 🆕 **新增**：metadata.json（记录预设引用和RAG范围）
- 🆕 **新增**：预设库结构（characters, backgrounds, summaries）
- 📝 **简化**：RAG检索基于 conversation_id（V1.0），预留语义标签扩展（V2.0+）

### v1.0 (2025-10-16)
- 初始版本（会话实例架构）

---

**文档版本**: v2.0
**最后更新**: 2025-10-16
