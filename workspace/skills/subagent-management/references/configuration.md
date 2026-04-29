# 子agent配置指南

## 配置文件位置
```
/home/czp/.openclaw/openclaw.json
```

## 配置步骤

### 1. 前提条件
- 子agent工作区必须已创建并配置完整
- 子agent必须在`agents.list`中定义
- 工作区路径和agentDir配置正确

### 2. 主agent配置示例
```json
{
  "id": "main",
  "name": "main",
  "workspace": "/home/czp/.openclaw/workspace",
  "agentDir": "/home/czp/.openclaw/agents/main",
  "subagents": {
    "allowAgents": ["inkstone", "technical-agent", "customer-service-agent"]
  },
  "tools": {
    "profile": "full",
    "deny": ["feishu_app_scopes"]
  }
}
```

### 3. 子agent配置示例（inkstone）
```json
{
  "id": "inkstone",
  "name": "inkstone",
  "workspace": "/home/czp/.openclaw/workspace-inkstone",
  "agentDir": "/home/czp/.openclaw/agents/inkstone/agent"
}
```

### 4. 配置验证命令

#### 检查当前允许的子agent
```javascript
agents_list()
```

#### 检查配置文件
```bash
openclaw config get agents.list
```

### 5. 应用配置方法

#### 方法一：使用config.patch（推荐）
```javascript
gateway(
  action="config.patch",
  raw='{
    "agents": {
      "list": [
        {
          "id": "main",
          "name": "main",
          "workspace": "/home/czp/.openclaw/workspace",
          "agentDir": "/home/czp/.openclaw/agents/main",
          "subagents": {
            "allowAgents": ["inkstone"]
          },
          "tools": {
            "profile": "full",
            "deny": ["feishu_app_scopes"]
          }
        }
      ]
    }
  }',
  note="配置inkstone为允许的子agent"
)
```

#### 方法二：重启OpenClaw
```bash
openclaw gateway restart
```

## 完整配置示例

### 主agent完整配置
```json
{
  "id": "main",
  "name": "main",
  "workspace": "/home/czp/.openclaw/workspace",
  "agentDir": "/home/czp/.openclaw/agents/main",
  "subagents": {
    "allowAgents": ["inkstone", "technical-agent", "customer-service-agent"],
    "maxConcurrent": 3,
    "runTimeoutSeconds": 600
  },
  "tools": {
    "profile": "full",
    "deny": ["feishu_app_scopes"]
  },
  "model": "deepseek/deepseek-chat"
}
```

### 路由决策逻辑（参考）
```javascript
// 根据消息内容选择子agent
function routeToSubagent(message) {
  if (message.includes("笔记") || message.includes("Obsidian")) {
    return "inkstone";
  } else if (message.includes("bug") || message.includes("技术")) {
    return "technical-agent";
  } else if (message.includes("帮助") || message.includes("客服")) {
    return "customer-service-agent";
  } else {
    return null; // 自行处理或使用通用agent
  }
}
```

## 配置字段说明

### 主agent配置字段
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | agent唯一标识 |
| `name` | string | 是 | agent名称 |
| `workspace` | string | 是 | 工作区路径 |
| `agentDir` | string | 是 | agent目录路径 |
| `subagents.allowAgents` | array | 是 | 允许的子agent列表 |
| `subagents.maxConcurrent` | number | 否 | 最大并发数 |
| `subagents.runTimeoutSeconds` | number | 否 | 运行超时时间 |
| `tools.profile` | string | 否 | 工具权限配置 |
| `tools.deny` | array | 否 | 禁止的工具列表 |
| `model` | string | 否 | 默认模型 |

### 子agent配置字段
| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `id` | string | 是 | 子agent唯一标识 |
| `name` | string | 是 | 子agent名称 |
| `workspace` | string | 是 | 子agent工作区路径 |
| `agentDir` | string | 是 | 子agent目录路径 |

## 配置最佳实践

### 1. 命名规范
- 使用小写字母和连字符
- 名称应描述agent的功能
- 保持一致性

### 2. 路径管理
- 工作区路径应绝对路径
- 确保路径存在且有正确权限
- 定期清理临时文件

### 3. 权限控制
- 遵循最小权限原则
- 明确禁止不需要的工具
- 定期审查权限配置

### 4. 性能优化
- 限制并发数量避免资源耗尽
- 设置合理的超时时间
- 监控资源使用情况

## 常见问题

### Q1: 配置后子agent仍然不可用
- 检查配置文件语法
- 验证工作区路径权限
- 重启OpenClaw服务

### Q2: 如何添加新的子agent
1. 创建子agent工作区
2. 在`agents.list`中添加子agent配置
3. 在主agent的`allowAgents`中添加子agent ID
4. 应用配置并重启

### Q3: 配置变更后需要重启吗？
- 使用`config.patch`通常不需要重启
- 某些配置变更可能需要重启生效
- 重启是最安全的确保变更生效的方式