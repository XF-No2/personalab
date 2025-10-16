# 数据存储结构

> **上下文**：本文档定义PersonaLab的所有数据结构格式。
> 概要架构见 [DESIGN_OVERVIEW.md](../DESIGN_OVERVIEW.md) 第2章

**文档版本**: v1.0
**最后更新**: 2025-10-16

---

## 目录结构

```
data/
├── config.json              # 全局配置文件（功能开关、参数设置）
│
├── characters/              # 角色库（全局）
│   └── {character_id}/
│       └── definition.json
│
├── backgrounds/             # 背景库（全局）
│   └── {background_id}/
│       └── background.json
│
├── instances/               # 会话实例（核心）
│   └── {instance_id}/
│       ├── instance_state.json     # 会话实例状态（剧情进度、当前会话等）
│       ├── character_state.json    # 角色状态
│       └── sessions/               # 会话文件
│           ├── session_001.jsonl
│           ├── session_002.jsonl
│           └── session_003.jsonl
│
├── event_library/           # 事件库
│   └── chroma_db/
│
└── prompt_templates/        # Prompt 模板（可配置）
```

---

## 会话文件格式（.jsonl）

### 首次会话示例

```jsonl
{"type":"metadata","instance_id":"inst_001","session_id":"sess_001","created_at":"2025-10-16T10:00:00Z","continued_from":null}
{"role":"user","content":"你好","turn":1,"timestamp":"2025-10-16T10:01:00Z"}
{"role":"assistant","content":"你好，有什么事吗？","turn":1,"timestamp":"2025-10-16T10:01:05Z"}
{"role":"user","content":"你还记得我之前说的话吗？","turn":2,"timestamp":"2025-10-16T10:02:00Z"}
```

### 汇总后的新会话示例

```jsonl
{"type":"metadata","instance_id":"inst_001","session_id":"sess_002","created_at":"2025-10-16T11:00:00Z","continued_from":"sess_001"}
{"type":"summary","content":"情报交换与信任建立。用户告知 Alserqi 北区据点有背叛者的情报..."}
{"type":"summary","content":"制定复仇计划。Alserqi 决定前往调查，制定了详细的潜入计划..."}
{"role":"user","content":"你确定这个计划可行吗？","turn":1,"timestamp":"2025-10-16T11:00:30Z"}
{"role":"assistant","content":"当然，我已经考虑了所有可能的风险...","turn":1,"timestamp":"2025-10-16T11:00:35Z"}
{"role":"user","content":"那我们现在出发？","turn":2,"timestamp":"2025-10-16T11:01:00Z"}
```

### 消息类型定义

| 类型 | 字段 | 说明 |
|------|------|------|
| `type:"metadata"` | `instance_id`, `session_id`, `created_at`, `continued_from` | 会话元数据（每个会话文件第一行） |
| `type:"summary"` | `content` | 历史剧情摘要（汇总后新会话才有） |
| `role:"user"` | `content`, `turn`, `timestamp` | 用户消息 |
| `role:"assistant"` | `content`, `turn`, `timestamp` | AI回复 |

**字段说明**：
- `instance_id`: 所属会话实例 ID
- `session_id`: 当前会话 ID
- `created_at`: 创建时间（ISO 8601格式）
- `continued_from`: 如果是汇总后的新会话，记录旧会话 ID（否则为 null）
- `content`: 消息内容（纯文本）
- `turn`: 轮次编号（一轮 = 一条 user + 一条 assistant）
- `timestamp`: 消息时间戳

**说明**：
- 新会话的 `turn` 字段从 1 重新开始编号
- 消息内容是纯净的对话，不包含任何元信息注释

---

## 角色状态格式（character_state.json）

### 结构定义

```json
{
  "base_persona": "角色的初始人格，永不改变",
  "evolved_persona": "角色经历成长后的状态，手动更新"
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `base_persona` | string | 角色的初始人格，永不改变。包含：原型、核心目标、核心特质、背景故事 |
| `evolved_persona` | string | 角色经历成长后的状态。包含：信念变化、行为模式、关系深度、当前情绪等。**纯定性描述**，不使用量化数值 |

### 初始化规则

- 创建会话实例时，从全局角色定义复制 `base_persona`
- `evolved_persona` 初始为空字符串（角色尚未成长）
- 用户手动触发"更新记忆"后，`evolved_persona` 才有内容

### 示例

```json
{
  "base_persona": "Alserqi，废土黑帮老大，曾经掌控北区，被心腹背叛后失去一切。核心目标是复仇，但内心深处渴望重建信任。性格坚毅、多疑、重情义。",
  "evolved_persona": "经历背叛后变得极度多疑，不再轻易相信他人。但在与玩家的互动中，开始展现出脆弱的一面，愿意分享内心的痛苦。对复仇的执念逐渐转变为对正义的追求。"
}
```

---

## 会话实例状态格式（instance_state.json）

### 结构定义

```json
{
  "instance_id": "inst_001",
  "character_id": "char_alserqi",
  "background_id": "bg_wasteland",
  "current_session_id": "sess_003",
  "created_at": "2025-10-10T10:00:00Z",
  "plot_state": {
    "current_plot_index": 3,
    "current_status": "in_progress",
    "no_update_count": 2
  }
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `instance_id` | string | 会话实例ID（唯一标识符） |
| `character_id` | string | 使用的角色ID（创建实例时指定，不可变） |
| `background_id` | string \| null | 当前使用的背景ID（可切换，null表示不使用背景） |
| `current_session_id` | string | 当前活跃的会话ID |
| `created_at` | string | 会话实例创建时间（ISO 8601格式） |
| `plot_state` | object | 剧情大纲控制使用的状态（见下表） |

### plot_state 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `current_plot_index` | number | 当前剧情点索引（1-based） |
| `current_status` | string | 当前剧情点状态：`completed`, `in_progress`, `pending` |
| `no_update_count` | number | 连续多少次AI回复没有输出`[PROGRESS]`标记（用于触发容错处理） |

---

## 背景设置格式（background.json）

### 结构定义

```json
{
  "background_id": "bg_wasteland",
  "name": "废土复仇记",
  "world_setting": "核战后的废土世界，资源匮乏，强者为王。幸存者聚集在几个主要据点，帮派之间争夺控制权。你是一个被背叛的前帮派老大，现在要复仇。",
  "story_outline": [
    {"index": 1, "content": "发现背叛者的线索"},
    {"index": 2, "content": "潜入敌人据点"},
    {"index": 3, "content": "与仇人对峙"},
    {"index": 4, "content": "做出关键选择（杀/放/合作）"},
    {"index": 5, "content": "应对选择的后果"}
  ]
}
```

### 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `background_id` | string | 背景ID（唯一标识符） |
| `name` | string | 背景名称 |
| `world_setting` | string | 世界设定（一段完整的文字描述） |
| `story_outline` | array | 故事主线大纲（关键剧情点列表） |

### story_outline 数组元素

| 字段 | 类型 | 说明 |
|------|------|------|
| `index` | number | 大纲点索引（1-based，顺序编号） |
| `content` | string | 大纲点内容描述 |

**说明**：
- 大纲点数量灵活，由用户自由编写（可以是5个、10个或更多）
- 大纲是建议路线，不是强制路线

---

## Chroma向量库格式

### Summaries Collection（情节点摘要）

**用途**：存储情节点摘要，用于延续历史和浅层RAG检索

**Document结构**：
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | `summary_{session_id}_{index}` |
| `content` | string | 情节点摘要文本 |
| `metadata` | object | 元数据（见下表） |

**Metadata字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| `related_plot_id` | string | 关联的详细素材ID（`plot_{session_id}_{index}`） |
| `session_id` | string | 所属会话ID |
| `instance_id` | string | 所属会话实例ID |
| `character_id` | string | 角色ID |
| `background_id` | string | 背景ID |

### Plots Collection（详细剧情素材）

**用途**：存储详细剧情素材，用于深度RAG检索

**Document结构**：
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | `plot_{session_id}_{index}` |
| `content` | string | 详细剧情素材文本 |
| `metadata` | object | 元数据（见下表） |

**Metadata字段**：
| 字段 | 类型 | 说明 |
|------|------|------|
| `related_summary_id` | string | 关联的摘要ID（`summary_{session_id}_{index}`） |
| `session_id` | string | 所属会话ID |
| `instance_id` | string | 所属会话实例ID |
| `character_id` | string | 角色ID |
| `background_id` | string | 背景ID |

**关联关系**：
- 通过 `related_plot_id` 和 `related_summary_id` 建立双向关联
- 同一情节点的摘要和素材使用相同的 `{session_id}_{index}`

---

## 配置文件格式（config.json）

### 结构定义

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
| `order` | string | `summary_first`, `last_n_first` | 汇总后新会话的初始内容顺序 |
| `last_n_turns` | number | - | 汇总后新会话保留多少轮对话 |

#### conversation_file

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `load_all` | boolean | `true` | 是否全量读取会话文件 |
| `max_tokens` | number | `100000` | 会话文件的最大token数（预留） |

---

**文档版本**: v1.0
**最后更新**: 2025-10-16
