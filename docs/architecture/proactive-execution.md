---
summary: "主动干活：Heartbeat、Cron Jobs、SystemEvent"
title: "Proactive Execution"
---

# OpenClaw 主动干活实现设计

本文档分析 OpenClaw 如何实现「主动干活」：定时唤醒、Cron 任务、Heartbeat 机制，便于借鉴其设计思想。

## 1. 整体机制概览

| 机制        | 用途                     | 触发方式           | 代码位置              |
|-------------|--------------------------|--------------------|-----------------------|
| Heartbeat   | 定期唤醒 agent 检查待办  | 定时轮询           | `infra/heartbeat-runner.ts` |
| Cron Jobs   | 用户定义的定时/一次性任务 | 调度器             | `cron/service/`       |
| SystemEvent | 异步事件注入（如 exec 完成） | 事件入队 + 下次 prompt | `infra/system-events.ts` |

## 2. Heartbeat 机制

### 2.1 作用

- 在**无新消息**时，按配置间隔定期「唤醒」agent
- 将 **SystemEvent**（如 exec 完成、cron 触发的 systemEvent）注入 prompt
- Agent 可主动回复用户（如「你之前的命令执行完了，结果是…」）

### 2.2 配置

```yaml
agents:
  defaults:
    heartbeat:
      every: "5m"        # 间隔
      prompt: "..."      # 唤醒时的 prompt
      ackMaxChars: 80    # HEARTBEAT_OK 类简短回复的最大字符
  list:
    - id: main
      heartbeat: { every: "3m" }  # 可覆盖
```

### 2.3 流程

```
定时器 (every N 秒)
        │
        ▼
resolveHeartbeatConfig(cfg, agentId)
        │
        ▼
peekSystemEventEntries(sessionKey)  ← 是否有待处理的 SystemEvent
        │
        ├─ 有事件 → 使用 EXEC_EVENT_PROMPT 等专用 prompt
        └─ 无事件 → 使用默认 heartbeat prompt
        │
        ▼
getReplyFromConfig → runReplyAgent (执行 agent 一轮)
        │
        ▼
解析回复:
  - HEARTBEAT_OK / 简短 ack → 可选发送 ack 或静默
  - 有实质内容 → deliverOutboundPayloads 投递到 channel
```

### 2.4 关键逻辑

- **isHeartbeatContentEffectivelyEmpty**：判断是否为「无实质内容」的 ack
- **resolveHeartbeatDeliveryTarget**：决定投递目标（last/指定 channel+to）
- **Active Hours**：可配置活跃时段，非活跃时段不发送

## 3. Cron Jobs

### 3.1 类型

| sessionTarget | payload.kind   | 说明                           |
|---------------|----------------|--------------------------------|
| main          | systemEvent    | 向主 session 注入系统事件      |
| isolated      | agentTurn      | 独立运行 agent 一轮，可投递结果 |

### 3.2 调度

- **every**：周期任务，如 `every: "0 * * * *"`（每小时）
- **at**：一次性任务，指定时间点
- `computeNextRunAtMs` 计算下次执行时间
- 定时器 `nextWakeAtMs` 取最近 due 的 job

### 3.3 Isolated Agent（主动执行并投递）

`cron/isolated-agent/run.ts`：

1. **resolveCronSession**：创建/复用 isolated session
2. **runEmbeddedPiAgent** 或 **runCliAgent**：执行 agent 一轮
3. **resolveCronDeliveryPlan**：根据 `delivery.mode`、`channel`、`to` 决定是否投递
4. **deliverOutboundPayloads**：将 agent 回复发送到 WhatsApp/Telegram 等

### 3.4 投递配置

```yaml
# Cron job 示例
sessionTarget: isolated
payload:
  kind: agentTurn
  message: "检查今日待办并简要汇报"
  deliver: true
  channel: whatsapp
  to: "+1234567890"
delivery:
  mode: announce   # none | announce | deliver
  channel: whatsapp
  to: "+1234567890"
  bestEffort: true  # 投递失败不阻塞
```

### 3.5 Cron Tool（Agent 可创建 Cron）

`agents/tools/cron-tool.ts`：

- Agent 可调用 `cron` 工具：`add`、`update`、`remove`、`run`、`list`、`status`
- 实现「你帮我每小时提醒一次」等用户意图
- 通过 Gateway RPC 与 `server-cron` 交互

## 4. SystemEvent 队列

### 4.1 作用

- 临时存储「应注入下次 prompt」的文本
- 不持久化，进程重启即清空
- 按 `sessionKey` 隔离

### 4.2 API

```typescript
enqueueSystemEvent(text, { sessionKey, contextKey? });
peekSystemEventEntries(sessionKey);  // 读取并消费
```

### 4.3 典型来源

- **Exec 完成**：异步 `exec` 命令结束后，将结果入队
- **Cron systemEvent**：`payload.kind=systemEvent` 的 cron 写入的文本
- **其他内部事件**：如需要「下次回复时带上某信息」

## 5. 协作关系

```
用户: "帮我每小时检查邮件并摘要发给我"
        │
        ▼
Agent 调用 cron 工具 add
        │
        ▼
Cron Job 创建 (sessionTarget=isolated, payload.kind=agentTurn)
        │
        ▼
定时触发 → runEmbeddedPiAgent
        │
        ▼
Agent 执行（查邮件、摘要）
        │
        ▼
deliverOutboundPayloads → WhatsApp/Telegram 发送给用户
```

```
用户: "运行这个长命令，完了告诉我"
        │
        ▼
Agent 调用 exec（异步）
        │
        ▼
Exec 完成 → enqueueSystemEvent(结果, { sessionKey })
        │
        ▼
下次 Heartbeat 或用户新消息
        │
        ▼
peekSystemEventEntries 取出事件，注入 prompt
        │
        ▼
Agent 回复 "命令执行完了，结果是..."
        │
        ▼
deliverOutboundPayloads 发送给用户
```

## 6. 借鉴要点

1. **Heartbeat + SystemEvent**：无消息时定时唤醒，通过事件队列注入上下文，实现「主动汇报」
2. **Cron 双模式**：main（注入事件）与 isolated（独立执行+投递），满足不同场景
3. **Cron Tool**：Agent 可编程式创建定时任务，形成「用户意图 → Cron → 定时执行 → 投递」闭环
4. **bestEffort 投递**：避免因 channel 不可达阻塞 cron 执行
5. **统一 Outbound**：Cron 与正常回复共用 `deliverOutboundPayloads`，复用 channel 适配逻辑
