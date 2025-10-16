# 配置系统详细设计

> **架构上下文**：见 [DESIGN_OVERVIEW.md](../DESIGN_OVERVIEW.md) 第8章

**文档版本**: v1.0
**最后更新**: 2025-10-16

---

## 模块职责

管理软件级别的功能开关和参数，支持运行时动态配置。

---

## 全局配置文件

### 文件位置

`data/config.json`

### 配置结构

格式定义：见 [data_structure.md](data_structure.md#配置文件格式configjson)

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

---

## 配置项说明

### features.director_plot_control

**剧情大纲控制功能开关**

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `enabled` | boolean | `true` | 剧情大纲控制功能开关 |
| `rag_fallback_threshold` | number | `3` | 触发容错处理的阈值（连续多少次未更新） |

**enabled 配置项的影响**：

- `true`（启用）：
  - Prompt包含 `story_outline`（JSON格式，放在头部4K）
  - 后端扫描AI回复中的 `[PROGRESS]` 标记
  - 触发RAG兜底机制（当 `no_update_count >= rag_fallback_threshold`）

- `false`（关闭）：
  - Prompt不包含 `story_outline`
  - 不扫描 `[PROGRESS]` 标记
  - 不触发RAG兜底机制
  - AI自由发挥，不受剧情大纲约束

**rag_fallback_threshold 配置项**：
- 定义连续多少次AI回复没有输出 `[PROGRESS]` 标记时，触发容错处理
- 默认值：3（连续3次未更新）
- 详见：[director.md](director.md) 第3节"容错处理（RAG兜底）"

---

### summary

**汇总功能配置**

| 配置项 | 类型 | 可选值 | 说明 |
|--------|------|--------|------|
| `order` | string | `summary_first`, `last_n_first` | 汇总后新会话的初始内容顺序 |
| `last_n_turns` | number | - | 汇总后新会话保留多少轮对话 |

**order 配置项**：

- `"summary_first"`（推荐）：
  ```jsonl
  {"type":"metadata",...}
  {"type":"summary","content":"摘要1"}
  {"type":"summary","content":"摘要2"}
  {"role":"user","content":"...","turn":1}
  {"role":"assistant","content":"...","turn":1}
  ...（最后N轮对话）
  ```
  - 优势：AI首先看到剧情概要，再看到具体对话

- `"last_n_first"`：
  ```jsonl
  {"type":"metadata",...}
  {"role":"user","content":"...","turn":1}
  {"role":"assistant","content":"...","turn":1}
  ...（最后N轮对话）
  {"type":"summary","content":"摘要1"}
  {"type":"summary","content":"摘要2"}
  ```
  - 优势：AI首先看到最近的对话细节，再看到历史概要

**last_n_turns 配置项**：
- 定义汇总后新会话保留多少轮对话
- 默认值：5
- 建议范围：3-10（根据对话长度调整）

---

### conversation_file

**会话文件配置**

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `load_all` | boolean | `true` | 是否全量读取会话文件 |
| `max_tokens` | number | `100000` | 会话文件的最大token数（预留） |

**load_all 配置项**：
- `true`（默认）：全量读取当前会话的所有对话
- `false`（预留）：未来可能支持滑动窗口机制（当前版本不支持）

**max_tokens 配置项**：
- 预留配置，用于未来限制会话文件大小
- 当前版本不使用此配置

---

## 前端控制

### 设置页面

前端提供设置页面，允许用户开关功能：

**UI示例**：
```
⚙️ 功能设置

[ ] 剧情大纲控制
    启用后，AI将遵循背景设定中的故事大纲推进剧情

    高级选项：
    - 容错阈值: [3] 轮

[v] 汇总功能
    初始内容顺序: [摘要优先 ▾]
    保留对话轮数: [5] 轮

[v] 全量读取会话
```

### 配置修改流程

```
用户修改配置
  ↓
[1] 前端发送API请求
  POST /api/config/update
  Body: {"features.director_plot_control.enabled": false}
  ↓
[2] 后端更新 config.json
  ↓
[3] 后端重新加载配置文件
  ↓
[4] 返回成功响应
  ↓
[5] 前端显示更新成功提示
```

**关键特性**：
- 配置变更立即生效，不需要重启服务
- 使用点路径语法修改嵌套配置（如 `features.director_plot_control.enabled`）

---

## 配置加载机制

### 后端加载流程

```python
# 启动时加载配置
def load_config():
    with open("data/config.json", "r", encoding="utf-8") as f:
        config = json.load(f)
    return config

# 配置修改后重新加载
def reload_config():
    global config
    config = load_config()
```

### 配置缓存

- 后端在内存中缓存配置，避免每次请求都读取文件
- 配置修改后，重新加载配置文件并更新缓存

---

## 与其他模块的交互

### 依赖：导演模块

- 读取 `features.director_plot_control.enabled` 判断是否启用剧情大纲控制
- 读取 `features.director_plot_control.rag_fallback_threshold` 判断何时触发容错处理
- 详见：[director.md](director.md)

### 依赖：汇总功能

- 读取 `summary.order` 确定新会话的初始内容顺序
- 读取 `summary.last_n_turns` 确定保留多少轮对话
- 详见：[DESIGN_OVERVIEW.md](../DESIGN_OVERVIEW.md) 第3.2章"汇总功能流程"

### 依赖：Prompt组装模块

- 根据 `features.director_plot_control.enabled` 决定是否在Prompt中包含 `story_outline`
- 详见：[DESIGN_OVERVIEW.md](../DESIGN_OVERVIEW.md) 第3.5章"Prompt组装策略"

---

## 扩展性

未来可扩展的配置项：

1. **RAG检索配置**：
   - `rag.retrieval_count`: 日常对话RAG检索返回多少条
   - `rag.enable_deep_search`: 是否启用深度搜索

2. **人格更新配置**：
   - `persona.auto_suggest`: 是否自动提示用户更新记忆
   - `persona.version_history`: 是否保存人格版本历史

3. **性能配置**：
   - `performance.max_concurrent_requests`: 最大并发请求数
   - `performance.stream_chunk_size`: 流式输出的块大小

4. **LLM配置**：
   - `llm.model`: 使用的模型名称
   - `llm.temperature`: 生成温度
   - `llm.max_tokens`: 最大生成token数

---

**文档版本**: v1.0
**最后更新**: 2025-10-16
