

---

# 📊 ModelInterceptor 详解

## 🎯 核心概念

`ModelInterceptor` 是一个**拦截器抽象类**，用于在每次调用 LLM（大语言模型）**前后**插入自定义逻辑。它实现了**责任链模式**，允许多个拦截器按顺序处理请求和响应。

---

## ⏰ 什么时候被调用？

### **调用时机：每次 Agent 调用 LLM 时**

```
用户调用 agent.call("问题")
  ↓
Agent 执行流程
  ↓
┌─────────────────────────────────┐
│ BEFORE_AGENT Hooks (1次)        │
└──────────────┬──────────────────┘
               ↓
      ╔═══════════════════════╗
      ║ ReAct 循环开始         ║
      ╚════════╤══════════════╝
               ↓
┌─────────────────────────────────┐
│ BEFORE_MODEL Hooks              │
└──────────────┬──────────────────┘
               ↓
╔══════════════════════════════════╗
║ ⭐ AGENT_MODEL_NODE              ║ ← 这里调用 ModelInterceptor!
║   ┌──────────────────────────┐  ║
║   │ 1. 构建 ModelRequest     │  ║
║   │ 2. 创建拦截器链           │  ║
║   │ 3. 执行拦截器链 ⭐⭐⭐  │  ║
║   │    interceptor1          │  ║
║   │      ↓                   │  ║
║   │    interceptor2          │  ║
║   │      ↓                   │  ║
║   │    ...                   │  ║
║   │      ↓                   │  ║
║   │    baseHandler (LLM API) │  ║
║   │ 4. 返回 ModelResponse    │  ║
║   └──────────────────────────┘  ║
╚══════════════════════════════════╝
               ↓
┌─────────────────────────────────┐
│ AFTER_MODEL Hooks               │
└──────────────┬──────────────────┘
               ↓
          需要工具？
        ┌───┴───┐
       YES      NO
        │        │
        ↓        │
  TOOL_NODE      │
        │        │
        └────────┘
               ↓
        (回到循环开始)
```


**关键点：**
- ✅ **每次 LLM 调用都执行**：如果 Agent 进行了 5 轮推理，拦截器会被调用 5 次
- ✅ **在 AgentLlmNode 内部**：具体在 `AgentLlmNode.apply()` 方法中
- ✅ **在实际调用 LLM API 之前**：可以修改请求、重试、降级等

---

## 🔍 调用位置代码详解

### **位置：AgentLlmNode.java 第 236-241 行（streaming）或 270-279 行（blocking）**

```java
// AgentLlmNode.apply() 方法中

// 1. 创建基础处理器（实际调用 LLM API）
ModelCallHandler baseHandler = request -> {
    try {
        // ⭐ 这里才是真正调用 LLM 的地方
        Flux<ChatResponse> chatResponseFlux = buildChatClientRequestSpec(request, config)
                .stream()
                .chatResponse();
        return ModelResponse.of(chatResponseFlux);
    } catch (Exception e) {
        logger.error("Exception during streaming model call: ", e);
        return ModelResponse.of(new AssistantMessage("Exception: " + e.getMessage()));
    }
};

// 2. ⭐⭐⭐ 链式包装所有拦截器
ModelCallHandler chainedHandler = InterceptorChain.chainModelInterceptors(
        modelInterceptors,  // 拦截器列表
        baseHandler         // 基础处理器
);

// 3. ⭐⭐⭐ 执行拦截器链（从这里开始调用 interceptModel）
ModelResponse modelResponse = chainedHandler.call(modelRequest);
```


---

## 🔗 拦截器链的工作原理

### **InterceptorChain.chainModelInterceptors() 实现**

```java
public static ModelCallHandler chainModelInterceptors(
        List<ModelInterceptor> interceptors,
        ModelCallHandler baseHandler) {
    
    if (interceptors == null || interceptors.isEmpty()) {
        return baseHandler;
    }
    
    // 从基础处理器开始
    ModelCallHandler current = baseHandler;
    
    // ⭐ 从后往前包装（右到左组合）
    for (int i = interceptors.size() - 1; i >= 0; i--) {
        ModelInterceptor interceptor = interceptors.get(i);
        ModelCallHandler nextHandler = current;
        
        // 创建包装器
        current = request -> interceptor.interceptModel(request, nextHandler);
    }
    
    return current;
}
```


### **示例：3 个拦截器的包装过程**

假设有拦截器 `[SkillsInterceptor, LoggingInterceptor, RetryInterceptor]`：

```java
// 初始状态
current = baseHandler

// 第 1 次循环 (i=2): RetryInterceptor
current = req -> RetryInterceptor.interceptModel(req, baseHandler)

// 第 2 次循环 (i=1): LoggingInterceptor
current = req -> LoggingInterceptor.interceptModel(req, 
            req -> RetryInterceptor.interceptModel(req, baseHandler))

// 第 3 次循环 (i=0): SkillsInterceptor
current = req -> SkillsInterceptor.interceptModel(req,
            req -> LoggingInterceptor.interceptModel(req,
                req -> RetryInterceptor.interceptModel(req, baseHandler)))
```


**最终调用顺序：**

```
request → SkillsInterceptor.interceptModel()
            ↓ (调用 handler.call())
          LoggingInterceptor.interceptModel()
            ↓ (调用 handler.call())
          RetryInterceptor.interceptModel()
            ↓ (调用 handler.call())
          baseHandler (实际调用 LLM)
            ↓
          LLM Response
            ↑
          RetryInterceptor 处理响应
            ↑
          LoggingInterceptor 处理响应
            ↑
          SkillsInterceptor 处理响应
            ↑
          返回最终响应
```


---

## 💡 使用场景详解

### **场景 1：Skills 管理（SkillsInterceptor）**

**用途：** 注入 skills 元数据到系统提示，实现渐进式披露

```java
public class SkillsInterceptor extends ModelInterceptor {
    
    @Override
    public ModelResponse interceptModel(ModelRequest request, ModelCallHandler handler) {
        // ① 获取所有 skills
        List<SkillMetadata> skills = skillRegistry.listAll();
        
        if (skills.isEmpty()) {
            return handler.call(request);
        }
        
        // ② 扫描消息历史，提取 read_skill 调用
        Set<String> readSkillNames = extractReadSkillNames(request.getMessages());
        
        // ③ 动态添加对应的工具
        List<ToolCallback> skillTools = new ArrayList<>(request.getDynamicToolCallbacks());
        for (String skillName : readSkillNames) {
            List<ToolCallback> toolsForSkill = groupedTools.get(skillName);
            if (toolsForSkill != null) {
                skillTools.addAll(toolsForSkill);
            }
        }
        
        // ④ 构建 skills prompt
        String skillsPrompt = buildSkillsPrompt(skills, skillRegistry, ...);
        SystemMessage enhanced = enhanceSystemMessage(request.getSystemMessage(), skillsPrompt);
        
        // ⑤ 修改请求并继续
        ModelRequest modified = ModelRequest.builder(request)
                .systemMessage(enhanced)
                .dynamicToolCallbacks(skillTools)
                .build();
        
        return handler.call(modified);  // ⭐ 调用下一个拦截器
    }
}
```


**何时使用：**
- 需要实现 Claude-style Skills 系统
- 需要按需加载技能内容
- 需要动态注入工具

---

### **场景 2：模型重试（ModelRetryInterceptor）**

**用途：** 当 LLM 调用失败时自动重试

```java
public class ModelRetryInterceptor extends ModelInterceptor {
    
    @Override
    public ModelResponse interceptModel(ModelRequest request, ModelCallHandler handler) {
        Exception lastException = null;
        long currentDelay = initialDelay;
        
        for (int attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                if (attempt > 1) {
                    log.info("Retry model call, on the {}th attempt", attempt);
                }
                
                // ⭐ 尝试调用 LLM
                ModelResponse response = handler.call(request);
                
                // 检查响应是否包含异常
                Message message = (Message) response.getMessage();
                if (message != null && message.getText().startsWith("Exception:")) {
                    throw new RuntimeException(message.getText());
                }
                
                // 成功，返回响应
                return response;
                
            } catch (Exception e) {
                lastException = e;
                log.warn("Attempt {} failed: {}", attempt, e.getMessage());
                
                if (attempt < maxAttempts && isRetryableException(e)) {
                    // ⭐ 指数退避等待
                    Thread.sleep(currentDelay);
                    currentDelay *= backoffMultiplier;
                } else {
                    break;
                }
            }
        }
        
        throw new RuntimeException("All attempts failed", lastException);
    }
}
```


**何时使用：**
- 网络不稳定，LLM API 可能超时
- 需要提高系统容错性
- 需要实现指数退避策略

**配置示例：**
```java
ModelRetryInterceptor retryInterceptor = ModelRetryInterceptor.builder()
    .maxAttempts(3)           // 最多重试 3 次
    .initialDelay(1000)       // 初始延迟 1 秒
    .maxDelay(10000)          // 最大延迟 10 秒
    .backoffMultiplier(2.0)   // 指数退避倍数
    .build();

ReactAgent agent = ReactAgent.builder()
    .model(chatModel)
    .interceptors(retryInterceptor)
    .build();
```


---

### **场景 3：模型降级（ModelFallbackInterceptor）**

**用途：** 主模型失败时切换到备用模型

```java
public class ModelFallbackInterceptor extends ModelInterceptor {
    
    private final List<ChatModel> fallbackModels;
    
    @Override
    public ModelResponse interceptModel(ModelRequest request, ModelCallHandler handler) {
        Exception lastException = null;
        
        // ① 尝试主模型
        try {
            ModelResponse response = handler.call(request);
            Message message = (Message) response.getMessage();
            
            if (message.getText().contains("Exception:")) {
                throw new RuntimeException(message.getText());
            }
            
            return response;  // 成功
            
        } catch (Exception e) {
            log.warn("Primary model failed: {}", e.getMessage());
            lastException = e;
        }
        
        // ② 尝试备用模型
        for (int i = 0; i < fallbackModels.size(); i++) {
            ChatModel fallbackModel = fallbackModels.get(i);
            try {
                log.info("Trying fallback model {} of {}", i + 1, fallbackModels.size());
                
                // ⭐ 直接调用备用模型（不经过拦截器链）
                Prompt prompt = new Prompt(request.getMessages(), request.getOptions());
                var response = fallbackModel.call(prompt);
                
                return ModelResponse.of(response.getResult().getOutput());
                
            } catch (Exception e) {
                log.warn("Fallback model {} failed: {}", i + 1, e.getMessage());
                lastException = e;
            }
        }
        
        throw new RuntimeException("All models failed", lastException);
    }
}
```


**何时使用：**
- 有多个 LLM 提供商（如 OpenAI + Anthropic + 本地模型）
- 需要高可用性保障
- 成本优化（主模型贵，备用模型便宜）

**配置示例：**
```java
ModelFallbackInterceptor fallbackInterceptor = ModelFallbackInterceptor.builder()
    .addFallbackModel(gpt4o)           // 主模型
    .addFallbackModel(claude35)        // 备用 1
    .addFallbackModel(localLlama)      // 备用 2
    .build();
```


---

### **场景 4：工具选择（ToolSelectionInterceptor）**

**用途：** 使用小模型筛选相关工具，减少 Token 消耗

```java
public class ToolSelectionInterceptor extends ModelInterceptor {
    
    private final ChatModel selectionModel;  // 小模型（如 GPT-4o-mini）
    private final int maxTools;
    
    @Override
    public ModelResponse interceptModel(ModelRequest request, ModelCallHandler handler) {
        List<String> allTools = request.getTools();
        
        if (allTools.size() <= maxTools) {
            // 工具不多，不需要筛选
            return handler.call(request);
        }
        
        // ⭐ 使用小模型筛选相关工具
        String prompt = buildSelectionPrompt(request.getMessages(), allTools);
        Prompt selectionPrompt = new Prompt(List.of(new UserMessage(prompt)));
        
        var selectionResponse = selectionModel.call(selectionPrompt);
        List<String> selectedTools = parseSelectedTools(selectionResponse);
        
        // 修改请求，只保留选中的工具
        ModelRequest filteredRequest = ModelRequest.builder(request)
                .tools(selectedTools)
                .build();
        
        log.info("Selected {} tools from {} available", selectedTools.size(), allTools.size());
        
        return handler.call(filteredRequest);
    }
}
```


**何时使用：**
- Agent 有大量工具（>10 个）
- 需要减少每次调用的 Token 消耗
- 提高 LLM 的工具选择准确性

**效果：**
```
原始: 50 个工具 → 5000 tokens
筛选后: 3 个工具 → 300 tokens
节省: 94% Token 消耗 ✅
```


---

### **场景 5：上下文编辑（ContextEditingInterceptor）**

**用途：** 清理旧的工具结果，控制上下文长度

```java
public class ContextEditingInterceptor extends ModelInterceptor {
    
    private final int trigger;      // 触发清理的 token 阈值
    private final int keep;         // 保留最近的 N 条工具结果
    private final String placeholder;
    
    @Override
    public ModelResponse interceptModel(ModelRequest request, ModelCallHandler handler) {
        List<Message> messages = request.getMessages();
        
        // 计算当前 token 数
        int tokenCount = countTokens(messages);
        
        if (tokenCount > trigger) {
            // ⭐ 超过阈值，清理旧的工具结果
            List<Message> cleanedMessages = cleanToolResults(messages, keep, placeholder);
            
            log.info("Context editing: reduced from {} to {} tokens", 
                    tokenCount, countTokens(cleanedMessages));
            
            ModelRequest cleanedRequest = ModelRequest.builder(request)
                    .messages(cleanedMessages)
                    .build();
            
            return handler.call(cleanedRequest);
        }
        
        return handler.call(request);
    }
    
    private List<Message> cleanToolResults(List<Message> messages, int keep, String placeholder) {
        // 找到所有 ToolResponseMessage
        List<Integer> toolResultIndices = findToolResults(messages);
        
        // 保留最近的 keep 条，其他的替换为占位符
        for (int i = 0; i < toolResultIndices.size() - keep; i++) {
            int idx = toolResultIndices.get(i);
            messages.set(idx, new ToolResponseMessage(
                List.of(new ToolResponseMessage.ToolResponse(
                    "...", placeholder  // ⭐ 替换为占位符
                ))
            ));
        }
        
        return messages;
    }
}
```


**何时使用：**
- 长对话场景
- 需要控制上下文窗口大小
- 避免超出 LLM 的 token 限制

---

### **场景 6：文件系统增强（FilesystemInterceptor）**

**用途：** 注入文件系统工具的指导说明

```java
public class FilesystemInterceptor extends ModelInterceptor {
    
    private final String systemPrompt;
    private final List<ToolCallback> tools;
    
    @Override
    public ModelResponse interceptModel(ModelRequest request, ModelCallHandler handler) {
        SystemMessage enhancedSystemMessage;
        
        if (request.getSystemMessage() == null) {
            enhancedSystemMessage = new SystemMessage(this.systemPrompt);
        } else {
            // ⭐ 增强系统提示，添加文件系统使用指南
            enhancedSystemMessage = new SystemMessage(
                request.getSystemMessage().getText() + "\n\n" + systemPrompt
            );
        }
        
        // 修改请求
        ModelRequest enhancedRequest = ModelRequest.builder(request)
                .systemMessage(enhancedSystemMessage)
                .build();
        
        return handler.call(enhancedRequest);
    }
    
    @Override
    public List<ToolCallback> getTools() {
        // ⭐ 提供文件系统工具
        return tools;  // ls, read_file, write_file, edit_file, glob, grep
    }
}
```


**何时使用：**
- 需要让 Agent 操作文件系统
- 需要提供文件操作的指导说明
- 需要注入内置工具

---

### **场景 7：子代理调用（SubAgentInterceptor）**

**用途：** 提供 task 工具，允许主 Agent 调用子 Agent

```java
public class SubAgentInterceptor extends ModelInterceptor {
    
    private final Map<String, ReactAgent> subAgents;
    
    @Override
    public ModelResponse interceptModel(ModelRequest request, ModelCallHandler handler) {
        // ⭐ 增强系统提示，说明如何使用子代理
        String subAgentGuidance = buildSubAgentGuidance(subAgents);
        
        SystemMessage enhanced = new SystemMessage(
            request.getSystemMessage().getText() + "\n\n" + subAgentGuidance
        );
        
        ModelRequest enhancedRequest = ModelRequest.builder(request)
                .systemMessage(enhanced)
                .build();
        
        return handler.call(enhancedRequest);
    }
    
    @Override
    public List<ToolCallback> getTools() {
        // ⭐ 提供 task 工具
        return List.of(createTaskTool(subAgents));
    }
}
```


**何时使用：**
- 复杂任务需要分解
- 需要隔离不同任务的上下文
- 需要专业化的子代理

---

## 🎨 如何注册 ModelInterceptor

### **方式 1：通过 Agent Builder 直接注册**

```java
ReactAgent agent = ReactAgent.builder()
    .name("my-agent")
    .model(chatModel)
    .interceptors(  // ⭐ 直接注册拦截器
        new SkillsInterceptor(...),
        ModelRetryInterceptor.builder().maxAttempts(3).build(),
        new LoggingInterceptor()
    )
    .build();
```


### **方式 2：通过 Hook 注册**

```java
@HookPositions(HookPosition.BEFORE_AGENT)
public class MyHook extends AgentHook {
    
    @Override
    public List<ModelInterceptor> getModelInterceptors() {
        // ⭐ Hook 可以提供拦截器
        return List.of(
            new CustomInterceptor(),
            new AnotherInterceptor()
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
ModelInterceptor agentLevelInterceptor = new AgentLevelInterceptor();

// Hook 级别的拦截器
@HookPositions(HookPosition.BEFORE_AGENT)
public class SkillsAgentHook extends AgentHook {
    @Override
    public List<ModelInterceptor> getModelInterceptors() {
        return List.of(new SkillsInterceptor(...));
    }
}

ReactAgent agent = ReactAgent.builder()
    .interceptors(agentLevelInterceptor)  // 先执行
    .hook(new SkillsAgentHook())          // 后执行
    .build();
```


**合并规则：**
```
最终拦截器顺序 = [Agent 级别拦截器] + [Hook 级别拦截器]
```


---

## 📊 完整执行流程图

```
用户: agent.call("帮我查北京天气")
  ↓
┌──────────────────────────────────────┐
│ BEFORE_AGENT Hooks                   │
│ - InstructionAgentHook               │
│ - SkillsAgentHook (加载 skills)      │
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
║ ⭐ AGENT_MODEL_NODE                    ║
║                                        ║
║  1. 构建 ModelRequest                  ║
║     - messages: [UserMessage(...)]    ║
║     - tools: [read_skill, weather]    ║
║                                        ║
║  2. 创建拦截器链                        ║
║     ┌─────────────────────────────┐   ║
║     │ SkillsInterceptor           │   ║
║     │  ├─ 注入 skills prompt      │   ║
║     │  └─ handler.call()          │   ║
║     │       ↓                     │   ║
║     │  LoggingInterceptor         │   ║
║     │  ├─ 记录请求日志            │   ║
║     │  └─ handler.call()          │   ║
║     │       ↓                     │   ║
║     │  RetryInterceptor           │   ║
║     │  ├─ try {                   │   ║
║     │  │   handler.call()         │   ║
║     │  │   ↓                      │   ║
║     │  │  baseHandler (LLM API) ⭐║   ║
║     │  │   ↓                      │   ║
║     │  │  返回响应                 │   ║
║     │  │ } catch { 重试 }         │   ║
║     │  └─ 返回响应                │   ║
║     │       ↑                     │   ║
║     │  LoggingInterceptor         │   ║
║     │  └─ 记录响应日志            │   ║
║     │       ↑                     │   ║
║     │  SkillsInterceptor          │   ║
║     │  └─ 返回响应                │   ║
║     └─────────────────────────────┘   ║
║                                        ║
║  3. 返回 ModelResponse                 ║
╚════════════════════════════════════════╝
               ↓
┌──────────────────────────────────────┐
│ AFTER_MODEL Hooks                    │
└──────────────┬───────────────────────┘
               ↓
          LLM 返回:
          read_skill("weather-skill")
               ↓
┌──────────────────────────────────────┐
│ AGENT_TOOL_NODE                      │
│ 执行 read_skill 工具                  │
└──────────────┬───────────────────────┘
               ↓
        (回到 Round 2)
```


---

## 🎯 总结

### **核心要点**

| 特性 | 说明 |
|------|------|
| **调用时机** | 每次调用 LLM API 前（AGENT_MODEL_NODE 内部） |
| **调用次数** | 每次 ReAct 循环调用 1 次（可能多次循环） |
| **设计模式** | 责任链模式（Chain of Responsibility） |
| **主要用途** | 修改请求/响应、重试、降级、日志、权限检查等 |
| **执行顺序** | 注册的顺序（第一个拦截器最外层） |
| **返回值** | `ModelResponse`（可以继续传递或终止） |

### **常见应用场景**

1. ✅ **Skills 管理**：SkillsInterceptor
2. ✅ **重试机制**：ModelRetryInterceptor
3. ✅ **模型降级**：ModelFallbackInterceptor
4. ✅ **工具选择**：ToolSelectionInterceptor
5. ✅ **上下文管理**：ContextEditingInterceptor
6. ✅ **功能增强**：FilesystemInterceptor, TodoListInterceptor
7. ✅ **子代理**：SubAgentInterceptor
8. ✅ **日志记录**：LoggingInterceptor
9. ✅ **权限检查**：AuthInterceptor
10. ✅ **缓存**：CacheInterceptor

### **关键优势**

- 🎯 **解耦**：业务逻辑与横切关注点分离
- 🔄 **可组合**：多个拦截器可以灵活组合
- 🔧 **可扩展**：轻松添加新的拦截器
- 🛡️ **健壮性**：统一的重试、降级、错误处理
- 📊 **可观测**：统一的日志、监控、追踪

`ModelInterceptor` 是 Agent 框架中非常强大的扩展机制，让你可以在不修改核心代码的情况下，为 Agent 添加各种高级功能！😊
