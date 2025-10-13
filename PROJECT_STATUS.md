# PersonaLab 项目当前状态

**最后更新**: 2025-10-10
**当前版本**: v0.1.0 (前端待生成)
**状态**: 🟡 筹备中

> 📌 **重要**：这是项目的"活文档"，记录**当前真实状态**，而非某个发布版本。
> 版本发布时，会归档快照到 `docs/archive/v{版本号}/`

---

## 🎯 项目概述

PersonaLab（人格实验室）是一个 **AI 角色对话平台**。

### 核心功能需求

1. **流式对话** - WebSocket 实时通信，打字机效果
2. **会话管理** - 创建、删除、导出会话（Markdown 格式）
3. **角色管理** - 自定义角色人格和描述（CRUD）
4. **背景管理** - 创建对话场景背景（CRUD）
5. **暗色主题** - 专业的暗色 UI 设计

### 技术选型（已确定）

**前端**：
- **框架**: Next.js 14 (App Router)
- **语言**: TypeScript
- **UI 库**: Shadcn/ui + Tailwind CSS
- **状态管理**: Zustand + localStorage 持久化
- **通信**: Socket.IO Client (WebSocket) + Fetch API (REST)

**后端**：
- 用户将自行进行颠覆性改造
- 前端需预留好后端接口对接

---

## 🏗️ 架构状态

### 目录结构（当前真实状态）

```
personalab/
├── src/                      # 前端源代码（待生成）
│   ├── app/                 # Next.js 页面
│   ├── components/          # UI 组件
│   ├── lib/                 # 工具库
│   └── types/               # TypeScript 类型
│
├── tests/                    # 测试
├── docs/                     # 文档
│   ├── archive/             # 版本归档
│   └── reports/             # 过程报告
├── temp/                     # 临时文件
│   ├── reports/
│   └── tests/
│
├── PROJECT_STATUS.md         # 项目状态（本文件）
├── 规范.md                   # 工作规范
├── README.md                 # 项目说明
└── CHANGELOG.md              # 变更日志
```

### 前端技术架构

**目录结构设计**：
```
src/
├── app/                     # Next.js 页面
│   ├── layout.tsx          # 根布局（暗色主题）
│   ├── page.tsx            # 首页（重定向到 /chat）
│   ├── chat/               # 对话页面
│   ├── characters/         # 角色管理页面
│   └── backgrounds/        # 背景管理页面
│
├── components/              # UI 组件
│   ├── ui/                 # Shadcn 基础组件
│   ├── layout/             # 布局组件（Navbar, Sidebar）
│   ├── chat/               # 对话相关组件
│   ├── character/          # 角色管理组件
│   └── background/         # 背景管理组件
│
├── lib/                     # 工具库
│   ├── api/                # REST API 客户端
│   ├── store/              # Zustand 状态管理
│   ├── websocket.ts        # WebSocket 管理
│   └── utils.ts            # 工具函数
│
└── types/                   # TypeScript 类型
    └── index.ts            # 类型定义
```

**状态管理方案**：
- 使用 Zustand 管理全局状态（会话、角色、背景）
- 使用 `persist` 中间件实现 localStorage 持久化
- 刷新页面数据不丢失

**通信方案**：
- REST API：会话/角色/背景的 CRUD 操作
- WebSocket：流式对话（打字机效果）

**主题方案**：
- Tailwind CSS dark mode
- 暗色主题配色
- 用户消息与 AI 消息不同背景色

---

## 🔧 核心模块状态

### 1. 前端模块（待生成）

| 模块 | 状态 | 说明 |
|------|------|------|
| 配置文件 | ⏳ 待生成 | package.json, tsconfig.json 等 |
| 类型定义 | ⏳ 待生成 | TypeScript 接口定义 |
| API 客户端 | ⏳ 待生成 | REST API 封装 |
| WebSocket | ⏳ 待生成 | 流式对话管理 |
| 状态管理 | ⏳ 待生成 | Zustand store |
| UI 组件 | ⏳ 待生成 | Shadcn 组件库 |
| 布局组件 | ⏳ 待生成 | Navbar, Sidebar |
| 功能组件 | ⏳ 待生成 | Chat, Character, Background |
| 页面 | ⏳ 待生成 | 对话、角色、背景页面 |

### 2. 后端接口预留

**说明**：前端需预留以下接口，具体实现由用户后端提供

**REST API**：
- `GET /api/sessions` - 获取会话列表
- `POST /api/sessions` - 创建会话
- `DELETE /api/sessions/:id` - 删除会话
- `GET /api/sessions/:id/export` - 导出会话
- `GET /api/characters` - 获取角色列表
- `POST /api/characters` - 创建角色
- `PUT /api/characters/:id` - 更新角色
- `DELETE /api/characters/:id` - 删除角色
- `GET /api/backgrounds` - 获取背景列表
- `POST /api/backgrounds` - 创建背景
- `PUT /api/backgrounds/:id` - 更新背景
- `DELETE /api/backgrounds/:id` - 删除背景

**WebSocket**：
- 事件：`chat` - 发送消息
- 事件：`chat_response` - 接收流式响应

---

## 📊 代码统计

**总行数**: 0 行（待生成）

---

## 📝 最近调整记录

### 2025-10-10 - 项目初始化 ✅

**调整内容**:
1. 创建标准目录结构
2. 从全局模板复制规范.md
3. 创建基础文档
4. 明确前端重新生成需求
5. 确定技术选型和架构设计

---

## 🤝 AI 协作指南

### 基本原则
1. **归档文档是历史快照**：`docs/archive/` 不代表最新状态
2. **问题查阅顺序**：PROJECT_STATUS.md → 规范.md

### 操作规范

**执行项目管理操作时，参考 `规范.md`**：
- 归档：`按照规范归档`
- 重构：`按照规范重构`
- 版本发布：`按照规范发布`
- 更新PS：`按照规范更新PS` 或 `更新PS`

---

## 📋 待办事项

### 高优先级（前端生成）
- [ ] 创建前端配置文件（package.json, tsconfig.json, tailwind.config.ts 等）
- [ ] 生成 TypeScript 类型定义
- [ ] 生成 API 客户端封装
- [ ] 生成 WebSocket 管理
- [ ] 生成 Zustand 状态管理
- [ ] 生成 Shadcn UI 基础组件
- [ ] 生成布局组件
- [ ] 生成对话功能组件
- [ ] 生成角色管理组件
- [ ] 生成背景管理组件
- [ ] 生成页面路由

### 中优先级
- [ ] 实现前端独立运行（使用假数据/本地状态）
- [ ] 优化暗色主题样式
- [ ] 添加加载状态和错误处理
- [ ] 实现会话导出功能
- [ ] 响应式布局优化

### 低优先级
- [ ] 编写组件文档
- [ ] 添加单元测试
- [ ] 性能优化

---

## ⚙️ 强制性开发规范

### 1. 文件位置规范

**明确指定各类文件的存放位置**：

| 文件类型 | 存放位置 | 示例 |
|---------|---------|------|
| 临时测试文件 | `temp/tests/` | test_*.* |
| 临时报告文件 | `temp/reports/` | *_report.md |
| 版本发布报告 | `docs/reports/` | v{版本号}_release_report.md |
| 归档文档 | `docs/archive/v{版本号}/` | PROJECT_STATUS.md 等 |
| 核心文档 | 项目根目录 | PROJECT_STATUS.md, README.md, CHANGELOG.md, 规范.md |
| 前端源代码 | `src/` | *.ts, *.tsx |
| 正式测试 | `tests/` | test_*.* |

**根目录文件清单**：
1. PROJECT_STATUS.md
2. README.md
3. CHANGELOG.md
4. 规范.md

---

## 🚀 前端生成准备

### 功能需求明确

**对话功能**：
- 左侧边栏：会话列表、角色选择、背景选择
- 右侧主区：消息展示区 + 消息输入框
- WebSocket 流式对话，打字机效果
- 消息格式：用户消息、AI 消息（不同样式）

**会话管理**：
- 创建新会话（指定角色和背景）
- 删除会话
- 会话列表展示
- 导出会话为 Markdown

**角色管理**：
- 角色列表（卡片式）
- 创建角色（名称 + 描述）
- 编辑角色
- 删除角色

**背景管理**：
- 背景列表（卡片式）
- 创建背景（名称 + 描述）
- 编辑背景
- 删除背景

### 技术实现要点

**独立运行**：
- 前端需支持独立运行（不依赖后端）
- 使用 Zustand + localStorage 存储状态
- API 调用失败时使用本地数据

**接口预留**：
- 使用环境变量配置后端地址
- API 客户端封装，便于后端对接
- WebSocket 连接配置化

**暗色主题**：
- 全局暗色主题
- 用户消息：蓝色背景
- AI 消息：灰色背景
- 响应式布局

---

**文档版本**: v0.1.0
**创建日期**: 2025-10-10
