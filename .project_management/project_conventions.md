# 项目约定

**文件路径**: `.project_management/project_conventions.md`
**用途**: 项目的编码规范、目录结构、时间格式、设计原则等约定
**最后更新**: 2025-10-16 23:15

---

## 📖 文档导航

**你现在在**: `project_conventions.md`（项目约定）

**其他项目管理文件**:
- **README.md** - 项目入口（根目录）
- **current_status.md** - 项目当前状态（.project_management/current_status.md）
- **ai_operation_manual.md** - AI 操作手册（.project_management/ai_operation_manual.md）

---

## 目录结构规范

**规则说明**：
1. **源代码** - 存放在 `frontend/`、`backend/` 目录
2. **测试代码** - 正式测试放 `tests/`，临时测试放 `temp/tests/`
3. **文档** - 核心文档放根目录，详细文档放 `docs/`
4. **归档** - 版本归档放 `docs/archive/v{版本号}/`
5. **临时文件** - 统一放 `temp/` 目录，不提交Git
6. **配置文件** - 根目录放置（如 `config.json`、`llm_config.json` 等）
7. **运行时数据** - 放在 `data/` 目录

**标准项目目录示例**:
```
personalab/
├── .project_management/      # 项目管理文件
│   ├── current_status.md     # 项目当前状态
│   ├── ai_operation_manual.md # AI 操作手册
│   └── project_conventions.md # 项目约定（本文件）
├── frontend/                 # 前端源码
├── backend/                  # 后端源码
├── data/                     # 运行时数据
├── docs/                     # 技术文档
│   ├── archive/             # 版本归档
│   ├── reports/             # 过程报告
│   ├── DESIGN_OVERVIEW.md   # 架构设计文档
│   └── REQUIREMENTS.md      # 需求文档
├── temp/                     # 临时文件（不提交Git）
│   ├── reports/             # 临时报告
│   └── tests/               # 临时测试
├── tests/                    # 正式测试
├── README.md                 # 项目门户
└── CHANGELOG.md              # 版本历史
```

**文件位置规范**:

| 文件类型 | 存放位置 | 说明 |
|---------|---------|------|
| 项目管理文件 | `.project_management/` | 项目状态、操作手册、约定 |
| 临时测试文件 | `temp/tests/` | 不提交Git |
| 临时报告文件 | `temp/reports/` | 分析、对比等临时报告 |
| 版本发布报告 | `docs/reports/` | 正式报告，提交Git |
| 归档文档 | `docs/archive/v{版本号}/` | 版本发布时的文档快照 |
| 核心文档 | 项目根目录 | README.md, CHANGELOG.md |
| 技术文档 | `docs/` | 设计文档、需求文档 |
| 源代码 | `frontend/`, `backend/` | 按项目语言组织子目录 |
| 正式测试 | `tests/` | 单元测试、集成测试 |
| 运行时数据 | `data/` | 角色库、背景库、会话数据 |

**配置文件规范**:
- 根据项目需求定义配置文件名和内容
- 敏感配置（API密钥等）必须添加到 `.gitignore`
- 代码中通过专门的配置加载函数读取
- **重构时必须保护配置文件，绝对不要删除**

**运行时数据目录规范**:
- 本项目运行时数据放在 `data/` 目录
- 具体结构：
  ```
  data/
  ├── characters/           # 角色库（全局）
  ├── backgrounds/          # 背景库（全局）
  ├── instances/            # 会话实例
  ├── event_library/        # 全局事件库（Chroma向量数据库）
  └── prompt_templates/     # Prompt模板
  ```
- **重构时必须保护这些目录，绝对不要删除**

---

## 文档时间格式规范

**适用范围**: 所有 Markdown 文档

**时间格式标准**:
- 所有文档中的时间必须精确到**分钟**
- 标准格式：`YYYY-MM-DD HH:MM`（24小时制）
- 示例：`2025-10-16 23:15`

**适用字段**:
- 文档头部的 `**最后更新**:`
- 文档头部的 `**创建日期**:`
- CHANGELOG.md 中的版本发布时间
- current_status.md 中的变更记录时间

**示例**:
```markdown
# 文档标题

**文件路径**: `.project_management/current_status.md`
**文档版本**: v1.0
**创建日期**: 2025-10-15 14:30
**最后更新**: 2025-10-16 23:15

...
```

```markdown
### 2025-10-16 23:15 - 重构项目管理文件体系 ✅

**背景**:
...

**调整内容**:
1. 拆分 DESIGN_OVERVIEW.md
2. 创建 5 个模块文档
```

**禁止格式**:
- ❌ `2025-10-16`（只到日期）
- ❌ `2025-10-16 23:15:35`（精确到秒，过于冗余）
- ❌ `10月16日 23:15`（使用中文）

---

## 设计原则（关键）

**绝对禁止量化**：
- ❌ 不使用 priority, intensity, level, score 等数字
- ❌ 不使用"每N轮-1"的降级机制
- ❌ 不使用数字阈值判断删除

**纯定性描述**：
- ✅ 所有内容都是文本描述
- ✅ 用 timestamp 记录时间
- ✅ 用定期维护任务清理数据

**示例**:
```json
// ❌ 错误：使用量化
{
  "emotion": "happy",
  "intensity": 8,
  "priority": 3
}

// ✅ 正确：纯定性描述
{
  "emotion": "欣喜若狂，兴奋得难以抑制",
  "timestamp": "2025-10-16T23:15:00Z"
}
```

---

## 配置文件保护规范

**重构时必须遵守**:
- ❌ 绝对不要删除/覆盖配置文件（如 `config.json`、`llm_config.json` 等）
- ❌ 绝对不要删除运行时数据目录（如 `data/` 目录及其子目录）
- ✅ 敏感配置添加到 `.gitignore`
- ✅ 配置文件变更前先备份

**保护清单**（本项目）:
- `data/` 目录及其所有子目录和文件
- 配置文件（如有）
- `.gitignore` 文件

---

## Git 配置规范

**代理配置**:
- 本项目使用 Clash for Windows 作为代理工具
- HTTP 代理端口：`8765`
- 推送命令：`git -c http.proxy=http://127.0.0.1:8765 -c https.proxy=http://127.0.0.1:8765 push`

**Git 操作规范**:
- 所有 Git 推送操作必须检查结果
- 推送失败时停止后续操作
- 使用 `git status` 验证最终状态

详细操作流程请查阅 [ai_operation_manual.md](ai_operation_manual.md#git-操作强制规范)

---

## 版本号规范

**语义化版本控制**:
- 格式：`vX.Y.Z`
- **X (major)**: 破坏性变更，不兼容的 API 修改
- **Y (minor)**: 新增功能，向后兼容
- **Z (patch)**: Bug 修复，向后兼容

**示例**:
- `v0.5.0` → `v0.6.0`：新增功能（如新增模块）
- `v0.5.0` → `v0.5.1`：Bug 修复
- `v0.5.0` → `v1.0.0`：重大架构变更

---

## 文档命名规范

**项目管理文件**:
- 使用小写字母 + 下划线：`current_status.md`, `ai_operation_manual.md`
- 放在 `.project_management/` 目录

**技术文档**:
- 使用大写字母 + 下划线：`DESIGN_OVERVIEW.md`, `REQUIREMENTS.md`
- 放在 `docs/` 目录

**临时文件**:
- 使用小写字母 + 日期：`refactor_checklist_20251016.md`
- 放在 `temp/` 目录

---

## Markdown 格式规范

**标题层级**:
- H1 (`#`)：文档标题（每个文件只有一个）
- H2 (`##`)：主要章节
- H3 (`###`)：子章节
- H4 (`####`)：详细说明

**代码块**:
- 必须指定语言：\`\`\`python, \`\`\`json, \`\`\`bash
- 不指定语言时使用：\`\`\`markdown

**列表**:
- 有序列表：用于步骤说明
- 无序列表：用于功能清单
- 任务列表：用于待办事项 `- [ ]` / `- [x]`

**强调**:
- **重要内容**：使用 `**粗体**`
- ✅ / ❌ / ⚠️：使用 emoji 标记状态
- `代码片段`：使用反引号

---

**文档版本**: v2.0
**创建日期**: 2025-10-16 23:15
**最后更新**: 2025-10-16 23:15
