
---

# 📊 HumanInTheLoopHook 详解

## 🎯 核心概念

**Human-in-the-Loop (HITL)** 是一种让人类在 Agent 执行过程中进行干预的机制。`HumanInTheLoopHook` 允许人类：
- ✅ **审批**工具调用（Approve）
- ✏️ **编辑**工具参数（Edit）
- ❌ **拒绝**工具调用（Reject）

这是一个**安全机制**，防止 Agent 执行危险或昂贵的操作。

---

## ⏰ 什么时候被调用？

### **调用时机：每次 LLM 返回后，工具执行前**

```
LLM 推理完成，返回工具调用
  ↓
┌─────────────────────────────────┐
│ AFTER_MODEL Hooks               │
│   ↓                             │
│ HumanInTheLoopHook.afterModel() │ ⭐ 这里检查是否需要人工审批
└──────────────┬──────────────────┘
               ↓
          需要审批？
        ┌───┴───┐
       YES      NO
        │        │
        ▼        │
   中断执行      │
   等待人类      │
   反馈          │
        │        │
        ▼        │
   人类审批      │
   (Approve/     │
    Edit/        │
    Reject)      │
        │        │
        ▼        │
   继续执行      │
        │        │
        └────────┘
               ↓
┌─────────────────────────────────┐
│ AGENT_TOOL_NODE                 │
│ 执行已批准的工具                 │
└─────────────────────────────────┘
```


**关键点：**
- ✅ **位置**：`AFTER_MODEL` 阶段（模型调用后，工具执行前）
- ✅ **条件触发**：只有配置了 `approvalOn` 的工具才会触发
- ✅ **中断机制**：需要审批时会中断执行，等待人类反馈

---

## 🔍 代码结构详解

### **1. 类定义与注解**

```java
@HookPositions(HookPosition.AFTER_MODEL)  // ⭐ 在模型调用后执行
public class HumanInTheLoopHook extends ModelHook 
        implements AsyncNodeActionWithConfig, InterruptableAction {
    
    public static final String HITL_NODE_NAME = "HITL";
    private Map<String, ToolConfig> approvalOn;  // ⭐ 需要审批的工具列表
}
```


**关键接口：**
- `ModelHook`: 模型级别的 Hook
- `AsyncNodeActionWithConfig`: 支持异步节点执行
- `InterruptableAction`: **可中断动作**（核心！）

---

### **2. 核心方法：interrupt() - 决定是否中断**

**位置：** 第 148-169 行

```java
@Override
public Optional<InterruptionMetadata> interrupt(
        String nodeId, 
        OverAllState state, 
        RunnableConfig config) {
    
    // ① 获取最后一条 AssistantMessage
    AssistantMessage lastMessage = getLastAssistantMessage(state);
    
    // ② 如果没有工具调用，不中断
    if (lastMessage == null || !lastMessage.hasToolCalls()) {
        return Optional.empty();
    }
    
    // ③ 检查是否已经有用户反馈
    Optional<Object> feedback = config.metadata(
        RunnableConfig.HUMAN_FEEDBACK_METADATA_KEY
    );
    
    if (feedback.isPresent()) {
        if (!(feedback.get() instanceof InterruptionMetadata)) {
            throw new IllegalArgumentException(
                "Human feedback metadata must be of type InterruptionMetadata."
            );
        }
        
        // ⭐ 验证反馈是否完整
        if (!validateFeedback(
                (InterruptionMetadata) feedback.get(), 
                lastMessage.getToolCalls())) {
            // 反馈不完整，继续中断
            return buildInterruptionMetadata(state, lastMessage);
        }
        
        // 反馈完整，不再中断
        return Optional.empty();
    }
    
    // ④ 没有反馈，构建中断元数据
    return buildInterruptionMetadata(state, lastMessage);
}
```


**工作流程：**

```
1. 检查最后一条消息是否有工具调用
   ↓
2. 检查 config 中是否有人类反馈
   ↓
3. 如果有反馈：
   ├─ 验证反馈是否完整
   ├─ 完整 → 不中断（继续执行）
   └─ 不完整 → 中断（等待更多反馈）
   ↓
4. 如果没有反馈：
   └─ 构建中断元数据（请求人类审批）
```


---

### **3. 构建中断元数据：buildInterruptionMetadata()**

**位置：** 第 190-211 行

```java
private Optional<InterruptionMetadata> buildInterruptionMetadata(
        OverAllState state, 
        AssistantMessage lastMessage) {
    
    boolean needsInterruption = false;
    InterruptionMetadata.Builder builder = 
        InterruptionMetadata.builder(Hook.getFullHookName(this), state);
    
    // ⭐ 遍历所有工具调用
    for (AssistantMessage.ToolCall toolCall : lastMessage.getToolCalls()) {
        
        // 检查这个工具是否需要审批
        if (approvalOn.containsKey(toolCall.name())) {
            
            ToolConfig toolConfig = approvalOn.get(toolCall.name());
            String description = toolConfig.getDescription();
            
            // 构建审批提示
            String content = "The AI is requesting to use the tool: " 
                    + toolCall.name() + ".\n"
                    + (description != null ? ("Description: " + description + "\n") : "")
                    + "With the following arguments: " + toolCall.arguments() + "\n"
                    + "Do you approve?";
            
            // ⭐ 添加工具反馈请求
            builder.addToolFeedback(
                ToolFeedback.builder()
                    .id(toolCall.id())
                    .name(toolCall.name())
                    .description(content)
                    .arguments(toolCall.arguments())
                    .build()
            );
            
            needsInterruption = true;
            
        } else {
            // 不需要审批的工具，自动批准
            builder.addToolsAutomaticallyApproved(toolCall);
        }
    }
    
    // 如果有需要审批的工具，返回中断元数据
    return needsInterruption ? Optional.of(builder.build()) : Optional.empty();
}
```


**示例输出：**

假设 LLM 调用了两个工具：
- `delete_file`（需要审批）
- `read_weather`（不需要审批）

生成的 `InterruptionMetadata`：

```json
{
  "hook_name": "agent.hook.HITL",
  "tool_feedbacks": [
    {
      "id": "call_abc123",
      "name": "delete_file",
      "arguments": "{\"path\": \"/important/data.txt\"}",
      "description": "The AI is requesting to use the tool: delete_file.\nDescription: Delete a file from the filesystem\nWith the following arguments: {\"path\": \"/important/data.txt\"}\nDo you approve?"
    }
  ],
  "automatically_approved_tools": [
    {
      "id": "call_def456",
      "name": "read_weather",
      "arguments": "{\"city\": \"Beijing\"}"
    }
  ]
}
```


---

### **4. 处理人类反馈：afterModel()**

**位置：** 第 67-145 行

```java
@Override
public CompletableFuture<Map<String, Object>> afterModel(
        OverAllState state, 
        RunnableConfig config) {
    
    // ① 从 config 中获取并移除人类反馈
    Optional<InterruptionMetadata> feedback = config.getMetadataAndRemove(
        RunnableConfig.HUMAN_FEEDBACK_METADATA_KEY, 
        new TypeRef<InterruptionMetadata>() {}
    );
    
    InterruptionMetadata interruptionMetadata = feedback.orElse(null);
    
    // ② 如果没有反馈，直接返回
    if (interruptionMetadata == null) {
        log.debug("No human feedback found...");
        return CompletableFuture.completedFuture(Map.of());
    }
    
    AssistantMessage assistantMessage = getLastAssistantMessage(state);
    
    if (assistantMessage != null && assistantMessage.hasToolCalls()) {
        
        List<AssistantMessage.ToolCall> newToolCalls = new ArrayList<>();
        List<ToolResponseMessage.ToolResponse> responses = new ArrayList<>();
        
        // ⭐ 遍历所有工具调用，根据人类反馈处理
        for (AssistantMessage.ToolCall toolCall : assistantMessage.getToolCalls()) {
            
            Optional<ToolFeedback> toolFeedbackOpt = interruptionMetadata.toolFeedbacks().stream()
                    .filter(tf -> tf.getId().equals(toolCall.id()))
                    .findFirst();
            
            if (toolFeedbackOpt.isPresent()) {
                ToolFeedback toolFeedback = toolFeedbackOpt.get();
                FeedbackResult result = toolFeedback.getResult();
                
                // ③ 根据反馈结果处理
                if (result == FeedbackResult.APPROVED) {
                    // ✅ 批准：保留原工具调用
                    newToolCalls.add(toolCall);
                    
                } else if (result == FeedbackResult.EDITED) {
                    // ✏️ 编辑：使用修改后的参数
                    AssistantMessage.ToolCall editedToolCall = 
                        new AssistantMessage.ToolCall(
                            toolCall.id(), 
                            toolCall.type(), 
                            toolCall.name(), 
                            toolFeedback.getArguments()  // ⭐ 使用编辑后的参数
                        );
                    newToolCalls.add(editedToolCall);
                    
                } else if (result == FeedbackResult.REJECTED) {
                    // ❌ 拒绝：添加拒绝消息
                    newToolCalls.add(toolCall);
                    
                    ToolResponseMessage.ToolResponse response = 
                        new ToolResponseMessage.ToolResponse(
                            toolCall.id(), 
                            toolCall.name(), 
                            String.format(
                                "Tool call request for %s has been rejected by human. " +
                                "The reason for why this tool is rejected and the suggestion " +
                                "for next possible tool choose is listed as below:\n %s.",
                                toolFeedback.getName(),
                                toolFeedback.getDescription()
                            )
                        );
                    responses.add(response);
                }
                
            } else {
                // 没有反馈的工具，默认批准
                newToolCalls.add(toolCall);
            }
        }
        
        // ④ 更新消息历史
        Map<String, Object> updates = new HashMap<>();
        List<Object> newMessages = new ArrayList<>();
        
        if (!newToolCalls.isEmpty()) {
            // 替换最后一条 AssistantMessage
            newMessages.add(AssistantMessage.builder()
                .content(assistantMessage.getText())
                .properties(assistantMessage.getMetadata())
                .toolCalls(newToolCalls)  // ⭐ 使用更新后的工具调用
                .media(assistantMessage.getMedia())
                .build());
            
            // 标记删除旧消息
            newMessages.add(new RemoveByHash<>(assistantMessage));
        }
        
        // 添加拒绝消息
        if (!responses.isEmpty()) {
            ToolResponseMessage rejectedMessage = 
                ToolResponseMessage.builder().responses(responses).build();
            newMessages.add(rejectedMessage);
        }
        
        updates.put("messages", newMessages);
        return CompletableFuture.completedFuture(updates);
    }
    
    return CompletableFuture.completedFuture(Map.of());
}
```


**三种反馈结果的处理：**

#### **场景 A：APPROVED（批准）**

```
原始工具调用: delete_file("/important/data.txt")
人类反馈: APPROVED
结果: 保留原调用，继续执行
```


#### **场景 B：EDITED（编辑）**

```
原始工具调用: delete_file("/important/data.txt")
人类反馈: EDITED, arguments = {"path": "/tmp/temp.txt"}
结果: 修改为 delete_file("/tmp/temp.txt")，然后执行
```


#### **场景 C：REJECTED（拒绝）**

```
原始工具调用: delete_file("/important/data.txt")
人类反馈: REJECTED, description = "这个文件很重要，不能删除"
结果: 
  1. 不执行工具
  2. 添加 ToolResponseMessage:
     "Tool call request for delete_file has been rejected by human. 
      The reason: 这个文件很重要，不能删除"
  3. LLM 看到拒绝消息，可能选择其他方案
```


---

### **5. 验证反馈：validateFeedback()**

**位置：** 第 213-269 行

```java
private boolean validateFeedback(
        InterruptionMetadata feedback, 
        List<AssistantMessage.ToolCall> toolCalls) {
    
    if (feedback == null || feedback.toolFeedbacks() == null) {
        return false;
    }
    
    // ① 找出需要审批的工具
    List<AssistantMessage.ToolCall> toolCallsNeedingApproval = toolCalls.stream()
            .filter(tc -> approvalOn.containsKey(tc.name()))
            .toList();
    
    // 如果没有需要审批的工具，验证通过
    if (toolCallsNeedingApproval.isEmpty()) {
        return true;
    }
    
    // ② 检查每个需要审批的工具是否有对应的反馈
    for (AssistantMessage.ToolCall call : toolCallsNeedingApproval) {
        InterruptionMetadata.ToolFeedback matchedFeedback = 
            feedback.toolFeedbacks().stream()
                .filter(tf -> tf.getName().equals(call.name()) 
                           && call.id().equals(tf.getId()))
                .findFirst()
                .orElse(null);
        
        if (matchedFeedback == null) {
            log.warn("Missing feedback for tool {} (id={})", 
                    call.name(), call.id());
            return false;  // ⭐ 缺少反馈，验证失败
        }
        
        if (matchedFeedback.getResult() == null) {
            log.warn("Feedback result for tool {} is null", call.name());
            return false;  // ⭐ 反馈结果为空，验证失败
        }
    }
    
    return true;  // ✅ 所有需要审批的工具都有反馈
}
```


**作用：** 确保人类对所有需要审批的工具都给出了明确的反馈。

---

## 💡 使用场景详解

### **场景 1：危险操作审批（文件删除）**

```java
// 配置需要审批的工具
HumanInTheLoopHook hitlHook = HumanInTheLoopHook.builder()
    .approvalOn("delete_file", "Delete a file from the filesystem")
    .approvalOn("execute_command", "Execute a shell command")
    .approvalOn("send_email", "Send an email to recipients")
    .build();

ReactAgent agent = ReactAgent.builder()
    .name("file-manager")
    .model(chatModel)
    .tools(deleteFileTool, executeCommandTool, sendEmailTool)
    .hook(hitlHook)
    .saver(new MemorySaver())  // ⭐ 需要保存状态
    .build();
```


**执行流程：**

```
用户: 帮我删除 /important/data.txt
  ↓
LLM: 我需要调用 delete_file("/important/data.txt")
  ↓
HumanInTheLoopHook.interrupt() 触发
  ↓
⚠️ 执行中断，等待人类反馈
  ↓
前端显示审批界面:
┌──────────────────────────────────────┐
│ AI 请求执行以下操作：                  │
│                                      │
│ 工具: delete_file                    │
│ 描述: Delete a file from the         │
│       filesystem                     │
│ 参数: {"path": "/important/data.txt"}│
│                                      │
│ [✅ 批准] [✏️ 编辑] [❌ 拒绝]       │
└──────────────────────────────────────┘
  ↓
人类选择: ✏️ 编辑，修改为 /tmp/temp.txt
  ↓
HumanInTheLoopHook.afterModel() 处理
  ↓
工具调用被修改为: delete_file("/tmp/temp.txt")
  ↓
继续执行工具
```


---

### **场景 2：昂贵 API 调用审批**

```java
HumanInTheLoopHook hitlHook = HumanInTheLoopHook.builder()
    .approvalOn("call_expensive_api", 
        "Call external API (costs $0.10 per request)")
    .approvalOn("database_write", 
        "Write to production database")
    .build();
```


**为什么需要审批？**
- 💰 成本控制：避免意外的高额 API 费用
- 🔒 数据安全：防止误操作生产数据库
- ✅ 质量保证：确保关键操作的准确性

---

### **场景 3：多工具批量审批**

```java
// LLM 一次调用多个工具
AssistantMessage:
  - delete_file("/file1.txt")
  - delete_file("/file2.txt")
  - read_weather("Beijing")  // 不需要审批

HumanInTheLoopHook 生成审批请求:
  - delete_file("/file1.txt") → 需要审批
  - delete_file("/file2.txt") → 需要审批
  - read_weather("Beijing") → 自动批准

前端显示:
┌──────────────────────────────────────┐
│ AI 请求执行 2 个需要审批的操作：       │
│                                      │
│ 1. delete_file("/file1.txt")         │
│    [✅] [✏️] [❌]                    │
│                                      │
│ 2. delete_file("/file2.txt")         │
│    [✅] [✏️] [❌]                    │
│                                      │
│ [全部批准] [全部拒绝]                 │
└──────────────────────────────────────┘
```


---

### **场景 4：参数编辑（修正错误）**

```java
// 场景：LLM 写错了文件路径
LLM: delete_file("/importnat/data.txt")  // 拼写错误
  ↓
人类发现错误，编辑参数:
  原参数: {"path": "/importnat/data.txt"}
  新参数: {"path": "/important/data.txt"}
  ↓
HumanInTheLoopHook 修改工具调用
  ↓
执行: delete_file("/important/data.txt")  // 正确的路径
```


---

### **场景 5：拒绝并提供建议**

```java
// 场景：人类拒绝并给出建议
LLM: delete_file("/production/config.yml")
  ↓
人类拒绝，原因: "这是生产配置文件，不能删除。建议使用 backup_file 工具备份。"
  ↓
HumanInTheLoopHook 添加拒绝消息:
  "Tool call request for delete_file has been rejected by human.
   The reason: 这是生产配置文件，不能删除。
   Suggestion: 建议使用 backup_file 工具备份。"
  ↓
LLM 看到拒绝消息，调整策略:
  "我理解了，让我先备份文件..."
  调用: backup_file("/production/config.yml")
```


---

## 🎨 完整使用示例

### **后端配置**

```java
@Configuration
public class AgentConfig {
    
    @Bean
    public ReactAgent createAgent(ChatModel chatModel) {
        
        // 1. 创建 HITL Hook
        HumanInTheLoopHook hitlHook = HumanInTheLoopHook.builder()
            .approvalOn("delete_file", "Delete a file permanently")
            .approvalOn("execute_sql", "Execute SQL query on database")
            .approvalOn("send_notification", "Send notification to users")
            .build();
        
        // 2. 创建 Agent
        return ReactAgent.builder()
            .name("safe-agent")
            .model(chatModel)
            .tools(
                deleteFileTool,
                executeSqlTool,
                sendNotificationTool,
                readFileTool  // 不需要审批
            )
            .hook(hitlHook)
            .saver(new MemorySaver())  // ⭐ 必须配置状态保存
            .build();
    }
}
```


### **前端交互（伪代码）**

```javascript
// 1. 调用 Agent
async function callAgent(userInput) {
    const config = {
        threadId: 'user-123',
        streamMode: 'values'
    };
    
    const stream = await agent.stream({ input: userInput }, config);
    
    for await (const chunk of stream) {
        // 2. 检查是否有中断
        if (chunk.metadata?.interrupted) {
            const interruption = chunk.metadata.interruption;
            
            // 3. 显示审批界面
            showApprovalUI(interruption.toolFeedbacks);
            
            // 4. 等待用户输入
            const userFeedback = await waitForUserFeedback();
            
            // 5. 提交反馈，继续执行
            await resumeWithFeedback(config.threadId, userFeedback);
        }
    }
}

// 6. 恢复执行
async function resumeWithFeedback(threadId, feedback) {
    const config = {
        threadId: threadId,
        metadata: {
            human_feedback: feedback  // ⭐ 传入人类反馈
        }
    };
    
    // 继续执行
    const stream = await agent.stream(null, config);
    // ...
}
```


### **审批界面示例（React）**

```jsx
function ApprovalUI({ toolFeedbacks, onApprove, onReject, onEdit }) {
    return (
        <div className="approval-panel">
            <h3>AI 请求执行以下操作：</h3>
            
            {toolFeedbacks.map((feedback) => (
                <div key={feedback.id} className="tool-request">
                    <h4>工具: {feedback.name}</h4>
                    <p>{feedback.description}</p>
                    <pre>{feedback.arguments}</pre>
                    
                    <div className="actions">
                        <button onClick={() => onApprove(feedback.id)}>
                            ✅ 批准
                        </button>
                        <button onClick={() => onEdit(feedback.id)}>
                            ✏️ 编辑
                        </button>
                        <button onClick={() => onReject(feedback.id)}>
                            ❌ 拒绝
                        </button>
                    </div>
                </div>
            ))}
        </div>
    );
}
```


---

## 🔄 完整执行流程图

```
用户输入: "删除重要文件"
  ↓
┌──────────────────────────────────────┐
│ Round 1: LLM 推理                    │
└──────────────┬───────────────────────┘
               ↓
          LLM 返回:
          delete_file("/important.txt")
               ↓
┌──────────────────────────────────────┐
│ AFTER_MODEL: HumanInTheLoopHook      │
│                                      │
│ interrupt() 检查:                    │
│ - 有工具调用？YES                    │
│ - 需要审批？YES (delete_file)        │
│ - 有反馈？NO                         │
│                                      │
│ → 返回 InterruptionMetadata          │
└──────────────┬───────────────────────┘
               ↓
          ⚠️ 执行中断
               ↓
      等待人类反馈...
               ↓
      人类查看审批界面:
      ┌────────────────────┐
      │ 工具: delete_file  │
      │ 参数: /important   │
      │                    │
      │ [批准][编辑][拒绝] │
      └────────────────────┘
               ↓
      人类选择: 编辑
      修改为: /tmp/temp.txt
               ↓
      前端提交反馈:
      {
        "tool_feedbacks": [{
          "id": "call_123",
          "result": "EDITED",
          "arguments": "{\"path\":\"/tmp/temp.txt\"}"
        }]
      }
               ↓
┌──────────────────────────────────────┐
│ Round 2: 继续执行                     │
│                                      │
│ afterModel() 处理反馈:               │
│ - 检测到 EDITED                      │
│ - 修改工具调用参数                    │
│ - 更新消息历史                        │
└──────────────┬───────────────────────┘
               ↓
┌──────────────────────────────────────┐
│ AGENT_TOOL_NODE                      │
│ 执行: delete_file("/tmp/temp.txt")   │
└──────────────┬───────────────────────┘
               ↓
          工具执行成功
               ↓
┌──────────────────────────────────────┐
│ Round 3: LLM 总结                    │
└──────────────┬───────────────────────┘
               ↓
          返回最终结果
```


---

## 📝 关键注意事项

### **1. 必须配置状态保存**

```java
// ❌ 错误：没有 saver，中断后无法恢复
ReactAgent agent = ReactAgent.builder()
    .hook(hitlHook)
    .build();

// ✅ 正确：配置 saver
ReactAgent agent = ReactAgent.builder()
    .hook(hitlHook)
    .saver(new MemorySaver())  // 或 RedisSaver, PostgresSaver
    .build();
```


**为什么？** 中断后需要保存当前状态，等人类反馈后再恢复。

---

### **2. 反馈验证很重要**

```java
// validateFeedback() 确保：
// 1. 所有需要审批的工具都有反馈
// 2. 每个反馈都有明确的结果（APPROVED/EDITED/REJECTED）
// 3. 反馈的工具 ID 匹配

// 如果验证失败，会继续中断，直到收到完整反馈
```


---

### **3. 自动批准 vs 需要审批**

```java
HumanInTheLoopHook hitlHook = HumanInTheLoopHook.builder()
    .approvalOn("delete_file", "...")  // ⭐ 需要审批
    // read_file 不在 approvalOn 中 → 自动批准
    .build();

// 执行时:
// - delete_file → 中断，等待审批
// - read_file → 自动执行，不中断
```


---

### **4. 拒绝后的 LLM 行为**

当工具被拒绝时，LLM 会收到拒绝消息，可以：
- 尝试其他方法
- 询问用户更多信息
- 解释为什么需要这个工具

**示例：**
```
LLM: 调用 delete_file("/prod/db.yml")
人类: 拒绝，原因"这是生产数据库配置"
LLM 看到拒绝消息:
  "我明白了，我不应该删除生产配置。
   让我先备份，或者您可以告诉我具体想做什么？"
```


---

## 🎯 总结

### **核心价值**

| 特性 | 说明 |
|------|------|
| **安全性** | 防止危险操作（删除文件、执行命令） |
| **成本控制** | 审批昂贵的 API 调用 |
| **质量保证** | 人类可以纠正 LLM 的错误 |
| **灵活性** | 支持批准、编辑、拒绝三种操作 |
| **透明度** | 人类可以看到 AI 的所有意图 |

### **适用场景**

1. ✅ **文件系统操作**：删除、移动、修改文件
2. ✅ **数据库操作**：写入、删除、更新数据
3. ✅ **外部 API**：付费 API、敏感数据访问
4. ✅ **通知发送**：邮件、短信、推送
5. ✅ **系统命令**：执行 shell 命令
6. ✅ **任何不可逆操作**

### **关键组件**

- `interrupt()`: 决定是否中断
- `buildInterruptionMetadata()`: 构建审批请求
- `afterModel()`: 处理人类反馈
- `validateFeedback()`: 验证反馈完整性

### **最佳实践**

1. 🎯 **只审批关键操作**：不要对所有工具都要求审批
2. 📝 **提供清晰的描述**：让人类理解操作的后果
3. 🔄 **支持编辑参数**：给人类修正错误的机会
4. 💬 **拒绝时提供建议**：帮助 LLM 学习更好的策略
5. 💾 **务必配置 saver**：否则中断后无法恢复

`HumanInTheLoopHook` 是构建**安全可靠**的 Agent 系统的关键组件，让人类在关键时刻保持控制权！😊
