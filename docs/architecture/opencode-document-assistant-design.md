---
summary: "基于 OpenCode 的智能文档助手设计：主动干活、记忆、外部聊天集成"
title: "OpenCode Document Assistant Design"
---

# 基于 OpenCode 的智能文档助手设计

本文档描述如何在基于 OpenCode 的智能文档助手中实现 OpenClaw 风格的三大能力：主动干活、记忆（越用越聪明）、外部聊天工具集成。便于将 OpenClaw 的设计思想迁移到 OpenCode 生态。

## 1. 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│  外部聊天 (Telegram / WhatsApp / Discord / ...)                   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  网关服务 (Node/Python)                                          │
│  - 接收消息、解析用户意图、维护会话映射                            │
│  - 调用 OpenCode SDK/API 执行 agent                               │
│  - 将回复发回聊天渠道                                             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  OpenCode + 插件                                                 │
│  - opencode-supermemory (跨会话记忆)                              │
│  - 自定义插件：主动任务调度、文档记忆增强                          │
└─────────────────────────────────────────────────────────────────┘
```

## 2. 记忆实现（越用越聪明）

### 2.1 使用 opencode-supermemory

- 安装：`bunx opencode-supermemory@latest install`
- 配置 `SUPERMEMORY_API_KEY`（环境变量或 `~/.config/opencode/supermemory.jsonc`）
- 提供跨会话、跨项目的持久记忆

### 2.2 文档场景增强

- **`.github/instructions/memory.instruction.md`**：OpenCode 内置，存用户偏好、常用术语
- **`.opencode/docs-memory/`**：自定义目录，存文档摘要、常见问题、检索模式

通过 `instructions` 加载：

```json
{
  "instructions": [
    ".github/instructions/memory.instruction.md",
    ".opencode/docs-memory/*.md"
  ]
}
```

### 2.3 可选：语义检索插件

仿 OpenClaw 的 `memory_search`，编写 OpenCode 插件：

- 用 embedding（OpenAI / Voyage / 本地）对文档和记忆做向量化
- 提供工具：`memory_search`、`memory_store`
- 在 agent 回答前根据用户问题检索相关记忆并注入 context

## 3. 主动干活实现

### 3.1 使用 opencode-scheduler

生态中的 [opencode-scheduler](https://github.com/different-ai/opencode-scheduler)：

- 使用 launchd（Mac）或 systemd（Linux）+ cron 语法
- 定时执行 OpenCode 任务（每日摘要、定期检查、提醒等）

### 3.2 自建调度服务

单独起一个调度服务，定期调用 OpenCode：

```bash
# 示例：每小时执行一次
0 * * * * cd /path/to/docs && opencode -c "检查今日待办事项，如有重要事项通过配置的渠道通知用户"
```

或通过 OpenCode SDK 以编程方式启动 session，将结果交给网关，由网关推送到 Telegram/WhatsApp。

### 3.3 主动推送流程

1. 调度器触发 → 调用 OpenCode 执行任务
2. OpenCode 输出结果 → 网关接收
3. 网关根据用户配置，将结果发到对应聊天渠道

## 4. 外部聊天工具集成

OpenCode 无内置 channel，需自建网关层。

### 4.1 网关职责

- 接收各平台消息（Telegram Bot API、WhatsApp Business API、Discord 等）
- 维护「平台用户 ↔ OpenCode 会话」映射
- 调用 OpenCode 执行任务，获取回复
- 将回复发回对应平台

### 4.2 调用 OpenCode 的方式

| 方式 | 说明 |
|------|------|
| CLI | `opencode -c "用户问题"` |
| SDK/API | 使用 @opencode-ai/sdk 或 OpenCode API 以编程方式创建 session、发送消息、获取回复 |
| 参考 kimaki | [kimaki](https://github.com/remorses/kimaki) 是 Discord bot，通过 SDK 控制 OpenCode，可参考其架构 |

### 4.3 实现顺序

1. 选一个平台（如 Telegram）做最小闭环
2. 实现：Telegram 消息 → 网关 → OpenCode → 回复回 Telegram
3. 抽象 channel 适配器，扩展到 WhatsApp、Discord 等

## 5. 实施路线图

| 阶段 | 内容 | 产出 |
|------|------|------|
| 1 | 记忆 | 安装 opencode-supermemory，配置文档记忆目录和 instructions | 跨会话记忆、文档偏好 |
| 2 | 单渠道 | 实现 Telegram bot + 网关，对接 OpenCode | 用户可在 Telegram 与文档助手对话 |
| 3 | 主动任务 | 用 opencode-scheduler 或自建 cron，定时执行 OpenCode 任务 | 定时摘要、提醒等 |
| 4 | 多渠道 | 抽象 channel 层，接入 WhatsApp、Discord 等 | 多平台统一入口 |
| 5 | 增强记忆 | 可选：语义检索插件、文档摘要自动写入记忆 | 更智能的文档理解与回答 |

## 6. 借鉴 OpenClaw 的设计

| 设计点 | OpenClaw 实现 | 迁移到 OpenCode 文档助手 |
|--------|---------------|--------------------------|
| Channel 插件化 | 每平台一插件，统一入站/出站 | 网关内每平台一适配器，统一接口 |
| SessionKey | `agent:channel:account:peer` | `platform:userId` 或 `platform:chatId` |
| Heartbeat | 定时唤醒 + SystemEvent 注入 | opencode-scheduler 定时执行 + 结果推送 |
| 记忆双路径 | 文件 + 向量检索 | supermemory + 可选自定义语义检索插件 |

## 7. 参考

- OpenCode 生态：https://github.com/anomalyco/opencode
- opencode-supermemory：https://github.com/supermemoryai/opencode-supermemory
- opencode-scheduler：https://github.com/different-ai/opencode-scheduler
- kimaki（Discord bot）：https://github.com/remorses/kimaki
- OpenClaw 架构分析：本目录 [channel-integration](./channel-integration.md)、[memory-design](./memory-design.md)、[proactive-execution](./proactive-execution.md)
