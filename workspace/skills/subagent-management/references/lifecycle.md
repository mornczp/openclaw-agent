# 子agent生命周期管理

## 生命周期阶段

### 1. 配置阶段
**时机**：系统初始化或添加新子agent时
**操作**：
- 在OpenClaw配置文件中定义子agent
- 配置主agent的`allowAgents`列表
- 创建子agent工作区和配置文件

**验证**：
```javascript
agents_list()  // 检查子agent是否在允许列表中
```

### 2. 启动阶段
**时机**：主agent调用`sessions_spawn`时
**操作**：
- 加载子agent配置文件（SOUL.md, USER.md等）
- 初始化工作区环境
- 建立会话上下文

**启动方式**：
```javascript
// 一次性任务启动
sessions_spawn(
    runtime="subagent",
    agentId="inkstone",
    task="初始任务",
    mode="run"
)

// 持久化会话启动
sessions_spawn(
    runtime="subagent",
    agentId="inkstone",
    task="初始任务",
    mode="session",
    thread=true
)
```

### 3. 运行阶段
**时机**：会话启动后到超时或任务完成前
**操作**：
- 处理主agent分配的任务
- 维护会话上下文
- 响应主agent的交互请求

**运行状态**：
- **活跃**：正在处理任务
- **空闲**：等待新任务
- **错误**：遇到处理错误

### 4. 监控阶段
**时机**：整个运行期间
**操作**：
- 监控资源使用情况
- 跟踪任务进度
- 记录性能指标

**监控命令**：
```javascript
// 查看运行状态
subagents(action="list")

// 查看会话详情
sessions_list(kinds=["subagent"])
```

### 5. 终止阶段
**时机**：
- 任务完成（一次性模式）
- 超时触发
- 手动终止
- 系统错误

**终止方式**：
```javascript
// 自动终止（任务完成）
// 一次性模式任务完成后自动销毁

// 手动终止
subagents(action="kill", target="session-key")

// 超时终止
// 达到runTimeoutSeconds或timeoutSeconds时自动终止
```

### 6. 清理阶段
**时机**：会话终止后
**操作**：
- 释放占用的资源
- 清理临时文件
- 保存会话日志

## 生命周期时间线

```
配置阶段 → 启动阶段 → 运行阶段 → 终止阶段 → 清理阶段
    ↑          ↑          ↑          ↑          ↑
  初始化     会话创建    任务处理    结束条件    资源回收
```

## 关键时间点

### 创建时间
- **配置时间**：子agent添加到配置文件的时间
- **启动时间**：`sessions_spawn`调用的时间
- **就绪时间**：子agent完成初始化准备接收任务的时间

### 运行时间
- **开始时间**：任务开始处理的时间
- **活动时间**：实际处理任务的时间
- **空闲时间**：等待任务的时间

### 终止时间
- **完成时间**：任务完成的时间
- **超时时间**：达到超时限制的时间
- **终止时间**：会话实际终止的时间

## 状态转换

### 正常流程
```
未配置 → 已配置 → 已启动 → 运行中 → 已完成 → 已清理
```

### 异常流程
```
运行中 → 错误状态 → 恢复尝试 → [成功:运行中] / [失败:已终止]
```

### 超时流程
```
运行中 → 超时警告 → 强制终止 → 已清理
```

## 资源管理

### 内存管理
- **启动时**：加载配置文件和工作区
- **运行时**：维护会话上下文和任务状态
- **终止时**：释放所有内存资源

### 文件管理
- **工作区文件**：子agent的配置和临时文件
- **会话日志**：记录会话活动和错误
- **备份文件**：重要操作的备份

### 网络资源
- **API连接**：如果需要外部服务
- **会话通道**：与主agent的通信通道
- **监控连接**：性能监控和数据收集

## 错误处理生命周期

### 1. 错误检测
- **运行时错误**：任务执行过程中的错误
- **配置错误**：配置文件或权限问题
- **资源错误**：内存、磁盘、网络问题

### 2. 错误响应
```javascript
// 错误处理策略
function handleSubagentError(error, sessionKey) {
    // 1. 记录错误日志
    // 2. 尝试恢复（如重试）
    // 3. 通知主agent
    // 4. 决定是否终止会话
}
```

### 3. 恢复机制
- **自动重试**：可恢复错误的自动重试
- **状态回滚**：恢复到错误前的状态
- **会话重建**：终止并重建会话

### 4. 错误报告
- **错误类型**：分类记录错误
- **错误上下文**：记录错误发生时的状态
- **解决方案**：记录解决方法和经验

## 监控指标

### 性能指标
| 指标 | 说明 | 监控频率 |
|------|------|----------|
| **启动时间** | 从调用到就绪的时间 | 每次启动 |
| **任务耗时** | 单个任务处理时间 | 每次任务 |
| **资源使用** | CPU、内存、磁盘使用 | 定期（如每分钟） |
| **错误率** | 错误任务比例 | 每次任务 |

### 健康指标
| 指标 | 健康标准 | 检查频率 |
|------|----------|----------|
| **响应时间** | < 5秒 | 每次交互 |
| **可用性** | > 95% | 定期检查 |
| **会话状态** | 活跃/正常 | 持续监控 |
| **资源占用** | 在限制范围内 | 定期检查 |

## 最佳实践

### 1. 生命周期规划
- **预估资源需求**：根据任务复杂度规划资源
- **设置合理超时**：避免资源泄漏
- **实现优雅终止**：保存状态，清理资源

### 2. 状态管理
- **明确状态转换**：定义清晰的状态机
- **状态持久化**：重要状态定期保存
- **状态恢复**：支持从保存状态恢复

### 3. 错误处理
- **预防性检查**：启动前检查配置和资源
- **快速失败**：无法恢复的错误快速终止
- **错误隔离**：防止错误扩散到其他组件

### 4. 监控优化
- **关键指标监控**：关注核心性能指标
- **预警机制**：提前发现潜在问题
- **性能分析**：定期分析优化机会

## 示例：完整生命周期管理

```javascript
// 1. 配置检查
const allowedAgents = agents_list();
if (!allowedAgents.includes("inkstone")) {
    // 配置子agent
    await configureSubagent("inkstone");
}

// 2. 启动会话
const session = sessions_spawn(
    runtime="subagent",
    agentId="inkstone",
    task="处理笔记任务",
    label="inkstone-session",
    mode="run",
    runTimeoutSeconds=600
);

// 3. 监控运行
setInterval(() => {
    const status = subagents(action="list");
    monitorSessionHealth(status);
}, 60000); // 每分钟检查一次

// 4. 错误处理
try {
    // 执行任务
    await executeTask(session);
} catch (error) {
    // 记录错误
    logError(error, session);
    
    // 尝试恢复
    if (isRecoverable(error)) {
        await recoverSession(session);
    } else {
        // 终止会话
        subagents(action="kill", target=session.key);
    }
}

// 5. 清理
function cleanupSession(session) {
    // 保存日志
    saveSessionLog(session);
    
    // 清理临时文件
    cleanupTempFiles(session);
    
    // 更新监控数据
    updateMonitoringStats(session);
}
```