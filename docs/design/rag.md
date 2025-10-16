# RAG检索策略详细设计

> **需求追溯**：解决 [REQUIREMENTS.md](../REQUIREMENTS.md) "需求3：长会话与跨会话的延续性"
>
> **架构上下文**：见 [DESIGN_OVERVIEW.md](../DESIGN_OVERVIEW.md) 第5章

**文档版本**: v1.0
**最后更新**: 2025-10-16

---

## 模块职责

通过语义检索历史事件，解决长会话记忆问题，支持跨会话的剧情延续。

---

## 核心子功能1：日常对话RAG检索（Conversational RAG）

### 触发时机

**每次用户输入时**触发（并发执行）

### 目的

当用户提到历史事件时，AI能够回忆起来。

### 检索流程

```
用户输入："你还记得我们在据点的那次冲突吗？"
  ↓
[1] 使用LLM重新表述查询
  - 输入：当前用户输入 + 最近5-10条对话历史
  - 输出：独立的查询语句（例如："据点冲突事件，涉及主角与某人的对抗"）
  ↓
[2] 语义检索（Chroma）
  - 检索当前conversation_id的历史事件
  - 返回20条相关事件（Summaries为主）
  ↓
[3] 注入到Prompt的32K+区域
  - 作为"历史事件回忆"展示给AI
```

### 关键特性

**语义检索**：
- 相比Silly Tavern的关键词匹配，不需要精确命中关键词
- 基于向量相似度，能理解同义表达

**实例隔离**：
- 只检索当前`conversation_id`的事件
- 严格隔离不同剧情线，避免混淆

**检索范围**：
- 只检索已汇总的会话事件（存储在Chroma中）
- 当前活跃会话直接全量读取，不走RAG

**首次对话**：
- Chroma为空时，返回空列表（正常情况）

### 与导演模块RAG的区别

| 特性 | 日常对话RAG | 导演模块RAG |
|------|------------|------------|
| 触发时机 | 每次用户输入 | AI连续3轮未推进剧情 |
| 检索范围 | 只检索当前实例 | 可跨实例检索（当前15条+其他5条） |
| 返回数量 | 20条 | 20条（分层：当前15+其他5） |
| Prompt位置 | 32K+区域（历史事件回忆） | 32K+区域（导演提醒+剧情经验） |

---

## 核心子功能2：深度搜索策略（分层摘要）

### 浅层搜索（默认）

```
用户输入："你还记得之前的复仇吗？"
  ↓
RAG 搜索 Summaries Collection
  ↓
返回相关摘要：
  - "复仇行动：小张对小玲进行了复仇，导致小玲重伤"
  ↓
组装 Prompt（只包含摘要）
```

**适用场景**：
- 日常对话，只需知道"发生过什么"
- 优点：简洁，节省 tokens

### 深度搜索（按需）

```
用户输入："你还记得复仇时是怎么设下陷阱的吗？"
  ↓
RAG 搜索 Summaries Collection
  ↓
找到相关摘要：summary_001
  ↓
通过 related_plot_id 查询 Plots Collection
  ↓
返回：
  - Summary: summary_001（摘要）
  - Plot: plot_001（详细素材："小张假装和解，邀请小玲到废弃工厂..."）
  ↓
组装 Prompt（包含摘要 + 详细素材）
```

**适用场景**：
- 需要具体细节、情节设计参考
- 导演模块需要参考类似剧情

**触发条件**：
1. 用户询问"怎么"、"如何"、"详细过程"
2. 导演模块需要参考类似剧情
3. 自动判断（相似度 > 阈值则展开 Plots）

### 分层存储机制

**Summaries Collection**（情节点摘要）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | `summary_{session_id}_{index}` |
| `content` | string | 情节点摘要文本 |
| `metadata.related_plot_id` | string | 关联的详细素材ID（`plot_{session_id}_{index}`） |
| `metadata.session_id` | string | 所属会话ID |
| `metadata.conversation_id` | string | 所属对话ID |
| `metadata.character_id` | string | 角色ID |
| `metadata.background_id` | string | 背景ID |

**Plots Collection**（详细剧情素材）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | string | `plot_{session_id}_{index}` |
| `content` | string | 详细剧情素材文本 |
| `metadata.related_summary_id` | string | 关联的摘要ID（`summary_{session_id}_{index}`） |
| `metadata.session_id` | string | 所属会话ID |
| `metadata.conversation_id` | string | 所属对话ID |
| `metadata.character_id` | string | 角色ID |
| `metadata.background_id` | string | 背景ID |

**关联关系**：
- 通过 `related_plot_id` 和 `related_summary_id` 建立双向关联
- 同一情节点的摘要和素材使用相同的 `{session_id}_{index}`

---

## 数据结构依赖

### Chroma向量库格式

格式定义：见 [data_structure.md](data_structure.md#chroma向量库格式)

**两个Collection**：
- Summaries Collection：存储情节点摘要，用于浅层检索
- Plots Collection：存储详细剧情素材，用于深度检索

### 查询参数

**日常对话RAG**：
```python
chroma_collection.query(
    query_embeddings=embed(reformulated_query),
    where={
        "conversation_id": current_instance,
        "character_id": current_character,
        "background_id": current_background
    },
    n_results=20
)
```

**深度搜索**：
```python
# 第一步：查询Summaries
summaries = summaries_collection.query(...)

# 第二步：通过related_plot_id查询Plots
for summary in summaries:
    plot_id = summary["metadata"]["related_plot_id"]
    plot = plots_collection.get(ids=[plot_id])
```

---

## 与其他模块的交互

### 依赖：汇总功能

- RAG检索的数据源来自汇总功能写入的Chroma数据
- 汇总功能流程：见 [DESIGN_OVERVIEW.md](../DESIGN_OVERVIEW.md) 第3.2章

### 依赖：导演模块

- 导演模块的容错处理使用RAG检索（见 [director.md](director.md) 第3节"容错处理"）
- 检索策略不同：导演模块可跨实例检索（当前15条+其他5条）

### 依赖：Prompt组装模块

- RAG检索结果注入到Prompt的32K+区域（见 [DESIGN_OVERVIEW.md](../DESIGN_OVERVIEW.md) 第3.5章）
- 作为"参考资料"，权重低于头部4K的角色人格和剧情状态

---

## 扩展性

未来可扩展的功能：

1. **智能触发策略**：根据用户输入自动判断是否需要深度搜索
2. **多模态检索**：支持图片、音频等多模态事件检索
3. **时间衰减**：根据事件发生时间调整检索权重
4. **情感标签**：为事件添加情感标签，支持情感维度检索

---

**文档版本**: v1.0
**最后更新**: 2025-10-16
