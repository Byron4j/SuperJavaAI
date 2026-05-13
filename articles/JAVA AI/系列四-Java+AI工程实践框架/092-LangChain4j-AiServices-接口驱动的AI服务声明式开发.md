---
title: LangChain4j AiServices：接口驱动的 AI 服务声明式开发，定义接口 = 完成 AI 功能
tags: [LangChain4j, AiServices, Java, LLM, 声明式开发, 动态代理, AI]
---

# LangChain4j AiServices：接口驱动的 AI 服务声明式开发

> "定义一个接口，AI 功能就完成了？" —— 这不是魔法，是 LangChain4j AiServices 的日常操作。本文带你深度拆解这个最优雅的 Java AI 编程模型。

---

## 一、开篇：你有没有写过这样的"垃圾代码"？

先来看看一个典型的 Java AI 调用代码：

```java
@Service
public class OrderAnalysisService {

    @Autowired
    private ChatLanguageModel model;

    public String analyzeOrder(String orderInfo) {
        String systemPrompt = """
            你是一个电商订单分析专家。
            请分析订单数据，识别异常订单、高价值客户和潜在风险。
            以 JSON 格式返回分析结果。
            """;

        String userPrompt = "请分析以下订单数据：\n" + orderInfo;

        Response<AiMessage> response = model.generate(
            SystemMessage.from(systemPrompt),
            UserMessage.from(userPrompt)
        );

        String text = response.content().text();
        // 手动解析 JSON...
        // 处理格式异常...
        // 转换类型...

        return text;
    }
}
```

这段代码的问题在哪里？

1. **Prompt 模板散落各处**：系统提示词硬编码在方法体里，不方便管理和复用
2. **输出格式靠约定**：你告诉 AI "返回 JSON"，但 AI 有时候返回纯文本、有时候 JSON 不完整
3. **类型转换全靠手写**：得自己解析 JSON、处理格式异常
4. **日志/监控全靠手动埋点**：每次调用的 Token 消耗、耗时都得自己记录
5. **模板代码多到窒息**：一个 AI 功能 20 行调用代码，10 个功能就是 200 行

**有没有一种方式，像写 MyBatis Mapper 一样写 AI 服务？**

有的！这就是 LangChain4j 的 **AiServices**——接口驱动的 AI 服务声明式开发。

---

## 二、AiServices 核心理念：你只管定义接口

### 2.1 一句话说清楚 AiServices

> **AiServices = MyBatis 的 @Mapper 之于数据库，就是 AiServices 的 @AiService 之于大模型。**

MyBatis 中你定义一个接口，MyBatis 通过动态代理帮你把接口方法翻译成 SQL 并执行。AiServices 同理——你定义一个接口，LangChain4j 通过动态代理帮你把接口方法翻译成 LLM 调用并返回结果。

### 2.2 最简单的例子

```java
// 1. 定义一个接口
interface Translator {
    @SystemMessage("你是一个专业的翻译助手，将用户输入翻译为英文")
    String translate(@UserMessage String text);
}

// 2. 使用 AiServices 创建代理实例
ChatLanguageModel model = OpenAiChatModel.builder()
        .apiKey(System.getenv("OPENAI_API_KEY"))
        .modelName("gpt-4o")
        .build();

Translator translator = AiServices.create(Translator.class, model);

// 3. 像调用普通方法一样调用 AI
String result = translator.translate("今天天气真好，适合出去走走。");
System.out.println(result);
// 输出: The weather is really nice today, perfect for going out for a walk.
```

三行代码，一个接口定义，AI 功能就完成了。不需要拼接 Prompt、不需要解析返回值、不需要处理异常——框架全帮你搞定了。

---

## 三、核心注解详解

AiServices 的核心就是三个注解和一个思维模型。

### 3.1 @SystemMessage —— 定义系统角色

`@SystemMessage` 用于设定 AI 的**角色、行为规则、输出格式约束**。它对应 LLM 调用中的 System Prompt。

```java
interface CodeReviewer {
    @SystemMessage("""
        你是一名资深Java代码审查专家。
        审查规则：
        1. 关注代码规范和命名
        2. 检查潜在的性能问题
        3. 识别安全漏洞（SQL注入、XSS等）
        4. 给出优化建议和修改后的代码
        
        请以以下格式返回：
        问题等级：[高危/中危/低危]
        问题描述：...
        建议修改：...
        修改后代码：```java ... ```
        """)
    String review(@UserMessage String code);
}
```

`@SystemMessage` 也支持从**资源文件**加载，便于集中管理 Prompt 模板：

```java
interface CodeReviewer {
    @SystemMessage(fromResource = "prompts/code-reviewer.txt")
    String review(@UserMessage String code);
}
```

```text
# src/main/resources/prompts/code-reviewer.txt
你是一名资深Java代码审查专家。
审查规则：
1. 关注代码规范和命名
2. 检查潜在的性能问题
3. 识别安全漏洞（SQL注入、XSS等）
4. 给出优化建议和修改后的代码
```

### 3.2 @UserMessage —— 定义用户消息模板

`@UserMessage` 用于构造发送给 AI 的用户消息。它支持**模板语法**，可以动态拼接参数。

```java
interface Translator {
    @SystemMessage("你是一个专业的翻译助手")
    @UserMessage("请将以下文本翻译为{targetLanguage}：\n\n{inputText}")
    String translate(@V("inputText") String text, @V("targetLanguage") String lang);
}

// 调用
String result = translator.translate("Hello World", "中文");
// 实际发送给AI的UserMessage：
// "请将以下文本翻译为中文：\n\nHello World"
```

模板语法说明：
- `{variableName}` 在 `@UserMessage` 中定义占位符
- `@V("variableName")` 绑定方法参数到占位符
- 如果方法只有一个参数且不关心占位符名字，可以直接用 `@UserMessage` 不指定模板

### 3.3 @V —— 参数绑定

`@V` 注解将方法参数绑定到 Prompt 模板中的占位符：

```java
interface ContentGenerator {
    @SystemMessage("你是一个{role}")
    @UserMessage("主题：{topic}\n风格：{style}\n字数：{wordCount}字以内")
    String generate(
        @V("role") String role,
        @V("topic") String topic,
        @V("style") String style,
        @V("wordCount") int wordCount
    );
}
```

`@V` 的 value 必须与 `@SystemMessage` 或 `@UserMessage` 中的模板变量名**精确匹配**。

### 3.4 三个注解的协作关系

```
┌─────────────────────────────────────────────────┐
│                  LLM Request                     │
├─────────────────────────────────────────────────┤
│  SystemMessage: @SystemMessage 定义              │
│  "你是一个{role}"  ← @V("role") 绑定参数值       │
├─────────────────────────────────────────────────┤
│  UserMessage: @UserMessage 定义                  │
│  "请翻译：{text}"  ← @V("text") 绑定参数值       │
├─────────────────────────────────────────────────┤
│  AI 返回文本                                     │
│  AiServices 自动做类型转换                       │
└─────────────────────────────────────────────────┘
```

---

## 四、返回值类型自动映射

AiServices 最强大的特性之一：**你定义什么返回值类型，它就自动帮你映射**。

### 4.1 String —— 返回原始文本

```java
interface GreetingBot {
    String sayHello(@UserMessage String name);
}

String result = greetingBot.sayHello("张三");
// result: "你好，张三！有什么我可以帮助你的吗？"
```

### 4.2 自定义 POJO —— 自动 JSON 反序列化

```java
@Description("客户意图分析结果")
record IntentAnalysisResult(
    @Description("用户的主要意图：咨询/投诉/购买/售后")
    String intent,

    @Description("用户的情绪：正面/中性/负面")
    String sentiment,

    @Description("紧急程度：高/中/低")
    String urgency,

    @Description("需要转接的部门，如果不需要则为null")
    String transferDepartment,

    @Description("回复建议")
    String replySuggestion
) {}

interface CustomerServiceAI {
    @SystemMessage("你是一个智能客服，分析客户消息的意图和情绪")
    IntentAnalysisResult analyze(@UserMessage String customerMessage);
}

// 调用
IntentAnalysisResult result = customerServiceAI.analyze("我的订单三天了还没发货，能退款吗？");
System.out.println(result.intent());    // "售后"
System.out.println(result.sentiment()); // "负面"
System.out.println(result.urgency());   // "中"
```

> 关键：AiServices 会自动在 System Prompt 中注入 JSON Schema 约束，并自动将 AI 的 JSON 响应反序列化为你的 POJO。你完全不需要手写 JSON 解析代码！

### 4.3 List / Set / Enum —— 自动适配

```java
enum RiskLevel { LOW, MEDIUM, HIGH, CRITICAL }

record AuditItem(
    RiskLevel riskLevel,
    String description,
    String suggestion
) {}

interface CodeAuditor {
    @SystemMessage("你是一个代码安全审计专家，列出所有发现的问题")
    List<AuditItem> audit(@UserMessage String code);
}

// 返回的 List 自动填充
List<AuditItem> issues = codeAuditor.audit(sourceCode);
for (AuditItem item : issues) {
    log.warn("[{}] {}: {}", item.riskLevel(), item.description(), item.suggestion());
}
```

### 4.4 Result 类型 —— 错误处理

LangChain4j 提供了 `Result<T>` 包装类型，当返回值可能因 Token 限制或其他原因不完整时，可以捕获错误：

```java
interface ContentWriter {
    @SystemMessage("根据主题写一篇技术文章")
    Result<String> writeArticle(@UserMessage String topic);
}

Result<String> result = contentWriter.writeArticle("Java虚拟线程原理");
if (result.hasError()) {
    log.error("生成中断: {}", result.error());
}
String content = result.content();
```

---

## 五、Chat Memory：多轮对话记忆

AiServices 天然支持对话记忆（Chat Memory），你只需在创建时指定 `ChatMemoryProvider`：

```java
import dev.langchain4j.memory.ChatMemory;
import dev.langchain4j.memory.chat.MessageWindowChatMemory;

ChatMemory memory = MessageWindowChatMemory.withMaxMessages(20);

Assistant assistant = AiServices.builder(Assistant.class)
        .chatLanguageModel(model)
        .chatMemory(memory)
        .build();

// 多轮对话，自动记忆上下文
assistant.chat("我叫张三");          // AI记住了
assistant.chat("我叫什么名字？");     // AI能回答"张三"
assistant.chat("今天天气怎么样？");   // AI知道还在跟张三对话
```

`MessageWindowChatMemory` 采用滑动窗口策略，只保留最近 N 条消息，避免 Token 超限。

---

## 六、Tools（工具调用）：AI 自动调用你的 Java 方法

这是 AiServices 最"智能"的特性——你可以定义 Java 方法作为 AI 的"工具"，AI 会自动决定何时调用它们。

```java
class WeatherTools {

    @Tool("获取指定城市的实时天气")
    String getWeather(@P("城市名称") String city) {
        // 实际对接天气 API
        return city + "今天晴，温度 18-25°C";
    }

    @Tool("获取指定城市的空气指数")
    String getAirQuality(@P("城市名称") String city) {
        return city + "空气质量指数 45，等级：优";
    }
}

interface WeatherAssistant {
    @SystemMessage("你是一个天气助手，可以使用工具查询天气信息")
    String ask(@UserMessage String question);
}

// 创建时注入工具
WeatherAssistant assistant = AiServices.builder(WeatherAssistant.class)
        .chatLanguageModel(model)
        .tools(new WeatherTools())
        .build();

// AI 会自动识别需要调用 getWeather 工具
String answer = assistant.ask("北京今天天气怎么样？需要带伞吗？");
// AI 内部流程：
// 1. 识别需要调用 getWeather("北京")
// 2. 获取返回的天气数据
// 3. 根据天气数据回答用户问题
```

这就是 **Agent（智能体）** 的雏形——AI 不再只是"说话"，它能"做事"。

---

## 七、动态代理原理：AiServices 底层揭秘

了解底层原理有助于你更好地使用和调试。

### 7.1 核心流程

```
┌──────────────────────────────────────────────────────┐
│                  AiServices.create(Class, Model)      │
├──────────────────────────────────────────────────────┤
│  1. Java 动态代理 (Proxy.newProxyInstance)            │
│     └── 为接口生成代理对象                            │
├──────────────────────────────────────────────────────┤
│  2. 方法拦截 (InvocationHandler.invoke)               │
│     ├── 读取 @SystemMessage，构造 SystemMessage       │
│     ├── 读取 @UserMessage + @V，构造 UserMessage      │
│     ├── 读取返回类型，注入 JSON Schema 约束           │
│     ├── 调用 ChatLanguageModel.generate()             │
│     └── 根据返回类型做反序列化                        │
├──────────────────────────────────────────────────────┤
│  3. 返回结果                                          │
│     └── 类型安全的业务对象                            │
└──────────────────────────────────────────────────────┘
```

### 7.2 简化版源码模拟

```java
public class AiServices {

    public static <T> T create(Class<T> aiServiceClass, ChatLanguageModel model) {
        return (T) Proxy.newProxyInstance(
            aiServiceClass.getClassLoader(),
            new Class[]{aiServiceClass},
            new AiServiceInvocationHandler(model)
        );
    }

    static class AiServiceInvocationHandler implements InvocationHandler {
        private final ChatLanguageModel model;

        AiServiceInvocationHandler(ChatLanguageModel model) {
            this.model = model;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) {
            // 1. 构造 SystemMessage
            String systemPrompt = method.getAnnotation(SystemMessage.class).value();
            SystemMessage systemMsg = SystemMessage.from(systemPrompt);

            // 2. 构造 UserMessage（处理模板 + @V 参数绑定）
            String userTemplate = method.getAnnotation(UserMessage.class).value();
            String userMsg = applyTemplate(userTemplate, method, args);

            // 3. 调用模型
            Response<AiMessage> response = model.generate(systemMsg, UserMessage.from(userMsg));
            String text = response.content().text();

            // 4. 类型转换
            Class<?> returnType = method.getReturnType();
            if (returnType == String.class) {
                return text;
            }
            return deserialize(text, returnType);  // JSON 反序列化
        }
    }
}
```

> 实际源码远比这个复杂（涉及流式输出、Tools、Memory、JSON Schema 注入等），但核心思想就是 **Java 动态代理 + 注解驱动的 Prompt 构造**。

---

## 八、完整实战：智能客服系统

下面是一个生产级别的示例，展示 AiServices 的真实威力：

### 8.1 接口定义

```java
public interface CustomerServiceAI {

    /**
     * 分析客户消息意图
     */
    @SystemMessage("""
        你是智能客服意图分析引擎。
        分析客户消息，判断意图类型和紧急程度。
        """)
    @UserMessage("客户消息：{message}")
    IntentResult analyzeIntent(@V("message") String message);

    /**
     * 生成回复
     */
    @SystemMessage("""
        你是{company}的智能客服{agentName}。
        回复规则：
        - 语气亲切专业
        - 先表达理解，再提供解决方案
        - 如果问题无法解决，建议转人工
        - 回复控制在100字以内
        """)
    @UserMessage("客户说：{message}\n分析结果：{analysis}")
    String generateReply(
        @V("company") String company,
        @V("agentName") String agentName,
        @V("message") String message,
        @V("analysis") String analysis
    );

    /**
     * 判断是否需要转人工
     */
    @SystemMessage("判断以下客户问题是否需要转人工客服处理。只需要回答 true 或 false。")
    boolean needHumanSupport(@UserMessage String message);
}
```

### 8.2 DTO 定义

```java
record IntentResult(
    @Description("意图类型")
    IntentType intent,

    @Description("紧急程度 1-5")
    int urgency,

    @Description("关键词列表")
    List<String> keywords
) {}

enum IntentType {
    @Description("咨询产品信息") PRODUCT_INQUIRY,
    @Description("投诉") COMPLAINT,
    @Description("售后问题") AFTER_SALES,
    @Description("订单查询") ORDER_INQUIRY,
    @Description("其他") OTHER
}
```

### 8.3 服务编排

```java
@Service
@Slf4j
public class CustomerServiceOrchestrator {

    private final CustomerServiceAI ai;

    public CustomerServiceOrchestrator(ChatLanguageModel model) {
        this.ai = AiServices.builder(CustomerServiceAI.class)
                .chatLanguageModel(model)
                .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
                .build();
    }

    public ServiceResponse handleCustomerMessage(String message, String sessionId) {
        // Step 1: 意图分析
        IntentResult intent = ai.analyzeIntent(message);
        log.info("[{}] 意图: {}, 紧急度: {}", sessionId, intent.intent(), intent.urgency());

        // Step 2: 判断是否需要转人工
        if (intent.urgency() >= 4 || ai.needHumanSupport(message)) {
            return ServiceResponse.transferToHuman("您的问题比较复杂，正在为您转接人工客服...", intent);
        }

        // Step 3: 生成回复
        String reply = ai.generateReply(
                "XX科技", "小智", message,
                "意图:" + intent.intent() + ", 紧急度:" + intent.urgency()
        );

        return ServiceResponse.autoReply(reply, intent);
    }
}
```

### 8.4 Controller 层

```java
@RestController
@RequestMapping("/api/customer-service")
@Slf4j
public class CustomerServiceController {

    private final CustomerServiceOrchestrator orchestrator;

    public CustomerServiceController(CustomerServiceOrchestrator orchestrator) {
        this.orchestrator = orchestrator;
    }

    @PostMapping("/message")
    public ResponseEntity<ServiceResponse> handleMessage(
            @RequestBody @Valid CustomerRequest request) {

        log.info("收到客户消息: {}", request.getMessage());
        ServiceResponse response = orchestrator.handleCustomerMessage(
                request.getMessage(), request.getSessionId());

        return ResponseEntity.ok(response);
    }

    @Data
    public static class CustomerRequest {
        @NotBlank private String message;
        @NotBlank private String sessionId;
    }
}
```

---

## 九、AiServices 最佳实践

### 9.1 接口粒度原则

一个 AiService 接口只做**一件事**。不要写成万能接口：

```java
// ❌ 不推荐：万能接口
interface AIAssistant {
    String translate(...);
    String summarize(...);
    String reviewCode(...);
    String generateArticle(...);
}

// ✅ 推荐：按职责拆分
interface TranslatorService { String translate(...); }
interface CodeReviewService { String review(...); }
interface ContentGeneratorService { String generate(...); }
```

### 9.2 Prompt 模板外部化

```java
interface EnglishTeacher {
    @SystemMessage(fromResource = "prompts/english-teacher-system.txt")
    @UserMessage(fromResource = "prompts/english-teacher-user.txt")
    String correct(@V("studentInput") String input);
}
```

### 9.3 日志与监控

```java
var assistant = AiServices.builder(Assistant.class)
        .chatLanguageModel(model)
        .chatMemory(memory)
        .listeners(new AiServiceListener() {
            @Override
            public void onRequest(AiServiceContext context) {
                log.info("AI 请求: {}, Token数: {}", context.method().getName(), context.tokenCount());
            }

            @Override
            public void onResponse(AiServiceContext context, Object result) {
                log.info("AI 响应: {}, 耗时: {}ms", context.method().getName(), context.duration().toMillis());
            }
        })
        .build();
```

### 9.4 流式输出

AiServices 同样支持流式输出，只需返回值类型设为 `TokenStream`：

```java
interface StreamingAssistant {
    @SystemMessage("你是一个友好的助手")
    TokenStream chat(@UserMessage String message);
}

TokenStream stream = streamingAssistant.chat("讲个笑话");
stream.onNext(token -> System.out.print(token))
      .onComplete(response -> System.out.println("\n[完成]"))
      .onError(Throwable::printStackTrace)
      .start();
```

---

## 十、总结

AiServices 是 LangChain4j 最亮眼的设计，它彻底改变了 Java 开发者编写 AI 代码的方式：

1. **定义接口 = 完成 AI 功能**：不再需要手写 Prompt 拼接和 JSON 解析
2. **三个注解走天下**：`@SystemMessage` + `@UserMessage` + `@V`，简单到让人怀疑
3. **自动类型映射**：返回值可以是 String、POJO、List、Enum、Result——全自动处理
4. **内置记忆和工具**：Chat Memory 和 Tool Calling 开箱即用
5. **底层是动态代理**：理解原理才能用好它，本质是 `Proxy + InvocationHandler + 注解驱动`

> 一句话：**如果说 Spring AI 让你用 Java 调用 AI，那 LangChain4j AiServices 让你用 Java 定义 AI。**

---

**下一篇预告**：《LangChain4j RAG 实战：Java 实现检索增强生成，让 AI 回答你的私有数据》—— 大模型只知道通用知识，如何让它看懂你的公司文档、产品手册、内部知识库？RAG 是答案。敬请期待！

---

> 作者：IT 老熊
> 标签：LangChain4j, AiServices, Java, LLM, 声明式开发, 动态代理
> 原文首发：CSDN 技术社区
