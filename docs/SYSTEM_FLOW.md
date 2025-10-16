# PersonaLab 系统运作机制总览

**文档版本**: v0.4.0
**创建日期**: 2025-10-16
**最后更新**: 2025-10-16
**状态**: ✅ 完成

> 📌 **文档目标**：用具体案例走通完整流程，验证设计的完整性和一致性

---

## 测试案例设定

**角色**: Alserqi（废土黑帮老大）
**背景**: 废土复仇记
**故事大纲**（10个点，只列出前5个）:
1. 发现背叛者的线索
2. 潜入敌人据点
3. 与仇人对峙
4. 做出关键选择
5. 应对选择的后果

**当前状态**:
- 会话实例: inst_001
- 当前会话: sess_003
- 已对话: 41轮
- 当前剧情点: 第3点"与仇人对峙"（in_progress）
- 未更新计数器: 2

**对话历史**（最近3轮）:
```
第39轮：
User: 我们已经潜入据点了，你看前面那个房间。
Assistant: （透过门缝）就是他...Victor，我曾经最信任的兄弟。[PROGRESS:3:in_progress]

第40轮：
User: 你想怎么做？直接冲进去？
Assistant: 不，太危险了。我数了一下，里面至少有五个人，都带着枪。

第41轮：
User: 你打算等到什么时候？
Assistant: 等他们分散。Victor不可能一直和他们在一起。
```

**用户当前输入**（第42轮）：
```
"你还记得我们之前的约定吗？"
```

---

## 完整流程（从点击"发送"到看到AI回复）

### 步骤1：追加用户输入到会话文件

**文件**: `data/instances/inst_001/sessions/sess_003.jsonl`

**动作**: 一次性写入（直接追加完整行）

**新增行**:
```json
{"role":"user","content":"你还记得我们之前的约定吗？","turn":42,"timestamp":"2025-10-16T10:30:00Z"}
```

---

### 步骤2：准备Prompt所需的所有数据

**系统并行读取以下文件**：

**错误处理规则**:
- **必需文件**（读取失败则终止，返回错误）：
  - `character_state.json`：没有角色无法对话
  - `instance_state.json`：没有实例状态无法继续
  - 当前会话文件（`.jsonl`）：没有对话历史无法继续
- **可选文件**（读取失败用默认值）：
  - `background.json`：使用空背景（无world_setting，无story_outline）
  - `config.json`：使用默认配置（所有功能关闭）

---

#### 2.1 角色状态
**文件**: `data/instances/inst_001/character_state.json`
```json
{
  "base_persona": "Alserqi，废土黑帮老大，曾经掌控北区，被心腹背叛...",
  "evolved_persona": "经历背叛后变得多疑，不再轻易相信他人..."
}
```

#### 2.2 背景设定
**文件**: `data/backgrounds/bg_wasteland/background.json`
```json
{
  "background_id": "bg_wasteland",
  "name": "废土复仇记",
  "world_setting": "2087年，核战后50年...",
  "story_outline": [
    {"index": 1, "content": "发现背叛者的线索"},
    {"index": 2, "content": "潜入敌人据点"},
    {"index": 3, "content": "与仇人对峙"},
    {"index": 4, "content": "做出关键选择"},
    {"index": 5, "content": "应对选择的后果"}
  ]
}
```

#### 2.3 会话实例状态
**文件**: `data/instances/inst_001/instance_state.json`
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

#### 2.4 当前会话完整对话
**文件**: `data/instances/inst_001/sessions/sess_003.jsonl`
**内容**: 全部42轮对话（包括刚追加的用户输入）

#### 2.5 日常对话RAG检索

**接口调用**: `history_events = rag.query_user_history(user_input, instance_id)`

**RAG模块内部逻辑**:
```python
def query_user_history(user_input, instance_id):
    # 内部判断是否需要检索
    keywords = ["还记得", "之前", "当时", "那次", "记得吗"]
    if not any(kw in user_input for kw in keywords):
        # 不需要检索，返回空列表
        return []

    # 需要检索
    query = extract_query(user_input)  # 提取关键词："约定"

    events = chroma_summaries.query(
        query_embeddings=embed(query),
        where={"instance_id": instance_id},
        n_results=20
    )

    return events
```

**当前情况**:
- 用户输入："你还记得我们之前的约定吗？"
- RAG检索返回：
  ```
  - "第12轮：玩家让Alserqi承诺不冲动送死"
  - "第20轮：Alserqi答应会冷静行动"
  ```

**如果Chroma为空**（首次对话，尚未汇总）:
- 返回空列表 `[]`
- Prompt中不显示"历史事件回忆"部分

**浅层/深度搜索**:
- 默认返回摘要（Summaries Collection）
- 如果用户问"怎么"、"如何"、"详细过程"，查询详细素材（Plots Collection）

---

### 步骤3：调用导演模块

**接口**: `director.prepare_context(instance_state, background)`

**导演模块内部逻辑**:
```python
def prepare_context(instance_state, background):
    # 检查模块自己的开关（不在全局config.json中）
    if not self.enabled:
        # 功能关闭，返回空
        return None

    # 功能开启，准备剧情大纲控制内容
    plot_control_context = {
        "story_outline": background["story_outline"],
        "current_index": instance_state["plot_state"]["current_plot_index"],
        "instruction": "如果剧情推进，请在回复中输出标记：[PROGRESS:X:status]"
    }

    # 检查是否需要触发容错处理
    # 从全局config.json读取阈值
    config = load_config()
    threshold = config["thresholds"]["rag_fallback_threshold"]

    if instance_state["plot_state"]["no_update_count"] >= threshold:
        # 触发容错处理，执行RAG检索
        fallback_events = rag_fallback(instance_state, background)
        plot_control_context["fallback_reminder"] = fallback_events

    return plot_control_context
```

**本次返回**:
- `story_outline`: 完整大纲（标记当前进度）
- `instruction`: 进度引导指令
- `fallback_reminder`: None（no_update_count=2，未达到阈值3）

---

### 步骤4：组装Prompt

```
【头部4K区域】
=== 角色初始人格 ===
{base_persona}

=== 角色成长人格 ===
{evolved_persona}

=== 背景设定 ===
世界观：{world_setting}

=== 故事大纲 ===
（导演模块提供，如果功能开启）
{
  "story_outline": [
    {"index": 1, "content": "发现背叛者的线索", "status": "completed"},
    {"index": 2, "content": "潜入敌人据点", "status": "completed"},
    {"index": 3, "content": "与仇人对峙", "status": "in_progress"},
    {"index": 4, "content": "做出关键选择", "status": "pending"},
    {"index": 5, "content": "应对选择的后果", "status": "pending"}
  ]
}

=== 进度引导指令 ===
（导演模块提供，如果功能开启）
如果剧情推进，请在回复中输出标记：[PROGRESS:X:status]

【中间区域】
=== 导演提醒 ===
（导演模块容错RAG，如果触发）
（本次未触发，no_update_count=2，未达到阈值3）

=== 历史事件回忆 ===
（日常对话RAG，如果用户提到历史）
- 第12轮：玩家让Alserqi承诺不冲动送死
- 第20轮：Alserqi答应会冷静行动

=== 当前会话完整对话 ===
（全部42轮对话）
...
第41轮：
User: 你打算等到什么时候？
Assistant: 等他们分散。Victor不可能一直和他们在一起。

第42轮：
User: 你还记得我们之前的约定吗？
```

**Prompt组装优先级**：
1. 头部4K（固定）：角色、背景、故事大纲、进度指令
2. 导演提醒（最高优先级，如果触发）
3. 历史事件回忆（日常RAG，如果检索到）
4. 当前会话对话（全部对话历史）

---

### 步骤5：调用LLM并流式处理

#### 5.1 初始化流式连接

**动作**:
1. 调用LLM API（流式模式）
2. 在会话文件准备写入位置
3. 初始化前端SSE连接

#### 5.2 流式写入 + SSE推送

**核心原则：先写文件，再推送前端**

```python
# 打开会话文件（追加模式）
file = open("sess_003.jsonl", "a")

# 初始化AI回复对象
ai_response = {
    "role": "assistant",
    "content": "",
    "turn": 42,
    "timestamp": current_timestamp()
}

# 记录当前行起始位置
current_line_start = file.tell()

# 流式处理
try:
    for chunk in llm.stream(prompt):
        token = chunk.text

        # 1. 追加到内存对象
        ai_response["content"] += token

        # 2. 先写入文件（覆盖当前行）
        file.seek(current_line_start)
        file.write(json.dumps(ai_response, ensure_ascii=False))
        file.flush()  # 强制刷新到磁盘

        # 3. 写入成功后，通过SSE推送到前端
        sse.send_event("token", {"content": token})

    # 流式完成，写入换行符
    file.write("\n")
    file.flush()

    # 通知前端完成
    sse.send_event("done", {})

except Exception as e:
    # 出错处理
    ai_response["error"] = str(e)
    file.seek(current_line_start)
    file.write(json.dumps(ai_response, ensure_ascii=False))
    file.write("\n")
    file.flush()

    # 通知前端错误
    sse.send_event("error", {"message": str(e)})

finally:
    file.close()
```

#### 5.3 前端接收（SSE）

```javascript
// 建立SSE连接
const eventSource = new EventSource('/api/chat/stream')

// 接收token
eventSource.addEventListener('token', (e) => {
    const data = JSON.parse(e.data)
    appendToDisplay(data.content)  // 追加显示
})

// 接收完成信号
eventSource.addEventListener('done', (e) => {
    eventSource.close()
})

// 接收错误
eventSource.addEventListener('error', (e) => {
    const data = JSON.parse(e.data)
    showError(data.message)
    eventSource.close()
})

// SSE连接断开（网络问题）
eventSource.onerror = () => {
    // 重新读取文件，恢复显示
    fetch(`/api/sessions/sess_003/current_response?turn=42`)
        .then(res => res.json())
        .then(data => {
            display(data.content)  // 显示文件中已保存的内容
        })
}
```

**关键点**：
- **先写文件，再推送**：确保前端显示的内容一定已持久化
- **实时性好**：token生成后立即推送，无延迟
- **效率高**：只在有新内容时推送，不浪费请求
- **断线重连**：SSE断开后，前端从文件恢复显示
- **单一数据源**：文件是唯一真相，SSE只是传输通道

**AI回复内容**（示例，逐token生成）:
```
我当然记得。（沉默片刻）我答应过你，不会冲动送死。但Victor必须付出代价，这是我活下去的唯一理由。我会等，等到最安全的时机。[PROGRESS:3:in_progress]
```

**前端展示**:
- 直接显示包含 `[PROGRESS:3:in_progress]` 标记的完整内容
- **不过滤标记**

**文件最终写入**:
```json
{"role":"assistant","content":"我当然记得。（沉默片刻）我答应过你，不会冲动送死。但Victor必须付出代价，这是我活下去的唯一理由。我会等，等到最安全的时机。[PROGRESS:3:in_progress]","turn":42,"timestamp":"2025-10-16T10:30:15Z"}
```

---

### 步骤6：调用导演模块处理AI回复

**接口**: `director.process_response(ai_response_content, instance_state)`

**导演模块内部逻辑**:
```python
def process_response(response_content, instance_state):
    # 检查模块自己的开关
    if not self.enabled:
        # 功能关闭，什么都不做
        return

    # 功能开启，扫描当前回复
    match = re.search(r'\[PROGRESS:(\d+):(completed|in_progress|pending)\]', response_content)

    if match:
        # 找到标记
        plot_index = int(match.group(1))
        status = match.group(2)

        # 更新状态
        instance_state["plot_state"]["current_plot_index"] = plot_index
        instance_state["plot_state"]["current_status"] = status
        instance_state["plot_state"]["no_update_count"] = 0  # 清零
    else:
        # 没找到标记
        instance_state["plot_state"]["no_update_count"] += 1

    # 保存更新后的状态
    save_instance_state(instance_state)
```

**本次执行**:
- 扫描到 `[PROGRESS:3:in_progress]`
- `no_update_count` 从 2 清零为 0
- 更新 `instance_state.json`

**更新后的文件**:
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
    "no_update_count": 0
  }
}
```

---

### 步骤7：流式完成

**前端**: 显示完整AI回复
**文件**: 已实时写入
**状态**: 已更新

**单一数据源**: 文件内容 = 前端显示内容

---

## 异常情况处理

### 情况1：用户中断（点击"停止"）

**前端动作**:
1. 发送中断信号到后端
2. 关闭SSE连接

**后端动作**:
1. 终止LLM流式连接
2. **立即完成当前行的写入**:
   ```json
   {"role":"assistant","content":"我当然记得。（沉默片刻）我答应过你，不会","turn":42,"timestamp":"2025-10-16T10:30:15Z","interrupted":true}
   ```
3. 写入换行符，关闭文件
4. **调用导演模块处理不完整的回复**（可能没有[PROGRESS]标记）

**结果**:
- 文件保存了部分回复（标记 `interrupted: true`）
- 前端显示部分回复
- 导演模块：如果没有[PROGRESS]，`no_update_count += 1`

### 情况2：连接断开（网络中断）

**后端检测**:
- SSE连接异常关闭
- LLM API超时

**处理方式**: 同"用户中断"

**前端重连**:
- 重新加载会话文件（单一数据源）
- 显示最后完整保存的内容

### 情况3：空回复（LLM返回空内容）

**LLM流式结束时检查**:
```python
if ai_response["content"].strip() == "":
    # 空回复，标记并保存
    ai_response["content"] = "(无回复)"
    ai_response["empty"] = true
```

**导演模块处理**:
- 空回复视为"没有[PROGRESS]标记"
- `no_update_count += 1`

### 情况4：LLM API错误

**错误捕获**:
```python
try:
    for chunk in llm.stream(prompt):
        # 流式处理
except APIError as e:
    # 保存错误信息
    ai_response["content"] = f"(系统错误: {e.message})"
    ai_response["error"] = true
    # 写入文件
    save_response(ai_response)
```

**结果**:
- 文件保存错误信息
- 前端显示错误提示
- 导演模块：`no_update_count += 1`

---

## 特殊情况：触发容错处理（RAG兜底）

假设第43、44、45轮AI都没有输出 `[PROGRESS]` 标记，第46轮会发生什么？

### 状态变化

```
第42轮: 找到[PROGRESS] → no_update_count = 0
第43轮: 没找到[PROGRESS] → no_update_count = 1
第44轮: 没找到[PROGRESS] → no_update_count = 2
第45轮: 没找到[PROGRESS] → no_update_count = 3
```

### 第46轮的步骤3：调用导演模块

**导演模块内部检测**:
```python
def prepare_context(instance_state, background):
    # 检查模块自己的开关
    if not self.enabled:
        return None

    # ... 准备基础内容 ...

    # 检查是否达到阈值（从全局config.json读取）
    config = load_config()
    threshold = config["thresholds"]["rag_fallback_threshold"]

    if instance_state["plot_state"]["no_update_count"] >= threshold:
        # 达到阈值3，触发容错处理
        fallback_events = rag_fallback_query(instance_state, background)
        plot_control_context["fallback_reminder"] = fallback_events

        # 注意：触发后不清零，等AI回复后再由process_response()处理

    return plot_control_context
```

**说明**:
- 触发容错后**不立即清零**
- AI回复后，在 `process_response()` 中：
  - 如果找到[PROGRESS] → 清零
  - 如果没找到[PROGRESS] → 继续累加（4, 5, 6...）
- 这样如果AI持续跑偏，每次都会提供RAG提醒，直到AI输出[PROGRESS]

### 容错处理RAG检索

**导演模块内部执行**:
```python
def rag_fallback_query(instance_state, background):
    current_plot = instance_state["plot_state"]["current_plot_index"]
    plot_content = background["story_outline"][current_plot - 1]["content"]

    query = f"故事大纲第{current_plot}点：{plot_content}"

    # 检索当前实例事件
    events_current = chroma.query(
        query_embeddings=embed(query),
        where={
            "instance_id": instance_state["instance_id"],
            "character_id": instance_state["character_id"],
            "background_id": instance_state["background_id"]
        },
        n_results=15
    )

    # 检索其他实例事件（剧情经验参考）
    events_others = chroma.query(
        query_embeddings=embed(query),
        where={
            "instance_id": {"$ne": instance_state["instance_id"]},
            "character_id": instance_state["character_id"],
            "background_id": instance_state["background_id"]
        },
        n_results=5
    )

    return {
        "current_events": events_current,
        "reference_events": events_others
    }
```

### 第46轮的Prompt会包含

```
【导演提醒】
你现在应该推进到故事大纲第3点：与仇人对峙

【当前剧情中已发生的事件】
- 事件1：Alserqi发现Victor的藏身地
- 事件2：Alserqi和玩家潜入敌人据点
...

【剧情经验参考】（其他剧情线的类似情节，仅供参考）
- 参考1：在其他剧情线中，对峙时Victor试图解释背叛原因
...
```

---

## 其他核心功能流程

### 汇总功能（用户触发）

**触发**: 用户点击"📝 汇总"按钮

**流程**:
1. 读取当前会话文件（sess_003.jsonl）
2. 调用LLM生成分层摘要：
   - Summaries（情节点摘要）
   - Plots（详细剧情素材）
3. 写入Chroma向量数据库
4. 创建新会话文件（sess_004.jsonl）
5. 新会话初始内容：
   - 根据配置决定顺序（`config.preferences.summary_order`）
   - 包含最后N轮对话（`config.thresholds.summary_last_n_turns`）
6. 更新 `instance_state.json` 中的 `current_session_id`

### 更新记忆功能（用户触发）

**触发**: 用户点击"🧠 更新记忆"按钮

**流程**:
1. 读取 `character_state.json`
2. 读取最近N轮对话（如最近20轮）
3. 调用LLM整合：
   ```
   base_persona（不变）+ 旧evolved_persona + 最近对话
   → 生成新的evolved_persona
   ```
4. 更新 `character_state.json` 中的 `evolved_persona`

### 切换背景

**触发**: 用户在前端选择新背景

**流程**:
1. 更新 `instance_state.json` 中的 `background_id`
2. 下次对话时：
   - Prompt使用新的 `background.json`（新的 `world_setting` 和 `story_outline`）
   - 如果导演模块开启，AI会按新的故事大纲推进

**注意**: `plot_state` 不需要重置，导演模块会根据新的story_outline工作

---

## Prompt边界控制与警示系统

### 配置文件

**文件**: `data/config.json`

```json
{
  "thresholds": {
    "rag_fallback_threshold": 3,
    "summary_last_n_turns": 5
  },
  "limits": {
    "max_total_tokens": 100000,
    "middle_section_warning_tokens": 20000,
    "conversation_max_tokens": 100000
  },
  "preferences": {
    "summary_order": "summary_first",
    "conversation_load_all": true
  }
}
```

**配置项说明**：

**阈值** (`thresholds`):
- `rag_fallback_threshold`: 容错处理触发阈值（连续N次未更新）
- `summary_last_n_turns`: 汇总后保留最后N轮对话

**限制** (`limits`):
- `max_total_tokens`: Prompt总长度上限
- `middle_section_warning_tokens`: 中间区域警告阈值
- `conversation_max_tokens`: 会话文件最大token数

**偏好设置** (`preferences`):
- `summary_order`: 汇总后新会话的初始内容顺序（`summary_first` / `last_n_first`）
- `conversation_load_all`: 是否全量读取会话文件

**注意**: 功能开关（如 `director_plot_control`）不在全局配置文件中，跟随各自模块管理

### 前端设置页面

**位置**: 前端导航栏"设置"标签页

**界面布局**：

```
┌─────────────────────────────────────────────┐
│ 设置                                         │
├─────────────────────────────────────────────┤
│                                              │
│ 【阈值】                                     │
│ 容错处理触发阈值                             │
│ [  3  ] 连续N次未更新后触发                 │
│                                              │
│ 汇总保留轮数                                 │
│ [  5  ] 轮                                   │
│                                              │
│ 【限制】                                     │
│ Prompt总长度上限                             │
│ [100000] tokens                              │
│                                              │
│ 中间区域警告阈值                             │
│ [ 20000] tokens                              │
│ ℹ️ 超过此值会显示警告，但不会截断            │
│                                              │
│ 会话文件最大token数                          │
│ [100000] tokens                              │
│                                              │
│ 【偏好设置】                                 │
│ 新会话初始内容顺序                           │
│ ○ 先放摘要，再放对话                        │
│ ● 先放对话，再放摘要                        │
│                                              │
│ ☑ 全量读取会话文件                          │
│                                              │
│          [恢复默认]  [保存]                  │
└─────────────────────────────────────────────┘
```

**交互逻辑**：
1. 页面加载时从 `config.json` 读取当前配置
2. 用户修改后点击"保存"
3. 后端更新 `config.json`
4. 配置立即生效（不需要重启服务）

**验证规则**：
- `rag_fallback_threshold`: 整数，范围 1-10
- `max_total_tokens`: 整数，范围 10000-200000
- `middle_section_warning_tokens`: 整数，范围 1000-50000
- `last_n_turns`: 整数，范围 1-20

**恢复默认**：
```json
{
  "thresholds": {
    "rag_fallback_threshold": 3,
    "summary_last_n_turns": 5
  },
  "limits": {
    "max_total_tokens": 100000,
    "middle_section_warning_tokens": 20000,
    "conversation_max_tokens": 100000
  },
  "preferences": {
    "summary_order": "summary_first",
    "conversation_load_all": true
  }
}
```

### 总量限制

**上限**: `max_total_tokens`（默认100000，可调）

**超过上限的处理**:
- 终止Prompt组装
- 返回错误给前端："Prompt总长度超过限制（{实际值} > {上限}），请执行汇总或调整配置"

### 中间区域警告

**阈值**: `middle_section_warning_tokens`（默认20000）

**中间区域包含**:
- 导演提醒（容错RAG）
- 历史事件回忆（日常RAG）
- 当前会话完整对话

**超过阈值的处理**:
- **不截断，继续执行**
- 发送警示到前端（通过SSE或响应头）
- 前端显示警示信息

**警示内容**:
```json
{
  "type": "warning",
  "category": "middle_section_overflow",
  "message": "当前对话历史过长",
  "current_value": 25000,
  "threshold": 20000,
  "suggestion": "建议执行汇总功能"
}
```

### 前端警示区域

**默认状态（收缩）**:
```
┌─────────────────────────┐
│ [⚠️ 2] 系统警示          │
└─────────────────────────┘
```

**点击展开**:
```
┌─────────────────────────────────────────┐
│ ⚠️ 当前对话历史过长（25000 tokens）     │
│ ⚠️ Prompt总长度接近上限（95000 tokens） │
│                         [双击查看详情]   │
└─────────────────────────────────────────┘
```

**双击弹窗**:
```
┌─────────────────────────────┐
│ 警示详情                     │
│ ━━━━━━━━━━━━━━━━━━━━━━━   │
│ 类型：当前对话历史过长       │
│ 当前值：25000 tokens         │
│ 建议阈值：20000 tokens       │
│ 建议操作：执行汇总功能       │
│                              │
│ 影响：可能导致响应变慢、     │
│       成本增加               │
└─────────────────────────────┘
```

**实现**:
- 警示角标默认在输入框上方
- 点击展开显示警示列表
- 双击警示项弹出详情窗口
- 警示自动去重（同类型只显示一条）

---

## 核心设计原则总结

### 1. 单一数据源
- **文件 = 前端显示**
- 流式写入确保一致性
- 异常时以文件为准

### 2. 职责分离
- **主流程**：负责数据准备、Prompt组装、流式处理
- **导演模块**：内部检查配置开关、管理自己的状态、决定是否触发容错
- **RAG模块**：提供日常对话检索和容错检索两种服务

### 3. 配置驱动
- 全局配置（阈值、限制、偏好）由 `config.json` 统一管理
- 功能开关跟随各自模块管理
- 主流程不关心具体逻辑

### 4. 流式处理
- 用户输入：一次性写入完整内容
- AI回复：流式写入（逐token）
- 前端显示：实时更新
- 异常处理：保证文件完整性

---

**文档版本**: v0.4.0
**创建日期**: 2025-10-16
**最后更新**: 2025-10-16
