---
summary: "记忆实现：双后端、混合检索、Session Memory Hook"
title: "Memory Design"
---

# OpenClaw 记忆实现设计

本文档分析 OpenClaw 的记忆（Memory）系统如何实现持久化、检索与注入，便于借鉴其设计思想。

## 1. 整体架构

### 1.1 记忆来源

| 来源           | 路径/位置                    | 说明                         |
|----------------|------------------------------|------------------------------|
| 主记忆文件     | `workspace/memory/MEMORY.md` | 用户/Agent 维护的长期记忆    |
| 扩展记忆目录   | `workspace/memory/*.md`       | 按主题/日期拆分的记忆文件    |
| 会话转录       | `~/.openclaw/sessions/`      | 可选纳入检索的会话历史       |
| Session Memory Hook | `/new` 命令触发         | 将当前会话摘要写入 memory   |

### 1.2 双后端支持

- **builtin**：自研 `MemoryIndexManager`，SQLite + FTS + 向量索引
- **qmd**：外部 QMD 工具，通过 `memory.qmd.command` 配置

```typescript
// config/types.memory.ts
backend?: "builtin" | "qmd";
citations?: "auto" | "on" | "off";
qmd?: { command, searchMode, paths, sessions, update, limits, scope };
```

## 2. 核心组件

### 2.1 MemoryIndexManager (builtin)

位置：`src/memory/manager.ts`

- **数据源**：`MEMORY.md`、`memory/*.md`、可选 session 转录
- **索引**：
  - **FTS**：SQLite FTS5，关键词检索
  - **Vector**：embedding 向量，支持 OpenAI、Gemini、Voyage、local
- **混合检索**：`mergeHybridResults` 结合 BM25 与向量相似度

### 2.2 检索流程

```
用户/Agent 调用 memory_search
        │
        ▼
getMemorySearchManager(cfg, agentId)
        │
        ├─ backend=qmd → QmdMemoryManager (外部进程)
        └─ backend=builtin → MemoryIndexManager
                │
                ▼
        manager.search(query, { maxResults, minScore, sessionKey })
                │
                ├─ 向量检索: searchVector (embedding + 相似度)
                ├─ 关键词检索: searchKeyword (FTS)
                └─ 混合排序: mergeHybridResults
                │
                ▼
        返回 MemorySearchResult[] (path, snippet, startLine, endLine, score)
```

### 2.3 Agent 工具

**memory_search**（`agents/tools/memory-tool.ts`）：

- 语义检索 MEMORY.md + memory/*.md（及可选 session）
- 返回 top snippets，带 path + lines
- 用于回答「之前的工作、决策、日期、人、偏好、待办」等问题

**memory_get**：

- 按 path + from/lines 安全读取片段
- 在 memory_search 之后按需拉取，控制 context 大小

### 2.4 配置解析

```typescript
// agents/memory-search.ts
resolveMemorySearchConfig(cfg, agentId) → {
  provider, remote, model, fallback, local,
  includeSessions, sessionScope, ...
}
```

- 按 agent 可配置不同 memory 策略
- 支持 OpenAI、Gemini、Voyage、local embedding

## 3. 同步与更新

### 3.1 文件监听

- `chokidar` 监听 `MEMORY.md`、`memory/*.md` 变更
- 变更后标记 `dirty`，异步执行 `sync`
- Session 转录变更通过 `sessionUnsubscribe` 订阅

### 3.2 同步流程

1. 读取文件内容，分块（chunk）
2. 计算 embedding（批量，支持 cache）
3. 写入 SQLite：`chunks_vec`（向量）、`chunks_fts`（FTS）
4. 去重、增量更新

### 3.3 QMD 后端

- 通过 `memory.qmd.command` 调用外部 QMD 进程
- 支持 `query`、`search`、`vsearch` 等模式
- 可配置 `includeDefaultMemory`、`paths`、`sessions`

## 4. Session Memory Hook

位置：`hooks/bundled/session-memory/handler.ts`

- **触发**：`/new` 命令
- **行为**：
  1. 读取当前 session 最近 N 条消息
  2. 可选调用 LLM 生成描述性 slug
  3. 创建 `memory/YYYY-MM-DD-<slug>.md`
  4. 写入会话摘要、sessionKey、source 等

- 将「对话上下文」沉淀为「可检索记忆」

## 5. 注入策略

### 5.1 Citations 模式

- **on**：始终附带引用（path#LstartLine）
- **off**：不附带
- **auto**：私聊显示，群聊/频道默认隐藏

### 5.2 字符预算

- `maxInjectedChars`：限制注入到 prompt 的记忆总字符数
- 超出时截断结果，避免 context 膨胀

## 6. 借鉴要点

1. **多源记忆**：文件 + 会话 + Hook 沉淀，统一检索入口
2. **双后端**：builtin 自包含，qmd 可扩展/替换
3. **混合检索**：FTS + 向量，兼顾关键词与语义
4. **按 agent 配置**：不同 agent 可有不同 memory 策略
5. **Session Memory Hook**：通过 `/new` 将对话沉淀为记忆，形成闭环
6. **工具化**：memory_search + memory_get 作为 Agent 工具，由 LLM 按需调用
