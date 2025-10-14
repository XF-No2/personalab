# SillyTavern Prompt 组装机制研究

**研究日期**: 2025-10-14
**目的**: 理解 SillyTavern 的 Prompt 组装机制，为 PersonaLab 提供参考

---

## 核心理解

SillyTavern 本质上是一个 **LLM 对话界面 + 灵活的 Prompt 组装引擎**。

用户通过可视化界面配置：
- 角色卡片（Character Card）
- 背景设定（Background/World Info）
- 其他预设（Presets/System Prompts）

系统将这些元素按照特定策略组装成最终发送给 LLM 的 Prompt。

---

## SillyTavern 核心组件

### 1. Story String（故事字符串）
- Prompt 的前导部分
- 使用 Handlebars 模板语法（如 `{{description}}`, `{{personality}}`）
- 放在对话历史之前

### 2. 角色卡片（Character Card）
包含字段：
- `description`: 角色描述
- `personality`: 性格特征
- `scenario`: 场景设定
- `first_mes`: 第一条消息
- `mes_example`: 示例对话

### 3. World Info（世界书）
- 基于**关键词触发**的背景信息注入系统
- 扫描最近 N 条消息，匹配关键词
- 可以设置注入位置：
  - `Before Char Defs`: 插入到角色描述之前（中等影响力）
  - `After Char Defs`: 插入到角色描述之后（更高影响力）
- 使用 `Order` 参数控制同深度元素的优先级

### 4. System Prompt
- 系统级指令
- 通常包含角色扮演规则、输出格式要求等

### 5. 对话历史
- 最近的用户-AI 对话记录
- 按时间倒序排列

---

## Prompt 组装顺序

**关键特性**: AI 从下往上阅读，注意力从下到上递减

```
[上层 - 低注意力]
├─ System Prompt
├─ Story String:
│  ├─ {{system}}
│  ├─ {{wiBefore}} (World Info - Before Char Defs)
│  ├─ {{description}} (角色描述)
│  ├─ {{personality}} (性格)
│  ├─ {{scenario}} (场景)
│  ├─ {{wiAfter}} (World Info - After Char Defs)
│  └─ {{mesExamples}} (示例对话)
├─ 早期对话历史
├─ 中期对话历史
└─ 最近对话历史 + 用户最新输入
[下层 - 高注意力]
```

**注意**: 实际顺序可以通过 Context Template 自定义

---

## World Info 的触发机制

### 关键词匹配策略
1. 系统扫描**最近 N 条消息**（N 可配置，如 10-20 条）
2. 如果消息中包含 World Info 条目的关键词，触发该条目
3. 将触发的条目按照配置的位置注入到 Prompt 中

### 示例
```json
{
  "keys": ["据点", "废土据点"],
  "content": "据点是废土中少数有防御设施的安全区，由不同帮派控制。",
  "position": "after_char",
  "order": 100
}
```

当最近对话中出现 "据点" 或 "废土据点" 时，这条信息会被注入到角色描述之后。

---

## Context Template 可定制性

SillyTavern 允许用户完全自定义 Prompt 的组装顺序：
- 通过可视化的**拖拽界面**调整各元素的位置
- 通过 `Order` 参数控制同深度元素的优先级
- 支持在特定深度（at-depth）注入内容

**核心理念**: 给用户完全的控制权，让用户根据不同模型、不同场景调整 Prompt 结构

---

## Instruct Mode

针对 Instruct 模型（如 Llama-2-Chat, Mistral-Instruct）的特殊模式：
- 将对话历史格式化为模型期望的格式
- 例如: `<s>[INST] user message [/INST] assistant response </s>`
- 自动适配不同模型的对话模板

---

## PersonaLab 的借鉴点

1. **灵活的 Prompt 组装**:
   - 采用模块化设计，各元素可独立配置
   - 支持按策略动态注入内容

2. **World Info 的改进**:
   - ST 使用关键词匹配（简单但不够智能）
   - PersonaLab 使用向量检索（Conversational RAG）
   - 更智能地找到相关历史事件

3. **注入位置的灵活性**:
   - 学习 ST 的分层注入策略
   - 根据内容重要性选择注入位置（头部/中间/尾部）

4. **用户控制权**:
   - ST 给用户很高的自由度
   - PersonaLab 也应该允许高级用户调整 Prompt 结构（通过配置文件或管理界面）

---

## 参考链接

- SillyTavern 官方文档: https://docs.sillytavern.app/
- Context Template 说明: https://docs.sillytavern.app/usage/prompts/context-template/
- World Info 说明: https://docs.sillytavern.app/usage/core-concepts/worldinfo/
- GitHub 仓库: https://github.com/SillyTavern/SillyTavern

---

**文档版本**: v1.0
**创建日期**: 2025-10-14
