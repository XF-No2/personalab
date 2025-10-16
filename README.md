# PersonaLab

PersonaLab（人格实验室）是一个 **AI 角色对话平台**，支持长篇、高一致性、角色可成长的互动叙事。

## 📊 项目状态

- **当前版本**: v0.5.0 (拆分设计文档，模块化详细设计)
- **状态**: 🟡 设计阶段
- **最后更新**: 2025-10-16 22:10

## 🚀 快速开始

### AI 新会话启动指南

1. **首先读取** → [PROJECT_STATUS.md](PROJECT_STATUS.md) (简称: **PS**) - 了解项目当前状态
2. **执行规范操作时** → [规范.md](规范.md) - 工作规范和操作指令
3. **了解设计** → [docs/DESIGN_OVERVIEW.md](docs/DESIGN_OVERVIEW.md) - 概要设计

### 简称约定

- **PS** = PROJECT_STATUS.md（项目状态文档）
- **规范** = 规范.md（工作规范）

## 📚 核心文档

### 项目管理文档
- [PROJECT_STATUS.md](PROJECT_STATUS.md) - 项目当前状态（新会话必读）
- [CHANGELOG.md](CHANGELOG.md) - 版本历史
- [规范.md](规范.md) - AI 协作规范

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

## 🏗️ 技术架构

**前端**: Next.js 14 + TypeScript + Shadcn/ui + Socket.IO
**后端**: Python 3.11+ + FastAPI + LangChain + Chroma

## 📝 开发阶段

- [x] 需求分析 (v3.0)
- [x] 架构设计 (v0.5.0)
- [x] 系统流程验证 (v2.0)
- [ ] 后端 API 接口设计
- [ ] 后端核心模块实现
- [ ] 前端 UI 实现
- [ ] 联调测试

## 📄 许可证

待定
