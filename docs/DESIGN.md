# PersonaLab 技术设计文档

**文档版本**: v0.1.0
**创建日期**: 2025-10-13
**最后更新**: 2025-10-13

---

## 目录

1. [系统总目标](#系统总目标)
2. [核心概念](#核心概念)
3. [数据结构设计](#数据结构设计)
4. [文件组织结构](#文件组织结构)
5. [核心交互流程](#核心交互流程)
6. [Prompt工程](#prompt工程)
7. [状态管理机制](#状态管理机制)
8. [事件归档机制](#事件归档机制)
9. [技术栈](#技术栈)
10. [待解决问题](#待解决问题)

---

## 系统总目标

实现一个支持**长篇、高一致性、角色可成长**的互动叙事引擎。

### 核心问题

解决LLM在长对话中的两大问题：
1. **上下文稀释** - 对话越长，早期的重要信息被稀释，LLM逐渐"忘记"
2. **指令遗忘** - 角色人格、行为准则在长对话中逐渐被忽略

### 解决方案

通过**显式状态管理 + RAG增强记忆**：
- 动态人格档案（dynamic_profile.json）- 存储角色当前状态
- 长期记忆 - 事件日志（event_log）- 向量检索历史事件
- 结构化Prompt工程 - 强制LLM始终关注核心状态

---

## 核心概念

### 人格成长的逻辑闭环

```
1. 主人公当前状态（dynamic_profile）
   ↓
2. 事件触发（用户输入/剧情推进）
   ↓
3. 主人公的反应（想法 + 行为）← LLM生成
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

### 两个核心积累

#### 1. 人格库（Character Library）
- 经过N次交互后形成的**丰富人格档案**
- 包含：核心人格特质、经历过的关键事件、形成的认知模式、关系网络、成长轨迹

#### 2. 事件库（Event Library）
- 提炼出的**事件模式**（抽象化，脱离具体角色）
- 跨角色共享，新角色可参考过去积累的经验

---

## 数据结构设计

### 1. 动态人格档案（dynamic_profile.json）

**存储位置**：`data/characters/{character_id}/dynamic_profile.json`

**用途**：存储角色的当前状态，每次AI回复后实时更新

**设计原则**：
- ✅ 定性描述（如 "Rage", "Complicated_Ally"）
- ❌ 不使用量化内容（如 trust_level: 5/10, anger: 9/10）
- ✅ 条目化累加（优先级队列）
- ❌ 不删除条目，只调整优先级

**结构示例**：

```json
{
  "character_id": "alserqi_v1",
  "core_identity": {
    "archetype": "Vengeful Anti-hero",
    "core_goal": "Destroy the 'Core'",
    "personality_traits": ["Cynical", "Pragmatic", "Loyal_to_few"]
  },
  "dynamic_state": {
    "emotions": {
      "items": [
        {
          "content": "Weary",
          "priority": 10,
          "timestamp": "2025-10-13T10:00:00Z"
        },
        {
          "content": "Rage",
          "priority": 8,
          "timestamp": "2025-10-12T15:30:00Z"
        }
      ]
    },
    "physical_condition": {
      "items": [
        {
          "content": "Exhausted, left arm injured",
          "priority": 10,
          "timestamp": "2025-10-13T09:00:00Z"
        }
      ]
    },
    "short_term_goals": {
      "items": [
        {
          "goal": "Find B",
          "reason": "Need his intel",
          "priority": 9,
          "timestamp": "2025-10-13T08:00:00Z"
        }
      ]
    },
    "relationships": {
      "items": [
        {
          "entity": "user",
          "status": "Trusted_Companion",
          "priority": 10,
          "timestamp": "2025-10-13T10:00:00Z"
        },
        {
          "entity": "character_B",
          "status": "Betrayer",
          "priority": 9,
          "timestamp": "2025-10-12T20:00:00Z"
        }
      ]
    },
    "learned_patterns": {
      "items": [
        {
          "pattern": "Trust must be earned through actions, not words",
          "priority": 8,
          "timestamp": "2025-10-12T18:00:00Z"
        }
      ]
    }
  }
}
```

**状态更新操作**：
- `add`：添加新条目
- `update_priority`：调整现有条目的优先级

**Prompt使用策略**：
- 只取每个字段的前N个高优先级条目（如前5个）
- 低优先级条目保留但不进入Prompt

**定时维护任务**：
- 每10轮对话后执行去重
- 合并重复内容（保留最高优先级）
- 降低旧条目的优先级
- 删除过低优先级条目（如 priority < 3）

---

### 2. 长期记忆 - 事件日志（event_log）

**存储位置**：`data/event_library/chroma_db/`（全局，所有角色共享）

**用途**：永久存储提炼出的关键事件，用于RAG检索

**设计原则**：
- 完全与角色脱钩（不记录character_id、session_id）
- 记录抽象化的"事件模式"
- 通过语义检索匹配相似情境

**结构示例**：

```json
{
  "event_id": "evt_uuid",
  "summary": "一个愤世嫉俗的角色，最初不信任陌生人的帮助承诺，但看到真实证据后震惊，关系从陌生人转为潜在盟友",
  "pattern": {
    "precondition": "角色具有不信任特质，面对陌生人的帮助承诺",
    "trigger": "对方出示了无法反驳的证据",
    "reaction": "从轻蔑 → 震惊",
    "impact": "关系质变，从陌生人 → 潜在盟友"
  },
  "impact": {
    "state_changes": [
      {
        "field": "relationships",
        "change": "Stranger → Potential_Ally"
      },
      {
        "field": "emotions",
        "change": "Contempt → Shock"
      }
    ]
  },
  "narrative_fragment": "他看到文件，瞳孔微缩。他的手不自觉地伸向文件，但又强迫自己收回...",
  "timestamp": "2025-10-13T10:30:00Z"
}
```

---

### 3. 角色定义（Character）

**存储位置**：`data/characters/{character_id}/definition.json`

**用途**：角色的静态定义，创建会话时使用

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
      "personality_traits": ["Cynical", "Pragmatic", "Loyal_to_few"]
    },
    "dynamic_state": {
      "emotions": {
        "items": [
          {"content": "Neutral", "priority": 5, "timestamp": "..."}
        ]
      },
      "relationships": {
        "items": []
      },
      "short_term_goals": {
        "items": []
      },
      "learned_patterns": {
        "items": []
      }
    }
  }
}
```

**初始化流程**：
- 用户创建角色时，生成 `definition.json`
- 第一次使用角色时，复制 `initial_profile` 到 `dynamic_profile.json`

---

### 4. 背景定义（Background）

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

### 5. 会话文件（Session）

**存储位置**：`data/sessions/sess_{uuid}.jsonl`

**用途**：记录完整对话历史

**格式**：
- 第一行：metadata（元数据）
- 后续行：对话消息（user/assistant交替）

**示例**：

```jsonl
{"type": "metadata", "character_id": "alserqi_v1", "background_id": "wasteland_world", "title": "与Alserqi的第一次对话", "created_at": "2025-10-13T10:00:00Z"}
{"role": "user", "content": "你这个骗子！", "archived": false, "timestamp": "2025-10-13T10:30:00Z"}
{"role": "assistant", "content": "Alserqi听了你的话，眼中闪过一丝轻蔑...", "archived": false, "timestamp": "2025-10-13T10:30:05Z"}
{"role": "user", "content": "我有证据", "archived": true, "timestamp": "2025-10-13T10:31:00Z"}
{"role": "assistant", "content": "他看到文件，瞳孔微缩...", "archived": true, "timestamp": "2025-10-13T10:31:08Z"}
```

**字段说明**：
- `archived`: false（未归档，进入Prompt）/ true（已归档，已提取到event_log）

**特点**：
- 一个会话一个文件
- 可能包含几十上百条消息
- 永久保存，不删除
- 支持导出功能

---

## 文件组织结构

```
data/
├── characters/                      # 角色库
│   ├── alserqi_v1/
│   │   ├── definition.json         # 角色定义（静态）
│   │   └── dynamic_profile.json    # 人格状态（动态，全局共享）
│   └── character_b/
│       ├── definition.json
│       └── dynamic_profile.json
│
├── backgrounds/                     # 背景库
│   ├── wasteland_world.json
│   └── cyberpunk_city.json
│
├── sessions/                        # 会话文件
│   ├── sess_001.jsonl              # 会话1（可能几十上百条消息）
│   ├── sess_002.jsonl              # 会话2
│   └── sess_003.jsonl
│
├── event_library/                   # 全局事件库
│   └── chroma_db/                  # 向量数据库
│
└── prompt_templates/                # Prompt模板管理
    └── default_v1.json
```

**关键特性**：
- **角色状态全局共享**：同一角色的所有会话共享 `dynamic_profile.json`
- **事件库全局共享**：所有角色、所有会话的事件存入同一个向量库
- **会话文件独立**：一个会话一个 `.jsonl` 文件

---

## 核心交互流程

### 完整流程（需求B：一次完整交互）

```
用户输入："你这个骗子！"
  ↓
Step 1: 追加消息到会话文件
  append_to_file("sess_001.jsonl", user_message)
  ↓
Step 2: 异步RAG检索（并行）
  - 读取会话文件，获取未归档消息
  - 将用户输入向量化
  - 从event_library检索Top-K=3相关事件
  ↓
Step 3: 构建Prompt（主线程等待RAG完成，超时1.5秒）
  - 读取会话metadata，获取character_id, background_id
  - 加载character的dynamic_profile.json
  - 加载background的description
  - 格式化RAG检索结果
  - 读取会话文件的未归档消息（最近20轮）
  - 组装最终Prompt
  ↓
Step 4: LLM API调用（单次请求）
  - 发送Prompt
  ↓
Step 5: 接收LLM输出
  - 返回：<narrative>...</narrative><state_update_json>...</state_update_json>
  ↓
Step 6: 输出解析与验证
  - 提取narrative_text
  - 提取state_update_json
  - 验证格式：如果解析失败 → 抛异常，事务回滚，返回错误给用户
  ↓
Step 7: 状态更新（原子替换）
  - 如果state_update_json不为空：
    * 加载dynamic_profile.json
    * 执行条目化累加（add操作）
    * 原子写回文件
  - 如果为空：不更新
  ↓
Step 8: 追加AI回复到会话文件
  append_to_file("sess_001.jsonl", assistant_message)
  ↓
Step 9: 异步归档（不阻塞，后台任务）
  - 判断最近未归档消息是否构成事件
  - 如果是：提取事件 → 写入event_library → 标记消息为archived
  ↓
Step 10: 返回前端（流式发送）
  - WebSocket分块发送narrative_text（打字机效果）
  - 发送完成信号
```

---

## Prompt工程

### Prompt模板结构（需求A）

**设计原则**：模块化、可配置、优先级分层

```
---SYSTEM_INSTRUCTION (META_LEVEL)---
## System Role ##
你是一个第三人称叙事者，描述主角[角色名]的行为、想法和遭遇。
用户的输入可能是：
1. 对主角说的话（对话）
2. 对剧情的推动（例如："这时，角色B出现了"）
3. 对世界的描述（例如："突然下起了暴雨"）
你需要根据主角的当前状态和人格特质，生成主角的反应。
---

---JAILBREAK_PREVENTION (META_LEVEL)---
## Critical Rules ##
你必须严格保持角色人设，不得脱离角色设定。
---

---BACKGROUND_CONTEXT (SYSTEM_LEVEL)---
## World Setting ##
{{background_description}}
{{world_rules}}
---

---IDENTITY_AND_STATE_BLOCK (HIGHEST_PRIORITY)---
## Core Identity & Current State ##
{{dynamic_profile_json_content}}
---

---LONG_TERM_MEMORY_BLOCK (RETRIEVED_VIA_RAG)---
## Relevant Past Events ##
{{rag_retrieved_events_formatted_text}}
---

---RECENT_CHAT_HISTORY---
## Conversation So Far ##
{{last_N_unarchived_messages}}
---

---USER_INPUT---
## User's Latest Action/Dialogue ##
{{user_input_text}}
---

---TASK_INSTRUCTIONS (CRITICAL)---
You MUST generate your response in two parts, enclosed in the specified XML tags.

**PART 1: THE NARRATIVE (<narrative>)**
Based on all the information above, generate the character's next action and dialogue.
The narrative must reflect the character's current state and be a logical continuation of the events.

**PART 2: THE STATE UPDATE (<state_update_json>)**
After the narrative, provide a JSON object describing ALL changes to the character's state that occurred within your narrative.
The JSON object must ONLY contain the fields that have changed.

Format:
{
  "dynamic_state": {
    "<field_name>": {
      "add": [
        {"content": "...", "priority": N}
      ],
      "update_priority": [
        {"content": "...", "new_priority": N}
      ]
    }
  }
}

If no state changes occurred (the character's reaction is a natural expression of existing state), return an empty object: {}
---

{{AI_GENERATION_STARTS_HERE}}
```

### Prompt模板管理

**存储位置**：`data/prompt_templates/default_v1.json`

**结构**：

```json
{
  "template_id": "default_v1",
  "modules": [
    {
      "name": "system_instruction",
      "priority": 1,
      "enabled": true,
      "content": "你是一个第三人称叙事者..."
    },
    {
      "name": "jailbreak_prevention",
      "priority": 2,
      "enabled": true,
      "content": "你必须严格保持角色人设..."
    },
    {
      "name": "background",
      "priority": 3,
      "enabled": true,
      "content_source": "dynamic",
      "template": "## World Setting ##\n{{background_description}}"
    },
    {
      "name": "character_state",
      "priority": 4,
      "enabled": true,
      "content_source": "dynamic",
      "template": "## Current State ##\n{{dynamic_profile_json}}"
    },
    {
      "name": "memory",
      "priority": 5,
      "enabled": true,
      "content_source": "rag",
      "template": "## Relevant Past Events ##\n{{rag_events}}"
    },
    {
      "name": "chat_history",
      "priority": 6,
      "enabled": true,
      "content_source": "messages",
      "template": "## Recent Conversation ##\n{{unarchived_messages}}"
    }
  ]
}
```

**前端管理功能**：
- `/prompt-templates` 页面
- 增删改查模块
- 拖拽排序
- 开关模块
- 实时预览

---

## 状态管理机制

### 状态更新触发条件

**只在发生"意外"时更新状态**

**意外的定义**：
- 主观预期（基于当前人格特质的推断）
- 客观结果（实际发生的事情）
- 两者产生差异

**示例A：有影响**
```
当前状态：Alserqi不信任陌生人（Cynical特质）
事件：陌生人声称能帮忙 → Alserqi轻蔑（符合预期，无需更新）
事件：陌生人拿出真实证据 → Alserqi震惊（意外！）
状态更新：relationships.user: Stranger → Potential_Ally
```

**示例B：无影响**
```
当前状态：Alserqi不信任陌生人
事件：陌生人声称能帮忙 → Alserqi轻蔑
事件：陌生人无法证明 → Alserqi离开（符合预期）
状态更新：无（state_update_json为空）
```

### 状态更新操作

**LLM输出的state_update_json格式**：

```json
{
  "dynamic_state": {
    "emotions": {
      "add": [
        {"content": "Shock", "priority": 9}
      ]
    },
    "relationships": {
      "add": [
        {"entity": "user", "status": "Potential_Ally", "priority": 7}
      ],
      "update_priority": [
        {"entity": "user", "status": "Stranger", "new_priority": 2}
      ]
    }
  }
}
```

**系统处理逻辑**：

```python
def update_profile(profile_path, state_update):
    profile = load_json(profile_path)

    for field_name, operations in state_update["dynamic_state"].items():
        # 处理add操作
        if "add" in operations:
            for item in operations["add"]:
                item["timestamp"] = now()
                profile["dynamic_state"][field_name]["items"].append(item)

        # 处理update_priority操作
        if "update_priority" in operations:
            for update in operations["update_priority"]:
                # 查找匹配的item，更新priority
                pass

        # 按priority降序排序
        profile["dynamic_state"][field_name]["items"].sort(
            key=lambda x: x["priority"],
            reverse=True
        )

    # 原子写回
    save_json(profile_path, profile)
```

### 定时维护任务

**触发条件**：每10轮对话

**操作**：
1. **去重**：合并相同内容的条目，保留最高优先级
2. **降级**：超过N天的条目，priority -= 1
3. **清理**：删除 priority < 3 的条目

---

## 事件归档机制

### 归档触发逻辑

**后台异步任务**，每次交互后执行

```python
async def archive_if_needed(session_id):
    # 读取最近未归档的消息
    unarchived = read_unarchived_messages(session_id)

    # 判断条件
    if state_update_was_applied:
        # 有状态变化，必定归档
        event = extract_event_with_impact(unarchived, state_update)
        save_to_event_library(event)
        mark_as_archived(session_id, unarchived)

    elif len(unarchived) >= 10:  # 5轮对话 = 10条消息
        # 无状态变化，但对话量足够
        event = extract_event_without_impact(unarchived)
        save_to_event_library(event)
        mark_as_archived(session_id, unarchived)

    else:
        # 继续累积
        pass
```

### 归档后的处理

**对话文件**：
- 标记消息为 `"archived": true`
- 消息不删除，仍保留在文件中
- 用于前端显示、导出功能

**Prompt构建**：
- 读取会话文件时，只取 `"archived": false` 的消息
- 已归档的消息不再进入Prompt的"Recent Conversation"部分
- 通过RAG检索，相关事件可能被检索回来（但是事件摘要，不是原文）

**数据流示意**：
```
对话历史（未归档）→ Prompt的"Recent Conversation"
事件日志（已归档）→ Prompt的"Relevant Past Events"（RAG检索）
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
- Chroma内置SQLite

### 前端

**框架**：
- Next.js 14 (App Router)
- TypeScript

**UI库**：
- Shadcn/ui
- Tailwind CSS（暗色主题）

**状态管理**：
- Zustand（前端临时状态）
- 不涉及后端核心逻辑

**通信**：
- Socket.IO Client（WebSocket，流式对话）
- Fetch API（REST，会话/角色/背景管理）

---

## 待解决问题

### 1. 会话文件的metadata字段

**问题**：会话文件第一行的metadata具体包含哪些字段？

**目前假设**：
```json
{
  "type": "metadata",
  "character_id": "...",
  "background_id": "...",
  "title": "...",
  "created_at": "..."
}
```

**待确认**：
- 是否需要 `title` 字段（用户自定义会话标题）？
- 是否需要其他字段？

---

### 2. 角色状态的初始化时机

**问题**：`dynamic_profile.json` 何时创建？

**可能性A**：创建角色时
- 用户在角色管理页创建角色
- 立即生成 `definition.json` 和 `dynamic_profile.json`

**可能性B**：第一次使用时
- 用户创建角色时，只生成 `definition.json`
- 第一次创建会话时，复制 `initial_profile` 到 `dynamic_profile.json`

**待确认**：采用哪种方式？

---

### 3. 事件归档是否记录来源会话

**问题**：event_log中的事件，是否需要记录 `source_session_id`？

**不记录的理由**：
- 事件完全脱离具体角色和会话，更抽象
- RAG检索只关心语义相似度

**记录的理由**：
- 调试时可追溯
- 未来可能需要"删除某个会话的所有事件"

**待确认**：是否记录？

---

### 4. 多个会话共享状态的并发问题

**问题**：
- 用户同时打开两个会话（都使用角色A）
- 在会话1对话，更新了 `dynamic_profile.json`
- 切换到会话2，状态已经变了
- 这是预期行为吗？

**可能的解决方案**：
- **方案A**：接受这种行为（状态是全局的，本就应该共享）
- **方案B**：每个会话独立复制状态（但失去了"人格成长"的连续性）
- **方案C**：前端提示用户"角色状态已更新"

**待确认**：采用哪种方案？

---

### 5. Prompt模板的前端管理功能优先级

**问题**：Prompt模板管理是V1.0必需功能，还是后期优化？

**V1.0简化方案**：
- 使用固定的默认模板
- 不提供前端管理页面

**完整方案**：
- 实现 `/prompt-templates` 管理页面
- 支持模块的增删改查、排序、开关

**待确认**：V1.0包含哪些功能？

---

### 6. 定时维护任务的触发机制

**问题**：dynamic_profile的去重/降级/清理任务，如何触发？

**可能方案**：
- **方案A**：每10轮对话后自动触发
- **方案B**：后台定时任务（如每天凌晨）
- **方案C**：手动触发（前端按钮）

**待确认**：采用哪种方案？

---

### 7. 角色数量与事件库规模

**问题**：随着使用时间增长，事件库会不断膨胀，是否需要限制？

**可能方案**：
- **方案A**：不限制，依赖向量数据库的性能
- **方案B**：设置事件数量上限（如1万条），超过后删除旧事件
- **方案C**：提供"清理事件库"功能

**待确认**：采用哪种方案？

---

## 附录

### 需求来源

本文档基于以下需求：
- 需求A：PersonaLab V1.0 核心需求与设计规格
- 需求B：一次完整交互的操作序列

### 修正记录

1. **删除所有量化内容**
   - 原需求A中包含 `metrics: [{"name": "anger", "value": 9}]`
   - 已修正为纯定性描述

2. **状态管理改为条目化累加**
   - 原需求A中是字段级替换
   - 已修正为优先级队列模式

3. **事件库完全脱离角色**
   - 原需求B中有 `character_id`, `session_id` 字段
   - 已删除，改为抽象化的事件模式

4. **会话文件结构明确**
   - 一个会话一个文件（.jsonl）
   - 第一行存metadata
   - 后续行存对话消息

---

**文档结束**
