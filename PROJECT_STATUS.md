# PersonaLab 项目当前状态

**最后更新**: 2025-10-14
**当前版本**: v0.2.0 (架构重新设计完成)
**状态**: 🟡 设计阶段

> 📌 **重要**：这是项目的"活文档"，记录**当前真实状态**，而非某个发布版本。
> 版本发布时，会归档快照到 `docs/archive/v{版本号}/`

---

## 🎯 项目概述

PersonaLab（人格实验室）是一个 **AI 角色对话平台**，支持长篇、高一致性、角色可成长的互动叙事。

### 核心功能需求

1. **故事线管理** - 创建独立的故事世界，状态隔离
2. **流式对话** - WebSocket 实时通信，打字机效果
3. **人格成长** - 三层人格机制（静态+慢动态+快动态）
4. **长期记忆** - RAG向量检索历史事件
5. **角色管理** - 自定义角色人格和描述（CRUD）
6. **背景管理** - 创建对话场景背景（CRUD）
7. **暗色主题** - 专业的暗色 UI 设计

### 技术选型（已确定）

**前端**：
- **框架**: Next.js 14 (App Router)
- **语言**: TypeScript
- **UI 库**: Shadcn/ui + Tailwind CSS
- **状态管理**: Zustand（前端临时状态）
- **通信**: Socket.IO Client (WebSocket) + Fetch API (REST)

**后端**：
- **语言**: Python 3.11+
- **框架**: FastAPI（REST API + WebSocket）
- **LLM**: LangChain + OpenAI/Claude API
- **向量库**: Chroma（本地部署）
- **存储**: 文件系统（JSON/JSONL）

---

## 🏗️ 架构状态

### 目录结构（最新设计）

```
personalab/
├── frontend/                 # 前端（待生成）
│   ├── src/
│   │   ├── app/            # Next.js 页面
│   │   │   ├── storylines/  # 故事线管理页面
│   │   │   ├── characters/  # 角色管理页面
│   │   │   └── backgrounds/ # 背景管理页面
│   │   ├── components/     # UI 组件
│   │   ├── lib/            # 工具库
│   │   └── types/          # TypeScript 类型
│   └── package.json
│
├── backend/                  # 后端（待生成）
│   ├── app/
│   │   ├── main.py         # FastAPI 入口
│   │   ├── models/         # Pydantic 数据模型
│   │   ├── core/           # 核心业务逻辑
│   │   │   ├── state_manager.py      # 状态管理
│   │   │   ├── event_manager.py      # 事件库管理
│   │   │   ├── prompt_engine.py      # Prompt组装
│   │   │   └── interaction_loop.py   # 核心交互流程
│   │   ├── api/            # REST API
│   │   └── websocket/      # WebSocket
│   └── requirements.txt
│
├── data/                     # 数据目录（运行时生成）
│   ├── characters/          # 角色库（全局）
│   ├── backgrounds/         # 背景库（全局）
│   ├── storylines/          # 故事线（核心）
│   │   └── {storyline_id}/
│   │       ├── metadata.json
│   │       ├── character_state.json
│   │       └── sessions/
│   ├── event_library/       # 全局事件库
│   │   └── chroma_db/
│   └── prompt_templates/
│
├── docs/
│   ├── DESIGN.md            # 技术设计文档 ✅ v0.2.0
│   ├── QUESTIONS.md         # 待解决问题清单 ✅
│   ├── archive/
│   └── reports/
│
├── temp/
│   ├── reports/
│   └── tests/
│
├── PROJECT_STATUS.md         # 项目状态（本文件）
├── 规范.md                   # 工作规范
├── README.md
└── CHANGELOG.md
```

---

## 🔧 核心架构设计

### 故事线架构（Storyline）

**核心概念**：
- 故事线是一条独立的、连续的故事世界
- 角色状态属于故事线，不属于全局
- 不同故事线完全隔离，互不影响
- 一个故事线内可以有多个会话（暂停/继续）

**数据流**：
```
角色定义（全局）→ 故事线1 → 角色状态1 → 会话1, 会话2, 会话3
                  → 故事线2 → 角色状态2 → 会话4, 会话5
```

### 人格三层机制

**核心原理**：数据的变化速度不同

1. **静态层（core_identity）** - 永不变化
   - archetype, core_goal, core_traits, background_story

2. **慢动态层（growth_state）** - 重大事件才变化
   - beliefs（信念）
   - behavioral_patterns（行为模式）
   - relationships（关系深度）

3. **快动态层（current_state）** - 频繁变化
   - emotions（当前情绪）
   - physical（身体状况）
   - immediate_goals（短期目标）

**维护机制**：
- 每10轮对话执行定期维护任务
- 清理过多累积（快动态层保留最新5个）
- 去重、合并（慢动态层）
- **纯定性描述，绝对禁止量化数值**

### 事件库与RAG

**设计要点**：
- 事件必须记录 `storyline_id` 和 `session_id`
- RAG检索时必须过滤 `storyline_id`，避免不同故事线混淆
- 不使用事件链表（previous_event），纯语义检索
- 只在状态变化时写入事件
- 异步写入，不阻塞返回

---

## 📊 当前状态

### 已完成

- [x] 完成核心技术设计（DESIGN.md v0.2.0）
- [x] 明确故事线架构
- [x] 设计人格三层机制（纯定性）
- [x] 设计事件库结构（带storyline_id过滤）
- [x] 设计性能优化方案（~2秒非LLM耗时）
- [x] 删除所有过度设计（量化、链表、归档、摘要）
- [x] 完成需求文档（REQUIREMENTS.md）
- [x] 研究Silly Tavern和Letta/MemGPT参考系统

### 进行中

- [ ] 待实现：后端核心模块
- [ ] 待实现：前端UI

---

## 📝 最近调整记录

### 2025-10-14 15:30 - 完成需求文档编写 ✅

**调整内容**:
1. 创建 [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) - 需求文档
   - 明确项目定位：AI角色对话平台,类似Silly Tavern
   - 核心需求1：主线剧情执行与遵守 (10个节点放入Prompt头部4K)
   - 核心需求2：人格丰富性与成长 (借鉴Letta/MemGPT三层记忆架构)
   - 核心需求3：长会话延续性 (使用Conversational RAG优化世界书机制)
2. 需求文档采用"需求+设计方案"结构
   - 每个需求都包含具体的设计方案
   - 使用具体数值 (每10轮、最近5-10条、检索20条)
   - 完整解释所有概念,避免缩略
3. 研究并参考了Silly Tavern和Letta/MemGPT系统
   - Silly Tavern World Book: 基于关键词匹配最近N条消息
   - Letta/MemGPT: 三层记忆架构 (Core/Archival/Recall Memory)
   - 采用Conversational RAG替代简单关键词匹配
4. 讨论AI协作规范的可执行性
   - 分析了AI无法自动执行的规范类型 (主观判断、自动触发)
   - 确认只有用户可验证的操作规范才可执行 (项目管理操作)

### 2025-10-14 - 架构重新设计 ✅

**重大调整**：
1. **引入故事线（Storyline）架构**
   - 解决原设计的状态污染问题
   - 角色状态从全局改为故事线级别
   - 一个故事线内可以有多个会话（支持暂停/继续）

2. **简化人格三层机制**
   - 删除所有量化数值（priority, intensity, level）
   - 改为纯定性描述 + timestamp
   - 用定期维护任务清理，而非数字降级

3. **简化事件库设计**
   - 删除事件链表（previous_event）
   - 改为纯RAG语义检索
   - 必须记录storyline_id和session_id
   - RAG检索时必须过滤storyline_id

4. **删除过度设计**
   - 删除归档机制（archived标记）
   - 删除会话摘要（session_summary）
   - 删除事件双层结构（会话事件+全局事件）
   - 删除数字降级机制（每N轮-1）

5. **性能优化**
   - 异步并行加载（asyncio.gather）
   - RAG超时保护（1.5秒）
   - 异步写入事件（不阻塞返回）
   - 总耗时：~2秒（非LLM）+ ~18秒（LLM）≈ 20秒 ✅

**设计原则确认**：
- ✅ 故事线隔离状态
- ✅ 纯定性描述，无量化
- ✅ 定期维护任务清理数据
- ✅ 事件库带storyline_id过滤
- ✅ 性能目标：总耗时 < 30秒

### 2025-10-13 - 完成初版技术设计 ✅

**调整内容**:
1. 明确系统总目标：解决LLM长对话中的上下文稀释和指令遗忘问题
2. 设计核心数据结构（已在v0.2.0中重构）
3. 确定核心机制（已在v0.2.0中简化）
4. 设计Prompt工程：模块化、可配置
5. 明确技术栈：后端 Python + FastAPI + LangChain + Chroma

---

## 📋 待办事项

### 高优先级（设计阶段）

- [x] 完成核心技术设计（DESIGN.md v0.2.0）
- [x] 解决架构层面的设计问题
- [x] 确定数据结构细节
- [x] 完成需求文档（REQUIREMENTS.md）
- [ ] 设计后端API接口规范（基于故事线架构）

### 高优先级（后端实现）

- [ ] 搭建FastAPI项目结构
- [ ] 实现数据模型（Pydantic）
- [ ] 实现StateManager（三层人格状态管理）
- [ ] 实现EventManager（RAG + Chroma + storyline过滤）
- [ ] 实现PromptEngine（Prompt组装）
- [ ] 实现LLMService（LangChain集成）
- [ ] 实现InteractionLoop（核心交互流程）
- [ ] 实现定期维护任务
- [ ] 实现REST API（故事线/角色/背景CRUD）
- [ ] 实现WebSocket（流式对话）

### 高优先级（前端实现）

- [ ] 创建Next.js项目结构
- [ ] 生成TypeScript类型定义（基于新架构）
- [ ] 生成API客户端封装（故事线API）
- [ ] 生成WebSocket管理
- [ ] 生成Shadcn UI基础组件
- [ ] 生成布局组件
- [ ] 生成故事线管理组件
- [ ] 生成对话功能组件
- [ ] 生成角色管理组件
- [ ] 生成背景管理组件
- [ ] 生成页面路由 

### 中优先级

- [ ] Prompt模板前端管理功能
- [ ] 故事线导出功能
- [ ] 角色状态展示
- [ ] 添加加载状态和错误处理
- [ ] 响应式布局优化

### 低优先级

- [ ] 编写组件文档
- [ ] 添加单元测试
- [ ] 性能监控和优化

---

## ⚙️ 强制性开发规范

### 1. 文件位置规范

| 文件类型 | 存放位置 | 示例 |
|---------|---------|------|
| 临时测试文件 | `temp/tests/` | test_*.* |
| 临时报告文件 | `temp/reports/` | *_report.md |
| 版本发布报告 | `docs/reports/` | v{版本号}_release_report.md |
| 归档文档 | `docs/archive/v{版本号}/` | PROJECT_STATUS.md 等 |
| 核心文档 | 项目根目录 | PROJECT_STATUS.md, README.md, CHANGELOG.md, 规范.md |
| 前端源代码 | `frontend/src/` | *.ts, *.tsx |
| 后端源代码 | `backend/app/` | *.py |
| 正式测试 | `tests/` | test_*.* |

### 2. 设计原则（关键）

**绝对禁止量化**：
- ❌ 不使用 priority, intensity, level, score 等数字
- ❌ 不使用"每N轮-1"的降级机制
- ❌ 不使用数字阈值判断删除

**纯定性描述**：
- ✅ 所有内容都是文本描述
- ✅ 用 timestamp 记录时间
- ✅ 用定期维护任务清理数据

---

## 🤝 AI 协作指南

### 基本原则

1. **归档文档是历史快照**：`docs/archive/` 不代表最新状态
2. **问题查阅顺序**：PROJECT_STATUS.md → DESIGN.md → 规范.md
3. **设计原则**：纯定性描述，绝对禁止量化

### 操作规范

**执行项目管理操作时，参考 `规范.md`**：
- 归档：`按照规范归档`
- 重构：`按照规范重构`
- 版本发布：`按照规范发布`
- 更新PS：`按照规范更新PS` 或 `更新PS`

---

## 🎯 下一步计划

### 短期目标（1-2周）

1. 设计后端API接口规范（基于故事线架构）
2. 搭建FastAPI后端项目结构
3. 实现核心模块（StateManager, EventManager, PromptEngine）
4. 实现基础REST API

### 中期目标（3-4周）

1. 完成后端核心功能实现
2. 搭建Next.js前端项目
3. 实现前端核心UI组件
4. 前后端联调

### 长期目标（1-2月）

1. 完成V1.0功能开发
2. 测试和优化
3. 编写用户文档
4. 准备发布

---

**文档版本**: v0.2.0
**创建日期**: 2025-10-13
**最后更新**: 2025-10-14
