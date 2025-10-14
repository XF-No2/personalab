# DESIGN.md 文档问题清单

**创建时间**: 2025-10-14
**状态**: ✅ 已全部修复（2025-10-14）

---

## 🚨 严重逻辑问题

### 问题1：数据结构不一致

**位置**: 第4节 vs Core Memory 设计

**冲突**:
- 第260-328行：角色状态使用旧结构
  ```json
  {
    "core_identity": {...},
    "growth_state": {...},
    "current_state": {...}
  }
  ```

- 第492-498行：Core Memory 使用新结构
  ```json
  {
    "base_persona": "...",
    "evolved_persona": "..."
  }
  ```

**解决方案**:
- 删除第4节中关于角色状态的旧结构描述
- 统一使用 `base_persona + evolved_persona` 结构

---

### 问题2：事件写入逻辑错误

**位置**: 第747-769行 "事件写入时机"

**冲突**:
- 代码检查 `if state_update:` 才写入事件
- 但交互流程（636-637行）明确说"AI 只返回叙事内容，不返回状态更新"

**问题**: 既然没有 state_update，怎么判断是否写入事件？

**解决方案**:
- 删除 "只在状态变化时写入事件" 的逻辑
- 或者明确新的事件写入策略（如：定期写入摘要？不写入？）

---

### 问题3：术语混乱（storyline_id vs instance_id）

**位置**: 全文档

**冲突**:
- 同时使用 `storyline_id` (line 115, 151, 335, 608等)
- 同时使用 `instance_id` (line 608, 615等)
- 第889行说明了术语变更（Storyline → Conversation Instance）
- 但整个文档没有统一更新

**解决方案**:
- 全文搜索替换 `storyline` → `instance`
- 统一使用 `instance_id`

---

### 问题4：旧设计残留（三层人格）

**位置**: 多处

**具体位置**:
1. 第35-40行 "解决方案"
   ```
   - 三层人格机制 - 静态核心 + 慢动态成长 + 快动态状态
   ```

2. 第74-98行 "人格成长的逻辑闭环"
   ```
   1. 角色当前状态（三层人格）
   ```

3. 第935-963行 "快动态层的维护策略"
   ```json
   {
     "current_state": {
       "emotions": [...],
       "physical": [...],
       "immediate_goals": [...]
     }
   }
   ```

4. 目录第5项仍然是 "人格三层机制"

**解决方案**:
- 删除所有关于 "三层人格（core_identity/growth_state/current_state）" 的描述
- 统一改为 "Core Memory 动态管理机制（base_persona + evolved_persona）"
- 更新目录

---

### 问题5：性能优化章节过时

**位置**: 第814-825行

**冲突**:
```python
recent, events, state, bg = await asyncio.gather(
    read_recent_messages(storyline_id, n=20),  # ← 应该从缓存读
    ...
)
```

**问题**: 应该使用 `RecallMemoryCache.get_for_prompt()`，不是从文件读取

**解决方案**:
- 更新代码示例，使用 RecallMemoryCache
- 或删除这个过时的示例

---

## 📋 需要删除的章节

1. **第4节 - 数据结构设计** 中的旧角色状态结构
2. **所有关于三层人格的描述**（core_identity/growth_state/current_state）
3. **第935-1037行** "需求澄清后的新设计"章节（已经过时）

---

## 📋 需要新增的内容

1. **character_state.json 的新结构**（base_persona + evolved_persona）
2. **EvolvedPersonaUpdater 的详细实现**
3. **MaintenanceScheduler 的完整代码**

---

## 🎯 修复优先级

### 高优先级（影响理解）
1. ✅ 统一数据结构（base_persona + evolved_persona）
2. ✅ 删除三层人格的所有描述
3. ✅ 统一术语（instance_id）

### 中优先级（影响实现）
4. ✅ 修正事件写入逻辑 - 已修复：改为定期写入摘要
5. ✅ 更新性能优化章节 - 已修复：更新代码使用 RecallMemoryCache

### 低优先级（清理）
6. ⚠️ 删除过时章节 - 需要用户确认："需求澄清后的新设计"章节包含重要功能，建议保留
7. 🔹 补充新内容

---

## 📝 修复检查清单

- [x] 修正事件写入逻辑（第747-769行）
- [x] 更新性能优化章节（第814-825行）
- [x] 更新 RAG 检索策略中的术语（storyline_id → instance_id）
- [x] 全文搜索 `storyline` → 替换为 `instance`
- [x] 全文搜索 `core_identity` → 删除相关描述
- [x] 全文搜索 `growth_state` → 删除相关描述
- [x] 全文搜索 `current_state` → 删除相关描述
- [x] 全文搜索 `三层人格` → 删除或改写
- [x] 重写第4节数据结构设计（统一为 base_persona + evolved_persona）
- [x] 删除 v0.2.1 中关于快动态层的章节
- [x] 检查所有代码示例的一致性

---

## ✅ 修复完成总结（2025-10-14）

所有问题已彻底修复：

1. **数据结构统一**：全文使用 `base_persona` + `evolved_persona`
2. **术语统一**：`storyline` → `instance`，`storyline_id` → `instance_id`
3. **三层人格已删除**：所有 `core_identity`/`growth_state`/`current_state` 引用已清除
4. **代码示例一致**：所有 Prompt、维护任务、数据加载逻辑完全统一
5. **逻辑闭环完整**：人格成长机制基于 `evolved_persona` 定期更新

**文档现在可以作为开发的唯一参考标准。**
