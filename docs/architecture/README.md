---
summary: "OpenClaw 架构设计分析索引"
title: "Architecture Design Analysis"
---

# OpenClaw 架构设计分析

本目录包含对 OpenClaw 项目关键设计的分析文档，便于借鉴其思想开发类似项目。

## 文档索引

| 文档 | 主题 |
|------|------|
| [channel-integration](./channel-integration.md) | 外部聊天工具对接：插件化、Monitor、Outbound、Bindings 路由 |
| [memory-design](./memory-design.md) | 记忆实现：双后端、混合检索、Session Memory Hook |
| [proactive-execution](./proactive-execution.md) | 主动干活：Heartbeat、Cron Jobs、SystemEvent |
| [session-and-routing](./session-and-routing.md) | 会话与路由：SessionKey、Bindings、Session Store |
| [agent-and-tools](./agent-and-tools.md) | Agent 与工具：执行模型、Skills、子 Agent、审批 |
| [hooks-and-extensibility](./hooks-and-extensibility.md) | Hooks 与扩展：Hook 机制、插件系统、配置扩展 |
| [gateway-core](./gateway-core.md) | Gateway 核心：进程模型、WebSocket、启动流程 |

## 核心设计思想摘要

1. **Channel 插件化**：每个聊天渠道独立 extension，通过 registry 注册，Dock 与 Plugin 分离
2. **记忆双后端**：builtin（SQLite+FTS+向量）与 qmd 可选，混合检索，Session Memory Hook 沉淀对话
3. **主动执行**：Heartbeat 定时唤醒 + SystemEvent 注入，Cron 支持 main（注入）与 isolated（执行+投递）
4. **多维路由**：Bindings 支持 channel/account/peer/guild/roles，SessionKey 贯穿会话
5. **工具即能力**：Agent 能力由工具集合决定，Skill 分组，支持子 Agent 与审批
6. **Hooks + 插件**：事件驱动扩展，插件即包，Registry 集中管理

## 参考

- 仓库：https://github.com/openclaw/openclaw
- 文档：https://docs.openclaw.ai/
