# 子agent错误处理指南

## 错误分类

### 1. 配置错误
**特征**：启动前发生的错误，通常与配置文件相关
**常见错误**：
- `agentId is not allowed for sessions_spawn`
- `Agent not found: [agent-name]`
- `Workspace path not accessible`

**解决方法**：
1. 检查`allowAgents`配置
2. 验证agent在`agents.list`中的定义
3. 检查工作区路径权限

### 2. 启动错误
**特征**：启动过程中发生的错误
**常见错误**：
- `Failed to load configuration files`
- `Missing required files (SOUL.md, USER.md)`
- `Permission denied when accessing workspace`

**解决方法**：
1. 检查配置文件完整性
2. 验证文件权限
3. 检查磁盘空间

### 3. 运行时错误
**特征**：任务执行过程中发生的错误
**常见错误**：
- `Task execution timeout`
- `Resource exhaustion (memory, CPU)`
- `External service unavailable`

**解决方法**：
1. 调整超时设置
2. 优化资源使用
3. 实现重试机制

### 4. 通信错误
**特征**：与主agent或外部服务通信失败
**常见错误**：
- `Connection lost with parent agent`
- `Message delivery failed`
- `Session channel closed unexpectedly`

**解决方法**：
1. 检查网络连接
2. 实现心跳机制
3. 建立重连逻辑

## 错误处理流程

### 1. 错误检测
```javascript
// 错误检测包装函数
async function safeSubagentCall(config) {
    try {
        const result = await sessions_spawn(config);
        return { success: true, data: result };
    } catch (error) {
        return { 
            success: false, 
            error: error.message,
            type: classifyError(error),
            timestamp: Date.now()
        };
    }
}

// 错误分类
function classifyError(error) {
    if (error.includes('not allowed')) return 'CONFIG_ERROR';
    if (error.includes('not found')) return 'CONFIG_ERROR';
    if (error.includes('timeout')) return 'RUNTIME_ERROR';
    if (error.includes('permission')) return 'PERMISSION_ERROR';
    return 'UNKNOWN_ERROR';
}
```

### 2. 错误响应策略

#### 配置错误响应
```javascript
function handleConfigError(error, agentId) {
    // 1. 记录错误
    logError('CONFIG_ERROR', { agentId, error });
    
    // 2. 提供修复建议
    const suggestions = [
        `检查 ${agentId} 是否在 allowAgents 列表中`,
        `验证 openclaw.json 配置文件`,
        `运行: agents_list() 查看允许的agent`
    ];
    
    // 3. 返回用户友好的错误信息
    return {
        userMessage: `无法启动 ${agentId}: 配置错误`,
        technicalDetails: error,
        suggestions: suggestions,
        canRetry: false  // 配置错误需要手动修复
    };
}
```

#### 运行时错误响应
```javascript
function handleRuntimeError(error, sessionKey, task) {
    // 1. 记录错误
    logError('RUNTIME_ERROR', { sessionKey, task, error });
    
    // 2. 决定重试策略
    const retryStrategy = {
        maxRetries: 3,
        retryDelay: 1000, // 1秒
        backoffFactor: 2
    };
    
    // 3. 检查是否可重试
    const canRetry = isRetryableError(error);
    
    return {
        userMessage: `任务执行失败: ${task.substring(0, 50)}...`,
        technicalDetails: error,
        canRetry: canRetry,
        retryStrategy: canRetry ? retryStrategy : null,
        fallbackAction: canRetry ? 'retry' : 'abort'
    };
}
```

### 3. 错误恢复机制

#### 自动恢复
```javascript
async function autoRecover(error, context) {
    const errorType = classifyError(error);
    
    switch (errorType) {
        case 'CONFIG_ERROR':
            // 配置错误需要人工干预
            return { recovered: false, action: 'manual_fix' };
            
        case 'RUNTIME_ERROR':
            // 运行时错误尝试重试
            return await retryWithBackoff(context);
            
        case 'PERMISSION_ERROR':
            // 权限错误检查并修复
            return await fixPermissions(context);
            
        default:
            // 未知错误记录并终止
            logError('UNKNOWN_ERROR', { error, context });
            return { recovered: false, action: 'terminate' };
    }
}

// 重试机制
async function retryWithBackoff(context, maxRetries = 3) {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
            // 等待指数退避时间
            const delay = Math.pow(2, attempt - 1) * 1000;
            await sleep(delay);
            
            // 重试操作
            const result = await retryOperation(context);
            return { recovered: true, attempt: attempt, result: result };
            
        } catch (retryError) {
            logError(`RETRY_ATTEMPT_${attempt}_FAILED`, { 
                context, 
                error: retryError 
            });
            
            if (attempt === maxRetries) {
                return { 
                    recovered: false, 
                    attempt: attempt, 
                    error: retryError 
                };
            }
        }
    }
}
```

### 4. 错误报告

#### 错误报告格式
```javascript
const errorReport = {
    // 基本信息
    timestamp: Date.now(),
    sessionId: 'agent:inkstone:subagent:session-id',
    agentId: 'inkstone',
    
    // 错误详情
    errorType: 'RUNTIME_ERROR',
    errorMessage: 'Task execution timeout after 300 seconds',
    stackTrace: error.stack,
    
    // 上下文信息
    taskDescription: '搜索笔记中关于时间管理的内容',
    taskDuration: 305, // 秒
    resourceUsage: {
        memory: '85%',
        cpu: '70%',
        disk: '45%'
    },
    
    // 处理信息
    recoveryAttempted: true,
    recoverySuccess: false,
    retryCount: 3,
    
    // 建议措施
    recommendations: [
        '增加任务超时时间',
        '优化搜索算法减少资源使用',
        '考虑分批处理大型搜索'
    ]
};
```

## 常见错误解决方案

### 错误1: `agentId is not allowed for sessions_spawn`
**原因**：子agent不在主agent的`allowAgents`列表中
**解决**：
1. 检查当前允许的agent：`agents_list()`
2. 更新主agent配置：
```javascript
gateway(
    action="config.patch",
    raw='{"agents":{"list":[{"id":"main","subagents":{"allowAgents":["inkstone"]}}]}}',
    note="添加inkstone到允许列表"
)
```

### 错误2: `Agent not found: inkstone`
**原因**：子agent未在`agents.list`中定义
**解决**：
1. 检查agent配置：`openclaw config get agents.list`
2. 添加子agent配置：
```javascript
gateway(
    action="config.patch",
    raw='{"agents":{"list":[{"id":"inkstone","name":"inkstone","workspace":"/path/to/workspace"}]}}',
    note="添加inkstone配置"
)
```

### 错误3: `Workspace path not accessible`
**原因**：工作区路径不存在或权限不足
**解决**：
1. 检查路径是否存在：`ls -la /path/to/workspace`
2. 修复权限：`chmod 755 /path/to/workspace`
3. 创建缺失的目录：`mkdir -p /path/to/workspace`

### 错误4: `Task execution timeout`
**原因**：任务执行时间超过`runTimeoutSeconds`
**解决**：
1. 增加超时时间：
```javascript
sessions_spawn(
    runtime="subagent",
    agentId="inkstone",
    task="任务描述",
    runTimeoutSeconds=600  // 增加到10分钟
)
```
2. 优化任务复杂度
3. 实现任务分片处理

### 错误5: `Connection lost with parent agent`
**原因**：网络问题或服务重启
**解决**：
1. 实现会话恢复机制
2. 添加心跳检测
3. 建立重连逻辑

## 预防措施

### 1. 配置验证
```javascript
// 启动前验证配置
async function validateSubagentConfig(agentId) {
    const checks = [
        checkAgentInAllowList(agentId),
        checkAgentConfigExists(agentId),
        checkWorkspaceAccessible(agentId),
        checkRequiredFilesExist(agentId)
    ];
    
    const results = await Promise.all(checks);
    const failures = results.filter(r => !r.passed);
    
    if (failures.length > 0) {
        throw new Error(`配置验证失败: ${failures.map(f => f.reason).join(', ')}`);
    }
    
    return true;
}
```

### 2. 资源监控
```javascript
// 资源使用监控
function monitorResources(sessionKey) {
    setInterval(async () => {
        const usage = await getResourceUsage(sessionKey);
        
        if (usage.memory > 0.9) {
            warnHighMemoryUsage(sessionKey, usage);
        }
        
        if (usage.cpu > 0.8) {
            warnHighCpuUsage(sessionKey, usage);
        }
    }, 30000); // 每30秒检查一次
}
```

### 3. 健康检查
```javascript
// 定期健康检查
async function healthCheck(sessionKey) {
    try {
        // 发送心跳请求
        const response = await sessions_send(
            sessionKey,
            "__HEARTBEAT__"
        );
        
        return {
            healthy: true,
            responseTime: response.duration,
            timestamp: Date.now()
        };
        
    } catch (error) {
        return {
            healthy: false,
            error: error.message,
            timestamp: Date.now()
        };
    }
}
```

## 错误日志管理

### 日志格式
```
[时间戳] [错误级别] [会话ID] [错误类型] [错误消息]
[上下文信息]
[堆栈跟踪]
[处理建议]
```

### 日志轮转
- 按日期分割日志文件
- 限制单个日志文件大小
- 自动清理旧日志

### 日志分析
- 统计错误频率和类型
- 识别常见错误模式
- 生成错误报告

## 最佳实践

### 1. 防御性编程
- 验证所有输入参数
- 检查资源可用性
- 实现边界条件处理

### 2. 优雅降级
- 主功能失败时提供备用方案
- 资源不足时降低服务质量
- 外部服务不可用时使用缓存

### 3. 快速失败
- 不可恢复的错误立即终止
- 避免无限重试循环
- 设置合理的重试限制

### 4. 透明错误
- 向用户提供清晰的错误信息
- 记录详细的错误上下文
- 提供具体的解决步骤

### 5. 持续改进
- 定期分析错误日志
- 根据错误模式优化代码
- 更新错误处理策略