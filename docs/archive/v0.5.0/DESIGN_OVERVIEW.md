# PersonaLab 概要设计文档

**文档类型**: 概要设计（Architecture Overview）
**版本**: v0.5.0
**最后更新**: 2025-10-16
**状态**: 🟡 设计中

> 📌 **重要**: 这是概要设计文档，只描述架构框架、核心流程和设计原则。
> 详细设计见 [docs/design/](design/) 目录下的模块文档。

---

## 📚 详细设计文档索引

| 模块 | 文档 | 说明 |
|------|------|------|
| **底层数据字典** | [data_structure.md](design/data_structure.md) | 所有数据结构格式定义（JSON、JSONL、Chroma） |
| **导演模块** | [director.md](design/director.md) | 剧情大纲控制（情节点设置、进度引导、RAG兜底） |
| **RAG检索策略** | [rag.md](design/rag.md) | 日常对话RAG、深度搜索、分层摘要机制 |
| **角色人格机制** | [persona.md](design/persona.md) | 两层人格、Prompt组装、更新记忆流程 |
| **配置系统** | [config.md](design/config.md) | 全局配置文件、功能开关、前端控制 |

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
│   - 详细设计：见 persona.md
│
├─ 事件库（Event Library）
│   - 存储：Chroma 向量数据库（分两个 Collection）
│   - 内容：分层摘要（Hierarchical Summary）的情节数据
│       ├─ Summaries Collection：情节点摘要
│       └─ Plots Collection：详细剧情素材
│   - 作用：RAG 检索历史事件（支持深度搜索）
│   - 写入：汇总时生成并写入
│   - 详细设计：见 rag.md
│
└─ 会话文件（Conversation File）
    - 存储：.jsonl 文件（当前会话）
    - 内容：用户和 AI 的完整对话记录
    - 读取：每次全量读取所有对话
    - 汇总：汇总后关闭旧会话，开启新会话
    - 详细格式：见 data_structure.md
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
[3] 组装 Prompt（详见 3.5 节）
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
  输出 JSON 格式：
    [
      {"summary": "情节点摘要", "details": "详细剧情素材"}
    ]
  ↓
[3] 解析 JSON（严格模式）
  - 解析成功 → 继续
  - 解析失败 → 报错，提示用户重试
  ↓
[4] 写入 Chroma（分层摘要）
  - Summaries Collection：存储摘要
  - Plots Collection：存储详细素材
  - 通过 ID 关联
  ↓
[5] 创建新会话
  - 初始化内容：当前会话的所有 Summaries + 最后5轮对话
  ↓
[6] 返回前端，自动切换到新会话
```

**关键设计**：
- **分层摘要**：一次 LLM 调用生成两层数据（Summaries + Plots）
- **严格解析**：JSON 格式输出，解析失败直接报错
- **分层存储**：摘要和素材分别存入 Chroma 的不同 Collection
- **剧情延续**：新会话包含摘要 + 最后5轮对话，保持剧情连贯性

**详细设计**：见 [rag.md](design/rag.md) 第2节"深度搜索策略"

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

**详细设计**：见 [persona.md](design/persona.md) 第4节"更新记忆流程"

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

让 AI 始终看到完整的故事大纲（放入 Prompt 头部 4K，JSON 格式提升权重）

#### 4.2 进度引导

AI 输出 `[PROGRESS:X:status]` 标记，后端正则扫描提取并更新状态

#### 4.3 容错处理（RAG兜底）

当 AI 连续3次没有推进剧情时，用 RAG 检索提醒 AI（分层检索：当前实例15条 + 其他实例5条）

**详细设计**：见 [director.md](design/director.md)

---

## 5. RAG 检索策略

> **需求追溯**：解决 [REQUIREMENTS.md](REQUIREMENTS.md) "需求3：长会话与跨会话的延续性"

### 5.1 日常对话RAG检索（Conversational RAG）

**触发时机**：每次用户输入时

**检索流程**：
1. 使用LLM重新表述查询
2. 语义检索（Chroma）
3. 注入到Prompt的32K+区域

**关键特性**：
- 语义检索（不需要精确匹配关键词）
- 实例隔离（只检索当前instance_id）
- 检索范围（只检索已汇总的会话事件）

### 5.2 深度搜索策略（分层摘要）

- **浅层搜索**：只返回 Summaries（摘要）
- **深度搜索**：通过 related_plot_id 展开 Plots（详细素材）

**详细设计**：见 [rag.md](design/rag.md)

---

## 6. 角色人格机制

> **需求追溯**：解决 [REQUIREMENTS.md](REQUIREMENTS.md) "需求2：人格的丰富性与成长"

### 两层人格

1. **base_persona（初始人格）**：角色的底色，永不改变
2. **evolved_persona（成长人格）**：随对话经历动态演化，由用户手动触发更新

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

**详细设计**：见 [persona.md](design/persona.md)

---

## 7. 数据存储结构

### 目录结构

```
data/
├── config.json              # 全局配置文件（功能开关、参数设置）
├── characters/              # 角色库（全局）
├── backgrounds/             # 背景库（全局）
├── instances/               # 会话实例（核心）
│   └── {instance_id}/
│       ├── instance_state.json
│       ├── character_state.json
│       └── sessions/
├── event_library/           # 事件库（Chroma向量数据库）
└── prompt_templates/        # Prompt 模板（可配置）
```

**详细格式**：见 [data_structure.md](design/data_structure.md)

---

## 8. 配置系统

### 全局配置文件（config.json）

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

**详细设计**：见 [config.md](design/config.md)

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

**文档版本**: v0.5.0
**创建日期**: 2025-10-15
**最后更新**: 2025-10-16
**文档类型**: 概要设计（不含详细实现）

---

## 变更记录

### v0.5.0 (2025-10-16)
- 🔄 **模块化重构**：将详细设计拆分到 docs/design/ 目录
  - 新增底层数据字典：data_structure.md（所有数据格式定义）
  - 新增模块文档：director.md, rag.md, persona.md, config.md
  - 精简 DESIGN_OVERVIEW.md：只保留架构概览和核心流程
  - 新增文档索引表格，便于查阅
- 📝 **零依赖设计**：各模块文档自包含，只依赖 data_structure.md
- 📝 **保留完整流程**：核心流程章节保留，方便理解全局运作机制

### v0.4.0 (2025-10-16)
- 🆕 **新增章节**：补充"日常对话RAG检索（Conversational RAG）"（第5.1章）
- 🆕 **新增章节**：补充"Prompt组装策略"（第3.5章）
- ✏️ **修正表述**：汇总功能"所有Summaries"明确为"当前会话的所有Summaries"
- ✏️ **删除冗余**：删除重复的"5.2 RAG检索范围"章节
- 📎 **需求追溯**：为第3.5、4、5、6章添加需求追溯链接

### v0.3.5 (2025-10-16)
- 🆕 **新增配置系统**：设计全局配置文件 `config.json`
- 📝 **更新目录结构**：在数据目录添加 `config.json`

### v0.3.4 (2025-10-16)
- 🔧 **修复术语**：配置系统 `recall_memory` → `conversation_file`
- 📝 **补充初始化**：明确角色状态创建时的初始化流程

### v0.3.3 (2025-10-16)
- 🗑️ **删除无用设计**：删除会话文件的 `status` 字段
- 🔄 **重新设计会话编辑**：改为 Prompt 编辑器理念

### v0.3.2 (2025-10-16)
- 🔧 **术语统一**："父子结构" → "分层摘要（Hierarchical Summary）"
- 🔧 **修复逻辑冲突**：明确更新记忆不写入 Chroma

### v0.3.1 (2025-10-15)
- 新增：汇总功能的分层摘要设计
- 新增：RAG 深度搜索策略

### v0.3.0 (2025-10-15)
- 创建概要设计文档
- 取消内存缓存和定期维护
