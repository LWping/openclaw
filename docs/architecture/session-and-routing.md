---
summary: "会话与路由：SessionKey、Bindings、Session Store"
title: "Session and Routing"
---

# OpenClaw 会话与路由设计

本文档分析 OpenClaw 的会话隔离、SessionKey 设计、Bindings 路由等，便于借鉴其设计思想。

## 1. SessionKey 设计

### 1.1 结构

`sessionKey` 是会话的唯一标识，用于：

- 会话文件路径（`~/.openclaw/sessions/<key>.json`）
- 转录文件（`~/.openclaw/sessions/<key>.jsonl`）
- 并发控制、记忆作用域等

### 1.2 生成逻辑

`routing/session-key.ts`：

```typescript
buildAgentSessionKey({
  agentId,
  channel,
  accountId,
  peer,        // { kind: "direct"|"group"|"channel", id }
  dmScope,     // main | per-peer | per-channel-peer | per-account-channel-peer
  identityLinks
}) → string
```

- **main**：同一 channel+account 下所有对话共用一个 session
- **per-peer**：每个 peer（用户/群）独立 session
- **per-channel-peer**：channel+peer 组合
- **per-account-channel-peer**：channel+account+peer 组合

### 1.3 示例

- DM：`agent:main:telegram:default:direct:12345`
- 群聊：`agent:main:telegram:default:group:-100123`
- 线程：`agent:main:slack:default:thread:C123_T456`

## 2. Bindings 路由

### 2.1 配置结构

```yaml
bindings:
  - agentId: support-bot
    match:
      channel: discord
      accountId: "*"
      guildId: "guild-123"
      roles: ["support-role-id"]
  - agentId: personal
    match:
      channel: telegram
      peer: { kind: direct, id: "12345" }
  - agentId: default
    match:
      channel: telegram
      accountId: "*"
```

### 2.2 匹配优先级

`resolve-route.ts` 中的 tiers（从高到低）：

1. **binding.peer**：精确匹配 peer
2. **binding.peer.parent**：线程场景，匹配父 peer
3. **binding.guild+roles**：Discord guild + 角色
4. **binding.guild**：仅 guild
5. **binding.team**：Slack team
6. **binding.account**：指定 account（非 `*`）
7. **binding.channel**：仅 channel（accountId=`*`）
8. **default**：使用 `agents.defaults.id` 或第一个 agent

### 2.3 输入到输出的映射

```
输入: channel, accountId, peer, guildId, teamId, memberRoleIds
        │
        ▼
getEvaluatedBindingsForChannelAccount(cfg, channel, accountId)
        │
        ▼
按 tier 顺序匹配 → 选中 binding.agentId
        │
        ▼
输出: agentId, sessionKey, mainSessionKey, matchedBy
```

## 3. Session Store

### 3.1 结构

`config/sessions/`：

- **SessionEntry**：`sessionId`、`sessionFile`、`updatedAt`、`spawnedBy`、`queueMode` 等
- **SessionStore**：按 `sessionKey` 索引的 Map
- 持久化到 `~/.openclaw/sessions/sessions.json`

### 3.2 关键字段

- `sessionFile`：转录文件路径
- `spawnedBy`：子 session 的父 sessionKey
- `spawnDepth`：子 agent 深度
- `queueMode`：steer、followup、collect、queue 等
- `lastHeartbeatText`：上次发送的 heartbeat 内容，用于去重

## 4. 会话生命周期

### 4.1 创建

- 首次消息到达 → `resolveAgentRoute` → `buildAgentSessionKey`
- `loadSessionStore` / `updateSessionStore` 创建或更新 SessionEntry
- 转录文件按需创建

### 4.2 转录管理

- 每条消息/回复追加到 `sessionFile`（JSONL）
- Compaction：超长时截断、摘要，见 `session-management-compaction`
- `resolveSessionTranscriptPath` 解析路径

### 4.3 隔离

- 不同 `sessionKey` 完全隔离：不同转录、不同 queue、不同 heartbeat 目标
- `identityLinks`：可配置跨 session 的身份关联（如同一用户的 Telegram 与 WhatsApp）

## 5. 借鉴要点

1. **SessionKey 即路由结果**：一次 `resolveAgentRoute` 得到 agentId + sessionKey，贯穿整个会话
2. **多维 Bindings**：channel/account/peer/guild/team/roles 灵活组合，支持企业多 bot 场景
3. **dmScope 可配置**：平衡「全局记忆」与「隐私隔离」
4. **Session Store 轻量**：只存元数据，转录单独文件，便于备份与迁移
