---
summary: "Gateway 核心：进程模型、WebSocket、启动流程"
title: "Gateway Core"
---

# OpenClaw Gateway 核心架构

本文档分析 OpenClaw Gateway 的进程模型、WebSocket、启动流程等核心架构，便于借鉴其设计思想。

## 1. Gateway 角色

- **单进程多职责**：HTTP + WebSocket + Channel Monitors + Cron + Heartbeat 同进程
- **本地优先**：默认 `gateway.mode=local`，绑定 loopback
- **可选远程**：可配置 `bind: 0.0.0.0` 供远程客户端连接

## 2. 服务组成

### 2.1 HTTP 服务

- `gateway/server-http.ts`：Express 风格 HTTP
- 健康检查、RPC 入口、静态资源、webhook 接收（Telegram 等）

### 2.2 WebSocket

- `gateway/server/ws-connection.ts`：管理客户端连接
- `message-handler.ts`：处理 RPC 消息
- 协议：JSON-RPC 风格，`method` + `params`

### 2.3 Channel Manager

- `gateway/server-channels.ts`：`createChannelManager`
- 启动时 `startChannels()` 启动所有已配置 channel
- 每个 channel 的 `startAccount` 在独立 task 中运行

### 2.4 Cron Service

- `gateway/server-cron.ts`：挂载到 Gateway
- 定时器驱动，到期执行 job
- 通过 RPC 暴露 `cron.list`、`cron.add` 等

### 2.5 Heartbeat Runner

- `infra/heartbeat-runner.ts`：定时唤醒
- 与 Gateway 生命周期绑定，`setHeartbeatWakeHandler` 注入

## 3. 启动流程

```
main / gateway run
        │
        ▼
loadConfig
        │
        ▼
createChannelManager
        │
        ▼
startChannels()  → 各 channel plugin.gateway.startAccount
        │
        ▼
startCronService
        │
        ▼
startHeartbeatRunner
        │
        ▼
HTTP listen + WebSocket upgrade
```

## 4. RPC 方法

| 方法类     | 示例                     | 说明           |
|------------|--------------------------|----------------|
| sessions   | list, send, create        | 会话管理       |
| send       | send                     | 发送消息       |
| cron       | list, add, update, run   | 定时任务       |
| config     | get, patch               | 配置读写       |
| skills     | list, update             | Skills 管理    |
| channels   | status, start, stop      | Channel 状态   |
| agent      | run                      | 执行 agent     |

## 5. 客户端连接

- **Mac 应用**：通过 WebSocket 连接本地 Gateway
- **CLI**：`openclaw message send` 等通过 HTTP 或 WebSocket
- **Mobile**：同上，可配置远程 Gateway URL

## 6. 借鉴要点

1. **单体 Gateway**：简化部署，适合个人/小团队
2. **Channel 即插件**：按需加载，失败隔离
3. **RPC 统一入口**：所有能力通过同一协议暴露，便于多端复用
4. **配置热重载**：`config-reload` 支持部分配置更新不重启
