---
summary: "外部聊天工具对接：插件化、Monitor、Outbound、Bindings 路由"
title: "Channel Integration"
---

# OpenClaw 外部聊天工具对接设计

本文档分析 OpenClaw 如何对接 Telegram、WhatsApp、Discord、Slack 等外部聊天工具，便于借鉴其设计思想。

## 1. 整体架构

### 1.1 分层设计

```
┌─────────────────────────────────────────────────────────────┐
│  Gateway (server-channels.ts)                               │
│  - ChannelManager: 生命周期管理、启动/停止、状态快照         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Channel Plugins (extensions/*)                              │
│  - 每个 channel 一个插件包 (telegram, discord, whatsapp...)  │
│  - 实现 ChannelPlugin 接口，注册到 plugin registry           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Channel Dock (channels/dock.ts)                            │
│  - 轻量级元数据：capabilities, allowFrom, 分块策略等          │
│  - 供共享逻辑使用，不加载 monitor/probe 等重逻辑             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Monitor + Outbound                                          │
│  - Monitor: 监听入站消息，解析后路由到 agent                 │
│  - Outbound: 将 agent 回复投递到各 channel 的 send 函数       │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 核心概念

- **ChannelPlugin**：插件接口，包含 `id`、`meta`、`config`、`gateway` 等
- **ChannelDock**：轻量级 channel 元数据，供 reply flow、command auth 等共享逻辑使用
- **ChannelManager**：统一管理所有 channel 的启动/停止，维护运行时状态

## 2. 插件注册与发现

### 2.1 插件注册

Channel 以 **extension 包** 形式存在，位于 `extensions/` 目录：

- `extensions/telegram/`、`extensions/discord/`、`extensions/whatsapp/` 等
- 通过 `plugins/runtime.js` 的 `requireActivePluginRegistry()` 获取 `registry.channels`
- `listChannelPlugins()` 返回去重、排序后的插件列表

### 2.2 ChannelPlugin 接口要点

```typescript
// channels/plugins/types.plugin.ts 核心结构
type ChannelPlugin = {
  id: ChannelId;           // 如 "telegram", "discord"
  meta: ChannelMeta;       // order, displayName 等
  config: ChannelConfigAdapter;   // listAccountIds, resolveAccount, isConfigured
  gateway?: ChannelGatewayAdapter; // startAccount, stopAccount
  // 可选: outbound, commands, status, directory, pairing...
};
```

- **config**：负责从 `OpenClawConfig` 解析该 channel 的账号配置
- **gateway**：负责 `startAccount`/`stopAccount`，即实际启动/停止监听

## 3. 入站流程 (Monitor)

### 3.1 消息监听

每个 channel 有独立的 **Monitor** 实现：

| Channel   | Monitor 位置              | 技术栈              |
|-----------|---------------------------|---------------------|
| Telegram  | `telegram/bot.ts`         | grammY + webhook/polling |
| WhatsApp  | `web/auto-reply/monitor.ts` | WhatsApp Web (puppeteer) |
| Discord   | `discord/monitor/`        | discord.js          |
| Slack     | `slack/monitor/`          | @slack/bolt         |

### 3.2 消息到 Agent 的路径

1. **Monitor 收到消息** → 解析为 `WebInboundMsg` / `TelegramMessage` 等统一结构
2. **resolveAgentRoute**：根据 `channel`、`accountId`、`peer`（群/私聊）、`guildId`/`teamId` 等，通过 **bindings** 解析出 `agentId` 和 `sessionKey`
3. **getReplyFromConfig**：获取该 session 的 reply 函数（执行 agent 对话）
4. **enqueueSystemEvent**：如有系统事件（如 exec 完成），先入队，下次 heartbeat 时注入

### 3.3 路由 (resolve-route.ts)

- **bindings**：配置中 `bindings[]` 定义 channel+account+peer/guild/team/roles 到 agent 的映射
- **匹配优先级**：`binding.peer` > `binding.peer.parent` > `binding.guild+roles` > `binding.guild` > `binding.team` > `binding.account` > `binding.channel` > `default`
- **sessionKey**：由 `buildAgentSessionKey` 生成，用于会话隔离和持久化

## 4. 出站流程 (Outbound)

### 4.1 统一投递入口

`infra/outbound/deliver.ts` 的 `deliverOutboundPayloads` 是核心：

- 接收 `ReplyPayload[]`（文本、媒体、reaction 等）
- 根据 `sessionKey` 解析出 `channel`、`accountId`、`to` 等投递目标
- 通过 `loadChannelOutboundAdapter` 加载各 channel 的 `sendPayload` 实现

### 4.2 各 Channel 的 Send 函数

```typescript
// OutboundSendDeps 中注入
sendWhatsApp?: typeof sendMessageWhatsApp;
sendTelegram?: typeof sendMessageTelegram;
sendDiscord?: typeof sendMessageDiscord;
sendSlack?: typeof sendMessageSlack;
sendSignal?: typeof sendMessageSignal;
sendIMessage?: typeof sendMessageIMessage;
sendMatrix?: SendMatrixMessage;
sendMSTeams?: ...
```

- 每个 channel 提供 `send*(to, text, opts)` 形式的函数
- 支持分块（`chunkByParagraph`、`chunkMarkdownTextWithMode`）、媒体、replyToId 等

### 4.3 分块与限制

- `resolveTextChunkLimit`：按 channel 配置限制单条消息长度（如 Telegram 4000）
- `blockStreaming`：部分 channel 不支持流式，需 coalesce 后一次性发送

## 5. 配置结构

### 5.1 ChannelsConfig

```yaml
channels:
  defaults:
    groupPolicy: mention | always
    heartbeat: { showOk, showAlerts, useIndicator }
  whatsapp:
    accounts:
      default: { ... }
  telegram:
    accounts:
      default: { token, allowFrom, ... }
  discord:
    accounts:
      default: { token, ... }
  # 扩展 channel 使用动态 key
```

### 5.2 Bindings（路由绑定）

```yaml
bindings:
  - agentId: my-agent
    match:
      channel: telegram
      accountId: "*"
      peer: { kind: group, id: "-100123" }
  - agentId: support-bot
    match:
      channel: discord
      guildId: "123"
      roles: ["support-role-id"]
```

## 6. 借鉴要点

1. **插件化**：每个 channel 独立 extension，通过 registry 注册，便于扩展新 channel
2. **Dock 与 Plugin 分离**：Dock 保持轻量，共享逻辑只依赖 Dock；Plugin 承载重逻辑
3. **统一入站/出站抽象**：Monitor 产出统一结构，Outbound 接收统一 Payload，便于多 channel 复用
4. **Bindings 路由**：灵活的多维匹配（channel/account/peer/guild/roles），支持复杂路由
5. **SessionKey 设计**：`agentId + channel + accountId + peer` 组合，支持 DM 与群聊隔离
