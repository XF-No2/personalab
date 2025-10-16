# PersonaLab

PersonaLab（人格实验室）是一个 **AI 角色对话平台**，支持长篇、高一致性、角色可成长的互动叙事。

---

## 🤖 AI 新会话启动指南

> **如果你是 AI 助手，请按以下顺序阅读文件**：

### 第 1 步：了解项目当前状态
```bash
阅读文件：.project_management/current_status.md
```
**包含内容**：
- 当前版本和开发阶段
- 最近的变更记录（最近5次）
- 待办事项（按优先级）
- 项目概述和架构状态

### 第 2 步：需要执行操作时
```bash
阅读文件：.project_management/ai_operation_manual.md
```
**何时阅读**：用户说"更新状态"、"发布版本"、"归档"时

**包含内容**：
- 更新状态的标准流程（5步骤）
- 发布版本的标准流程（11步骤）
- 归档的标准流程（5步骤）
- Git 操作规范（代理配置、状态检查）

### 第 3 步：查阅项目约定
```bash
阅读文件：.project_management/project_conventions.md
```
**何时阅读**：不确定目录结构、时间格式、设计原则时

**包含内容**：
- 目录结构规范
- 文档时间格式（YYYY-MM-DD HH:MM）
- 设计原则（禁止量化）
- 配置文件保护规范

### 文件简称（用于快速引用）
- **STATUS** = `.project_management/current_status.md`
- **MANUAL** = `.project_management/ai_operation_manual.md`
- **CONVENTIONS** = `.project_management/project_conventions.md`

---

## 📊 项目状态

- **当前版本**: v0.5.0 (拆分设计文档，模块化详细设计)
- **状态**: 🟡 设计阶段
- **最后更新**: 2025-10-16 23:15

---

## 👨‍💻 人类开发者快速开始

- **查看当前进度** → [.project_management/current_status.md](.project_management/current_status.md)
- **技术栈**: Next.js 14 + FastAPI + LangChain + Chroma
- **架构设计** → [docs/DESIGN_OVERVIEW.md](docs/DESIGN_OVERVIEW.md)
- **版本历史** → [CHANGELOG.md](CHANGELOG.md)

---

## 📚 完整文档列表

### 项目管理文档
- [.project_management/current_status.md](.project_management/current_status.md) - 项目当前状态
- [.project_management/ai_operation_manual.md](.project_management/ai_operation_manual.md) - AI 操作手册
- [.project_management/project_conventions.md](.project_management/project_conventions.md) - 项目约定
- [CHANGELOG.md](CHANGELOG.md) - 版本历史

### 需求与设计文档
- [docs/REQUIREMENTS.md](docs/REQUIREMENTS.md) (v3.0) - 需求文档
- [docs/DESIGN_OVERVIEW.md](docs/DESIGN_OVERVIEW.md) (v0.5.0) - 概要设计
- [docs/SYSTEM_FLOW.md](docs/SYSTEM_FLOW.md) (v2.0) - 系统流程验证
- [docs/design/](docs/design/) - 详细设计模块文档（5个模块）

### 详细设计模块
- [data_structure.md](docs/design/data_structure.md) - 底层数据字典
- [director.md](docs/design/director.md) - 导演模块
- [rag.md](docs/design/rag.md) - RAG检索策略
- [persona.md](docs/design/persona.md) - 角色人格机制
- [config.md](docs/design/config.md) - 配置系统

---

## 🏗️ 技术架构

**前端**: Next.js 14 + TypeScript + Shadcn/ui + Socket.IO
**后端**: Python 3.11+ + FastAPI + LangChain + Chroma

### 核心功能

1. **会话实例管理** - 创建独立的对话实例，状态隔离
2. **导演模块** - 剧情推进控制，确保AI不偏离主线
3. **流式对话** - WebSocket 实时通信，打字机效果
4. **人格成长** - 三层人格机制（静态+慢动态+快动态）
5. **长期记忆** - RAG向量检索历史事件
6. **角色管理** - 自定义角色人格和描述（CRUD）
7. **背景管理** - 创建对话场景背景（CRUD）
8. **暗色主题** - 专业的暗色 UI 设计

---

## 📝 开发阶段

详细待办事项请查看 [.project_management/current_status.md - 待办事项章节](.project_management/current_status.md#📋-待办事项)

**当前进度**：
- [x] 需求分析 (v3.0)
- [x] 架构设计 (v0.5.0)
- [x] 系统流程验证 (v2.0)
- [ ] 后端 API 接口设计
- [ ] 后端核心模块实现
- [ ] 前端 UI 实现
- [ ] 联调测试

---

## 📄 许可证

待定
