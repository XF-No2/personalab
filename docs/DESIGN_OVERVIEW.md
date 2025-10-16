# PersonaLab 概要设计文档

**文档类型**: 概要设计（Architecture Overview）
**版本**: v0.3.5
**最后更新**: 2025-10-16
**状态**: 🟡 设计中

> 📌 **重要**: 这是概要设计文档，只描述架构框架、核心流程和设计原则。
> 不包含具体实现细节、代码示例和性能数值。

---

## 1. 项目概述

PersonaLab（人格实验室）是一个 **AI 角色对话平台**，支持长篇、高一致性、角色可成长的互动叙事。

### 核心目标

1. **支持长对话** - 对话可持续数百轮，通过汇总功能延续
2. **角色成长** - 角色人格随对话经历动态演化
3. **剧情控制** - 导演模块确保 AI 遵循主线大纲
4. **用户控制** - 所有维护操作（汇总、更新记忆）由用户主动触发

### 技术选型

**前端**: Next.js 14 + TypeScript + Shadcn/ui + Socket.IO
**后端**: Python 3.11+ + FastAPI + LangChain + Chroma

---

## 2. 核心架构

### 2.1 数据架构（三层）

```
PersonaLab 数据架构
├─ 角色状态（Character State）
│   - 存储：character_state.json
│   - 内容：base_persona（初始人格）+ evolved_persona（成长人格）
│   - 更新：用户手动触发（"🧠 更新记忆"按钮）
│
├─ 事件库（Event Library）
│   - 存储：Chroma 向量数据库（分两个 Collection）
│   - 内容：分层摘要（Hierarchical Summary）的情节数据
│       ├─ Summaries Collection：情节点摘要
│       └─ Plots Collection：详细剧情素材
│   - 作用：RAG 检索历史事件（支持深度搜索）
│   - 写入：汇总时生成并写入
│
└─ 会话文件（Conversation File）
    - 存储：.jsonl 文件（当前会话）
    - 内容：用户和 AI 的完整对话记录
    - 读取：每次全量读取所有对话
    - 汇总：汇总后关闭旧会话，开启新会话
```

### 2.2 会话实例架构

**核心概念**：会话实例是状态隔离容器

```
角色定义（全局）
  └─ 会话实例 1
      ├─ 角色状态（独立）
      └─ 多个会话
          ├─ session_001（已汇总）
          ├─ session_002（已汇总）
          └─ session_003（当前活跃）
```

**关键点**：
- 同一角色可以创建多个会话实例（不同剧情线）
- 每个会话实例的角色状态完全独立
- 一个会话实例内可以有多个会话（通过汇总功能延续）

---

## 3. 核心流程

### 3.1 对话交互流程

```
用户输入
  ↓
[1] 追加到会话文件
  ↓
[2] 并行加载数据
  - RAG 检索已汇总的历史会话事件（Chroma，只查当前 instance_id）
  - 加载角色状态（base_persona + evolved_persona）
  - 加载背景设定
  - 全量读取当前会话对话历史（.jsonl 文件）
  ↓
[3] 组装 Prompt
  - 角色人格
  - 背景设定
  - RAG 检索到的历史事件
  - 当前会话的全部对话
  - 用户输入
  ↓
[4] 调用 LLM（流式输出）
  ↓
[5] 追加 AI 回复到会话文件
  ↓
[6] 返回前端
```

**设计原则**：
- ✅ 单一数据源（只维护 .jsonl 文件）
- ✅ 全量读取对话历史（不限制数量）
- ✅ 无自动维护任务（不自动更新人格、不自动汇总）

---

### 3.2 汇总功能流程（分层摘要）

**触发**: 用户点击"📝 汇总对话"按钮

```
汇总功能流程
  ↓
[1] 全量读取当前会话所有对话
  ↓
[2] 调用汇总 Agent（一次 LLM 调用，生成分层摘要）
  Prompt 要求:
    "请把以下对话按照情节点进行汇总。
     每个情节点包含：
     1. summary（摘要）：一句话概括情节点
     2. details（详细素材）：具体的剧情过程

     输出 JSON 格式：
     [
       {
         "summary": "情节点摘要",
         "details": "详细剧情素材"
       }
     ]"

  LLM 返回示例:
    [
      {
        "summary": "情报交换与信任建立",
        "details": "用户告知 Alserqi 北区据点有背叛者的情报。Alserqi 一开始很戒备..."
      },
      {
        "summary": "制定复仇计划",
        "details": "Alserqi 决定前往调查。他制定了详细的潜入计划..."
      }
    ]
  ↓
[3] 解析 JSON（严格模式）
  - 解析成功 → 继续
  - 解析失败 → 报错，提示用户重试
  - 不使用降级方案（避免产生错误数据）
  ↓
[4] 写入 Chroma（分层摘要）
  for each 情节点:
    ├─ Summaries Collection
    │   - ID: summary_{session_id}_{index}
    │   - 内容: summary（摘要）
    │   - 元数据: related_plot_id, session_id, instance_id
    │
    └─ Plots Collection
        - ID: plot_{session_id}_{index}
        - 内容: details（详细素材）
        - 元数据: related_summary_id, session_id, instance_id
  ↓
[5] 创建新会话
  - 初始化内容（可配置顺序）:
      当前会话的所有 Summaries + 最后5轮对话
  - 写入新会话文件：
      {"type":"summary","content":"摘要文本1"}
      {"type":"summary","content":"摘要文本2"}
      ...（当前会话生成的所有摘要）
      ...（最后5轮的 user/assistant 消息）
  ↓
[6] 返回前端，自动切换到新会话
```

**关键设计**：
- **分层摘要**：一次 LLM 调用生成两层数据（Summaries + Plots）
- **严格解析**：JSON 格式输出，解析失败直接报错，由用户决定重试
- **分层存储**：摘要和素材分别存入 Chroma 的不同 Collection，通过 ID 关联
- **剧情延续**：新会话包含摘要（纯文本）+ 最后5轮对话，保持剧情连贯性
- **可复用素材**：Plots 保存在 Chroma，可被导演模块、RAG 检索等功能复用

---

### 3.3 手动更新记忆流程

**触发**: 用户点击"🧠 更新记忆"按钮

```
更新记忆流程
  ↓
[1] 加载当前角色状态
  ↓
[2] 全量读取当前会话对话
  ↓
[3] 调用 LLM 更新 evolved_persona
  - 输入: base_persona + old_evolved_persona + 对话历史
  - 输出: new_evolved_persona
  ↓
[4] 保存新的 evolved_persona 到 character_state.json
  ↓
[5] 返回前端
```

**与汇总功能的区别**：
- **汇总功能**: 总结对话内容，生成分层摘要写入 Chroma，创建新会话延续剧情
- **更新记忆**: 更新角色人格状态（evolved_persona），只修改 character_state.json

---

### 3.4 会话编辑功能（Prompt 编辑器）

**核心理念**：
- 对话界面本质上是一个 **Prompt 编辑器**
- 操作单位是"一条消息"（user 或 assistant）
- 用户可以随意编辑对话历史，然后继续对话

**支持的编辑操作**：
1. **删除消息** - 删除任意一条 user 或 assistant 消息
2. **编辑消息** - 修改任意一条消息的内容
3. **重新生成** - 编辑 user 消息后，删除后续内容，重新生成 AI 回复

**编辑流程示例**：
```
原始对话：
  1. User: 你好
  2. Assistant: 你好，有什么事吗？
  3. User: 今天天气怎么样？
  4. Assistant: 今天天气很好。
  5. User: 那我们出去玩吧
  6. Assistant: 好的，去哪里玩？

用户操作：
  - 删除消息 3 和 4（关于天气的讨论）
  - 编辑消息 5 为"我们讨论一下项目吧"
  - 重新生成

结果：
  1. User: 你好
  2. Assistant: 你好，有什么事吗？
  5. User: 我们讨论一下项目吧
  → AI 基于 1、2、5 生成新回复
```

**后端实现**：
```
用户编辑操作
  ↓
[1] 前端发送编辑后的消息列表
  ↓
[2] 后端重写会话文件（覆盖旧内容）
  ↓
[3] 读取新的会话内容
  ↓
[4] 基于现有消息生成 AI 回复
  ↓
[5] 追加新回复到会话文件
```

**设计原则**：
- 无任何限制：用户可以编辑任意会话文件的任意消息
- 单一数据源：只需修改 .jsonl 文件
- 简单直接：不需要状态检查、不需要缓存同步

---

### 3.5 Prompt组装策略

> **需求追溯**：应对 [REQUIREMENTS.md](REQUIREMENTS.md) 中"LLM注意力机制的U型特点"约束

基于LLM注意力机制的U型特点，Prompt区域分配如下：

**头部0-4K**（高权重）：
- 系统指令
- 角色状态（base_persona + evolved_persona）
- 剧情状态（导演模块，如果开启）
- 故事大纲（JSON格式提升权重）

**中间4K-32K**（中等权重）：
- 背景世界设定
- 当前会话完整对话历史

**32K+区域**（能看到事实，语义分析能力弱）：
- 日常对话RAG检索的历史事件
- 导演模块RAG拉回的剧情事件

**尾部4K**（高权重）：
- 用户最新输入

**说明**：
- 头部4K使用JSON格式提升权重，确保AI始终记得角色人格和剧情状态
- 32K+区域的RAG事件作为"参考资料"，AI能看到但权重较低
- 尾部用户输入权重最高，确保AI优先响应当前对话

---

## 4. 导演模块（Director）

> **需求追溯**：解决 [REQUIREMENTS.md](REQUIREMENTS.md) "需求1：主线剧情的执行与遵守"

### 核心职责

控制 AI 按照故事大纲推进剧情，确保不偏离主线。

### 子功能：剧情大纲控制

通过三个部分协同工作，控制 AI 按照预设的故事大纲推进剧情。

#### 4.1 情节点设置

**作用**：让 AI 始终看到完整的故事大纲。

**实现**：
- 将背景设定中的 `story_outline`（10个情节点）放入 Prompt 头部 4K
- 使用 JSON 格式提升权重
- 每次对话都包含完整大纲

#### 4.2 进度引导

**作用**：让 AI 主动报告剧情推进情况。

**实现**：
- 在 Prompt 中告诉 AI：如果剧情推进，输出 `[PROGRESS:X:status]` 标记
  - X 是大纲点索引（1-10）
  - status 可以是：completed / in_progress / pending
- 每次 AI 回复后，后端正则扫描提取标记
- 如果找到标记：
  - 记录当前剧情点和状态（存入 instance_state.json）
  - 清零"未更新计数器"
- 如果没找到标记：
  - "未更新计数器" +1

#### 4.3 容错处理（RAG兜底）

**作用**：当 AI 连续多次没有推进剧情时，用 RAG 检索提醒 AI。

**触发条件**：
- "未更新计数器" >= 3（连续3次 AI 回复都没有 [PROGRESS] 标记）

**触发后的动作**：
1. 用当前大纲点构造查询：`"故事大纲第X点：{大纲点内容}"`
2. RAG 检索相关事件：
   - 当前实例：15条
   - 其他实例：5条（可选，作为剧情经验参考）
3. 在 Prompt 中加入"导演提醒"：
   ```
   【导演提醒】
   你现在应该推进到故事大纲第X点：{大纲点内容}

   【当前剧情中已发生的事件】
   - 事件1...
   - 事件2...

   【剧情经验参考】（其他剧情线的类似情节，仅供参考）
   - 参考1...
   - 参考2...
   ```
4. 清零"未更新计数器"

**注意**：跨实例检索时需明确标注"剧情经验参考"，避免混淆当前剧情的事实。

---

**扩展性**：导演模块是框架，未来可扩展更多子功能（如角色行为控制、冲突管理等）。

---

## 5. RAG 检索策略

> **需求追溯**：解决 [REQUIREMENTS.md](REQUIREMENTS.md) "需求3：长会话与跨会话的延续性"

### 5.1 日常对话RAG检索（Conversational RAG）

**触发时机**：每次用户输入时

**目的**：当用户提到历史事件时，AI能够回忆起来

**检索流程**：

```
用户输入："你还记得我们在据点的那次冲突吗？"
  ↓
[1] 使用LLM重新表述查询
  - 输入：当前用户输入 + 最近5-10条对话历史
  - 输出：独立的查询语句（例如："据点冲突事件，涉及主角与某人的对抗"）
  ↓
[2] 语义检索（Chroma）
  - 检索当前instance_id的历史事件
  - 返回20条相关事件（Summaries为主）
  ↓
[3] 注入到Prompt的32K+区域
  - 作为"历史事件回忆"展示给AI
```

**关键特性**：
- **语义检索**：相比Silly Tavern的关键词匹配，不需要精确命中关键词
- **实例隔离**：只检索当前`instance_id`的事件，严格隔离不同剧情线
- **检索范围**：只检索已汇总的会话事件（存储在Chroma中），当前活跃会话直接全量读取
- **首次对话**：Chroma为空时，返回空列表（正常情况）

**与导演模块RAG的区别**：
- **日常对话RAG**：用户主动提到历史时触发，只检索当前实例，返回20条
- **导演模块RAG**：AI连续3轮未推进剧情时触发，可跨实例检索（当前15条+其他5条）

---

### 5.2 深度搜索策略（分层摘要）

**浅层搜索（默认）**：
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
- 适用：日常对话，只需知道"发生过什么"
- 优点：简洁，节省 tokens

**深度搜索（按需）**：
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
- 适用：需要具体细节、情节设计参考
- 触发条件：
  - 用户询问"怎么"、"如何"、"详细过程"
  - 导演模块需要参考类似剧情
  - 自动判断（相似度 > 阈值则展开 Plots）

---

## 6. 角色人格机制

> **需求追溯**：解决 [REQUIREMENTS.md](REQUIREMENTS.md) "需求2：人格的丰富性与成长"

### 数据结构

```json
{
  "base_persona": "角色的初始人格，永不改变",
  "evolved_persona": "角色经历成长后的状态，手动更新"
}
```

**初始化**：
- 创建会话实例时，从全局角色定义复制 `base_persona`
- `evolved_persona` 初始为空字符串（角色尚未成长）
- 用户手动触发"更新记忆"后，`evolved_persona` 才有内容

### 两层人格

1. **base_persona（初始人格）**
   - 角色的底色，永不改变
   - 包含：原型、核心目标、核心特质、背景故事

2. **evolved_persona（成长人格）**
   - 随对话经历动态演化
   - 由用户手动触发更新
   - 包含：信念变化、行为模式、关系深度、当前情绪等
   - **纯定性描述**，不使用量化数值

### Prompt 组装

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

---

## 7. 数据存储结构

### 目录结构

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

### 会话文件格式（.jsonl）

**首次会话示例**：
```jsonl
{"type":"metadata","instance_id":"inst_001","session_id":"sess_001","created_at":"...","continued_from":null}
{"role":"user","content":"你好","turn":1,"timestamp":"..."}
{"role":"assistant","content":"你好，有什么事吗？","turn":1,"timestamp":"..."}
{"role":"user","content":"你还记得我之前说的话吗？","turn":2,"timestamp":"..."}
```

**汇总后的新会话示例**：
```jsonl
{"type":"metadata","instance_id":"inst_001","session_id":"sess_002","created_at":"...","continued_from":"sess_001"}
{"type":"summary","content":"情报交换与信任建立。用户告知 Alserqi 北区据点有背叛者的情报..."}
{"type":"summary","content":"制定复仇计划。Alserqi 决定前往调查，制定了详细的潜入计划..."}
{"role":"user","content":"你确定这个计划可行吗？","turn":1,"timestamp":"..."}
{"role":"assistant","content":"当然，我已经考虑了所有可能的风险...","turn":1,"timestamp":"..."}
{"role":"user","content":"那我们现在出发？","turn":6,"timestamp":"..."}
```

**说明**：
- 新会话的前几条消息是从旧会话复制的最后5轮对话
- `turn` 字段重新编号（从 1 开始）
- 消息内容是纯净的对话，不包含任何元信息注释

**消息类型定义**：
- `type:"metadata"` - 会话元数据（每个会话文件第一行）
  - `instance_id`: 所属会话实例 ID
  - `session_id`: 当前会话 ID
  - `created_at`: 创建时间
  - `continued_from`: 如果是汇总后的新会话，记录旧会话 ID（否则为 null）
- `type:"summary"` - 历史剧情摘要（纯文本，汇总后新会话才有）
  - `content`: 摘要文本（给 AI 看的剧情延续信息）
- `role:"user"` / `role:"assistant"` - 对话消息
  - `content`: 消息内容
  - `turn`: 轮次编号（一轮 = 一条 user + 一条 assistant）
  - `timestamp`: 时间戳

### 会话实例状态格式（instance_state.json）

**职责**：记录会话实例的运行状态。

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

**字段说明**：
- `instance_id`: 会话实例ID
- `character_id`: 使用的角色ID（创建实例时指定，不可变）
- `background_id`: 当前使用的背景ID（可以切换，也可以为null表示不使用背景）
- `current_session_id`: 当前活跃的会话ID
- `created_at`: 会话实例创建时间
- `plot_state`: 剧情大纲控制使用的状态
  - `current_plot_index`: 当前剧情点索引
  - `current_status`: 当前剧情点状态（completed/in_progress/pending）
  - `no_update_count`: 连续多少次AI回复没有输出[PROGRESS]标记（用于触发容错处理）


---

## 8. 配置系统

### 全局配置文件（config.json）

**位置**：`data/config.json`

**职责**：管理软件级别的功能开关和参数。

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

**字段说明**：

- `features.director_plot_control.enabled`: 剧情大纲控制功能开关
  - `true`: 启用（Prompt包含story_outline，扫描[PROGRESS]，触发RAG兜底）
  - `false`: 关闭（不放story_outline，不扫描[PROGRESS]，不触发RAG兜底）
- `features.director_plot_control.rag_fallback_threshold`: 触发容错处理的阈值（连续多少次未更新）
- `summary.order`: 汇总后新会话的初始内容顺序
  - `"summary_first"`: 先放摘要，再放最后N轮对话
  - `"last_n_first"`: 先放最后N轮对话，再放摘要
- `summary.last_n_turns`: 汇总后新会话保留多少轮对话
- `conversation_file.load_all`: 是否全量读取会话文件
- `conversation_file.max_tokens`: 会话文件的最大token数（预留）

**前端控制**：
- 前端提供设置页面，允许用户开关功能
- 修改配置后，后端重新加载配置文件
- 配置变更立即生效，不需要重启服务

---

## 9. 核心设计原则

### 单一数据源原则
- 只维护 .jsonl 文件，不维护内存缓存
- 避免数据同步问题
- 支持会话编辑功能（用户可自由编辑对话历史）

### 完全手动触发原则
- 汇总功能：用户主动触发
- 更新记忆：用户主动触发
- 不自动维护，用户完全控制

### 预留扩展性原则
- 汇总摘要格式待定，预留扩展字段
- Prompt 模板可配置
- 配置系统支持运行时调整

### 纯定性描述原则
- 角色状态使用自然语言描述
- 不使用 priority、intensity、level 等量化数值
- 符合人类理解习惯

---

## 10. 关键技术决策

### 已确定

1. ✅ **取消 RecallMemoryCache**（内存缓存）→ 全量读取对话
2. ✅ **取消定期自动维护**（Core Memory、事件写入）→ 完全手动触发
3. ✅ **取消滑动窗口机制** → 不限制对话数量
4. ✅ **汇总功能**：独立 LLM Agent，一次调用生成分层摘要
5. ✅ **分层摘要存储**：
   - Summaries Collection（摘要）：情节点概括，用于延续历史
   - Plots Collection（详细素材）：剧情过程，可复用的剧情设计
   - Chroma 分两个 Collection 存储，通过 ID 关联
6. ✅ **RAG 深度搜索**：支持浅层（只查 Summaries）和深度（展开 Plots）
7. ✅ **严格解析**：JSON 格式输出，解析失败报错，不降级
8. ✅ **会话编辑**：用户可以编辑/删除任意消息，自由操作对话历史

### 待确定

1. ⏳ 深度搜索的触发策略（关键词检测 vs 自动判断）
2. ⏳ Prompt 模板的详细设计
3. ⏳ 导演模块"拉回主线"的具体策略
4. ⏳ 性能优化的具体措施

---

## 11. 后续工作

### 短期

1. 确认概要设计（本文档）
2. 设计后端 API 接口规范
3. 设计前端 UI 交互流程

### 中期

1. 实现后端核心模块
2. 实现前端核心组件
3. 前后端联调

### 长期

1. 完善详细设计文档（按需）
2. 性能测试和优化
3. V1.0 发布

---

**文档版本**: v0.4.0
**创建日期**: 2025-10-15
**最后更新**: 2025-10-16
**文档类型**: 概要设计（不含详细实现）

---

## 变更记录

### v0.4.0 (2025-10-16)
- 🆕 **新增章节**：补充"日常对话RAG检索（Conversational RAG）"（第5.1章）
  - 详细描述触发时机、检索流程和关键特性
  - 与导演模块RAG的区别说明
- 🆕 **新增章节**：补充"Prompt组装策略"（第3.5章）
  - 基于LLM注意力机制U型特点的区域分配
  - 明确头部4K、中间区域、32K+区域、尾部4K的内容
- ✏️ **修正表述**：汇总功能"所有Summaries"明确为"当前会话的所有Summaries"
- ✏️ **删除冗余**：删除重复的"5.2 RAG检索范围"章节（内容已在5.1中说明）
- 📎 **需求追溯**：为第3.5、4、5、6章添加需求追溯链接
  - 明确每个设计解决哪个需求

### v0.3.5 (2025-10-16)
- 🆕 **新增配置系统**：设计全局配置文件 `config.json`
  - 管理软件级别的功能开关（如剧情大纲控制）
  - 前端可管理开关设置，修改后立即生效
- 📝 **更新目录结构**：在数据目录添加 `config.json`

### v0.3.4 (2025-10-16)
- 🔧 **修复术语**：配置系统 `recall_memory` → `conversation_file`
- 📝 **补充初始化**：明确角色状态创建时的初始化流程
  - `base_persona` 从全局角色定义复制
  - `evolved_persona` 初始为空字符串
  - 删除无用的 `last_maintenance_turn` 字段
- 📝 **修正描述**："支持回退功能" → "支持会话编辑功能"
- 📝 **清理示例**：删除会话文件示例中的元信息注释，补充说明
- 📝 **修正变更记录**：标注 v0.3.2 中错误添加的回退约束

### v0.3.3 (2025-10-16)
- 🗑️ **删除无用设计**：删除会话文件的 `status` 字段（完全没必要）
- 🔄 **重新设计会话编辑**：
  - 删除"回退功能"及其所有约束条件
  - 改为"会话编辑功能"：用户可以编辑/删除任意消息
  - 明确核心理念：对话界面是 Prompt 编辑器
- 📝 **修复命名**：
  - "粗粒度层/细粒度层" → "Summaries Collection / Plots Collection"
  - 提高可扩展性，避免层级增加时的命名问题
- 📝 **删除状态引用**：删除所有对 `status` 字段的引用和检查

### v0.3.2 (2025-10-16)
- 🔧 **术语统一**："父子结构" → "分层摘要（Hierarchical Summary）"
- 🔧 **修复逻辑冲突**：明确更新记忆不写入 Chroma（避免与汇总功能重复写入）
- 📝 **补充文档**：会话文件格式新增 `type:"summary"` 消息类型定义
- 📝 **明确 RAG 范围**：只检索已汇总会话，当前活跃会话不走 RAG
- 📝 **优化描述**：导演模块跨实例检索标注为"可选"，明确语义
- ⚠️ 注：本版本错误地添加了回退约束（已在 v0.3.3 删除）

### v0.3.1 (2025-10-15 01:30)
- 新增：汇总功能的分层摘要设计（摘要 + 详细素材）
- 新增：RAG 深度搜索策略
- 术语调整："对话历史" → "会话文件"
- 明确：严格解析模式，不使用降级方案

### v0.3.0 (2025-10-15 01:00)
- 创建概要设计文档
- 取消内存缓存和定期维护
- 确定全量读取和完全手动触发策略
