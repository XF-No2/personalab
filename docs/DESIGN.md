# PersonaLab 技术设计文档

**文档版本**: v0.2.0
**创建日期**: 2025-10-13
**最后更新**: 2025-10-14

---

## 目录

1. [系统总目标](#系统总目标)
2. [核心概念](#核心概念)
3. [架构设计](#架构设计)
4. [数据结构设计](#数据结构设计)
5. [人格三层机制](#人格三层机制)
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

通过**故事线隔离 + 三层人格 + RAG增强记忆**：
- 故事线（Storyline）- 独立的故事世界，状态隔离
- 三层人格机制 - 静态核心 + 慢动态成长 + 快动态状态
- 事件库（Event Library）- 向量检索历史事件
- 结构化Prompt工程 - 强制LLM始终关注核心状态

---

## 核心概念

### 故事线（Storyline）

**定义**：一条独立的、连续的故事世界。

**特点**：
- 用户创建故事线时，选择角色和背景
- 故事线内可以有多个会话（暂停/继续）
- 角色状态属于故事线，不属于全局
- 不同故事线完全隔离，互不影响

**类比**：
```
角色：Alserqi

故事线A："盟友之路"
  - 会话1：初次见面
  - 会话2：建立信任（继续故事线A）
  - 会话3：共同战斗（继续故事线A）
  → 角色状态：与用户是盟友

故事线B："敌对之路"（独立故事）
  - 会话4：敌对相遇
  - 会话5：冲突升级（继续故事线B）
  → 角色状态：与用户是敌人

两条故事线的角色状态完全独立，互不干扰
```

### 人格成长的逻辑闭环

```
1. 角色当前状态（三层人格）
   ↓
2. 事件触发（用户输入/剧情推进）
   ↓
3. 角色的反应（想法 + 行为）← LLM生成
   ↓
4. 客观结果（事实）
   ↓
5. 主观预期 vs 客观结果 → 产生差异
   ↓
6. 差异触发状态变化（情绪、认知、关系改变）
   ↓
7. 新状态 → 影响下一次事件的反应方式
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
├── storylines/                          # 故事线（核心）
│   └── {storyline_id}/
│       ├── metadata.json                # 故事线元数据
│       ├── character_state.json         # 故事线的角色状态
│       └── sessions/
│           ├── sess_001.jsonl           # 会话1
│           ├── sess_002.jsonl           # 会话2（继续故事）
│           └── sess_003.jsonl           # 会话3（继续故事）
│
├── event_library/                       # 全局事件库
│   └── chroma_db/                       # 向量数据库
│       # 存储所有事件，带storyline_id和session_id过滤标记
│
└── prompt_templates/                    # Prompt模板
    └── default_v1.json
```

### ID关联关系

```
角色定义（全局唯一）
  └── character_id: "alserqi_v1"

故事线（独立世界）
  └── storyline_id: "story_001"
      ├── character_id: "alserqi_v1"     ← 引用角色
      ├── background_id: "wasteland"     ← 引用背景
      ├── character_state.json           ← 状态属于故事线
      └── sessions/
          ├── sess_001.jsonl
          │   └── session_id: "sess_001"
          └── sess_002.jsonl
              └── session_id: "sess_002"

事件库（全局，但带标记）
  └── event_id: "evt_story001_sess001_030"
      ├── storyline_id: "story_001"      ← 必须！用于过滤
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
  "initial_profile": {
    "core_identity": {
      "archetype": "Vengeful Anti-hero",
      "core_goal": "Destroy the 'Core'",
      "core_traits": ["Cynical", "Pragmatic", "Loyal_to_few"],
      "background_story": "曾是组织的精英战士，因发现真相被背叛"
    },
    "growth_state": {
      "beliefs": [],
      "behavioral_patterns": [],
      "relationships": []
    },
    "current_state": {
      "emotions": ["Neutral"],
      "physical": "健康",
      "immediate_goals": []
    }
  }
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

### 3. 故事线元数据（Storyline Metadata）

**存储位置**：`storylines/{storyline_id}/metadata.json`

**用途**：记录故事线的基本信息

**结构示例**：

```json
{
  "storyline_id": "story_001",
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

**存储位置**：`storylines/{storyline_id}/character_state.json`

**用途**：存储角色在当前故事线的状态

**结构示例**（详见"人格三层机制"章节）：

```json
{
  "storyline_id": "story_001",
  "character_id": "alserqi_v1",
  "last_updated_turn": 250,
  "last_maintenance_turn": 240,

  "core_identity": {
    "archetype": "Vengeful Anti-hero",
    "core_goal": "Destroy the 'Core'",
    "core_traits": ["Cynical", "Pragmatic", "Loyal_to_few"],
    "background_story": "曾是组织的精英战士，因发现真相被背叛"
  },

  "growth_state": {
    "beliefs": [
      {
        "content": "Trust must be earned through actions, not words",
        "formed_from": "多次被背叛的经历",
        "timestamp": "2025-10-12T18:00:00Z"
      }
    ],
    "behavioral_patterns": [
      {
        "pattern": "Always verify information before acting",
        "timestamp": "2025-10-12T19:00:00Z"
      }
    ],
    "relationships": [
      {
        "entity": "user",
        "status": "Trusted_Companion",
        "history": "经历了考验、背叛、解释、原谅的完整循环",
        "timestamp": "2025-10-13T10:00:00Z"
      }
    ]
  },

  "current_state": {
    "emotions": [
      {
        "content": "Weary",
        "context": "长时间的战斗和逃亡",
        "timestamp": "2025-10-13T10:00:00Z"
      },
      {
        "content": "Cautious_Hope",
        "context": "与user的合作进展顺利",
        "timestamp": "2025-10-13T09:00:00Z"
      }
    ],
    "physical": {
      "condition": "Exhausted, left arm injured",
      "timestamp": "2025-10-13T09:00:00Z"
    },
    "immediate_goals": [
      {
        "goal": "Find safe shelter",
        "reason": "Need to rest and treat wounds",
        "timestamp": "2025-10-13T10:00:00Z"
      },
      {
        "goal": "Contact B for intel",
        "reason": "Need information about Core's movements",
        "timestamp": "2025-10-13T09:30:00Z"
      }
    ]
  }
}
```

---

### 5. 会话消息（Session Messages）

**存储位置**：`storylines/{storyline_id}/sessions/{session_id}.jsonl`

**用途**：记录完整对话历史

**格式**：
- 第一行：metadata（元数据）
- 后续行：对话消息（user/assistant交替）

**示例**：

```jsonl
{"type": "metadata", "session_id": "sess_001", "storyline_id": "story_001", "started_at": "2025-10-01T10:00:00Z"}
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

**用途**：存储重要事件，用于RAG检索

**结构示例**：

```json
{
  "event_id": "evt_story001_sess001_030",
  "storyline_id": "story_001",
  "session_id": "sess_001",
  "turn": 30,
  "summary": "用户展示证据，Alserqi从怀疑转为信任",
  "full_context": "用户：'我有证据' → Alserqi看到文件，瞳孔微缩... → 关系改变",
  "state_changes": {
    "relationships": "Stranger → Potential_Ally"
  },
  "timestamp": "2025-10-01T10:30:00Z",
  "embedding": [0.1, 0.2, 0.3, ...]
}
```

**关键字段**：
- `storyline_id` - **必须**，用于RAG检索时过滤，避免不同故事线的事件混淆
- `session_id` - **必须**，用于追溯和调试
- `turn` - 会话内的第几轮，用于精确定位
- 不需要 `previous_event`（不使用链表结构，纯RAG）

---

## 人格三层机制

### 核心原理

**三层的本质**：不是"分几个字段"，而是**数据的变化速度不同**。

```
第1层：静态（永不变化）
  ↓
第2层：慢动态（重大事件才变化）
  ↓
第3层：快动态（频繁变化）
```

### 实现方式

```json
{
  // 第1层：静态核心（永不变化）
  "core_identity": {
    "archetype": "Vengeful Anti-hero",
    "core_goal": "Destroy the 'Core'",
    "core_traits": ["Cynical", "Pragmatic", "Loyal_to_few"],
    "background_story": "..."
  },

  // 第2层：慢动态（成长层）
  "growth_state": {
    "beliefs": [...],           // 形成的深层信念
    "behavioral_patterns": [...],// 行为模式
    "relationships": [...]       // 关系深度
  },

  // 第3层：快动态（状态层）
  "current_state": {
    "emotions": [...],           // 当前情绪
    "physical": {...},           // 身体状况
    "immediate_goals": [...]     // 短期目标
  }
}
```

### 设计原则

**❌ 绝对禁止量化**：
- 不使用 `priority`, `intensity`, `level` 等数字
- 不使用 `每N轮-1` 的降级机制
- 不使用数字阈值判断删除

**✅ 纯定性描述**：
- 所有内容都是文本描述
- 用 `timestamp` 记录时间
- 用定期维护任务清理过时数据

### 变化机制

```python
# 第1层：永不变化
# core_identity 从 definition.json 复制后，不再修改

# 第2层：重大事件才变化
if is_major_event(state_update):  # 如：关系质变、核心信念动摇
    growth_state["beliefs"].append({
        "content": "新信念",
        "formed_from": "事件描述",
        "timestamp": now()
    })

# 第3层：每次交互可能变化
current_state["emotions"].append({
    "content": "Betrayed",
    "context": "用户承认一直在欺骗",
    "timestamp": now()
})
```

### 定期维护任务

**触发条件**：每10轮对话执行一次

**操作内容**：

```python
def cleanup_character_state(state, current_turn):
    """
    定期维护：清理、去重、整理动态状态
    """
    # 1. 快动态层：清理过多数据
    # emotions只保留最新的5个
    if len(state["current_state"]["emotions"]) > 5:
        state["current_state"]["emotions"] = \
            state["current_state"]["emotions"][-5:]

    # immediate_goals只保留最新的3个
    if len(state["current_state"]["immediate_goals"]) > 3:
        state["current_state"]["immediate_goals"] = \
            state["current_state"]["immediate_goals"][-3:]

    # 2. 慢动态层：去重、合并
    # 去重beliefs（相同内容合并）
    beliefs_dict = {}
    for belief in state["growth_state"]["beliefs"]:
        content = belief["content"]
        if content not in beliefs_dict:
            beliefs_dict[content] = belief
        else:
            # 保留最新的timestamp
            if belief["timestamp"] > beliefs_dict[content]["timestamp"]:
                beliefs_dict[content] = belief
    state["growth_state"]["beliefs"] = list(beliefs_dict.values())

    # 3. 合并同一实体的关系（保留最新）
    relationships_dict = {}
    for rel in state["growth_state"]["relationships"]:
        entity = rel["entity"]
        if entity not in relationships_dict:
            relationships_dict[entity] = rel
        else:
            if rel["timestamp"] > relationships_dict[entity]["timestamp"]:
                relationships_dict[entity] = rel
    state["growth_state"]["relationships"] = list(relationships_dict.values())

    # 4. 更新维护轮次
    state["last_maintenance_turn"] = current_turn
```

**关键**：
- 不是"衰减"，是"清理过多累积"
- 不使用数字降级，而是控制数量上限
- 慢动态层去重、合并
- 快动态层只保留最新N个

---

## 核心交互流程

### 一次完整交互（性能优化版）

**性能目标**：
- 总耗时：< 30秒
- 非LLM耗时：< 12秒（实际约2秒）
- LLM耗时：15-20秒

**流程**：

```
用户输入："你还记得我说的话吗？"
  ↓
[1] 追加消息到会话文件 (~50ms)
  append_to_file("storylines/story_001/sessions/sess_002.jsonl", user_message)
  ↓
[2] 并行执行（总耗时取最慢的）
  await asyncio.gather(
    read_recent_messages(storyline_id, n=20),         # ~100ms
    rag_query_with_timeout(user_input, storyline_id), # ~500-1500ms
    load_character_state(storyline_id),               # ~50ms
    load_background(background_id)                    # ~50ms
  )
  并行耗时：max(100, 1500, 50, 50) = ~1500ms
  ↓
[3] 组装Prompt (~100ms)
  prompt = build_prompt(state, background, rag_events, recent_msgs, user_input)
  ↓
[4] LLM API调用 (~15-20秒，不计入12秒预算)
  response = await llm.generate(prompt)
  ↓
[5] 解析LLM输出 (~50ms)
  narrative, state_update = parse_response(response)
  ↓
[6] 更新状态 (~100ms)
  if state_update:
      apply_state_update("storylines/story_001/character_state.json", state_update)
  ↓
[7] 检查是否需要定期维护 (~50ms)
  if current_turn % 10 == 0:
      cleanup_character_state(state, current_turn)
  ↓
[8] 追加AI回复到会话文件 (~50ms)
  append_to_file("sess_002.jsonl", assistant_message)
  ↓
[9] 异步写入事件（不阻塞返回）
  if state_update:
      asyncio.create_task(write_event_to_chroma_async(...))
  ↓
[10] 返回前端（流式发送）
  websocket.send_stream(narrative_text)

总计（非LLM）：~1.9秒 ✅
```

---

## Prompt工程

### Prompt模板结构

**设计原则**：模块化、清晰、无量化

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

---CHARACTER_CORE---
## Core Identity (Immutable) ##
Archetype: {{archetype}}
Core Goal: {{core_goal}}
Core Traits: {{core_traits}}
Background Story: {{background_story}}
---

---CHARACTER_GROWTH---
## Growth & Deep Patterns ##
Beliefs:
{{beliefs_list}}

Behavioral Patterns:
{{patterns_list}}

Relationships:
{{relationships_list}}
---

---CHARACTER_CURRENT_STATE---
## Current State ##
Emotions: {{emotions_list}}
Physical Condition: {{physical_condition}}
Immediate Goals: {{immediate_goals_list}}
---

---RELEVANT_PAST_EVENTS---
## Relevant Past Events (RAG Retrieved) ##
{{rag_events_formatted}}
---

---RECENT_CONVERSATION---
## Recent Conversation (Last 20 turns) ##
{{recent_messages}}
---

---USER_INPUT---
## User's Latest Input ##
{{user_input}}
---

---TASK_INSTRUCTIONS---
You MUST generate your response in two parts, enclosed in XML tags:

**PART 1: THE NARRATIVE (<narrative>)**
Generate the character's next action, dialogue, and inner thoughts.
The narrative must reflect the character's current state.

**PART 2: THE STATE UPDATE (<state_update_json>)**
Provide a JSON object describing changes to the character's state.
Only include fields that have changed.

Format:
{
  "growth_state": {
    "beliefs": {
      "add": [{"content": "...", "formed_from": "..."}]
    },
    "relationships": {
      "update": [{"entity": "...", "status": "...", "history": "..."}]
    }
  },
  "current_state": {
    "emotions": {
      "add": [{"content": "...", "context": "..."}]
    },
    "physical": {
      "condition": "..."
    }
  }
}

If no state changes occurred, return an empty object: {}
---

{{AI_GENERATION_STARTS_HERE}}
```

---

## 事件库与RAG

### 事件写入时机

**只在状态变化时写入事件**：

```python
async def handle_llm_response(storyline_id, session_id, turn, state_update):
    if state_update:
        # 有状态变化，说明发生重要事件
        event = {
            "event_id": f"evt_{storyline_id}_{session_id}_{turn}",
            "storyline_id": storyline_id,
            "session_id": session_id,
            "turn": turn,
            "summary": extract_summary(recent_messages),
            "state_changes": state_update,
            "timestamp": now()
        }

        # 异步写入，不阻塞返回
        asyncio.create_task(write_event_to_chroma(event))
```

### RAG检索策略

**关键**：必须过滤 `storyline_id`，避免不同故事线的事件混淆

```python
async def rag_query_with_timeout(user_input, storyline_id, timeout=1.5):
    """
    RAG检索，带超时保护
    """
    try:
        events = await asyncio.wait_for(
            chroma.query(
                query_text=user_input,
                filter={"storyline_id": storyline_id},  # ← 必须过滤
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
| RAG检索慢 | 事件库规模大 | 1. 过滤storyline_id<br>2. 超时保护(1.5s)<br>3. 限制事件库规模(每故事线<1000) | ~500-1500ms |
| 事件写入阻塞 | Embedding API调用 | 异步写入，不等待完成 | ~0ms（不阻塞） |
| 多次文件读取 | 串行加载 | 并行加载（asyncio.gather） | 取最慢项 |

### 异步优化

```python
# 并行执行多个IO操作
recent, events, state, bg = await asyncio.gather(
    read_recent_messages(storyline_id, n=20),
    rag_query_with_timeout(user_input, storyline_id),
    load_character_state(storyline_id),
    load_background(background_id)
)

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

1. **故事线架构** - 状态隔离，解决多会话状态污染问题
2. **三层人格机制** - 静态 + 慢动态 + 快动态，纯定性描述
3. **事件库过滤** - 必须记录storyline_id和session_id
4. **定期维护任务** - 每10轮清理、去重、整理状态
5. **性能优化** - 异步并行、超时保护、~2秒非LLM耗时

### ❌ 删除的设计

1. **量化数值** - 不使用priority、intensity、level等数字
2. **数字降级** - 不使用"每N轮-1"的机制
3. **事件链表** - 不使用previous_event，纯RAG
4. **归档机制** - 不使用archived标记，简化设计
5. **会话摘要** - 不需要session_summary，信任RAG

---

**文档结束**
