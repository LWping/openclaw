---
summary: "Agent 与工具：执行模型、Skills、子 Agent、审批"
title: "Agent and Tools"
---

# OpenClaw Agent 与工具设计

本文档分析 OpenClaw 的 Agent 执行模型、工具系统、Skills 等，便于借鉴其设计思想。

## 1. Agent 执行模型

### 1.1 核心入口

- **runEmbeddedPiAgent**：基于 Pi 的嵌入式 agent（主要路径）
- **runCliAgent**：CLI 模式，通过子进程调用 `openclaw agent`
- **runReplyAgent**：auto-reply 场景，单轮对话执行

### 1.2 配置

```yaml
agents:
  defaults:
    id: default
    model: claude-3-5-sonnet
    provider: anthropic
    heartbeat: { ... }
    skills: { filter: [...], allowRemote: true }
  list:
    - id: coding
      model: claude-3-5-sonnet
      skills: { filter: ["exec", "web", "apply-patch"] }
```

### 1.3 作用域

- `resolveAgentConfig(cfg, agentId)`：按 id 解析配置
- `resolveAgentWorkspaceDir`：每个 agent 独立 workspace
- `resolveAgentDir`：agent 专属目录（skills、hooks 等）

## 2. 工具系统

### 2.1 工具来源

| 来源       | 说明                         |
|------------|------------------------------|
| openclaw-tools | 核心工具集（exec、web、memory、sessions 等） |
| pi-tools   | Pi 内置工具（coding、browser 等）      |
| channel-tools | 按 channel 注入（telegram_*、discord_* 等） |
| skills     | 插件/扩展提供的工具                  |
| cron-tool  | 定时任务管理                        |

### 2.2 工具注册与过滤

- `resolveAgentSkillsFilter`：按 agent 配置过滤可用 skills
- `buildWorkspaceSkillSnapshot`：构建当前 workspace 的 skill 列表
- 工具通过 `AnyAgentTool` 接口：`name`、`description`、`parameters`、`execute`

### 2.3 典型工具

- **memory_search** / **memory_get**：记忆检索
- **cron**：Cron 任务增删改查、立即执行
- **sessions_send**：向指定 session 发送消息
- **sessions_spawn**：创建子 agent
- **exec**：执行 shell 命令（支持审批、沙箱）
- **web_fetch** / **web_search**：网页抓取与搜索

## 3. Skills

### 3.1 概念

- Skill = 一组工具 + 可选配置
- 位于 `skills/` 或通过插件注册
- 可配置 `filter` 白名单，控制 agent 可用范围

### 3.2 内置 Skills

- exec、web、apply-patch、browser、firecrawl 等
- 每个 skill 有 `tools` 数组和 `config` schema

### 3.3 远程 Skill

- `getRemoteSkillEligibility`：判断是否允许调用远程 skill
- 支持 ClawHub 等远程 skill 仓库

## 4. 子 Agent (Subagent)

### 4.1 机制

- `sessions_spawn` 工具：创建子 agent 会话
- `spawnedBy`、`spawnDepth` 记录层级
- `countActiveDescendantRuns`、`listDescendantRunsForRequester` 管理生命周期

### 4.2 宣布流程

- `runSubagentAnnounceFlow`：子 agent 完成后向父 session 汇报
- 通过 `sessions_send` 或系统事件注入父会话

## 5. 审批与沙箱

### 5.1 Exec 审批

- `execAsk`：always | never | dangerous
- 危险命令需用户确认后才执行
- 通过 channel 的 reaction 或按钮交互

### 5.2 沙箱

- Docker 沙箱：隔离执行环境
- `execHost`、`execSecurity` 配置
- `sandbox-create-args` 构建容器参数

## 6. 借鉴要点

1. **工具即能力**：Agent 能力由工具集合决定，通过 filter 精细控制
2. **Skill 抽象**：工具按 skill 分组，便于复用与权限管理
3. **子 Agent**：复杂任务拆分子 agent，结果回传父会话
4. **审批流程**：危险操作需显式确认，降低误操作风险
5. **多来源工具**：核心 + channel + skills 组合，扩展性强
