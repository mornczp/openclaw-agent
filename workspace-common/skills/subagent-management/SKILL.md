---
name: subagent-management
description: 管理OpenClaw子agent的启动和基本管理。触发条件：当用户提到"子agent"、"subagent"、"启动agent"、"agent会话"时使用此技能。
---

# Subagent Management Skill

## 概述

此技能提供OpenClaw子agent的基本启动和管理功能。

## 快速开始

### 1. 检查可用子agent
```javascript
agents_list()
```

### 2. 启动子agent会话
```javascript
// 基础启动方式（当前唯一可用）
sessions_spawn(
    runtime="subagent",
    agentId="inkstone",      // 或 "searcher"
    task="处理任务描述",
    label="会话标签",
    mode="run",
    runTimeoutSeconds=300
)
```

### 3. 管理运行中的子agent
```javascript
// 查看运行状态
subagents(action="list")

// 向子agent发送消息（如果知道会话key）
sessions_send(
    sessionKey="agent:inkstone:subagent:session-id",
    message="继续处理..."
)

// 终止子agent
subagents(action="kill", target="session-key")
```

## 详细指南

### 配置管理
- **基础配置**：参见 [references/configuration.md](references/configuration.md)

### 会话管理
- **会话类型**：参见 [references/session-types.md](references/session-types.md)

### 错误处理
- **常见错误**：参见 [references/error-handling.md](references/error-handling.md)

## 使用流程

1. **检查可用性**：使用 `agents_list()` 查看可用的子agent
2. **启动会话**：使用 `sessions_spawn()` 启动子agent处理任务
3. **监控状态**：使用 `subagents(action="list")` 查看运行状态

## 注意事项

1. **超时设置**：根据任务复杂度合理设置 `runTimeoutSeconds`
2. **会话管理**：记录会话key以便后续发送消息

## 更新记录

- **2026-03-17**：清理多余内容，只保留实际可用功能
