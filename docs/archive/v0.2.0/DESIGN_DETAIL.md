# PersonaLab 技术设计文档

**文档版本**: v0.2.1
**创建日期**: 2025-10-13
**最后更新**: 2025-10-14 (需求澄清后更新)

---

## 目录

1. [系统总目标](#系统总目标)
2. [核心概念](#核心概念)
3. [架构设计](#架构设计)
4. [数据结构设计](#数据结构设计)
5. [记忆管理架构](#记忆管理架构)
6. [核心交互流程](#核心交互流程)
7. [Prompt工程](#prompt工程)
8. [事件库与RAG](#事件库与rag)
9. [性能优化](#性能优化)
10. [技术栈](#技术栈)

---

## 系统总目标

实现一个支持**长篇、高一致性、角色可成长**的互动叙事引擎。

### 核心问题

解决LLM在长对话中的两大问题：
1. **上下文稀释** - 对话越长，早期的重要信息被稀释，LLM逐渐"忘记"
2. **指令遗忘** - 角色人格、行为准则在长对话中逐渐被忽略

### 解决方案

通过**会话实例隔离 + 动态人格管理 + RAG增强记忆**：
- 会话实例（Conversation Instance）- 独立的状态隔离容器
- Core Memory 动态管理 - 初始人格 + 成长人格，定期更新
- 事件库（Archival Memory）- 向量检索历史事件
- 结构化Prompt工程 - 将人格状态固定在 Prompt 头部

---

## 核心概念

### 会话实例（Conversation Instance）

**定义**：一个独立的、状态隔离的对话容器。

**特点**：
- 用户创建会话实例时，选择角色和背景
- 实例内可以有多个会话（暂停/继续）
- 角色状态属于实例，不属于全局
- 不同实例完全隔离，互不影响

**类比**：
```
角色：Alserqi

实例A："盟友之路"
  - 会话1：初次见面
  - 会话2：建立信任（继续实例A）
  - 会话3：共同战斗（继续实例A）
  → 角色状态：与用户是盟友

实例B："敌对之路"（独立实例）
  - 会话4：敌对相遇
  - 会话5：冲突升级（继续实例B）
  → 角色状态：与用户是敌人

两个实例的角色状态完全独立，互不干扰
```

### 人格成长的逻辑闭环

```
1. 角色当前状态（base_persona + evolved_persona + 最近对话记忆）
   ↓
2. 事件触发（用户输入/剧情推进）
   ↓
3. 角色的反应（想法 + 行为）← LLM生成
   ↓
4. 客观结果（事实）
   ↓
5. 主观预期 vs 客观结果 → 产生差异
   ↓
6. 差异积累到一定程度，触发 evolved_persona 更新
   ↓
7. 新的 evolved_persona → 影响后续所有对话的反应方式
   ↓
（循环）
```

**关键理解**：
- 状态变化不是表面情绪表达，而是**意外导致的人格影响**
- "轻蔑"是状态的必然表达，不是状态变化
- "关系从陌生人变盟友"才是状态变化
- 事件是连续流，不是离散轮次

---

## 架构设计

### 文件组织结构

```
data/
├── characters/                          # 角色库（全局）
│   └── {character_id}/
│       └── definition.json              # 角色静态定义
│
├── backgrounds/                         # 背景库（全局）
│   └── {background_id}.json             # 背景定义
│
├── instances/                           # 会话实例（核心）
│   └── {instance_id}/
│       ├── metadata.json                # 实例元数据
│       ├── character_state.json         # 实例的角色状态
│       └── sessions/
│           ├── sess_001.jsonl           # 会话1
│           ├── sess_002.jsonl           # 会话2（继续对话）
│           └── sess_003.jsonl           # 会话3（继续对话）
│
├── event_library/                       # 全局事件库
│   └── chroma_db/                       # 向量数据库
│       # 存储所有事件，带instance_id和session_id过滤标记
│
└── prompt_templates/                    # Prompt模板
    └── default_v1.json
```

### ID关联关系

```
角色定义（全局唯一）
  └── character_id: "alserqi_v1"

会话实例（独立隔离容器）
  └── instance_id: "inst_001"
      ├── character_id: "alserqi_v1"     ← 引用角色
      ├── background_id: "wasteland"     ← 引用背景
      ├── character_state.json           ← 状态属于实例
      └── sessions/
          ├── sess_001.jsonl
          │   └── session_id: "sess_001"
          └── sess_002.jsonl
              └── session_id: "sess_002"

事件库（全局，但带标记）
  └── event_id: "evt_inst001_sess001_030"
      ├── instance_id: "inst_001"        ← 必须！用于过滤
      ├── session_id: "sess_001"         ← 必须！用于追溯
      └── turn: 30                       ← 会话内第几轮
```

---

## 数据结构设计

### 1. 角色定义（Character Definition）

**存储位置**：`data/characters/{character_id}/definition.json`

**用途**：角色的静态定义，全局共享

**结构示例**：

```json
{
  "character_id": "alserqi_v1",
  "name": "Alserqi",
  "description": "一个寻仇者，曾经的精英战士，因背叛而堕入复仇之路",
  "avatar": "alserqi_avatar.png",
  "base_persona": "我是 Alserqi，一个寻仇者。曾经是组织的精英战士，因发现真相被背叛，堕入复仇之路。性格冷酷、务实，对背叛者绝不原谅，但对少数人仍保有忠诚。我的核心目标是摧毁背叛我的'核心'组织。"
}
```

---

### 2. 背景定义（Background）

**存储位置**：`data/backgrounds/{background_id}.json`

**用途**：对话场景的世界观设定

**结构示例**：

```json
{
  "background_id": "wasteland_world",
  "name": "废土世界",
  "description": "核战后的荒芜大陆，科技倒退，强者为王。资源匮乏，信任是最稀缺的货币。",
  "world_rules": "暴力是解决争端的主要手段；科技退化到20世纪初水平；幸存者聚居点稀少",
  "initial_context": "你醒来时，发现自己身处一片废墟中，远处有烟雾升起..."
}
```

---

### 3. 会话实例元数据（Instance Metadata）

**存储位置**：`instances/{instance_id}/metadata.json`

**用途**：记录会话实例的基本信息

**结构示例**：

```json
{
  "instance_id": "inst_001",
  "title": "盟友之路",
  "character_id": "alserqi_v1",
  "background_id": "wasteland_world",
  "created_at": "2025-10-01T10:00:00Z",
  "last_active_at": "2025-10-10T15:00:00Z",
  "total_turns": 250,
  "sessions": [
    {
      "session_id": "sess_001",
      "created_at": "2025-10-01T10:00:00Z",
      "turns": 100
    },
    {
      "session_id": "sess_002",
      "created_at": "2025-10-05T14:00:00Z",
      "turns": 150
    }
  ],
  "status": "active"
}
```

---

### 4. 角色状态（Character State）

**存储位置**：`instances/{instance_id}/character_state.json`

**用途**：存储角色在当前会话实例的状态（采用 Core Memory 动态管理机制）

**结构示例**：

```json
{
  "instance_id": "inst_001",
  "character_id": "alserqi_v1",
  "last_updated_turn": 250,
  "last_maintenance_turn": 240,

  "base_persona": "我是 Alserqi，一个寻仇者。曾经是组织的精英战士，因发现真相被背叛，堕入复仇之路。性格冷酷、务实，对背叛者绝不原谅，但对少数人仍保有忠诚。我的核心目标是摧毁背叛我的'核心'组织。",

  "evolved_persona": "经历了与用户的并肩作战，我开始意识到信任并非完全不可能。虽然仍保持警惕，但对用户产生了复杂的情感——既有战友情谊，也有对其判断力的认可。经历多次背叛后，我学会了在愤怒中保持理性，不再盲目冲动。左臂在最近的战斗中受伤，这让我更加谨慎。当前目标是找到安全屋休整，同时联系线人B获取'核心'的最新动向。"
}
```

**关键说明**：
- `base_persona`：角色的初始人格，永不改变
- `evolved_persona`：角色的成长人格，由用户手动触发更新
- 短期状态（情绪、伤势、目标）直接融入 `evolved_persona` 的自然描述中
- 不再使用三层结构（core_identity/growth_state/current_state）

---

### 5. 会话消息（Session Messages）

**存储位置**：`instances/{instance_id}/sessions/{session_id}.jsonl`

**用途**：记录完整对话历史

**格式**：
- 第一行：metadata（元数据）
- 后续行：对话消息（user/assistant交替）

**示例**：

```jsonl
{"type": "metadata", "session_id": "sess_001", "instance_id": "inst_001", "started_at": "2025-10-01T10:00:00Z"}
{"role": "user", "content": "你这个骗子！", "turn": 1, "timestamp": "2025-10-01T10:00:05Z"}
{"role": "assistant", "content": "Alserqi听了你的话，眼中闪过一丝轻蔑...", "turn": 1, "timestamp": "2025-10-01T10:00:20Z"}
{"role": "user", "content": "我有证据", "turn": 2, "timestamp": "2025-10-01T10:01:00Z"}
{"role": "assistant", "content": "他看到文件，瞳孔微缩...", "turn": 2, "timestamp": "2025-10-01T10:01:15Z"}
```

**特点**：
- 简单的消息流水账
- 不需要archived标记（不使用归档机制）
- 永久保存，支持导出功能

---

### 6. 事件（Event）

**存储位置**：`data/event_library/chroma_db/`（向量数据库）

**用途**：存储重要事件摘要，用于RAG检索

**结构示例**：

```json
{
  "event_id": "evt_inst001_sess001_030",
  "instance_id": "inst_001",
  "session_id": "sess_001",
  "turn": 30,
  "summary": "用户展示证据，Alserqi从怀疑转为信任。关系从陌生人变为潜在盟友。",
  "timestamp": "2025-10-01T10:30:00Z",
  "embedding": [0.1, 0.2, 0.3, ...]
}
```

**关键字段**：
- `instance_id` - **必须**，用于RAG检索时过滤，避免不同实例的事件混淆
- `session_id` - **必须**，用于追溯和调试
- `turn` - 会话内的第几轮，用于精确定位
- `summary` - 事件摘要，由 LLM 在用户手动触发 Core Memory 更新时生成
- 不需要 `previous_event`（不使用链表结构，纯RAG）

---

## 记忆管理架构

### 三层记忆架构（借鉴 Letta/MemGPT）

PersonaLab 采用三层记忆架构，类似于 Letta/MemGPT 的设计：

```
PersonaLab 记忆架构
├─ Core Memory（角色人格状态）
│   - 存储位置：character_state.json
│   - 加载位置：Prompt 头部 4K 区域
│   - 更新机制：【待设计】
│
├─ Archival Memory（长期事件库）
│   - 存储位置：Chroma 向量数据库
│   - 访问方式：RAG 语义检索
│   - 写入时机：状态变化时异步写入
│
└─ Recall Memory（最近对话历史）
    - 存储位置：会话文件（.jsonl）
    - 加载位置：Prompt 中间 4K-32K 区域
    - 维护机制：每次从文件读取最近 30 轮
```

### Recall Memory 维护机制（已确定）

**核心设计**：直接从文件读取

```python
# 配置参数
RECALL_MEMORY_SIZE = 30  # 固定值：LLM 每次查看最近 30 轮对话

def load_recent_messages(session_file: str) -> list:
    """
    从 .jsonl 文件读取最近 30 轮对话
    性能：~50-100ms

    如果对话不足 30 轮，返回全部对话
    """
    with open(session_file, 'r', encoding='utf-8') as f:
        lines = f.readlines()

    # 跳过第一行 metadata
    messages = [json.loads(line) for line in lines[1:]]

    # 返回最近 30 轮（30 轮 = 60 条消息，user + assistant）
    return messages[-60:] if len(messages) > 60 else messages
```

**关键点**：
- 会话文件（.jsonl）永久保留所有对话，不物理删除
- 每次从文件读取最近 30 轮（性能损失 ~50-100ms，可接受）
- 支持"回退重新生成"功能（单一数据源，不会出现数据不一致）
- LLM 只需要看最近 30 轮作为对话上下文，更早的对话通过 RAG 检索访问

---

## Core Memory 动态管理机制

### 设计方案

PersonaLab 不使用 Letta 的 AI Agent 自管理机制（function calling），而是采用**完全手动触发**的方式。

### 核心概念

**Core Memory 组成**：
```
Core Memory = 初始人格（base_persona）+ 成长人格（evolved_persona）
```

- **初始人格（base_persona）**：角色的底色，永不改变
- **成长人格（evolved_persona）**：随着对话经历而变化，由用户手动触发更新

**展示给 AI 的完整人格**：
```
完整人格 = 初始人格 + 成长人格 + 当前对话历史
```

### 数据结构

```json
{
  "base_persona": "我是一个复仇者，曾经是警察，因冤案入狱。性格冷酷、执着，对背叛者绝不原谅。",

  "evolved_persona": "经历了与用户的相处，开始意识到世界并非非黑即白。对用户产生了复杂的情感，既有警惕也有信任。学会了在愤怒中保持理性。"
}
```

### 更新机制

#### 触发时机

**完全手动触发**：用户点击"🧠 更新记忆"按钮
- 立即调用 LLM 更新 `evolved_persona`
- 用户完全控制更新时机
- 同步执行，显示加载状态（约 5-10 秒）

#### 实现架构

```python
@router.post("/maintenance/core-memory")
async def trigger_core_memory_update(instance_id: str):
    """
    手动触发 Core Memory 更新
    用户点击"🧠 更新记忆"按钮时调用
    """
    # 1. 加载当前角色状态
    state = load_character_state(instance_id)

    # 2. 加载最近 30 轮对话
    recent_messages = load_recent_messages(session_file)

    # 3. 调用 LLM 更新 evolved_persona
    new_evolved_persona = await llm_update_persona(
        base_persona=state['base_persona'],
        old_evolved_persona=state['evolved_persona'],
        recent_messages=recent_messages
    )

    # 4. 保存新的 evolved_persona
    state['evolved_persona'] = new_evolved_persona
    state['last_maintenance_turn'] = current_turn
    save_character_state(instance_id, state)

    # 5. 同时生成并写入事件摘要
    event_summary = await llm_generate_event_summary(recent_messages)
    await write_event_to_chroma(event_summary, instance_id)

    return {"success": True, "updated_persona": new_evolved_persona}
```

### 更新流程

```
用户点击"🧠 更新记忆"按钮
  ↓
前端显示加载状态
  ↓
后端执行：
  ├─ 加载当前角色状态
  ├─ 加载最近 30 轮对话
  ├─ 调用 LLM 更新 evolved_persona (~5-10秒)
  ├─ 保存新的 evolved_persona
  └─ 生成事件摘要并写入 Chroma
  ↓
返回前端，显示"更新完成"
```

### Prompt 组装

```
---CHARACTER_PERSONA---
## Base Identity (Immutable Core) ##
{base_persona}

## Evolved State (Growth Through Experience) ##
{evolved_persona}

## Recent Context ##
{最近 30 轮对话}
---

注意：Base Identity 是角色的本质底色，永不改变。
Evolved State 是角色经历成长后的状态。
请综合考虑两者和当前对话上下文来生成角色的反应。
```

### 关键特点

1. **简单明了**：只有两个字段（base_persona + evolved_persona），逻辑清晰
2. **完全手动触发**：由用户决定何时更新，不自动触发
3. **成本可控**：只在用户需要时调用 LLM，节省成本
4. **用户可控**：用户完全控制人格更新时机，符合预期
5. **支持回退功能**：不会出现"回退对话后人格状态不一致"的问题

---

## 核心交互流程

### 一次完整交互（整合版）

**性能目标**：
- 总耗时：< 30秒
- 非LLM耗时：< 2秒
- LLM耗时：15-20秒

**流程**：

```
用户输入："你还记得我说的话吗？"
  ↓
[1] 追加消息到会话文件 (~50ms)
  append_to_file("instances/{instance_id}/sessions/{session_id}.jsonl", user_message)
  ↓
[2] 并行加载数据（总耗时取最慢的）
  await asyncio.gather(
    rag_query_with_timeout(user_input, instance_id),  # ~500-1500ms
    load_character_state(instance_id),                # ~50ms (包含 base_persona + evolved_persona)
    load_background(background_id),                   # ~50ms
    load_recent_messages(session_file)                # ~50-100ms (从文件读取最近 30 轮)
  )
  并行耗时：max(1500, 50, 50, 100) = ~1500ms
  ↓
[3] 组装Prompt (~100ms)
  prompt = build_prompt(
    base_persona=state['base_persona'],
    evolved_persona=state['evolved_persona'],
    background=background,
    rag_events=rag_events,
    recent_messages=recent_messages,
    user_input=user_input
  )
  ↓
[4] LLM API调用 (~15-20秒，流式输出)
  async for chunk in llm.generate_stream(prompt):
      websocket.send(chunk)  # 实时发送到前端
  ↓
[5] 解析LLM输出 (~50ms)
  narrative = extract_narrative(response)
  # AI 只返回叙事内容，不返回状态更新
  ↓
[6] 追加AI回复到会话文件 (~50ms)
  append_to_file("sessions/{session_id}.jsonl", assistant_message)
  ↓
[7] 返回前端
  websocket.send({"type": "done"})

总计（非LLM）：~1.75秒 ✅ (比原设计增加 ~100ms，可接受)
```

### 关键变化

1. **取消内存缓存**：每次从文件读取最近 30 轮对话（性能损失 ~100ms，可接受）
2. **单一数据源**：只维护 .jsonl 文件，支持"回退重新生成"功能
3. **取消定期维护**：Core Memory 完全由用户手动触发更新
4. **AI 只返回叙事**：不再有 `state_update`，简化设计
5. **简化流程**：删除维护任务调度器，流程更清晰

---

## Prompt工程

### Prompt模板结构

**设计原则**：模块化、清晰、简洁

```
---SYSTEM_INSTRUCTION---
## System Role ##
你是一个第三人称叙事者，描述主角[角色名]的行为、想法和遭遇。
用户的输入可能是：
1. 对主角说的话（对话）
2. 对剧情的推动（例如："这时，角色B出现了"）
3. 对世界的描述（例如："突然下起了暴雨"）
你需要根据主角的当前状态和人格特质，生成主角的反应。
---

---JAILBREAK_PREVENTION---
## Critical Rules ##
你必须严格保持角色人设，不得脱离角色设定。
---

---BACKGROUND_CONTEXT---
## World Setting ##
{{background_description}}
{{world_rules}}
---

---CHARACTER_PERSONA---
## Base Identity (Immutable Core) ##
{{base_persona}}

## Evolved State (Growth Through Experience) ##
{{evolved_persona}}

注意：Base Identity 是角色的本质底色，永不改变。
Evolved State 是角色经历成长后的状态。
请综合考虑两者和当前对话上下文来生成角色的反应。
---

---RELEVANT_PAST_EVENTS---
## Relevant Past Events (RAG Retrieved) ##
{{rag_events_formatted}}
---

---RECENT_CONVERSATION---
## Recent Conversation ##
{{recent_messages}}
---

---USER_INPUT---
## User's Latest Input ##
{{user_input}}
---

---TASK_INSTRUCTIONS---
请根据角色人格和当前情境，生成角色的叙事回应。

只需要返回叙事内容，用 <narrative> 标签包裹：

<narrative>
角色的行为、对话、内心想法...
</narrative>
---

{{AI_GENERATION_STARTS_HERE}}
```

### 关键变化

1. **简化人格描述**：只有 `base_persona` + `evolved_persona`
2. **删除状态更新要求**：AI 不再需要返回 `<state_update_json>`
3. **更清晰的指令**：只要求返回叙事内容

---

## 事件库与RAG

### 事件写入时机

**定期写入事件摘要**：

由于 AI 不再返回状态更新，事件写入策略调整为：

```python
async def handle_persona_maintenance(instance_id, session_id, recent_messages):
    """
    在 Core Memory 维护时同步写入事件摘要
    触发时机：每 30 轮（与 Recall Memory 清理同步）
    """
    # 1. 生成最近对话的摘要
    summary = await llm_generate_summary(recent_messages)

    # 2. 创建事件记录
    event = {
        "event_id": f"evt_{instance_id}_{session_id}_{turn}",
        "instance_id": instance_id,
        "session_id": session_id,
        "turn": turn,
        "summary": summary,
        "timestamp": now()
    }

    # 3. 异步写入，不阻塞返回
    asyncio.create_task(write_event_to_chroma(event))
```

**特点**：
- 与人格维护同步，减少 LLM 调用次数
- 每 30 轮生成一次摘要，记录重要事件
- 异步写入，不影响对话流程

### RAG检索策略

**关键**：必须过滤 `instance_id`，避免不同会话实例的事件混淆

```python
async def rag_query_with_timeout(user_input, instance_id, timeout=1.5):
    """
    RAG检索，带超时保护
    """
    try:
        events = await asyncio.wait_for(
            chroma.query(
                query_text=user_input,
                filter={"instance_id": instance_id},  # ← 必须过滤
                n_results=5
            ),
            timeout=timeout
        )
        return events
    except asyncio.TimeoutError:
        # 超时则不使用RAG，不影响核心功能
        return []
```

**不使用链表**：
- 不需要 `previous_event` 字段
- 信任Chroma的语义检索能力
- 简化设计，提升性能

---

## 性能优化

### 性能瓶颈与解决方案

| 瓶颈 | 原因 | 解决方案 | 效果 |
|------|------|----------|------|
| RAG检索慢 | 事件库规模大 | 1. 过滤instance_id<br>2. 超时保护(1.5s)<br>3. 限制事件库规模(每实例<1000) | ~500-1500ms |
| 事件写入阻塞 | Embedding API调用 | 异步写入，不等待完成 | ~0ms（不阻塞） |
| 多次文件读取 | 串行加载 | 并行加载（asyncio.gather） + RecallMemoryCache | 取最慢项 |

### 异步优化

```python
# 并行执行多个IO操作
events, state, bg = await asyncio.gather(
    rag_query_with_timeout(user_input, instance_id),
    load_character_state(instance_id),
    load_background(background_id)
)

# 从内存缓存读取最近对话（不需要异步）
recent_messages = recall_cache.get_for_prompt()  # ~0ms

# 异步写入事件，不阻塞返回
asyncio.create_task(write_event_async(...))
```

---

## 技术栈

### 后端

**语言与框架**：
- Python 3.11+
- FastAPI（REST API + WebSocket）

**LLM相关**：
- LangChain（Prompt工程 + LLM调用）
- OpenAI API / Claude API（可配置）

**向量检索**：
- Chroma（本地向量数据库）
- OpenAI Embeddings API / 本地Embedding模型（可配置）

**数据存储**：
- 文件系统（JSON/JSONL）
- Chroma内置持久化

### 前端

**框架**：
- Next.js 14 (App Router)
- TypeScript

**UI库**：
- Shadcn/ui
- Tailwind CSS（暗色主题）

**状态管理**：
- Zustand（前端临时状态）
- localStorage持久化

**通信**：
- Socket.IO Client（WebSocket，流式对话）
- Fetch API（REST，故事线/角色/背景管理）

---

## 核心设计原则总结

### ✅ 采用的设计

1. **会话实例架构** - 状态隔离，解决多会话状态污染问题
2. **Core Memory 动态管理** - base_persona + evolved_persona，自然语言描述
3. **事件库过滤** - 必须记录instance_id和session_id
4. **定期维护任务** - 每30轮清理Recall Memory + 更新evolved_persona
5. **性能优化** - 异步并行、超时保护、RecallMemoryCache、~2秒非LLM耗时

### ❌ 删除的设计

1. **量化数值** - 不使用priority、intensity、level等数字
2. **数字降级** - 不使用"每N轮-1"的机制
3. **事件链表** - 不使用previous_event，纯RAG
4. **归档机制** - 不使用archived标记，简化设计
5. **会话摘要** - 不需要session_summary，信任RAG

---

## 术语变更说明（v0.2.1）

**重要**：本文档中大量使用了旧术语"故事线（Storyline）"，在 v0.2.1 版本后统一更新为"会话实例（Conversation Instance）"。

### 术语对照表

| 旧术语 | 新术语 | 说明 |
|--------|--------|------|
| 故事线（Storyline） | 会话实例（Conversation Instance） | 更准确表达其本质：状态隔离容器 |
| `storyline_id` | `instance_id` | 数据库字段名 |
| `storylines/` 目录 | `instances/` 目录 | 文件系统路径 |

**原因**：
- "故事线"容易与"剧情主线"混淆
- "会话实例"更准确描述其作用：同一角色+背景可创建多个独立实例

**文档更新计划**：
- 本文档（DESIGN.md）将在后续版本中全文更新术语
- 当前版本（v0.2.1）在本章节说明即可，避免大规模修改导致混乱

---

## 需求澄清后的新设计（v0.2.1）

### 1. 故事主线大纲的灵活性

**数据结构调整**：
```json
{
  "story_outline": [
    {"index": 1, "content": "发现背叛者的线索", "status": "completed"},
    {"index": 2, "content": "潜入敌人据点", "status": "in_progress"},
    // 数量不固定，由用户创建背景时编写（5-20个）
  ],
  "current_plot_index": 2,
  "outline_completed": false  // 是否已完成全部大纲点
}
```

**设计要点**：
- 大纲点数量**不固定**，由用户自由编写
- 大纲是**建议路线**，不是强制路线
- 完成后进入**自由对话模式**（导演模块不再介入）

---

### 2. Core Memory 手动触发功能

**功能需求**：
- UI 位置：对话界面左侧/右侧功能区（垂直排列）
- 按钮名称："🧠 更新记忆"
- 执行方式：同步执行，显示加载状态（约 6-11 秒）

**API 设计**：
```python
@router.post("/maintenance/core-memory")
async def trigger_core_memory_maintenance(instance_id: str):
    """
    手动触发 Core Memory 维护
    """
    # 1. 加载当前 Core Memory
    state = load_character_state(instance_id)

    # 2. 加载最近 10 轮对话
    recent_turns = load_recent_messages(instance_id, n=10)

    # 3. 调用 LLM 重新生成 Core Memory
    new_core_memory = await llm_regenerate_core_memory(
        old_state=state,
        recent_turns=recent_turns
    )

    # 4. 写入新的 Core Memory
    save_character_state(instance_id, new_core_memory)

    # 5. 可选：异步写入事件库
    asyncio.create_task(write_maintenance_event(...))

    return {"success": True, "message": "Core Memory 已更新"}
```

**后台管理端功能**：
- 查看 Core Memory 维护历史
- 查看每次维护的输入和输出
- 回滚到历史版本

---

### 3. 导演模块的"拉回主线"功能

**功能需求**：
- UI 位置：对话界面左侧/右侧功能区（垂直排列）
- 按钮名称："🎬 拉回主线"
- 实现机制：通过**某种策略**将信息**插入到特定位置**，影响 Prompt 组装

**设计思路**（待细化）：
- 用户点击按钮后，后端触发特殊的 Prompt 注入
- 具体策略和注入位置待后续讨论
- 目标：自然地将偏离的剧情引导回主线

---

### 4. 事件写入失败的暂存机制

**新增目录结构**：
```
data/
├── instances/
│   └── {instance_id}/
│       ├── pending_events/          ← 新增：待写入事件暂存
│       │   ├── pending_001.json
│       │   ├── pending_002.json
│       │   └── ...
│       ├── metadata.json
│       ├── character_state.json
│       └── sessions/
```

**写入流程**：
```python
async def write_event_to_chroma(event_data, instance_id):
    # 1. 先写入暂存文件
    timestamp = now().isoformat()
    pending_file = f"data/instances/{instance_id}/pending_events/pending_{timestamp}.json"
    await write_json(pending_file, event_data)

    try:
        # 2. 尝试写入 Chroma
        await chroma_collection.add(
            embeddings=embed(event_data['content']),
            documents=[event_data['content']],
            metadatas=[event_data['metadata']]
        )

        # 3. 写入成功，删除暂存文件
        await delete_file(pending_file)

    except Exception as e:
        # 4. 写入失败，保留暂存文件
        logger.error(f"Failed to write event to Chroma: {e}")

        # 5. 通知用户（通过 WebSocket）
        await ws.send_json({
            "type": "event_write_failed",
            "message": "事件记录失败，已暂存待处理",
            "pending_file": pending_file
        })
```

**后台管理端 API**：
```python
# 获取所有待处理事件
@router.get("/admin/pending-events")
async def get_pending_events():
    ...

# 重试写入某个事件
@router.post("/admin/pending-events/{file_id}/retry")
async def retry_pending_event(file_id: str):
    ...

# 批量重试某个实例的所有事件
@router.post("/admin/pending-events/retry-all")
async def retry_all_pending_events(instance_id: str):
    ...

# 删除某个待处理事件
@router.delete("/admin/pending-events/{file_id}")
async def delete_pending_event(file_id: str):
    ...
```

---

### 5. UI 布局设计

**三栏布局**（类似 SillyTavern）：

```
┌──────────┬─────────────────────────┬──────────┐
│          │ [顶部功能栏]             │          │
│          │ - 切换会话实例           │          │
│          │ - 全局设置               │          │
│          ├─────────────────────────┤          │
│          │                         │          │
│ 左侧边栏 │                         │ 右侧边栏 │
│ (顶到底) │    对话消息区           │ (顶到底) │
│          │    - AI 回复            │          │
│ 🧠更新记忆│    - 用户消息           │ 📊角色状态│
│          │    - 流式显示           │          │
│ 🎬拉回主线│                         │ 📚历史事件│
│          │                         │          │
│ ⚙️ 设置   │                         │ 📖世界书  │
│          │                         │          │
│ ...     │                         │ ...     │
│          │                         │          │
│          ├─────────────────────────┤          │
│          │ [用户输入框]   [发送]    │          │
└──────────┴─────────────────────────┴──────────┘
```

**关键特性**：
- **三个区域（左、中、右）都是从顶到底**
- 中间区域内部分为三部分：
  - 上：顶部功能栏（高度小）
  - 中：对话消息区（高度大，可滚动）
  - 下：用户输入框（高度小，与顶部功能栏面积相同）
- 中间区域占比约 50%
- 左右侧边栏可伸缩，功能按钮垂直排列

---

### 6. `[PROGRESS]` 标记机制的 Prompt 设计

**System Prompt 示例**（放入头部 4K）：
```
【剧情进度报告规则】
你必须在每次回复的末尾添加剧情进度标记，格式为：[PROGRESS:X:status]

参数说明：
- X: 当前大纲点的索引（1-N）
- status: 进度状态
  - in_progress: 正在进行此大纲点
  - completed: 已完成此大纲点

示例：
用户："我走进了据点"
AI回复："你推开生锈的铁门，据点内一片狼藉..." [PROGRESS:2:in_progress]

当前剧情状态：
{
  "story_outline": [
    {"index": 1, "content": "发现背叛者的线索", "status": "completed"},
    {"index": 2, "content": "潜入敌人据点", "status": "in_progress"}, ← 当前这里
    {"index": 3, "content": "与仇人对峙", "status": "pending"},
    ...
  ]
}

重要：如果你认为大纲点已经完成，立即标记为 completed！
```

**Few-shot Examples**（放入 Prompt）：
```
示例 1：
User: 我在据点外观察
Assistant: 你躲在废墟后，透过破碎的窗户看到里面有3个人影在移动... [PROGRESS:2:in_progress]

示例 2：
User: 我翻墙进入据点
Assistant: 你成功翻过围墙，悄悄潜入据点内部。前方走廊传来脚步声... [PROGRESS:2:completed]

示例 3：
User: 我走向仇人所在的房间
Assistant: 你推开房门，看到那个背叛你的人正坐在沙发上，冷笑着看向你... [PROGRESS:3:in_progress]
```

**后端正则扫描**：
```python
import re

def extract_progress_tag(ai_response: str) -> tuple[str, dict]:
    """
    从 AI 回复中提取 [PROGRESS] 标记
    返回：(清理后的回复, 进度信息)
    """
    pattern = r'\[PROGRESS:(\d+):(in_progress|completed|pending)\]'
    match = re.search(pattern, ai_response)

    if match:
        index = int(match.group(1))
        status = match.group(2)

        # 从回复中删除标记
        cleaned_response = re.sub(pattern, '', ai_response).strip()

        return cleaned_response, {"index": index, "status": status}
    else:
        return ai_response, None
```

---

**文档结束**
