# 导演模块（Director）详细设计

> **需求追溯**：解决 [REQUIREMENTS.md](../REQUIREMENTS.md) "需求1：主线剧情的执行与遵守"
>
> **架构上下文**：见 [DESIGN_OVERVIEW.md](../DESIGN_OVERVIEW.md) 第2章

**文档版本**: v1.0
**最后更新**: 2025-10-16

---

## 模块职责

控制AI按照故事大纲推进剧情，确保不偏离主线。

---

## 核心子功能：剧情大纲控制

通过三个部分协同工作，控制AI按照预设的故事大纲推进剧情。

### 1. 情节点设置

**作用**：让AI始终看到完整的故事大纲。

**实现**：
- 将背景设定中的 `story_outline`（10个情节点）放入 Prompt 头部 4K
- 使用 JSON 格式提升权重
- 每次对话都包含完整大纲

**示例**（Prompt中的格式）：
```json
{
  "story_outline": [
    {"index": 1, "content": "发现背叛者的线索", "status": "completed"},
    {"index": 2, "content": "潜入敌人据点", "status": "completed"},
    {"index": 3, "content": "与仇人对峙", "status": "in_progress"},
    {"index": 4, "content": "做出关键选择", "status": "pending"},
    {"index": 5, "content": "应对选择的后果", "status": "pending"}
  ],
  "current_plot_index": 3
}
```

---

### 2. 进度引导

**作用**：让AI主动报告剧情推进情况。

**实现**：

#### 2.1 Prompt指令
在 Prompt 中告诉 AI：如果剧情推进，输出 `[PROGRESS:X:status]` 标记

**标记格式**：`[PROGRESS:X:status]`
- `X`: 大纲点索引（1-10）
- `status`: 剧情点状态
  - `completed`: 已完成
  - `in_progress`: 进行中
  - `pending`: 待进行

**AI回复示例**：
```
"你的枪口对准了他，他眼中闪过一丝恐惧..." [PROGRESS:3:in_progress]
```

#### 2.2 后端处理流程

```
AI生成回复
  ↓
[1] 正则扫描提取 [PROGRESS] 标记
  ↓
[2] 如果找到标记：
   - 解析 plot_index 和 status
   - 更新 plot_state.json
     * current_plot_index = plot_index
     * current_status = status
     * no_update_count = 0 (清零)
   - 从用户看到的回复中删除此标记
  ↓
[3] 如果没找到标记：
   - plot_state.no_update_count += 1
  ↓
[4] 返回给用户的回复（已删除标记）
```

#### 2.3 容错机制

- AI 不必每次都输出标记，3轮内出现即可
- 标记可以出现在回复任何位置（开头/中间/结尾）
- 后端正则扫描是文本处理，性能影响可忽略（<1ms）

**AI实现可行性**：
- 通过精心设计的 System Prompt + Few-shot Examples，GPT-4/Claude-3.5 的成功率约 70-80%
- 关键技术：
  1. 在 Prompt 头部 4K 明确指示输出规则
  2. 提供 3-5 个示例对话展示正确格式
  3. 在当前剧情状态 JSON 中标记当前进度，引导 AI 判断
- 即使 AI 偶尔忘记输出，兜底机制（RAG）会在 3 轮后介入

---

### 3. 容错处理（RAG兜底）

**作用**：当AI连续多次没有推进剧情时，用RAG检索提醒AI。

#### 3.1 触发条件

```python
if plot_state.no_update_count >= config.features.director_plot_control.rag_fallback_threshold:
    # 触发容错处理
    trigger_rag_fallback()
```

**默认阈值**：3（连续3次 AI 回复都没有 `[PROGRESS]` 标记）

#### 3.2 检索策略

**分层检索策略**（利用历史剧情经验）：

```python
# 构造查询
current_plot = plot_state.current_plot_index
plot_content = background.story_outline[current_plot - 1].content
query = f"故事大纲第{current_plot}点：{plot_content}"

# 第一次检索：当前实例的事件（高权重，当前剧情事实）
events_current = chroma_collection.query(
    query_embeddings=embed(query),
    where={
        "conversation_id": current_instance,
        "character_id": current_character,
        "background_id": current_background
    },
    n_results=15
)

# 第二次检索：其他实例的事件（低权重，剧情经验参考）
events_others = chroma_collection.query(
    query_embeddings=embed(query),
    where={
        "$and": [
            {"conversation_id": {"$ne": current_instance}},
            {"character_id": current_character},
            {"background_id": current_background}
        ]
    },
    n_results=5
)
```

**设计思路**：
- **需求矛盾**：既想利用历史剧情经验帮助AI理解剧情走向，又不想历史事件污染当前剧情的内部一致性
- **解决方案**：分层展示，明确区分"当前剧情事实"和"其他剧情经验参考"
- **权重分配**：当前实例15条（高权重），其他实例5条（低权重）

#### 3.3 Prompt注入

```
【导演提醒】
你现在应该推进到故事大纲第X点：{大纲点内容}

【当前剧情中已发生的事件】（这些是当前剧情中实际发生的事件）
- 事件1...
- 事件2...
...（15条）

【剧情经验参考】（其他剧情中的类似情节，仅供参考剧情走向，不代表当前剧情的事实）
- 参考1...
- 参考2...
...（5条）
```

**Prompt指示**：
- 在系统指令中明确告诉AI，"剧情经验参考"只是帮助理解剧情方向，不要当作当前剧情的事实

#### 3.4 清零机制

```python
# 触发容错后不立即清零
# 等AI回复后，在进度引导的步骤2中：
#   - 如果找到[PROGRESS] → 清零
#   - 如果没找到[PROGRESS] → 继续累加（4, 5, 6...）
# 这样如果AI持续跑偏，每次都会提供RAG提醒，直到AI输出[PROGRESS]
```

#### 3.5 效果

- 平时AI自由发挥，自己判断剧情进度
- 当AI跑偏时，分层检索提供支撑：
  - 当前实例的历史事件：提醒AI已经发生了什么
  - 其他实例的经验：参考类似剧情应该怎么走
- 像一根绳子，拉住AI不要偏离太远，同时利用历史经验增强剧情连贯性

---

## 数据结构依赖

### plot_state.json 文件

格式定义：见 [data_structure.md](data_structure.md#plot_statejson剧情进度可选)

```json
{
  "enabled": true,
  "current_plot_index": 3,
  "current_status": "in_progress",
  "no_update_count": 2
}
```

---

## 配置依赖

### config.json 中的阈值

格式定义：见 [data_structure.md](data_structure.md#配置文件格式configjson)

```json
"features": {
  "director_plot_control": {
    "enabled": true,
    "rag_fallback_threshold": 3
  }
}
```

**配置项说明**：
- `enabled`: 剧情大纲控制功能开关
  - `true`: 启用（Prompt包含story_outline，扫描[PROGRESS]，触发RAG兜底）
  - `false`: 关闭（不放story_outline，不扫描[PROGRESS]，不触发RAG兜底）
- `rag_fallback_threshold`: 触发容错处理的阈值（连续多少次未更新）

---

## 与其他模块的交互

### 依赖：背景设定
- 读取 `background.json` 中的 `story_outline`
- 格式定义：见 [data_structure.md](data_structure.md#背景设置格式backgroundjson)

### 依赖：RAG检索模块
- 容错处理使用RAG检索（见 [rag.md](rag.md)）
- 检索当前实例和其他实例的历史事件
- 返回分层检索结果（当前剧情事实 + 剧情经验参考）

### 依赖：Prompt组装模块
- 情节点设置放入Prompt头部4K（见 [DESIGN_OVERVIEW.md](../DESIGN_OVERVIEW.md) 第3.5章）
- 容错处理的提醒内容注入到Prompt中间区域

---

## 扩展性

导演模块是框架，未来可扩展更多子功能：
- 角色行为控制（确保角色行为符合人格设定）
- 冲突管理（检测剧情冲突并提醒）
- 节奏控制（控制剧情推进的快慢）
- ...

---

**文档版本**: v1.0
**最后更新**: 2025-10-16
