# AI 操作手册

**文件路径**: `.project_management/ai_operation_manual.md`
**用途**: AI 执行标准化操作的流程手册
**使用方式**: 用户说"更新状态"、"发布版本"、"归档"时，阅读本文件对应章节
**最后更新**: 2025-10-16 23:15

---

## 📖 文档导航

**你现在在**: `ai_operation_manual.md`（AI 操作手册）

**其他项目管理文件**:
- **README.md** - 项目入口（根目录）
- **current_status.md** - 项目当前状态（.project_management/current_status.md）
- **project_conventions.md** - 项目约定（.project_management/project_conventions.md）

---

## 🤖 使用说明

当用户说以下指令时，查阅本文件对应章节：

| 用户指令 | 对应章节 | 说明 |
|---------|---------|------|
| `更新状态` 或 `按照规范更新状态` | [更新状态](#更新状态) | 更新 current_status.md |
| `发布版本` 或 `按照规范发布` | [发布版本](#发布版本) | 版本发布流程 |
| `归档` 或 `按照规范归档` | [归档](#归档) | 归档当前版本 |
| `重构` 或 `按照规范重构` | [重构](#重构) | 代码重构流程 |

---

## 更新状态

**触发指令**: "更新状态" 或 "按照规范更新状态"

**执行步骤**:
1. 回顾对话历史，找出最近的变更操作
2. 总结变更内容（2-5条）
3. 更新 `.project_management/current_status.md`:
   - 添加到"最近变更（最近 5 次）"章节
   - 格式:
     ```markdown
     ### {日期} {时间} - {变更标题} ✅

     **背景**:
     （说明为什么要做这个变更）

     **调整内容**:
     1. 变更1
     2. 变更2
     ...

     **效果**:
     - ✅ 效果1
     - ✅ 效果2
     ```
   - 如果涉及模块状态变化，更新"当前状态"章节
   - 更新"进行中"章节（标记已完成的项）
   - **【强制检查】更新"📋 待办事项"章节**：
     * 是否有新的待办事项需要添加？（包含具体子问题）
     * 是否有待办事项已完成需要标记为完成？
     * 是否有待办事项需要展开详细的讨论点？
     * **重要**：待办事项要写详细，新会话时要能看懂要做什么
   - 更新文件顶部的"最后更新"时间（格式：YYYY-MM-DD HH:MM）
4. **Git 操作（强制状态检查）**:
   - 执行 `git add .` 添加所有更改
   - 执行 `git commit` 创建提交
   - 提交信息格式:
     ```
     更新项目状态

     {变更摘要}

     🤖 Generated with [Claude Code](https://claude.com/claude-code)

     Co-Authored-By: Claude <noreply@anthropic.com>
     ```
   - 执行 `git -c http.proxy=http://127.0.0.1:8765 -c https.proxy=http://127.0.0.1:8765 push` 推送到远程仓库
   - **必须检查推送结果**：
     * 如果推送成功：显示 "✅ Git 推送成功"
     * 如果推送失败：显示 "❌ Git 推送失败：[错误信息]" 并报告异常
   - 执行 `git status` 验证最终状态
5. **明确反馈更新状态**:
   - ✅ 成功：显示 "📝 项目状态已更新并推送到 GitHub"
   - ❌ 失败：显示 "⚠️ 状态更新失败，Git 推送未成功，本地文件已更新但未同步到远程"

---

## 发布版本

**触发指令**: "发布版本" 或 "按照规范发布"

**执行步骤**:
1. 读取 `.project_management/current_status.md` 顶部的"当前版本"和"最近变更"章节
2. 推断下一个版本号（语义化版本）:
   - 新功能 → minor版本 (X.Y.Z → X.(Y+1).0)
   - 仅修复 → patch版本 (X.Y.Z → X.Y.(Z+1))
   - 破坏性变更 → major版本 (X.Y.Z → (X+1).0.0)
3. 更新 `.project_management/current_status.md` 顶部的"当前版本"为新版本号
4. 更新 `CHANGELOG.md`:
   - 添加新版本条目
   - 日期、版本号、变更内容
5. 执行"归档"（归档新版本）
6. **清理临时文件**:
   - 检查 `docs/` 下的临时问题文档（`*_ISSUES.md`, `*_FIXES_NEEDED.md`, `*_PROBLEMS.md` 等）
   - 如果标记为"✅ 已修复/已解决"：
     * 创建目录: `docs/archive/v{版本号}/issues/`
     * 移动这些文件到归档目录的 issues/ 子文件夹
   - 如果是活跃的问题文档（如 `QUESTIONS.md` 仍有未解决问题）：保留在 `docs/`
   - 检查 `temp/` 目录，询问用户是否需要保留临时文件
7. **归档历史变更记录**:
   - 在步骤5的归档目录中，已包含完整的 `current_status.md`（含历史记录）
   - 清理当前 current_status.md 文件的"最近变更"章节，只保留最近 5 次变更
   - 将清理出的历史记录追加到 `docs/PROJECT_HISTORY.md`
8. 生成发布报告: `docs/reports/v{新版本号}_release_report.md`
9. 发布报告内容:
   - 版本号、日期
   - 主要变更
   - 功能列表
   - 已知问题
10. **Git 操作（强制状态检查）**:
   - 执行 `git add .` 添加所有更改
   - 执行 `git commit` 创建发布提交
   - 提交信息格式:
     ```
     发布 v{版本号}

     {主要变更摘要}

     🤖 Generated with [Claude Code](https://claude.com/claude-code)

     Co-Authored-By: Claude <noreply@anthropic.com>
     ```
   - 执行 `git -c http.proxy=http://127.0.0.1:8765 -c https.proxy=http://127.0.0.1:8765 push` 推送到远程仓库
   - **必须检查推送结果**：
     * 如果推送成功：显示 "✅ Git 推送成功"
     * 如果推送失败：显示 "❌ Git 推送失败：[错误信息]" 并停止后续操作
   - 执行 `git status` 验证最终状态
11. **明确反馈发布状态**:
   - ✅ 成功：显示 "🎉 版本 v{版本号} 发布完成并已推送到 GitHub"
   - ❌ 失败：显示 "⚠️ 版本发布失败，Git 推送未成功，请检查网络连接"

**必须遵守**:
- 发布报告位置: `docs/reports/`
- 历史归档：完整版本归档到 `docs/archive/v{版本号}/`，清理后的历史追加到 `docs/PROJECT_HISTORY.md`

---

## 归档

**触发指令**: "归档" 或 "按照规范归档"

**执行步骤**:
1. 读取 `.project_management/current_status.md` 顶部的"当前版本"（格式：`**当前版本**: vX.Y.Z`）
2. 创建目录: `docs/archive/v{当前版本}/`
3. 复制以下文件到归档目录:
   - `.project_management/current_status.md`
   - `.project_management/ai_operation_manual.md`
   - `.project_management/project_conventions.md`
   - `README.md`
   - `CHANGELOG.md`
   - `docs/DESIGN_OVERVIEW.md`（如果存在）
   - `docs/REQUIREMENTS.md`（如果存在）
   - （其他项目核心文档根据需要添加）
4. 确认所有文件复制完成
5. 告知用户归档完成（包括版本号）

**必须遵守**:
- 归档是快照，不要修改内容
- 版本号格式: `vX.Y.Z`
- 归档位置: `docs/archive/v{版本号}/`

---

## 重构

**触发指令**: "重构" 或 "按照规范重构"

**执行步骤**:
1. 读取 `docs/DESIGN_OVERVIEW.md`，找到"核心功能清单"或类似章节
2. 列出现有功能列表
3. 创建检查表: `temp/reports/refactor_checklist_{日期}.md`
4. 检查表格式:
   ```markdown
   # 重构功能检查表

   生成时间: {时间}
   重构目标: {说明}

   ## 功能清单
   - [ ] 功能1
   - [ ] 功能2
   ...
   ```
5. 告知用户检查表已创建，等待逐项实现
6. 全部完成后生成对比报告: `temp/reports/refactor_report_{日期}.md`

**必须遵守（配置保护）**:
- 绝对不要删除/覆盖配置文件（如 `config.json`、`llm_config.json` 等，根据项目具体定义）
- 绝对不要删除运行时数据目录（如 `.sessions/`、`.cache/` 等，根据项目具体定义）
- 临时文件放 `temp/`

---

## Git 操作强制规范

**适用范围**: 所有涉及 Git 推送的规范操作

**网络环境配置**:
- 本项目使用 Clash for Windows 作为代理工具
- HTTP代理端口：`8765`
- 推送命令：`git -c http.proxy=http://127.0.0.1:8765 -c https.proxy=http://127.0.0.1:8765 push`
- 确保 Clash 已开启 "Allow LAN"（允许局域网连接）

**强制要求**:
1. **推送结果必须检查**：
   - 每次 `git push` 后必须检查返回码
   - 推送成功：显示 "✅ Git 推送成功"
   - 推送失败：显示 "❌ Git 推送失败：[具体错误信息]"

2. **失败处理**：
   - 推送失败时，停止后续操作
   - 明确告知用户推送失败及原因
   - 提供解决建议（检查网络、代理配置等）

3. **状态验证**：
   - 使用 `git status` 验证最终状态
   - 确认是否有未推送的提交

4. **用户反馈**：
   - 成功：用绿色 ✅ 标记
   - 失败：用红色 ❌ 标记
   - 警告：用黄色 ⚠️ 标记

**禁止行为**:
- ❌ 推送失败后继续执行其他操作
- ❌ 不检查推送结果就报告"完成"
- ❌ 模糊的状态反馈（如"可能成功"）

---

## 规范模板管理

### 更新规范模板

**触发指令**: "更新规范模板" 或 "按照规范更新规范模板"

**执行步骤**:
1. 读取当前项目的 `.project_management/ai_operation_manual.md`
2. 覆盖写入到 `d:/AI_Projects/.project_templates/ai_operation_manual.md`
3. 读取当前项目的 `.project_management/project_conventions.md`
4. 覆盖写入到 `d:/AI_Projects/.project_templates/project_conventions.md`
5. 提示：已将当前项目的规范更新到全局模板

**使用场景**:
- 当前项目改进了某个规范
- 希望其他项目也使用这个改进后的版本

### 下载规范模板

**触发指令**: "下载规范模板" 或 "按照规范下载规范模板"

**执行步骤**:
1. 读取 `d:/AI_Projects/.project_templates/ai_operation_manual.md`
2. 备份当前项目的 `.project_management/ai_operation_manual.md` 到 `ai_operation_manual.md.backup`
3. 覆盖当前项目的 `.project_management/ai_operation_manual.md`
4. 读取 `d:/AI_Projects/.project_templates/project_conventions.md`
5. 备份当前项目的 `.project_management/project_conventions.md` 到 `project_conventions.md.backup`
6. 覆盖当前项目的 `.project_management/project_conventions.md`
7. 提示：已从全局模板下载最新规范，旧版本备份为 `.backup` 文件

**使用场景**:
- 其他项目更新了规范模板
- 当前项目希望同步最新标准

---

**文档版本**: v2.0
**创建日期**: 2025-10-16 23:15
**最后更新**: 2025-10-16 23:15
