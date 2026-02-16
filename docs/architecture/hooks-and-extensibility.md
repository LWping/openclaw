---
summary: "Hooks 与扩展：Hook 机制、插件系统、配置扩展"
title: "Hooks and Extensibility"
---

# OpenClaw Hooks 与扩展设计

本文档分析 OpenClaw 的 Hooks 机制、插件系统、扩展点等，便于借鉴其设计思想。

## 1. Hooks 机制

### 1.1 概念

- Hook 在特定事件发生时执行自定义逻辑
- 不修改核心代码即可扩展行为
- 位于 `~/.openclaw/hooks/` 或 `hooks/bundled/`

### 1.2 内置 Hooks

| Hook              | 触发时机       | 作用                         |
|-------------------|----------------|------------------------------|
| session-memory     | `/new` 命令    | 将当前会话摘要写入 memory    |
| bootstrap-extra-files | 启动时      | 生成额外引导文件             |

### 1.3 Hook 接口

```typescript
type HookHandler = (event: {
  type: string;      // "command" | ...
  action?: string;   // "new" | ...
  sessionKey?: string;
  timestamp?: number;
  context?: Record<string, unknown>;
}) => Promise<void> | void;
```

### 1.4 调用入口

- `getGlobalHookRunner()`：获取全局 hook 执行器
- 在 command 处理、session 切换等节点调用

## 2. 插件系统

### 2.1 插件类型

- **Channel 插件**：对接新聊天渠道（extensions/telegram、discord 等）
- **Skill 插件**：提供新工具
- **Provider 插件**：新 LLM/embedding 提供商
- **通用插件**：如 diagnostics-otel、llm-task

### 2.2 注册机制

- `plugins/loader.ts`：加载 `extensions/*` 和配置的 plugin 路径
- `plugins/runtime.js`：`requireActivePluginRegistry()` 返回 `registry`
- `registry.channels`、`registry.skills` 等

### 2.3 插件结构

```
extensions/my-plugin/
  package.json    # name, peerDependencies: openclaw
  index.ts        # 导出 plugin 对象
```

- 避免 `workspace:*` 在 dependencies，用 `peerDependencies` 或 `devDependencies`
- 运行时通过 `openclaw/plugin-sdk` 解析

## 3. 配置扩展

### 3.1 动态 Key

- `ChannelsConfig` 支持 `[key: string]: any`，扩展 channel 可加自定义 key
- `ExtensionChannelConfig` 作为扩展 channel 的配置基类

### 3.2 Schema 与校验

- `config/schema.ts`、`zod-schema*.ts`：配置校验
- 扩展通过 `plugin.config` 声明自己的 schema

## 4. Gateway 扩展点

### 4.1 RPC 方法

- Gateway 暴露 `server-methods`：sessions、send、cron、config、skills 等
- 插件可注册新的 RPC 方法（通过 server 的扩展机制）

### 4.2 HTTP 插件

- `gateway/server/plugins-http.ts`：插件可挂载 HTTP 路由
- 用于 OAuth、webhook 等

## 5. 借鉴要点

1. **Hook 即事件回调**：在关键节点触发，不侵入核心流程
2. **插件即包**：每个扩展是独立 npm 包，便于版本管理
3. **Registry 集中注册**：channel、skill、provider 统一从 registry 获取
4. **配置可扩展**：动态 key + 基类，兼容新 channel 配置
5. **轻量 Dock**：共享逻辑只依赖轻量 Dock，避免加载重插件
