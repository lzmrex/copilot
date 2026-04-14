
---

# 📊 SkillsInterceptor 详解

## 🎯 核心概念

`SkillsInterceptor` 是一个 **ModelInterceptor**，用于实现 **Claude-style Skills** 系统。它的核心理念是**渐进式披露（Progressive Disclosure）**：

- 📋 **第一步**：只给 LLM 看 skills 的轻量级列表（名称 + 描述）
- 📖 **第二步**：LLM 需要时，调用 `read_skill` 工具读取完整内容
- 🔧 **第三步**：根据读取的 skill，动态注入相关工具

这与传统方式（一次性把所有 skills 内容都放入 prompt）相比，**大幅节省 Token**。

---

## ⏰ 什么时候被调用？

### **调用时机：每次 LLM 调用前**

```
用户输入问题
  ↓
┌──────────────────────────────────────┐
│ BEFORE_AGENT Hooks                   │
│ SkillsAgentHook.beforeAgent()        │
│   └─ 加载/重载 skills                │
└──────────────┬───────────────────────┘
               ↓
┌──────────────────────────────────────┐
│ BEFORE_MODEL Hooks                   │
└──────────────┬───────────────────────┘
               ↓
╔══════════════════════════════════════╗
║ ⭐ AGENT_MODEL_NODE                  ║
║                                      ║
║ 1. 构建 ModelRequest                 ║
║ 2. 创建拦截器链                       ║
║ 3. 执行 SkillsInterceptor ⭐⭐⭐   ║
║    interceptModel()                  ║
║    ├─ 注入 skills prompt            ║
║    ├─ 提取 read_skill 调用          ║
║    ├─ 动态添加工具                   ║
║    └─ handler.call()                ║
║         ↓                            ║
║      调用 LLM API                    ║
║ 4. 返回 ModelResponse                ║
╚══════════════════════════════════════╝
```


**关键点：**
- ✅ **位置**：AGENT_MODEL_NODE 内部，实际调用 LLM 之前
- ✅ **频率**：每次 LLM 调用都执行（ReAct 循环中可能多次）
- ✅ **作用**：修改 ModelRequest，增强系统提示和动态工具

---

## 🔍 代码结构详解

### **1. 类定义与字段**

```java
public class SkillsInterceptor extends ModelInterceptor {
    
    private static final Logger logger = LoggerFactory.getLogger(SkillsInterceptor.class);
    
    // ⭐ SkillRegistry：存储所有 skills 元数据
    private final SkillRegistry skillRegistry;
    
    // ⭐ groupedTools：skill 名称 → 工具列表的映射
    // 用于动态注入工具
    private final Map<String, List<ToolCallback>> groupedTools;
    
    private SkillsInterceptor(Builder builder) {
        if (builder.skillRegistry == null) {
            throw new IllegalArgumentException(
                "SkillRegistry must be provided. Use SkillsAgentHook to load skills."
            );
        }
        this.skillRegistry = builder.skillRegistry;
        this.groupedTools = builder.groupedTools != null
                ? builder.groupedTools
                : Collections.emptyMap();
    }
}
```


**关键字段说明：**

| 字段 | 类型 | 作用 |
|------|------|------|
| `skillRegistry` | `SkillRegistry` | 存储和管理所有 skills 的元数据 |
| `groupedTools` | `Map<String, List<ToolCallback>>` | skill 名称到工具列表的映射，用于动态注入 |

---

### **2. 核心方法：interceptModel()**

**位置：** 第 117-154 行

这是整个类的**核心逻辑**，实现了渐进式披露和动态工具注入。

```java
@Override
public ModelResponse interceptModel(
        ModelRequest request, 
        ModelCallHandler handler) {
    
    // ① 从 SkillRegistry 获取所有 skills
    List<SkillMetadata> skills = skillRegistry.listAll();
    
    // 如果没有 skills，直接调用下一个处理器
    if (skills.isEmpty()) {
        return handler.call(request);
    }
    
    // ② 扫描消息历史，提取 read_skill 工具调用中的 skill_name
    Set<String> readSkillNames = extractReadSkillNames(request.getMessages());
    
    // ③ 根据提取的 skill_name，从 groupedTools 中获取对应的工具
    List<ToolCallback> skillTools = new ArrayList<>(request.getDynamicToolCallbacks());
    Map<String, List<ToolCallback>> grouped = getGroupedTools();
    
    for (String skillName : readSkillNames) {
        List<ToolCallback> toolsForSkill = grouped.get(skillName);
        
        if (toolsForSkill != null && !toolsForSkill.isEmpty()) {
            // ⭐ 动态添加工具到请求中
            skillTools.addAll(toolsForSkill);
            
            if (logger.isInfoEnabled()) {
                logger.info("SkillsInterceptor: added {} tool(s) for skill '{}' to dynamicToolCallbacks",
                        toolsForSkill.size(), skillName);
            }
        }
    }
    
    // ④ 构建 skills prompt（轻量级元数据）
    String skillsPrompt = buildSkillsPrompt(
        skills, 
        skillRegistry, 
        skillRegistry.getSystemPromptTemplate()
    );
    
    // ⑤ 增强系统消息
    SystemMessage enhanced = enhanceSystemMessage(
        request.getSystemMessage(), 
        skillsPrompt
    );
    
    if (logger.isDebugEnabled()) {
        logger.debug("Enhanced system message:\n{}", enhanced.getText());
    }
    
    // ⑥ 构建修改后的请求
    ModelRequest modified = ModelRequest.builder(request)
            .systemMessage(enhanced)           // ⭐ 增强的系统消息
            .dynamicToolCallbacks(skillTools)  // ⭐ 动态添加的工具
            .build();
    
    // ⑦ 继续调用链
    return handler.call(modified);
}
```


**工作流程图解：**

```
输入: ModelRequest
  ↓
┌─────────────────────────────────────┐
│ 1. 获取所有 skills                  │
│    skills = skillRegistry.listAll() │
└──────────────┬──────────────────────┘
               ↓
          有 skills？
        ┌───┴───┐
       NO       YES
        │        │
        │        ▼
        │   ┌─────────────────────────────┐
        │   │ 2. 提取 read_skill 调用     │
        │   │    extractReadSkillNames()  │
        │   │                             │
        │   │ 扫描消息历史，找到:          │
        │   │ - read_skill("weather")    │
        │   │ - read_skill("calculator") │
        │   └──────────┬──────────────────┘
        │              ↓
        │   ┌─────────────────────────────┐
        │   │ 3. 动态添加工具             │
        │   │                             │
        │   │ groupedTools:               │
        │   │ {                           │
        │   │   "weather": [getWeather],  │
        │   │   "calculator": [calc]      │
        │   │ }                           │
        │   │                             │
        │   │ 提取到: ["weather"]         │
        │   │ → 添加 getWeather 工具      │
        │   └──────────┬──────────────────┘
        │              ↓
        │   ┌─────────────────────────────┐
        │   │ 4. 构建 skills prompt       │
        │   │                             │
        │   │ [AVAILABLE SKILLS]          │
        │   │ - weather: 查询天气         │
        │   │ - calculator: 数学计算      │
        │   │ - ...                       │
        │   └──────────┬──────────────────┘
        │              ↓
        │   ┌─────────────────────────────┐
        │   │ 5. 增强系统消息             │
        │   │                             │
        │   │ 原始: "你是一个助手"        │
        │   │ 增强: "你是一个助手\n\n     │
        │   │ [AVAILABLE SKILLS]..."      │
        │   └──────────┬──────────────────┘
        │              ↓
        │   ┌─────────────────────────────┐
        │   │ 6. 构建新的 ModelRequest    │
        │   │                             │
        │   │ - systemMessage: 增强版     │
        │   │ - dynamicToolCallbacks:     │
        │   │   [getWeather]              │
        │   └──────────┬──────────────────┘
        │              ↓
        └──────────────┘
               ↓
输出: ModelResponse (经过 handler.call())
```


---

### **3. 提取 read_skill 调用：extractReadSkillNames()**

**位置：** 第 160-180 行

```java
private Set<String> extractReadSkillNames(List<Message> messages) {
    if (messages == null || messages.isEmpty()) {
        return Set.of();
    }
    
    Set<String> names = new LinkedHashSet<>();
    
    // ⭐ 遍历所有消息
    for (Message message : messages) {
        
        // 只处理 AssistantMessage 且有工具调用的消息
        if (!(message instanceof AssistantMessage assistantMessage) 
                || !assistantMessage.hasToolCalls()) {
            continue;
        }
        
        // 遍历所有工具调用
        for (AssistantMessage.ToolCall toolCall : assistantMessage.getToolCalls()) {
            
            // ⭐ 只关心 read_skill 工具
            if (!ReadSkillTool.READ_SKILL.equals(toolCall.name())) {
                continue;
            }
            
            // 解析参数，提取 skill_name
            String skillName = parseSkillNameFromArguments(toolCall.arguments());
            
            if (skillName != null && !skillName.isEmpty()) {
                names.add(skillName);
            }
        }
    }
    
    return names;
}
```


**示例：**

假设消息历史中有：

```json
[
  {
    "role": "user",
    "content": "帮我查北京天气"
  },
  {
    "role": "assistant",
    "tool_calls": [
      {
        "name": "read_skill",
        "arguments": "{\"skill_name\": \"weather-skill\"}"
      }
    ]
  },
  {
    "role": "tool",
    "content": "SKILL.md 内容..."
  }
]
```


`extractReadSkillNames()` 会返回：`["weather-skill"]`

---

### **4. 解析 skill_name：parseSkillNameFromArguments()**

**位置：** 第 182-199 行

```java
private static String parseSkillNameFromArguments(String arguments) {
    if (arguments == null || arguments.isBlank()) {
        return null;
    }
    
    try {
        // ⭐ 解析 JSON 参数
        Object parsed = JsonParser.fromJson(arguments, Map.class);
        
        if (parsed instanceof Map<?, ?> map) {
            // 提取 skill_name 字段
            Object v = map.get("skill_name");
            return v != null ? v.toString().trim() : null;
        }
    }
    catch (Exception e) {
        if (logger.isDebugEnabled()) {
            logger.debug("Failed to parse read_skill arguments: {}", e.getMessage());
        }
    }
    
    return null;
}
```


**示例：**

```java
// 输入
arguments = "{\"skill_name\": \"weather-skill\", \"other\": \"value\"}"

// 解析过程
JsonParser.fromJson(arguments, Map.class)
  → {"skill_name": "weather-skill", "other": "value"}

map.get("skill_name")
  → "weather-skill"

// 输出
"weather-skill"
```


---

### **5. 增强系统消息：enhanceSystemMessage()**

**位置：** 第 210-215 行

```java
private SystemMessage enhanceSystemMessage(SystemMessage existing, String skillsSection) {
    if (existing == null) {
        // 没有现有系统消息，直接返回 skills 部分
        return new SystemMessage(skillsSection);
    }
    
    // ⭐ 将 skills 部分追加到现有系统消息后面
    return new SystemMessage(existing.getText() + "\n\n" + skillsSection);
}
```


**示例：**

```java
// 输入
existing = "你是一个有用的助手，可以帮助用户解决问题。"
skillsSection = "[AVAILABLE SKILLS]\n- weather: 查询天气\n- calc: 数学计算"

// 输出
"你是一个有用的助手，可以帮助用户解决问题。\n\n[AVAILABLE SKILLS]\n- weather: 查询天气\n- calc: 数学计算"
```


---

## 💡 使用场景详解

### **场景 1：自动注册（推荐方式）**

通过 `SkillsAgentHook` 自动创建和注册 `SkillsInterceptor`。

```java
// 1. 创建 SkillRegistry
FileSystemSkillRegistry registry = FileSystemSkillRegistry.builder()
    .userSkillsDirectory("~/saa/skills")      // 用户技能目录
    .projectSkillsDirectory("./skills")        // 项目技能目录
    .build();

// 2. 创建 SkillsAgentHook
SkillsAgentHook hook = SkillsAgentHook.builder()
    .skillRegistry(registry)
    .autoReload(true)  // 每次调用前重新加载 skills
    .build();

// 3. 创建 Agent（拦截器自动注册）
ReactAgent agent = ReactAgent.builder()
    .name("skill-agent")
    .model(chatModel)
    .hook(hook)  // ⭐ SkillsInterceptor 会自动注册
    .build();
```


**优点：**
- ✅ 简单，开箱即用
- ✅ 自动管理 `read_skill` 工具
- ✅ 自动创建 `SkillsInterceptor`

---

### **场景 2：手动注册 + 动态工具注入**

当你需要将特定工具与 skills 关联时使用。

```java
// 1. 创建 SkillRegistry
FileSystemSkillRegistry registry = FileSystemSkillRegistry.builder()
    .userSkillsDirectory("~/skills")
    .build();

// 2. 准备工具
ToolCallback getWeatherTool = createGetWeatherTool();
ToolCallback forecastTool = createForecastTool();
ToolCallback calculatorTool = createCalculatorTool();

// 3. 配置 groupedTools
Map<String, List<ToolCallback>> groupedTools = Map.of(
    "weather-skill", List.of(getWeatherTool, forecastTool),
    "math-skill", List.of(calculatorTool)
);

// 4. 手动创建 SkillsInterceptor
SkillsInterceptor interceptor = SkillsInterceptor.builder()
    .skillRegistry(registry)
    .groupedTools(groupedTools)  // ⭐ 配置动态工具注入
    .build();

// 5. 创建 Agent
ReactAgent agent = ReactAgent.builder()
    .name("skill-agent")
    .model(chatModel)
    .interceptors(interceptor)  // ⭐ 手动注册
    .build();
```


**工作流程：**

```
Round 1:
用户: 帮我查天气
  ↓
LLM: 我需要先了解 weather-skill
     调用: read_skill("weather-skill")
  ↓
SkillsInterceptor:
  - 注入 skills prompt
  - 检测到 read_skill("weather-skill")
  - 但这是第一次调用，还没有历史记录
  - 不添加工具
  ↓
执行 read_skill 工具
  ↓
返回 SKILL.md 内容

Round 2:
用户: (继续对话)
  ↓
LLM: 现在我了解了 weather-skill，可以使用它的工具
     调用: get_weather("Beijing")
  ↓
SkillsInterceptor:
  - 扫描历史，发现 Round 1 的 read_skill("weather-skill")
  - 从 groupedTools 获取 weather-skill 的工具
  - 动态添加: [getWeatherTool, forecastTool]
  ↓
执行 get_weather 工具 ✅
```


---

### **场景 3：多技能组合**

一个 Agent 可以使用多个 skills，每个 skill 有自己的工具集。

```java
// 配置多个 skills 的工具
Map<String, List<ToolCallback>> groupedTools = Map.of(
    "weather-skill", List.of(getWeatherTool, forecastTool),
    "calendar-skill", List.of(createEventTool, listEventsTool),
    "email-skill", List.of(sendEmailTool, readEmailTool)
);

SkillsInterceptor interceptor = SkillsInterceptor.builder()
    .skillRegistry(registry)
    .groupedTools(groupedTools)
    .build();
```


**执行流程：**

```
用户: 帮我查明天北京的天气，如果有雨就发邮件提醒我

Round 1:
LLM: 我需要了解 weather-skill 和 email-skill
     调用: 
       - read_skill("weather-skill")
       - read_skill("email-skill")

Round 2:
SkillsInterceptor 检测到两个 read_skill 调用
  → 动态添加: [getWeatherTool, forecastTool, sendEmailTool, readEmailTool]

LLM: 现在我有所有需要的工具
     调用: 
       - get_weather("Beijing", "tomorrow")
       - 如果预报有雨: send_email(...)
```


---

### **场景 4：按需加载工具（节省 Token）**

**传统方式的问题：**

```
系统提示包含所有工具的详细描述:
- get_weather: Get current weather... (500 tokens)
- forecast: Get weather forecast... (500 tokens)
- send_email: Send an email... (500 tokens)
- read_email: Read emails... (500 tokens)
- create_event: Create calendar event... (500 tokens)
- ...

总计: 50+ 工具 × 500 tokens = 25000 tokens ❌
```


**SkillsInterceptor 的方式：**

```
Round 1:
系统提示只包含 skills 列表:
[AVAILABLE SKILLS]
- weather-skill: Query weather information (50 tokens)
- email-skill: Send and read emails (50 tokens)
- calendar-skill: Manage calendar events (50 tokens)

总计: 150 tokens ✅

Round 2 (LLM 调用 read_skill 后):
动态注入 weather-skill 的工具:
- get_weather: Get current weather... (500 tokens)
- forecast: Get weather forecast... (500 tokens)

总计: 150 + 1000 = 1150 tokens ✅

节省: 25000 - 1150 = 23850 tokens (95% 节省!) 🎉
```


---

## 🔄 完整执行流程示例

### **示例：天气查询**

```
用户: 帮我查北京明天的天气

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Round 1: LLM 探索阶段
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. BEFORE_AGENT: SkillsAgentHook
   - 加载 skills: [weather-skill, email-skill, calendar-skill]

2. BEFORE_MODEL: (无)

3. AGENT_MODEL_NODE:
   
   SkillsInterceptor.interceptModel():
   ├─ 获取 skills: [weather-skill, email-skill, calendar-skill]
   ├─ 提取 read_skill 调用: [] (第一次调用，没有历史)
   ├─ 动态工具: []
   ├─ 构建 skills prompt:
   │  "[AVAILABLE SKILLS]
   │   - weather-skill: Query weather information
   │   - email-skill: Send and read emails
   │   - calendar-skill: Manage calendar events"
   ├─ 增强系统消息
   └─ 调用 LLM
   
   LLM 返回:
   "我需要先了解 weather-skill 的详细信息"
   工具调用: read_skill("weather-skill")

4. AGENT_TOOL_NODE:
   执行 read_skill 工具
   返回: weather-skill/SKILL.md 的完整内容

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Round 2: LLM 使用工具阶段
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. BEFORE_MODEL: (无)

2. AGENT_MODEL_NODE:
   
   SkillsInterceptor.interceptModel():
   ├─ 获取 skills: [weather-skill, email-skill, calendar-skill]
   ├─ 提取 read_skill 调用: ["weather-skill"] ⭐
   ├─ 从 groupedTools 获取 weather-skill 的工具:
   │  [getWeatherTool, forecastTool]
   ├─ 动态工具: [getWeatherTool, forecastTool] ⭐
   ├─ 构建 skills prompt (同上)
   ├─ 增强系统消息
   └─ 调用 LLM (这次 LLM 可以看到 getWeatherTool)
   
   LLM 返回:
   "我现在可以使用 getWeatherTool 了"
   工具调用: get_weather("Beijing", "tomorrow")

3. AGENT_TOOL_NODE:
   执行 get_weather 工具
   返回: "北京明天晴，25°C"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Round 3: LLM 总结阶段
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. AGENT_MODEL_NODE:
   
   SkillsInterceptor.interceptModel():
   ├─ 获取 skills
   ├─ 提取 read_skill 调用: ["weather-skill"]
   ├─ 动态工具: [getWeatherTool, forecastTool]
   ├─ 构建 skills prompt
   └─ 调用 LLM
   
   LLM 返回:
   "北京明天天气晴朗，气温 25°C。适合出行！"

2. 返回最终结果给用户
```


---

## 🎨 Builder 模式详解

### **必需参数**

```java
SkillsInterceptor interceptor = SkillsInterceptor.builder()
    .skillRegistry(registry)  // ⭐ 必须提供
    .build();
```


### **可选参数**

```java
SkillsInterceptor interceptor = SkillsInterceptor.builder()
    .skillRegistry(registry)
    .groupedTools(groupedTools)  // ⭐ 可选，用于动态工具注入
    .build();
```


### **参数说明**

| 参数 | 类型 | 必需 | 说明 |
|------|------|------|------|
| `skillRegistry` | `SkillRegistry` | ✅ 是 | 存储 skills 元数据的注册表 |
| `groupedTools` | `Map<String, List<ToolCallback>>` | ❌ 否 | skill 名称到工具列表的映射 |

---

## 📝 关键设计思想

### **1. 渐进式披露（Progressive Disclosure）**

```
传统方式:
┌──────────────────────────────────────┐
│ System Prompt (25000 tokens)         │
│ ├─ Skill 1 完整内容 (5000 tokens)    │
│ ├─ Skill 2 完整内容 (5000 tokens)    │
│ ├─ Skill 3 完整内容 (5000 tokens)    │
│ ├─ 所有工具详细描述 (10000 tokens)   │
│ └─ ...                               │
└──────────────────────────────────────┘
  ↓ Token 消耗大，LLM 容易迷失

SkillsInterceptor 方式:
┌──────────────────────────────────────┐
│ Round 1: System Prompt (150 tokens)  │
│ └─ Skills 列表 (名称+简短描述)        │
│      ↓ LLM 调用 read_skill           │
│                                      │
│ Round 2: System Prompt (1150 tokens) │
│ ├─ Skills 列表 (150 tokens)          │
│ └─ 动态注入相关工具 (1000 tokens)     │
│      ↓ LLM 调用具体工具               │
│                                      │
│ Round 3: 生成最终回复                 │
└──────────────────────────────────────┘
  ✅ 按需加载，节省 95% Token
```


---

### **2. 动态工具注入**

```java
// 初始状态
ModelRequest.dynamicToolCallbacks = []

// LLM 调用 read_skill("weather-skill")
  ↓
// SkillsInterceptor 检测到
  ↓
// 从 groupedTools 提取
groupedTools.get("weather-skill")
  → [getWeatherTool, forecastTool]

// 添加到请求
ModelRequest.dynamicToolCallbacks = [getWeatherTool, forecastTool]
  ↓
// LLM 下一轮可以看到并使用这些工具
```


---

### **3. 解耦设计**

```
SkillsAgentHook (Agent 级别)
  ├─ 职责: 加载/重载 skills
  ├─ 提供: read_skill 工具
  └─ 创建: SkillsInterceptor
  
SkillsInterceptor (Model 级别)
  ├─ 职责: 注入 skills prompt
  ├─ 职责: 动态添加工具
  └─ 依赖: SkillRegistry (共享)

SkillRegistry (数据存储)
  ├─ 职责: 存储 skills 元数据
  ├─ 实现: FileSystemSkillRegistry
  └─ 实现: DatabaseSkillRegistry (可扩展)
```


**优势：**
- ✅ **单一职责**：每个组件只做一件事
- ✅ **可替换**：可以轻松替换 SkillRegistry 实现
- ✅ **可测试**：每个组件可以独立测试

---

## 🎯 总结

### **核心价值**

| 特性 | 说明 |
|------|------|
| **Token 节省** | 按需加载，节省 90%+ Token |
| **动态工具** | 根据 LLM 需求动态注入工具 |
| **渐进式披露** | 先给概览，再给详情 |
| **解耦设计** | Hook、Interceptor、Registry 分离 |
| **灵活性** | 支持多种 SkillRegistry 实现 |

### **适用场景**

1. ✅ **大量 skills**：超过 10 个 skills 时效果显著
2. ✅ **复杂工具集**：每个 skill 有多个工具
3. ✅ **成本敏感**：需要减少 LLM Token 消耗
4. ✅ **性能要求**：减少 prompt 长度，提高响应速度
5. ✅ **模块化设计**：skills 可以独立开发和测试

### **最佳实践**

1. 🎯 **使用自动注册**：优先使用 `SkillsAgentHook`
2. 📦 **合理分组工具**：将相关工具归到同一个 skill
3. 📝 **编写清晰的 SKILL.md**：帮助 LLM 理解何时使用该 skill
4. 🔍 **监控 Token 使用**：对比使用前后的 Token 消耗
5. 🔄 **启用 autoReload**：开发时方便调试 skills

### **关键代码位置**

| 功能 | 代码位置 |
|------|---------|
| 核心逻辑 | `interceptModel()` (117-154 行) |
| 提取 skill 名称 | `extractReadSkillNames()` (160-180 行) |
| 解析参数 | `parseSkillNameFromArguments()` (182-199 行) |
| 增强系统消息 | `enhanceSystemMessage()` (210-215 行) |
| 获取分组工具 | `getGroupedTools()` (201-207 行) |

`SkillsInterceptor` 是实现高效、可扩展 Skills 系统的核心组件，通过渐进式披露和动态工具注入，大幅提升了 Agent 的性能和可用性！😊
