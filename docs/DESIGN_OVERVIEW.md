# PersonaLab 概要设计文档

**文档类型**: 概要设计（Architecture Overview）
**版本**: v0.3.2
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
│       ├─ 粗粒度层（Summaries Collection）：情节点摘要
│       └─ 细粒度层（Plots Collection）：详细剧情素材
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
     1. 摘要：一句话概括（粗粒度）
     2. 详细素材：具体的剧情过程（细粒度）

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
    ├─ 粗粒度层 → Summaries Collection
    │   - ID: summary_{session_id}_{index}
    │   - 内容: summary（摘要）
    │   - 元数据: related_plot_id, session_id, instance_id
    │
    └─ 细粒度层 → Plots Collection
        - ID: plot_{session_id}_{index}
        - 内容: details（详细素材）
        - 元数据: related_summary_id, session_id, instance_id
  ↓
[5] 关闭当前会话（标记为 summarized）
  ↓
[6] 创建新会话
  - 初始化内容（可配置顺序）:
      所有摘要（粗粒度层）+ 最后5轮对话
  - 写入新会话文件：
      {"type":"summary","content":"摘要文本1"}
      {"type":"summary","content":"摘要文本2"}
      ...（最后5轮的 user/assistant 消息）
  ↓
[7] 返回前端，自动切换到新会话
```

**关键设计**：
- **分层摘要**：一次 LLM 调用生成两层数据（粗粒度摘要 + 细粒度素材）
- **严格解析**：JSON 格式输出，解析失败直接报错，由用户决定重试
- **分层存储**：摘要和素材分别存入 Chroma 的不同 Collection，通过 ID 关联
- **剧情延续**：新会话包含摘要（纯文本）+ 最后5轮对话，保持剧情连贯性
- **可复用素材**：细粒度素材保存在 Chroma，可被导演模块、RAG 检索等功能复用

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

### 3.4 回退重新生成流程

**触发**: 用户点击"⬅️ 回退 N 步"按钮

**前置条件**：
- 只有 status: "active" 的会话才能回退
- status: "summarized" 或 "closed" 的会话不可回退（前端禁用按钮）

```
回退流程
  ↓
[1] 检查会话状态（必须是 "active"）
  ↓
[2] 读取会话文件
  ↓
[3] 删除最后 N 轮对话（N 轮 = 2N 条消息）
  ↓
[4] 重写会话文件
  ↓
[5] 返回前端，用户可重新输入
```

**设计原则**：
- 单一数据源，只需修改 .jsonl 文件
- 不需要同步更新缓存（因为没有缓存）
- 已汇总的会话不可回退（避免与 Chroma 数据不一致）

---

## 4. 导演模块

### 核心职责

控制剧情推进，确保 AI 不偏离主线大纲。

### 三层控制机制

```
1. 静态约束
   - 主线大纲（10个情节点）全量放入 Prompt 头部 4K
   - JSON 格式，始终可见

2. 动态引导
   - AI 每次回复输出 [PROGRESS:X:status] 标记
   - 后端正则扫描，更新剧情状态

3. RAG 兜底（可选）
   - 连续 3 轮未更新剧情状态 → 触发兜底机制
   - 检索策略：
     * 优先检索当前实例的历史事件（15 条）
     * 可选：跨实例检索（5 条，借鉴其他剧情线的推进方式）
   - 注入 Prompt 提醒 AI 回归主线
   - 注：跨实例检索时需明确标注"剧情经验参考"，避免混淆事实
```

**扩展性**: 导演模块是框架，未来可扩展更多控制机制。

---

## 5. RAG 检索策略

### 5.1 深度搜索策略（分层摘要）

**浅层搜索（默认）**：
```
用户输入："你还记得之前的复仇吗？"
  ↓
RAG 搜索 Summaries Collection（只查粗粒度层）
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
  - 粗粒度：summary_001（摘要）
  - 细粒度：plot_001（详细素材："小张假装和解，邀请小玲到废弃工厂..."）
  ↓
组装 Prompt（包含摘要 + 详细素材）
```
- 适用：需要具体细节、情节设计参考
- 触发条件：
  - 用户询问"怎么"、"如何"、"详细过程"
  - 导演模块需要参考类似剧情
  - 自动判断（相似度 > 阈值则展开细粒度层）

### 5.2 RAG 检索范围

**检索对象**：
- 只检索**已汇总的会话**（status: "summarized"）的事件
- 当前活跃会话不走 RAG（直接全量读取 .jsonl 文件）
- 首次对话时 Chroma 为空（正常情况，RAG 返回空列表）

**实例隔离**：
- 日常对话：只检索当前 `instance_id` 的事件（严格隔离）
- 返回 20 条相关事件（粗粒度层为主）

**导演兜底（可选）**：
- 跨实例检索（借鉴其他实例的剧情推进经验）
- 分两次查询：
  - 当前实例 15 条
  - 其他实例 5 条
- Prompt 中明确区分"当前剧情事实"和"剧情经验参考"

---

## 6. 角色人格机制

### 数据结构

```json
{
  "base_persona": "角色的初始人格，永不改变",
  "evolved_persona": "角色经历成长后的状态，手动更新",
  "last_maintenance_turn": 150
}
```

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
│       ├── metadata.json
│       ├── character_state.json
│       └── sessions/
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
{"type":"metadata","instance_id":"inst_001","session_id":"sess_001","status":"active","created_at":"...","continued_from":null}
{"role":"user","content":"你好","turn":1,"timestamp":"..."}
{"role":"assistant","content":"你好，有什么事吗？","turn":1,"timestamp":"..."}
{"role":"user","content":"你还记得我之前说的话吗？","turn":2,"timestamp":"..."}
```

**汇总后的新会话示例**：
```jsonl
{"type":"metadata","instance_id":"inst_001","session_id":"sess_002","status":"active","created_at":"...","continued_from":"sess_001"}
{"type":"summary","content":"情报交换与信任建立。用户告知 Alserqi 北区据点有背叛者的情报..."}
{"type":"summary","content":"制定复仇计划。Alserqi 决定前往调查，制定了详细的潜入计划..."}
{"role":"user","content":"（旧会话第46轮）你确定这个计划可行吗？","turn":1,"timestamp":"..."}
{"role":"assistant","content":"（旧会话第46轮）当然，我已经...","turn":1,"timestamp":"..."}
{"role":"user","content":"（新会话第51轮）那我们现在出发？","turn":6,"timestamp":"..."}
```

**消息类型定义**：
- `type:"metadata"` - 会话元数据（每个会话文件第一行）
  - `status`: "active" / "closed" / "summarized"
  - `continued_from`: 如果是汇总后的新会话，记录旧会话 ID
- `type:"summary"` - 历史剧情摘要（纯文本，汇总后新会话才有）
  - `content`: 摘要文本（粗粒度层，给 AI 看的剧情延续信息）
- `role:"user"` / `role:"assistant"` - 对话消息
  - `turn`: 轮次编号（一轮 = 一条 user + 一条 assistant）

---

## 8. 配置系统

### 配置文件（预留）

```json
{
  "summary": {
    "order": "summary_first",  // 或 "last_5_first"
    "last_n_turns": 5
  },
  "recall_memory": {
    "load_all": true,
    "max_tokens": 100000
  },
  "director": {
    "enabled": true,
    "rag_fallback_threshold": 3
  }
}
```

---

## 9. 核心设计原则

### 单一数据源原则
- 只维护 .jsonl 文件，不维护内存缓存
- 避免数据同步问题
- 支持回退功能

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
   - 粗粒度层（摘要）：情节点概括，用于延续历史
   - 细粒度层（详细素材）：剧情过程，可复用的剧情设计
   - Chroma 分两个 Collection 存储，通过 ID 关联
6. ✅ **RAG 深度搜索**：支持浅层（只查粗粒度）和深度（展开细粒度）
7. ✅ **严格解析**：JSON 格式输出，解析失败报错，不降级
8. ✅ **回退功能**：支持删除最近 N 轮对话

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

**文档版本**: v0.3.2
**创建日期**: 2025-10-15
**最后更新**: 2025-10-16
**文档类型**: 概要设计（不含详细实现）

---

## 变更记录

### v0.3.2 (2025-10-16)
- 🔧 **术语统一**："父子结构" → "分层摘要（Hierarchical Summary）"
- 🔧 **修复逻辑冲突**：明确更新记忆不写入 Chroma（避免与汇总功能重复写入）
- 📝 **补充文档**：会话文件格式新增 `type:"summary"` 消息类型定义
- 📝 **明确 RAG 范围**：只检索已汇总会话，当前活跃会话不走 RAG
- 🔒 **新增约束**：回退功能只对 status:"active" 的会话有效
- 📝 **优化描述**：导演模块跨实例检索标注为"可选"，明确语义

### v0.3.1 (2025-10-15 01:30)
- 新增：汇总功能的分层摘要设计（摘要 + 详细素材）
- 新增：RAG 深度搜索策略
- 术语调整："对话历史" → "会话文件"
- 明确：严格解析模式，不使用降级方案

### v0.3.0 (2025-10-15 01:00)
- 创建概要设计文档
- 取消内存缓存和定期维护
- 确定全量读取和完全手动触发策略
