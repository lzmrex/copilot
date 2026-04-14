

---

# 📊 ToolInterceptor 详解

## 🎯 核心概念

`ToolInterceptor` 是一个**工具调用拦截器抽象类**，用于在 Agent 执行工具（Tool）**前后**插入自定义逻辑。它与 `ModelInterceptor` 对应，但作用于**工具调用层面**而非 LLM 调用层面。

---

## ⏰ 什么时候被调用？

### **调用时机：每次 Agent 执行工具时**

```
用户: agent.call("帮我查北京天气")
  ↓
┌──────────────────────────────────────┐
│ BEFORE_AGENT Hooks                   │
└──────────────┬───────────────────────┘
               ↓
      ╔═══════════════════════╗
      ║ Round 1: ReAct 循环   ║
      ╚════════╤══════════════╝
               ↓
┌──────────────────────────────────────┐
│ BEFORE_MODEL Hooks                   │
└──────────────┬───────────────────────┘
               ↓
╔════════════════════════════════════════╗
║ AGENT_MODEL_NODE (LLM 推理)            ║
║ ModelInterceptor 在这里调用 ⭐          ║
║                                        ║
║ LLM 返回: get_weather("Beijing")       ║
╚════════════════════════════════════════╝
               ↓
┌──────────────────────────────────────┐
│ AFTER_MODEL Hooks                    │
└──────────────┬───────────────────────┘
               ↓
          需要工具？✅ YES
               ↓
╔════════════════════════════════════════╗
║ ⭐⭐⭐ AGENT_TOOL_NODE                 ║
║   ┌────────────────────────────────┐  ║
║   │ 遍历所有工具调用                │  ║
║   │ for each toolCall:             │  ║
║   │   ┌────────────────────────┐   │  ║
║   │   │ 1. 创建 ToolCallRequest│   │  ║
║   │   │ 2. 创建拦截器链        │   │  ║
║   │   │ 3. 执行拦截器链 ⭐⭐⭐│   │  ║
║   │   │    interceptor1        │   │  ║
║   │   │      ↓                 │   │  ║
║   │   │    interceptor2        │   │  ║
║   │   │      ↓                 │   │  ║
║   │   │    baseHandler         │   │  ║
║   │   │      ↓                 │   │  ║
║   │   │    实际执行工具         │   │  ║
║   │   │ 4. 返回 ToolCallResponse│  │  ║
║   │   └────────────────────────┘   │  ║
║   └────────────────────────────────┘  ║
╚════════════════════════════════════════╝
               ↓
┌──────────────────────────────────────┐
│ 将工具结果添加到消息列表              │
└──────────────┬───────────────────────┘
               ↓
        (回到 Round 2)
```


**关键点：**
- ✅ **每次工具调用都执行**：如果 LLM 一次性调用 3 个工具，拦截器会被调用 3 次
- ✅ **在 AgentToolNode 内部**：具体在 `executeToolCallWithInterceptors()` 方法中
- ✅ **在实际执行工具代码之前**：可以修改参数、重试、缓存、降级等

---

## 🔍 调用位置代码详解

### **位置：AgentToolNode.java 第 509-600+ 行**

```java
private ToolCallResponse executeToolCallWithInterceptors(
        AssistantMessage.ToolCall toolCall, 
        OverAllState state,
        RunnableConfig config, 
        Map<String, Object> extraStateFromToolCall, 
        boolean inParallelExecution,
        Map<Integer, DefaultCancellationToken> cancellationTokens, 
        int toolIndex) {

    // 1. 创建 ToolCallRequest
    ToolCallRequest request = ToolCallRequest.builder()
            .toolCall(toolCall)  // 从 LLM 返回的工具调用
            .context(config.metadata().orElse(new HashMap<>()))
            .executionContext(new ToolCallExecutionContext(config, state))
            .build();

    // 2. ⭐ 创建基础处理器（实际执行工具）
    ToolCallHandler baseHandler = req -> {
        ToolCallback toolCallback = resolve(req.getToolName(), config);
        
        if (toolCallback == null) {
            throw new IllegalStateException("Tool not found: " + req.getToolName());
        }
        
        // 实际调用工具
        ToolExecutionResult result = toolCallback.call(req.getArguments());
        
        return ToolCallResponse.of(
            req.getToolCallId(),
            req.getToolName(),
            result.getResult()
        );
    };

    // 3. ⭐⭐⭐ 链式包装所有拦截器
    ToolCallHandler chainedHandler = InterceptorChain.chainToolInterceptors(
            toolInterceptors,  // 拦截器列表
            baseHandler        // 基础处理器
    );

    // 4. ⭐⭐⭐ 执行拦截器链（从这里开始调用 interceptToolCall）
    ToolCallResponse response = chainedHandler.call(request);
    
    return response;
}
```


---

## 🔗 拦截器链的工作原理

与 `ModelInterceptor` 完全相同，使用责任链模式：

```java
// InterceptorChain.chainToolInterceptors()
public static ToolCallHandler chainToolInterceptors(
        List<ToolInterceptor> interceptors,
        ToolCallHandler baseHandler) {
    
    if (interceptors == null || interceptors.isEmpty()) {
        return baseHandler;
    }
    
    ModelCallHandler current = baseHandler;
    
    // 从后往前包装
    for (int i = interceptors.size() - 1; i >= 0; i--) {
        ToolInterceptor interceptor = interceptors.get(i);
        ToolCallHandler nextHandler = current;
        
        current = request -> interceptor.interceptToolCall(request, nextHandler);
    }
    
    return current;
}
```


**调用顺序示例：**

假设有拦截器 `[LoggingInterceptor, RetryInterceptor, CacheInterceptor]`：

```
request → LoggingInterceptor.interceptToolCall()
            ↓ (记录日志)
            ↓ (调用 handler.call())
          RetryInterceptor.interceptToolCall()
            ↓ (尝试执行)
            ↓ (调用 handler.call())
          CacheInterceptor.interceptToolCall()
            ↓ (检查缓存)
            ↓ (调用 handler.call())
          baseHandler (实际执行工具)
            ↓
          工具执行结果
            ↑
          CacheInterceptor 处理响应（可能缓存）
            ↑
          RetryInterceptor 处理响应（失败则重试）
            ↑
          LoggingInterceptor 处理响应（记录结果）
            ↑
          返回最终响应
```


---

## 💡 使用场景详解

### **场景 1：工具重试（ToolRetryInterceptor）**

**用途：** 当工具执行失败时自动重试

```java
public class ToolRetryInterceptor extends ToolInterceptor {
    
    private final int maxRetries;
    private final double backoffFactor;
    private final long initialDelayMs;
    private final Predicate<Exception> retryOn;
    
    @Override
    public ToolCallResponse interceptToolCall(ToolCallRequest request, ToolCallHandler handler) {
        Exception lastException = null;
        long delayMs = initialDelayMs;
        
        for (int attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                if (attempt > 0) {
                    log.info("Retrying tool '{}' (attempt {}/{})", 
                            request.getToolName(), attempt, maxRetries);
                    
                    // 指数退避等待
                    Thread.sleep(delayMs);
                    delayMs *= backoffFactor;
                }
                
                // ⭐ 尝试执行工具
                ToolCallResponse response = handler.call(request);
                
                // 检查是否是错误响应
                if (response.isError()) {
                    throw new RuntimeException(response.getResult());
                }
                
                // 成功，返回响应
                return response;
                
            } catch (Exception e) {
                lastException = e;
                log.warn("Tool '{}' attempt {} failed: {}", 
                        request.getToolName(), attempt, e.getMessage());
                
                // 检查是否应该重试
                if (attempt < maxRetries && retryOn.test(e)) {
                    continue;  // 重试
                } else {
                    break;  // 不重试
                }
            }
        }
        
        // 所有重试都失败，返回错误响应
        return ToolCallResponse.error(
            request.getToolCallId(),
            request.getToolName(),
            lastException
        );
    }
}
```


**何时使用：**
- 调用外部 API 可能超时或失败
- 网络不稳定
- 需要提高工具执行的可靠性

**配置示例：**
```java
ToolRetryInterceptor retryInterceptor = ToolRetryInterceptor.builder()
    .maxRetries(3)              // 最多重试 3 次
    .backoffFactor(2.0)         // 指数退避倍数
    .initialDelay(1000)         // 初始延迟 1 秒
    .retryOn(e -> e instanceof TimeoutException)  // 只重试超时异常
    .build();

ReactAgent agent = ReactAgent.builder()
    .model(chatModel)
    .tools(weatherApiTool)
    .toolInterceptors(retryInterceptor)
    .build();
```


---

### **场景 2：工具模拟（ToolEmulatorInterceptor）**

**用途：** 用 LLM 模拟工具执行，用于测试或避免真实调用

```java
public class ToolEmulatorInterceptor extends ToolInterceptor {
    
    private final ChatModel emulatorModel;
    private final Set<String> toolsToEmulate;
    private final boolean emulateAll;
    
    @Override
    public ToolCallResponse interceptToolCall(ToolCallRequest request, ToolCallHandler handler) {
        String toolName = request.getToolName();
        
        // 检查是否应该模拟这个工具
        boolean shouldEmulate = emulateAll || toolsToEmulate.contains(toolName);
        
        if (!shouldEmulate) {
            // ⭐ 不模拟，正常执行
            return handler.call(request);
        }
        
        log.info("Emulating tool call: {}", toolName);
        
        try {
            // ⭐ 构建提示词，让 LLM 模拟工具执行
            String prompt = String.format(
                "Simulate the execution of a tool:\n" +
                "Tool Name: %s\n" +
                "Tool Description: %s\n" +
                "Arguments: %s\n\n" +
                "Please provide a realistic response as if this tool was executed.",
                toolName,
                "No description available",
                request.getArguments()
            );
            
            // 调用 LLM 模拟工具执行
            ChatResponse response = emulatorModel.call(
                new Prompt(new UserMessage(prompt))
            );
            
            String emulatedResult = response.getResult().getOutput().getText();
            
            log.debug("Emulated tool '{}' returned: {}", toolName, emulatedResult);
            
            // 返回模拟结果
            return ToolCallResponse.of(
                request.getToolCallId(),
                toolName,
                emulatedResult
            );
            
        } catch (Exception e) {
            log.error("Failed to emulate tool '{}': {}", toolName, e.getMessage());
            // 模拟失败，回退到真实执行
            return handler.call(request);
        }
    }
}
```


**何时使用：**
- 开发测试阶段，避免调用真实 API
- 演示环境，不需要真实数据
- 成本优化，某些工具用 LLM 模拟更便宜
- 离线环境，无法访问外部服务

**配置示例：**
```java
// 模拟所有工具
ToolEmulatorInterceptor emulator = ToolEmulatorInterceptor.builder()
    .model(gpt4oMini)  // 用小模型模拟，成本低
    .build();

// 只模拟特定工具
ToolEmulatorInterceptor selectiveEmulator = ToolEmulatorInterceptor.builder()
    .model(gpt4oMini)
    .addTool("expensive_api")     // 模拟昂贵的 API
    .addTool("slow_database")     // 模拟慢查询
    .emulateAll(false)            // 其他工具正常执行
    .build();
```


---

### **场景 3：大结果清理（LargeResultEvictionInterceptor）**

**用途：** 自动将大型工具结果保存到文件，避免上下文爆炸

```java
public class LargeResultEvictionInterceptor extends ToolInterceptor {
    
    private final int toolTokenLimit;
    private final Set<String> excludeTools;
    private final String evictionDir;
    
    @Override
    public ToolCallResponse interceptToolCall(ToolCallRequest request, ToolCallHandler handler) {
        // 1. 先执行工具
        ToolCallResponse response = handler.call(request);
        
        String toolName = request.getToolName();
        
        // 2. 检查是否应该排除这个工具
        if (excludeTools.contains(toolName)) {
            return response;  // 文件系统工具自己处理结果
        }
        
        // 3. 检查结果大小
        String result = response.getResult();
        int tokenCount = countTokens(result);
        
        if (tokenCount <= toolTokenLimit) {
            return response;  // 结果不大，直接返回
        }
        
        // 4. ⭐ 结果太大，保存到文件
        log.info("Tool '{}' result too large ({} tokens), evicting to file", 
                toolName, tokenCount);
        
        // 生成文件名
        String fileName = sanitizeFileName(request.getToolCallId()) + ".txt";
        Path filePath = Paths.get(evictionDir, fileName);
        
        try {
            // 保存完整结果到文件
            Files.writeString(filePath, result, StandardOpenOption.CREATE);
            
            // 5. 创建摘要（前 10 行）
            String sample = createSample(result, 10);
            
            // 6. 返回截断的响应，指向文件
            String truncatedResult = String.format(
                "[Result too large (%d tokens). Full result saved to: %s]\n\n" +
                "Sample (first 10 lines):\n%s",
                tokenCount,
                filePath.toAbsolutePath(),
                sample
            );
            
            return ToolCallResponse.builder()
                    .content(truncatedResult)
                    .toolName(toolName)
                    .toolCallId(request.getToolCallId())
                    .status("evicted")
                    .metadata(Map.of(
                        "originalTokenCount", tokenCount,
                        "evictionFile", filePath.toString(),
                        "evicted", true
                    ))
                    .build();
            
        } catch (IOException e) {
            log.error("Failed to evict large result: {}", e.getMessage());
            // 保存失败，返回原始结果
            return response;
        }
    }
    
    private String createSample(String content, int lineCount) {
        String[] lines = content.split("\n");
        int sampleLines = Math.min(lines.length, lineCount);
        
        StringBuilder sample = new StringBuilder();
        for (int i = 0; i < sampleLines; i++) {
            String line = lines[i];
            if (line.length() > 1000) {
                line = line.substring(0, 1000) + "...";
            }
            sample.append(String.format("%6d | %s\n", i + 1, line));
        }
        
        if (lines.length > lineCount) {
            sample.append(String.format("... (%d more lines)\n", lines.length - lineCount));
        }
        
        return sample.toString();
    }
}
```


**何时使用：**
- 工具可能返回大量数据（如搜索 API、数据库查询）
- 需要控制上下文窗口大小
- 避免超出 LLM 的 token 限制
- 节省 Token 成本

**效果：**
```
原始结果: 50000 tokens (完整 HTML 页面)
  ↓
清理后: 200 tokens
  ├─ 提示信息: "结果已保存到 /path/to/file.txt"
  └─ 摘要: 前 10 行内容
```


---

### **场景 4：工具错误处理（ToolErrorInterceptor）**

**用途：** 捕获工具执行异常，返回友好的错误信息

```java
public class ToolErrorInterceptor extends ToolInterceptor {
    
    @Override
    public ToolCallResponse interceptToolCall(ToolCallRequest request, ToolCallHandler handler) {
        try {
            // ⭐ 尝试执行工具
            return handler.call(request);
            
        } catch (Exception e) {
            log.error("Tool '{}' execution failed: {}", 
                    request.getToolName(), e.getMessage(), e);
            
            // ⭐ 返回友好的错误信息，而不是抛出异常
            return ToolCallResponse.error(
                request.getToolCallId(),
                request.getToolName(),
                "Tool failed: " + e.getMessage()
            );
        }
    }
}
```


**何时使用：**
- 希望工具失败时不中断整个 Agent 流程
- 需要给 LLM 提供错误信息，让它决定如何处理
- 生产环境需要健壮性

**对比：**
```
不使用拦截器:
  工具失败 → 抛出异常 → Agent 崩溃 ❌

使用 ToolErrorInterceptor:
  工具失败 → 返回错误消息 → LLM 看到错误 → 尝试其他方式 ✅
```


---

### **场景 5：工具缓存（自定义实现）**

**用途：** 缓存工具执行结果，避免重复调用

```java
public class ToolCacheInterceptor extends ToolInterceptor {
    
    private final Cache<String, ToolCallResponse> cache;
    private final Duration ttl;
    
    @Override
    public ToolCallResponse interceptToolCall(ToolCallRequest request, ToolCallHandler handler) {
        // 1. 生成缓存键
        String cacheKey = generateCacheKey(request);
        
        // 2. 检查缓存
        ToolCallResponse cached = cache.getIfPresent(cacheKey);
        if (cached != null) {
            log.info("Cache hit for tool '{}'", request.getToolName());
            return cached;  // ⭐ 返回缓存结果
        }
        
        // 3. 缓存未命中，执行工具
        log.info("Cache miss for tool '{}', executing...", request.getToolName());
        ToolCallResponse response = handler.call(request);
        
        // 4. 缓存结果（如果不是错误）
        if (!response.isError()) {
            cache.put(cacheKey, response);
            log.info("Cached result for tool '{}'", request.getToolName());
        }
        
        return response;
    }
    
    private String generateCacheKey(ToolCallRequest request) {
        // 工具名 + 参数的哈希值
        return request.getToolName() + ":" + 
               Integer.toHexString(request.getArguments().hashCode());
    }
}
```


**何时使用：**
- 工具调用成本高（API 费用、时间）
- 相同参数可能多次调用
- 数据变化不频繁

**效果：**
```
第 1 次: get_weather("Beijing") → 调用 API → 缓存结果
第 2 次: get_weather("Beijing") → 返回缓存 ✅ 节省时间和成本
```


---

### **场景 6：工具权限检查（自定义实现）**

**用途：** 检查用户是否有权限执行某个工具

```java
public class ToolAuthInterceptor extends ToolInterceptor {
    
    private final AuthService authService;
    private final Set<String> restrictedTools;
    
    @Override
    public ToolCallResponse interceptToolCall(ToolCallRequest request, ToolCallHandler handler) {
        String toolName = request.getToolName();
        
        // 1. 检查是否是受限工具
        if (!restrictedTools.contains(toolName)) {
            return handler.call(request);  // 不受限，直接执行
        }
        
        // 2. 获取用户 ID
        String userId = request.getExecutionContext()
                .flatMap(ctx -> ctx.config().threadId())
                .orElse("unknown");
        
        // 3. 检查权限
        if (!authService.hasToolPermission(userId, toolName)) {
            log.warn("User '{}' denied access to tool '{}'", userId, toolName);
            
            // ⭐ 返回权限拒绝错误
            return ToolCallResponse.error(
                request.getToolCallId(),
                toolName,
                "Access denied: You don't have permission to use this tool"
            );
        }
        
        // 4. 有权限，执行工具
        log.info("User '{}' authorized to use tool '{}'", userId, toolName);
        return handler.call(request);
    }
}
```


**何时使用：**
- 多租户环境
- 敏感操作需要权限控制
- 不同用户有不同的工具访问权限

---

### **场景 7：工具执行监控（自定义实现）**

**用途：** 记录工具执行的指标和日志

```java
public class ToolMonitoringInterceptor extends ToolInterceptor {
    
    private final MeterRegistry meterRegistry;
    private final Tracer tracer;
    
    @Override
    public ToolCallResponse interceptToolCall(ToolCallRequest request, ToolCallHandler handler) {
        String toolName = request.getToolName();
        long startTime = System.currentTimeMillis();
        
        // 1. 创建追踪 span
        Span span = tracer.spanBuilder("tool.execution")
                .setAttribute("tool.name", toolName)
                .setAttribute("tool.call_id", request.getToolCallId())
                .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            // 2. 执行工具
            ToolCallResponse response = handler.call(request);
            
            // 3. 记录指标
            long duration = System.currentTimeMillis() - startTime;
            
            meterRegistry.counter("tool.calls.total", 
                    "tool", toolName, 
                    "status", response.isError() ? "error" : "success")
                    .increment();
            
            meterRegistry.timer("tool.execution.duration", "tool", toolName)
                    .record(duration, TimeUnit.MILLISECONDS);
            
            // 4. 记录到 span
            span.setAttribute("tool.duration_ms", duration);
            span.setAttribute("tool.result_length", 
                    response.getResult().length());
            
            if (response.isError()) {
                span.setStatus(StatusCode.ERROR);
                span.recordException(new RuntimeException(response.getResult()));
            } else {
                span.setStatus(StatusCode.OK);
            }
            
            return response;
            
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR);
            span.recordException(e);
            throw e;
            
        } finally {
            span.end();
        }
    }
}
```


**何时使用：**
- 生产环境需要可观测性
- 需要监控工具执行性能
- 需要追踪问题根源

---

## 🎨 如何注册 ToolInterceptor

### **方式 1：通过 Agent Builder 直接注册**

```java
ReactAgent agent = ReactAgent.builder()
    .name("my-agent")
    .model(chatModel)
    .tools(weatherTool, searchTool)
    .toolInterceptors(  // ⭐ 注册工具拦截器
        new ToolRetryInterceptor(...),
        new ToolCacheInterceptor(...),
        new ToolMonitoringInterceptor(...)
    )
    .build();
```


### **方式 2：通过 Hook 注册**

```java
@HookPositions(HookPosition.BEFORE_AGENT)
public class MyToolHook extends AgentHook {
    
    @Override
    public List<ToolInterceptor> getToolInterceptors() {
        // ⭐ Hook 可以提供工具拦截器
        return List.of(
            new CustomToolInterceptor(),
            new AnotherToolInterceptor()
        );
    }
}

ReactAgent agent = ReactAgent.builder()
    .hook(new MyToolHook())  // 拦截器会自动注册
    .build();
```


### **方式 3：混合使用**

```java
// Agent 级别的拦截器（优先级更高）
ToolInterceptor agentLevelInterceptor = new AgentLevelToolInterceptor();

// Hook 级别的拦截器
@HookPositions(HookPosition.BEFORE_AGENT)
public class RetryHook extends AgentHook {
    @Override
    public List<ToolInterceptor> getToolInterceptors() {
        return List.of(new ToolRetryInterceptor(...));
    }
}

ReactAgent agent = ReactAgent.builder()
    .toolInterceptors(agentLevelInterceptor)  // 先执行
    .hook(new RetryHook())                    // 后执行
    .build();
```


**合并规则：**
```
最终拦截器顺序 = [Agent 级别拦截器] + [Hook 级别拦截器]
同名拦截器：Agent 级别优先
```


---

## 📊 ModelInterceptor vs ToolInterceptor 对比

| 特性 | ModelInterceptor | ToolInterceptor |
|------|------------------|-----------------|
| **作用对象** | LLM 调用 | 工具调用 |
| **调用位置** | AgentLlmNode | AgentToolNode |
| **请求类型** | `ModelRequest` | `ToolCallRequest` |
| **响应类型** | `ModelResponse` | `ToolCallResponse` |
| **处理方法** | `interceptModel()` | `interceptToolCall()` |
| **典型场景** | 重试、降级、Skills、日志 | 重试、缓存、模拟、权限、监控 |
| **执行频率** | 每次 LLM 调用 | 每次工具调用 |

---

## 🔄 完整执行流程图

```
LLM 返回: get_weather("Beijing"), search_web("北京天气")
  ↓
┌────────────────────────────────────────┐
│ AGENT_TOOL_NODE                        │
│                                        │
│ 并行/串行执行工具调用                   │
│                                        │
│ ┌──────────────────────────────────┐  │
│ │ Tool Call 1: get_weather         │  │
│ │                                  │  │
│ │ 1. 创建 ToolCallRequest          │  │
│ │    - toolName: "get_weather"     │  │
│ │    - arguments: {"city":"Beijing"}│ │
│ │    - toolCallId: "call_abc123"   │  │
│ │                                  │  │
│ │ 2. 创建拦截器链                   │  │
│ │    ┌─────────────────────────┐   │  │
│ │    │ MonitoringInterceptor   │   │  │
│ │    │  ├─ 开始计时            │   │  │
│ │    │  └─ handler.call()      │   │  │
│ │    │       ↓                 │   │  │
│ │    │  CacheInterceptor       │   │  │
│ │    │  ├─ 检查缓存            │   │  │
│ │    │  └─ handler.call()      │   │  │
│ │    │       ↓                 │   │  │
│ │    │  RetryInterceptor       │   │  │
│ │    │  ├─ try {               │   │  │
│ │    │  │   handler.call()     │   │  │
│ │    │  │   ↓                  │   │  │
│ │    │  │  AuthInterceptor     │   │  │
│ │    │  │  ├─ 检查权限         │   │  │
│ │    │  │  └─ handler.call()   │   │  │
│ │    │  │       ↓              │   │  │
│ │    │  │  baseHandler ⭐      │   │  │
│ │    │  │   ├─ 解析工具        │   │  │
│ │    │  │   ├─ 调用工具代码    │   │  │
│ │    │  │   └─ 返回结果        │   │  │
│ │    │  │   ↓                  │   │  │
│ │    │  │  返回响应             │   │  │
│ │    │  │ } catch { 重试 }     │   │  │
│ │    │  └─ 返回响应            │   │  │
│ │    │       ↑                 │   │  │
│ │    │  CacheInterceptor       │   │  │
│ │    │  └─ 缓存结果            │   │  │
│ │    │       ↑                 │   │  │
│ │    │  MonitoringInterceptor  │   │  │
│ │    │  ├─ 记录指标            │   │  │
│ │    │  └─ 返回响应            │   │  │
│ │    └─────────────────────────┘   │  │
│ │                                  │  │
│ │ 3. 返回 ToolCallResponse         │  │
│ └──────────────────────────────────┘  │
│                                        │
│ ┌──────────────────────────────────┐  │
│ │ Tool Call 2: search_web          │  │
│ │ (同样的拦截器链流程)              │  │
│ └──────────────────────────────────┘  │
└────────────────────────────────────────┘
  ↓
将工具结果添加到消息列表
  ↓
回到 LLM 推理...
```


---

## 🎯 总结

### **核心要点**

| 特性 | 说明 |
|------|------|
| **调用时机** | 每次执行工具前（AGENT_TOOL_NODE 内部） |
| **调用次数** | 每个工具调用 1 次（可能多个工具并行） |
| **设计模式** | 责任链模式（Chain of Responsibility） |
| **主要用途** | 重试、缓存、模拟、权限、监控、错误处理 |
| **执行顺序** | 注册的顺序（第一个拦截器最外层） |
| **返回值** | `ToolCallResponse`（可以继续传递或终止） |

### **完美！现在我有足够的信息来详细解读 `ToolInterceptor`。让我为你全面解析：

---

# 📊 ToolInterceptor 详解

## 🎯 核心概念

`ToolInterceptor` 是一个**工具调用拦截器抽象类**，用于在 Agent 执行工具调用**前后**插入自定义逻辑。它与 `ModelInterceptor` 对应，但作用对象不同：

- **ModelInterceptor**: 拦截 LLM 调用
- **ToolInterceptor**: 拦截工具调用

---

## ⏰ 什么时候被调用？

### **调用时机：每次 Agent 执行工具时**

```
LLM 返回: 我需要调用 weather_tool
  ↓
┌─────────────────────────────────┐
│ AFTER_MODEL Hooks               │
└──────────────┬──────────────────┘
               ↓
          检测到工具调用
               ↓
╔══════════════════════════════════╗
║ ⭐ AGENT_TOOL_NODE               ║ ← 这里调用 ToolInterceptor!
║   ┌──────────────────────────┐  ║
║   │ 1. 解析工具调用           │  ║
║   │ 2. 创建 ToolCallRequest  │  ║
║   │ 3. 创建拦截器链           │  ║
║   │ 4. 执行拦截器链 ⭐⭐⭐  │  ║
║   │    interceptor1          │  ║
║   │      ↓                   │  ║
║   │    interceptor2          │  ║
║   │      ↓                   │  ║
║   │    ...                   │  ║
║   │      ↓                   │  ║
║   │    baseHandler (实际工具) │  ║
║   │ 5. 返回 ToolCallResponse │  ║
║   └──────────────────────────┘  ║
╚══════════════════════════════════╝
               ↓
┌─────────────────────────────────┐
│ 将工具结果添加到消息历史         │
└──────────────┬──────────────────┘
               ↓
        (回到 ReAct 循环)
```


**关键点：**
- ✅ **每次工具调用都执行**：如果 LLM 一次调用 3 个工具，拦截器会被调用 3 次
- ✅ **在 AgentToolNode 内部**：具体在 `executeToolCallWithInterceptors()` 方法中
- ✅ **在实际执行工具代码之前**：可以修改参数、重试、缓存等

---

## 🔍 调用位置代码详解

### **位置：AgentToolNode.java 第 509-600+ 行**

```java
private ToolCallResponse executeToolCallWithInterceptors(
        AssistantMessage.ToolCall toolCall, 
        OverAllState state,
        RunnableConfig config, 
        Map<String, Object> extraStateFromToolCall, 
        boolean inParallelExecution,
        Map<Integer, DefaultCancellationToken> cancellationTokens, 
        int toolIndex) {

    // 1. 创建 ToolCallRequest
    ToolCallRequest request = ToolCallRequest.builder()
            .toolCall(toolCall)  // 从 LLM 的响应中提取
            .context(config.metadata().orElse(new HashMap<>()))
            .executionContext(new ToolCallExecutionContext(config, state))
            .build();

    // 2. 创建基础处理器（实际执行工具）
    ToolCallHandler baseHandler = req -> {
        ToolCallback toolCallback = resolve(req.getToolName(), config);
        
        if (toolCallback == null) {
            throw new IllegalStateException("Tool not found: " + req.getToolName());
        }
        
        // ⭐ 这里才是真正调用工具的地方
        ToolExecutionResult result = toolCallback.call(req.getArguments());
        
        return ToolCallResponse.of(
            req.getToolCallId(),
            req.getToolName(),
            result.getResult()
        );
    };

    // 3. ⭐⭐⭐ 链式包装所有拦截器
    ToolCallHandler chainedHandler = InterceptorChain.chainToolInterceptors(
            toolInterceptors,  // 拦截器列表
            baseHandler        // 基础处理器
    );

    // 4. ⭐⭐⭐ 执行拦截器链（从这里开始调用 interceptToolCall）
    ToolCallResponse response = chainedHandler.call(request);
    
    return response;
}
```


---

## 🔗 拦截器链的工作原理

与 `ModelInterceptor` 完全相同，使用责任链模式：

```java
// InterceptorChain.chainToolInterceptors()
public static ToolCallHandler chainToolInterceptors(
        List<ToolInterceptor> interceptors,
        ToolCallHandler baseHandler) {
    
    if (interceptors == null || interceptors.isEmpty()) {
        return baseHandler;
    }
    
    ModelCallHandler current = baseHandler;
    
    // 从后往前包装
    for (int i = interceptors.size() - 1; i >= 0; i--) {
        ToolInterceptor interceptor = interceptors.get(i);
        ToolCallHandler nextHandler = current;
        
        current = request -> interceptor.interceptToolCall(request, nextHandler);
    }
    
    return current;
}
```


**调用顺序示例：**

假设有拦截器 `[ToolRetryInterceptor, LoggingInterceptor]`：

```
request → ToolRetryInterceptor.interceptToolCall()
            ↓ (调用 handler.call())
          LoggingInterceptor.interceptToolCall()
            ↓ (调用 handler.call())
          baseHandler (实际执行工具)
            ↓
          工具执行结果
            ↑
          LoggingInterceptor 处理响应
            ↑
          ToolRetryInterceptor 处理响应
            ↑
          返回最终响应
```


---

## 💡 使用场景详解

### **场景 1：工具重试（ToolRetryInterceptor）**

**用途：** 当工具调用失败时自动重试

```java
public class ToolRetryInterceptor extends ToolInterceptor {
    
    private final int maxRetries;
    private final double backoffFactor;
    private final long initialDelayMs;
    private final Predicate<Exception> retryOn;
    
    @Override
    public ToolCallResponse interceptToolCall(ToolCallRequest request, ToolCallHandler handler) {
        Exception lastException = null;
        long delayMs = initialDelayMs;
        
        for (int attempt = 0; attempt <= maxRetries; attempt++) {
            try {
                if (attempt > 0) {
                    log.info("Retrying tool '{}' (attempt {}/{})", 
                            request.getToolName(), attempt, maxRetries);
                    
                    // 指数退避等待
                    Thread.sleep(delayMs);
                    delayMs *= backoffFactor;
                }
                
                // ⭐ 尝试调用工具
                ToolCallResponse response = handler.call(request);
                
                // 检查是否是错误响应
                if (!response.isError()) {
                    return response;  // 成功
                }
                
                lastException = new RuntimeException(response.getResult());
                
            } catch (Exception e) {
                lastException = e;
                log.warn("Tool call failed (attempt {}): {}", attempt + 1, e.getMessage());
                
                // 检查是否应该重试
                if (attempt < maxRetries && retryOn.test(e)) {
                    continue;  // 继续重试
                } else {
                    break;  // 不重试
                }
            }
        }
        
        // 所有重试都失败，返回错误
        return ToolCallResponse.error(
            request.getToolCallId(),
            request.getToolName(),
            lastException
        );
    }
}
```


**何时使用：**
- 网络工具可能超时（API 调用）
- 数据库连接可能暂时不可用
- 需要提高工具调用的可靠性

**配置示例：**
```java
ToolRetryInterceptor retryInterceptor = ToolRetryInterceptor.builder()
    .maxRetries(3)              // 最多重试 3 次
    .backoffFactor(2.0)         // 指数退避倍数
    .initialDelay(1000)         // 初始延迟 1 秒
    .retryOn(e -> e instanceof TimeoutException)  // 只重试超时异常
    .build();

ReactAgent agent = ReactAgent.builder()
    .model(chatModel)
    .tools(weatherApiTool)
    .interceptors(retryInterceptor)
    .build();
```


---

### **场景 2：工具模拟（ToolEmulatorInterceptor）**

**用途：** 测试时用 LLM 模拟工具，避免真实调用

```java
public class ToolEmulatorInterceptor extends ToolInterceptor {
    
    private final ChatModel emulatorModel;
    private final Set<String> toolsToEmulate;
    private final boolean emulateAll;
    
    @Override
    public ToolCallResponse interceptToolCall(ToolCallRequest request, ToolCallHandler handler) {
        String toolName = request.getToolName();
        
        // 检查是否需要模拟这个工具
        boolean shouldEmulate = emulateAll || toolsToEmulate.contains(toolName);
        
        if (!shouldEmulate) {
            // ⭐ 不需要模拟，正常执行
            return handler.call(request);
        }
        
        log.info("Emulating tool call: {}", toolName);
        
        try {
            // ⭐ 构建提示，让 LLM 模拟工具行为
            String prompt = String.format(
                "Simulate the following tool call:\n" +
                "Tool: %s\n" +
                "Arguments: %s\n" +
                "\nProvide a realistic response as if the tool was executed.",
                toolName,
                request.getArguments()
            );
            
            // 调用 LLM 生成模拟结果
            ChatResponse response = emulatorModel.call(
                new Prompt(new UserMessage(prompt))
            );
            
            String emulatedResult = response.getResult().getOutput().getText();
            
            log.debug("Emulated result: {}", emulatedResult);
            
            return ToolCallResponse.of(
                request.getToolCallId(),
                toolName,
                emulatedResult
            );
            
        } catch (Exception e) {
            log.error("Failed to emulate tool: {}", e.getMessage());
            // 模拟失败，降级到真实调用
            return handler.call(request);
        }
    }
}
```


**何时使用：**
- 单元测试（避免真实 API 调用）
- 开发环境（节省成本）
- 演示环境（稳定可控的结果）

**配置示例：**
```java
// 测试环境：模拟所有工具
ToolEmulatorInterceptor emulator = ToolEmulatorInterceptor.builder()
    .model(gpt4oMini)  // 用小模型模拟
    .emulateAll(true)
    .build();

// 或者只模拟特定工具
ToolEmulatorInterceptor partialEmulator = ToolEmulatorInterceptor.builder()
    .model(gpt4oMini)
    .addTool("expensive_api")  // 只模拟昂贵的 API
    .addTool("slow_database")   // 只模拟慢查询
    .build();
```


---

### **场景 3：大结果清理（LargeResultEvictionInterceptor）**

**用途：** 自动将超大的工具结果保存到文件，避免上下文爆炸

```java
public class LargeResultEvictionInterceptor extends ToolInterceptor {
    
    private static final int DEFAULT_TOOL_TOKEN_LIMIT = 20000;
    private static final String LARGE_RESULTS_DIR = "./large_tool_results/";
    
    private final int tokenLimit;
    private final Set<String> excludedTools;
    
    @Override
    public ToolCallResponse interceptToolCall(ToolCallRequest request, ToolCallHandler handler) {
        // 1. 先执行工具
        ToolCallResponse response = handler.call(request);
        
        // 2. 检查结果大小
        String result = response.getResult();
        int tokenCount = countTokens(result);
        
        if (tokenCount <= tokenLimit) {
            // 结果不大，直接返回
            return response;
        }
        
        // 3. ⭐ 结果太大，保存到文件
        log.info("Tool '{}' result too large ({} tokens), evicting to file", 
                request.getToolName(), tokenCount);
        
        // 生成文件名
        String filename = sanitizeToolCallId(request.getToolCallId()) + ".txt";
        Path filePath = Paths.get(LARGE_RESULTS_DIR, filename);
        
        try {
            // 保存完整结果到文件
            Files.createDirectories(filePath.getParent());
            Files.writeString(filePath, result);
            
            // 4. 生成摘要（前 10 行）
            String summary = generateSummary(result, 10);
            
            // 5. 返回指向文件的响应
            String evictionMessage = String.format(
                "[Tool result too large (%d tokens). Full result saved to: %s]\n\n" +
                "Preview (first 10 lines):\n%s",
                tokenCount,
                filePath.toAbsolutePath(),
                summary
            );
            
            return ToolCallResponse.of(
                request.getToolCallId(),
                request.getToolName(),
                evictionMessage
            );
            
        } catch (IOException e) {
            log.error("Failed to save large result to file", e);
            // 保存失败，返回截断的结果
            return ToolCallResponse.of(
                request.getToolCallId(),
                request.getToolName(),
                result.substring(0, Math.min(result.length(), 5000)) + "...[truncated]"
            );
        }
    }
    
    private String generateSummary(String content, int maxLines) {
        String[] lines = content.split("\n");
        int linesToShow = Math.min(lines.length, maxLines);
        
        StringBuilder summary = new StringBuilder();
        for (int i = 0; i < linesToShow; i++) {
            String line = lines[i];
            if (line.length() > 1000) {
                line = line.substring(0, 1000) + "...";
            }
            summary.append(line).append("\n");
        }
        
        if (lines.length > maxLines) {
            summary.append(String.format("\n... (%d more lines)", lines.length - maxLines));
        }
        
        return summary.toString();
    }
}
```


**何时使用：**
- 工具返回大量数据（如搜索结果显示 1000 条）
- 需要控制上下文窗口大小
- 避免超出 LLM 的 token 限制

**效果：**
```
原始结果: 50000 tokens (完整的 API 响应)
清理后: 500 tokens (摘要 + 文件路径)
节省: 99% Token 消耗 ✅
```


---

### **场景 4：工具错误处理（ToolErrorInterceptor）**

**用途：** 捕获工具异常，返回友好的错误消息

```java
public class ToolErrorInterceptor extends ToolInterceptor {
    
    @Override
    public ToolCallResponse interceptToolCall(ToolCallRequest request, ToolCallHandler handler) {
        try {
            // ⭐ 尝试执行工具
            return handler.call(request);
            
        } catch (Exception e) {
            log.error("Tool '{}' failed: {}", request.getToolName(), e.getMessage(), e);
            
            // ⭐ 返回友好的错误消息（而不是抛出异常）
            return ToolCallResponse.error(
                request.getToolCallId(),
                request.getToolName(),
                "Tool failed: " + e.getMessage()
            );
        }
    }
}
```


**为什么需要这个？**

如果不捕获异常：
```
工具抛出异常 → Agent 崩溃 ❌
```


使用错误拦截器：
```
工具抛出异常 → 返回错误消息 → LLM 看到错误 → 尝试其他方法 ✅
```


**LLM 可以看到错误并调整策略：**
```
用户: 帮我查天气
LLM: 调用 weather_tool
ToolErrorInterceptor: 返回 "Tool failed: API timeout"
LLM: 我看到天气 API 超时了，让我尝试备用方法...
     调用 backup_weather_tool
```


---

### **场景 5：工具调用日志（LoggingInterceptor）**

**用途：** 记录所有工具调用，便于调试和审计

```java
public class LoggingToolInterceptor extends ToolInterceptor {
    
    private static final Logger log = LoggerFactory.getLogger(LoggingToolInterceptor.class);
    
    @Override
    public ToolCallResponse interceptToolCall(ToolCallRequest request, ToolCallHandler handler) {
        long startTime = System.currentTimeMillis();
        
        log.info("🔧 Tool call started: {}", request.getToolName());
        log.debug("   Arguments: {}", request.getArguments());
        log.debug("   Call ID: {}", request.getToolCallId());
        
        try {
            // 执行工具
            ToolCallResponse response = handler.call(request);
            
            long duration = System.currentTimeMillis() - startTime;
            
            log.info("✅ Tool call completed: {} ({}ms)", 
                    request.getToolName(), duration);
            log.debug("   Result length: {} chars", 
                    response.getResult().length());
            
            return response;
            
        } catch (Exception e) {
            long duration = System.currentTimeMillis() - startTime;
            
            log.error("❌ Tool call failed: {} ({}ms)", 
                    request.getToolName(), duration);
            log.error("   Error: {}", e.getMessage());
            
            throw e;
        }
    }
}
```


**输出示例：**
```
🔧 Tool call started: get_weather
   Arguments: {"city": "Beijing"}
   Call ID: call_abc123
✅ Tool call completed: get_weather (234ms)
   Result length: 156 chars
```


---

### **场景 6：工具缓存（CachingInterceptor）**

**用途：** 缓存工具结果，避免重复调用

```java
public class CachingToolInterceptor extends ToolInterceptor {
    
    private final Cache<String, ToolCallResponse> cache;
    private final Duration ttl;
    
    public CachingToolInterceptor(Duration ttl) {
        this.ttl = ttl;
        this.cache = Caffeine.newBuilder()
            .expireAfterWrite(ttl)
            .maximumSize(1000)
            .build();
    }
    
    @Override
    public ToolCallResponse interceptToolCall(ToolCallRequest request, ToolCallHandler handler) {
        // 生成缓存键
        String cacheKey = generateCacheKey(request);
        
        // 检查缓存
        ToolCallResponse cached = cache.getIfPresent(cacheKey);
        if (cached != null) {
            log.info("Cache hit for tool: {}", request.getToolName());
            return cached;
        }
        
        // ⭐ 缓存未命中，执行工具
        ToolCallResponse response = handler.call(request);
        
        // 存入缓存（只缓存成功结果）
        if (!response.isError()) {
            cache.put(cacheKey, response);
            log.info("Cached result for tool: {}", request.getToolName());
        }
        
        return response;
    }
    
    private String generateCacheKey(ToolCallRequest request) {
        return request.getToolName() + ":" + request.getArguments();
    }
}
```


**何时使用：**
- 工具调用成本高（付费 API）
- 相同参数会频繁调用
- 结果在短时间内不会变化

**效果：**
```
第 1 次: get_weather("Beijing") → 调用 API (200ms) → 缓存
第 2 次: get_weather("Beijing") → 从缓存 (1ms) ✅ 节省 199ms
第 3 次: get_weather("Beijing") → 从缓存 (1ms) ✅ 节省 199ms
```


---

### **场景 7：权限检查（AuthInterceptor）**

**用途：** 检查工具调用权限

```java
public class AuthToolInterceptor extends ToolInterceptor {
    
    private final AuthService authService;
    private final Map<String, Set<String>> toolPermissions;  // tool -> required roles
    
    @Override
    public ToolCallResponse interceptToolCall(ToolCallRequest request, ToolCallHandler handler) {
        String toolName = request.getToolName();
        
        // 获取当前用户
        String userId = request.getExecutionContext()
            .flatMap(ctx -> ctx.threadId())
            .orElse("anonymous");
        
        // 检查权限
        Set<String> requiredRoles = toolPermissions.get(toolName);
        if (requiredRoles != null && !requiredRoles.isEmpty()) {
            if (!authService.hasRoles(userId, requiredRoles)) {
                log.warn("User {} denied access to tool {}", userId, toolName);
                
                return ToolCallResponse.error(
                    request.getToolCallId(),
                    toolName,
                    "Access denied: insufficient permissions"
                );
            }
        }
        
        // 权限通过，执行工具
        return handler.call(request);
    }
}
```


**何时使用：**
- 多租户系统
- 敏感操作（删除文件、支付等）
- 需要细粒度权限控制

---

## 🎨 如何注册 ToolInterceptor

### **方式 1：通过 Agent Builder 直接注册**

```java
ReactAgent agent = ReactAgent.builder()
    .name("my-agent")
    .model(chatModel)
    .tools(weatherTool, searchTool)
    .interceptors(  // ⭐ 注册工具拦截器
        new ToolRetryInterceptor(...),
        new LoggingToolInterceptor(),
        new CachingToolInterceptor(Duration.ofMinutes(5))
    )
    .build();
```


### **方式 2：通过 Hook 注册**

```java
@HookPositions(HookPosition.BEFORE_AGENT)
public class MyHook extends AgentHook {
    
    @Override
    public List<ToolInterceptor> getToolInterceptors() {
        // ⭐ Hook 可以提供工具拦截器
        return List.of(
            new AuthToolInterceptor(),
            new RateLimitingInterceptor()
        );
    }
}

ReactAgent agent = ReactAgent.builder()
    .hook(new MyHook())  // 拦截器会自动注册
    .build();
```


### **方式 3：混合使用**

```java
// Agent 级别的拦截器（优先级更高）
ToolInterceptor agentLevelInterceptor = new AgentLevelInterceptor();

// Hook 级别的拦截器
@HookPositions(HookPosition.BEFORE_AGENT)
public class RetryHook extends AgentHook {
    @Override
    public List<ToolInterceptor> getToolInterceptors() {
        return List.of(new ToolRetryInterceptor(...));
    }
}

ReactAgent agent = ReactAgent.builder()
    .interceptors(agentLevelInterceptor)  // 先执行
    .hook(new RetryHook())                // 后执行
    .build();
```


---

## 📊 ModelInterceptor vs ToolInterceptor 对比

| 特性 | ModelInterceptor | ToolInterceptor |
|------|-----------------|----------------|
| **拦截对象** | LLM 调用 | 工具调用 |
| **调用时机** | AGENT_MODEL_NODE | AGENT_TOOL_NODE |
| **请求类型** | ModelRequest | ToolCallRequest |
| **响应类型** | ModelResponse | ToolCallResponse |
| **核心方法** | `interceptModel()` | `interceptToolCall()` |
| **典型用途** | 重试、降级、Skills | 重试、缓存、权限 |
| **执行频率** | 每次 LLM 调用 | 每次工具调用 |

---

## 🔄 完整执行流程示例

```
用户: agent.call("帮我查北京和上海天气")
  ↓
Round 1:
┌──────────────────────────────────────┐
│ AGENT_MODEL_NODE                     │
│ ModelInterceptor 链执行              │
└──────────────┬───────────────────────┘
               ↓
          LLM 返回:
          我需要调用 get_weather 两次
               ↓
┌──────────────────────────────────────┐
│ AGENT_TOOL_NODE                      │
│                                      │
│ 工具 1: get_weather("Beijing")       │
│   ↓                                  │
│   ToolInterceptor 链:                │
│   ├─ CachingInterceptor              │
│   │  └─ 缓存未命中                   │
│   ├─ LoggingInterceptor              │
│   │  └─ 记录日志                     │
│   ├─ ToolRetryInterceptor            │
│   │  └─ try {                        │
│   │       handler.call() ⭐          │
│   │       ↓                          │
│   │       实际执行工具                │
│   │       ↓                          │
│   │       返回结果                    │
│   │     } catch { 重试 }             │
│   └─ 返回响应                        │
│                                      │
│ 工具 2: get_weather("Shanghai")      │
│   ↓                                  │
│   ToolInterceptor 链:                │
│   ├─ CachingInterceptor              │
│   │  └─ 缓存未命中                   │
│   ├─ LoggingInterceptor              │
│   ├─ ToolRetryInterceptor            │
│   └─ 返回响应                        │
└──────────────┬───────────────────────┘
               ↓
        (回到 Round 2)
```


---

## 📝 总结

### **核心要点**

| 特性 | 说明 |
|------|------|
| **调用时机** | 每次执行工具前（AGENT_TOOL_NODE 内部） |
| **调用次数** | 每个工具调用 1 次（可能并行多个） |
| **设计模式** | 责任链模式 |
| **主要用途** | 重试、缓存、日志、权限、模拟、错误处理 |
| **执行顺序** | 注册的顺序 |
| **返回值** | `ToolCallResponse` |

### **常见应用场景**

1. ✅ **工具重试**：ToolRetryInterceptor
2. ✅ **工具模拟**：ToolEmulatorInterceptor
3. ✅ **大结果清理**：LargeResultEvictionInterceptor
4. ✅ **错误处理**：ToolErrorInterceptor
5. ✅ **日志记录**：LoggingInterceptor
6. ✅ **结果缓存**：CachingInterceptor
7. ✅ **权限检查**：AuthInterceptor
8. ✅ **速率限制**：RateLimitingInterceptor
9. ✅ **监控追踪**：MonitoringInterceptor
10. ✅ **参数验证**：ValidationInterceptor

### **关键优势**

- 🎯 **解耦**：工具逻辑与横切关注点分离
- 🔄 **可组合**：多个拦截器灵活组合
- 🔧 **可扩展**：轻松添加新拦截器
- 🛡️ **健壮性**：统一的重试、错误处理
- 📊 **可观测**：统一的日志、监控

`ToolInterceptor` 是 Agent 框架中管理工具调用的强大机制，让你可以在不修改工具代码的情况下，为所有工具添加通用功能！😊
