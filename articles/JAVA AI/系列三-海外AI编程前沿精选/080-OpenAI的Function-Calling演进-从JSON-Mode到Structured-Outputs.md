# OpenAI 的 Function Calling 演进：从 JSON Mode 到 Structured Outputs，Java开发者如何利用这些能力

## 引言：三代进化，一个趋势

如果你从2023年初就开始关注OpenAI的API，你会见证一个有趣的演进过程：

- **2023年6月**：Function Calling 首次发布（GPT-4-0613 / GPT-3.5-turbo-0613），开发者终于可以让模型输出结构化的函数调用参数
- **2023年11月**：JSON Mode 发布，允许强制模型输出有效JSON（但需要自己在prompt中描述schema）
- **2024年8月**：Structured Outputs 正式发布，模型可以严格遵循开发者提供的JSON Schema，正确率达到100%（在评测中）

从"尽力而为"到"严格遵循"，从"期待输出JSON"到"保证输出合法JSON"，这三次迭代解决的是同一个核心问题：**如何让大语言模型的输出变得可预测、可解析、可编程**。

对Java开发者来说，这意味着不再需要写一长串try-catch来处理LLM输出的各种"可能的格式"，也不再需要写复杂的正则表达式来解析"差不多"但"不完全是"JSON的响应。本文将全面梳理这三次演进，并给出Java实战示例。

## 第一部分：Function Calling三代演进对比

### 1.1 第一代：原始Function Calling（2023年6月）

第一代Function Calling的核心机制是：在API请求中传入函数定义（名称、描述、参数schema），模型在需要时会返回一个`function_call`对象。

```java
// 第一代 Function Calling - 使用Spring AI实现
// 注意：这是2023年中期的用法

// 定义函数
FunctionDefinition weatherFunction = FunctionDefinition.builder()
    .name("getWeather")
    .description("获取指定城市的天气信息")
    .parameters("""
        {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "城市名称，如'北京'、'上海'"
                },
                "unit": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "温度单位"
                }
            },
            "required": ["city"]
        }
        """)
    .build();

// 调用
ChatResponse response = chatClient.call(
    new Prompt(
        "北京今天天气怎么样？",
        OpenAiChatOptions.builder()
            .withFunction("getWeather")
            .withFunctions(List.of(weatherFunction))
            .build()
    )
);

// 第一代的问题：
// 1. 模型不一定会调用函数——可能直接回答"抱歉，我是语言模型..."
// 2. 参数可能不合法——city有可能是"我不知道"
// 3. 需要自己验证和重试
```

**第一代的核心问题**：
- 模型可能在不需要调用函数时强行调用
- 模型可能在需要调用时却直接回答
- 参数合法性无法保证
- 需要大量的防御性代码来处理各种边界情况

### 1.2 第二代：JSON Mode（2023年11月）

JSON Mode允许开发者通过在系统消息中要求模型输出JSON，并在API调用时设置`response_format: { type: "json_object" }`来强制输出有效JSON。

```java
// 第二代 JSON Mode - 更可控的JSON输出
// 在你的System Prompt中描述JSON Schema

String systemPrompt = """
    你是一个天气查询助手。回复必须始终使用以下JSON格式：
    {
        "weather": {
            "city": "城市名称",
            "temperature": 温度数值,
            "condition": "天气状况",
            "humidity": 湿度百分比
        }
    }
    如果不知道某个信息，使用null。
    """;

ChatResponse response = chatClient.call(
    new Prompt(
        new SystemMessage(systemPrompt),
        new UserMessage("北京今天天气怎么样？")
    ),
    OpenAiChatOptions.builder()
        .withResponseFormat("json_object")  // 关键：强制JSON输出
        .build()
);

String content = response.getResult().getOutput().getContent();
// content 保证是合法的JSON，但：
// 1. Schema不一定匹配——可能多了字段或少了字段
// 2. 类型不一定正确——temperature可能是字符串"25度"
// 3. 需要在prompt中描述schema，容易出错
```

**JSON Mode的改进与局限**：
- 改进：保证输出是合法JSON（解决了"能不能解析"的问题）
- 局限：不保证JSON的结构符合预期（没解决"解析出来对不对"的问题）
- 局限：需要在prompt中自己描述schema，容易遗漏或出错
- 一个实际痛点：如果schema复杂（嵌套对象、枚举、约束条件），prompt描述很容易遗漏细节

### 1.3 第三代：Structured Outputs（2024年8月）

Structured Outputs是游戏规则的改变者。开发者可以直接传入JSON Schema，模型会严格按Schema输出——不仅格式正确，字段和类型也完全匹配。

```java
// 第三代 Structured Outputs - 使用Spring AI
// 模型严格遵循提供的JSON Schema

// Step 1: 定义响应Schema（使用JsonSchema格式）
String responseSchema = """
    {
        "type": "object",
        "properties": {
            "weather": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称"
                    },
                    "temperature": {
                        "type": "number",
                        "description": "当前温度（摄氏度）"
                    },
                    "condition": {
                        "type": "string",
                        "enum": ["sunny", "cloudy", "rainy", "snowy", "foggy"],
                        "description": "天气状况"
                    },
                    "humidity": {
                        "type": "integer",
                        "minimum": 0,
                        "maximum": 100,
                        "description": "湿度百分比"
                    },
                    "forecast": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "date": {"type": "string", "format": "date"},
                                "high": {"type": "number"},
                                "low": {"type": "number"},
                                "condition": {"type": "string"}
                            },
                            "required": ["date", "high", "low", "condition"]
                        },
                        "description": "未来几天天气预报"
                    }
                },
                "required": ["city", "temperature", "condition", "humidity"]
            }
        },
        "required": ["weather"]
    }
    """;

// Step 2: 调用
ChatResponse response = chatClient.call(
    new Prompt("北京今天天气怎么样？"),
    OpenAiChatOptions.builder()
        .withResponseFormat(
            new ResponseFormat(ResponseFormatType.JSON_SCHEMA, responseSchema)
        )
        .build()
);

// Step 3: 直接反序列化——不需要额外的验证！
String jsonContent = response.getResult().getOutput().getContent();
WeatherResponse weatherResponse = objectMapper.readValue(
    jsonContent, WeatherResponse.class
);
// weatherResponse.getWeather().getCondition() 保证是 "sunny"/"cloudy"/...
// weatherResponse.getWeather().getHumidity() 保证在 0-100 之间
```

## 第二部分：Structured Outputs的底层原理——Constrained Decoding

### 2.1 问题本质

要理解Structured Outputs为什么能保证100%的Schema合规，需要先理解一个关键概念：LLM生成文本是"逐token"的。

```
LLM生成 "天气是" → 下一个token可能是 "晴" / "雨" / "阴"
LLM生成 "温度是" → 下一个token可能是 "2" / "3" / "-"
```

传统方法中，LLM的每个token是从整个词汇表（通常5万-10万个token）中按概率采样。所以当LLM输出JSON时，它可能在生成`"temperature":`之后输出`"twenty-five"`而不是`25`——因为从语言模型的角度，"twenty-five"也是个合理的续写。

**Constrained Decoding的核心思想是：在生成每个token时，动态限制可选的token集合，只允许那些符合目标Schema的token。**

### 2.2 有限状态机（FSM）方案

一个常见的实现是将JSON Schema编译为一个**有限状态机**：

```java
// 概念性示例：JSON Schema → FSM 的转换逻辑
// 实际实现在模型的推理引擎中，而非Java层

public class ConceptualFSMExample {
    
    // Schema: {"type": "object", "properties": {"name": {"type": "string"}}}
    // 对应的状态转移（简化）：
    //
    // State 0: 等待 "{" 
    // State 1: 等待 "name" 或 "}"
    // State 2: 等待 ":"
    // State 3: 等待字符串值
    // State 4: 等待 "," 或 "}"
    //
    // 在每个状态，只有符合Schema的token被允许
    
    public Set<String> getAllowedTokens(FsmState state) {
        return switch (state) {
            case START -> Set.of("{");
            case AFTER_OPEN_BRACE -> Set.of("\"name\"", "}");
            case AFTER_KEY -> Set.of(":");
            case AFTER_COLON -> allValidStringTokens();
            case AFTER_VALUE -> Set.of(",", "}");
            // ...
        };
    }
}
```

在推理时，模型的softmax输出（词汇表上的概率分布）会被FSM掩码过滤——不符合Schema的token的概率被设为0，然后在剩余的合法token中重新归一化概率分布，再进行采样。

### 2.3 为什么是"生成时约束"而不是"生成后验证"

有人可能会问：为什么不直接让模型生成文本，然后再验证和修正？

答案是：**生成后修正的效率远不如生成时约束**。

- 生成后验证：模型生成 → 验证失败 → 重新生成 → 再验证 → 可能再失败（反复重试消耗大量token和时间）
- 生成时约束：模型只在合法token中采样 → 生成的每一步都是合法的 → 不需要验证和重试

对于复杂Schema（比如包含`enum`、`minimum`/`maximum`、`pattern`等约束的），后验证方案可能需要在10个约束点上反复重试，token消耗是指数级的。

### 2.4 实际限制和注意事项

Structured Outputs不是万能的：

```
支持：
- JSON Schema的type/object/array/string/number/boolean/null
- required字段
- enum枚举值
- 嵌套对象
- 数组项定义
- 简单约束（minLength, maxLength, minimum, maximum）

不支持（或者说效果不好）：
- 复杂的依赖关系（"如果A字段是X，则B字段必须是Y"）
- 过于复杂的约束组合
- "oneOf"/"anyOf"等复杂的JSON Schema组合关键字
- 动态数量的属性

最佳实践：
- Schema不要太深层（不超过5层嵌套）
- 不要把业务逻辑全部写成Schema约束（太难维护）
- 配合应用层的验证使用（Schema保证结构，代码保证业务逻辑）
```

## 第三部分：Java中如何优雅地实现Function Calling

### 3.1 Spring AI的@Tool注解方案

Spring AI提供了一种声明式的方式来定义Function——通过`@Tool`注解：

```java
// Spring AI的@Tool注解方案
// 方式一：使用@Tool注解（推荐）
@Component
public class WeatherTools {
    
    private final WeatherService weatherService;
    
    public WeatherTools(WeatherService weatherService) {
        this.weatherService = weatherService;
    }
    
    @Tool(name = "getCurrentWeather", 
          description = "获取指定城市的当前天气信息")
    public WeatherInfo getCurrentWeather(
            @ToolParam(description = "城市名称，如'北京'、'上海'")
            String city,
            @ToolParam(description = "温度单位，celsius或fahrenheit")
            String unit) {
        return weatherService.getCurrentWeather(city, unit);
    }
    
    @Tool(name = "getWeatherForecast",
          description = "获取指定城市未来7天的天气预报")
    public List<ForecastDay> getWeatherForecast(
            @ToolParam(description = "城市名称")
            String city,
            @ToolParam(description = "未来天数，1-7")
            @ToolParamConstraint(min = 1, max = 7)
            int days) {
        return weatherService.getForecast(city, days);
    }
}

@Configuration
public class ChatConfiguration {
    
    @Bean
    public ChatClient chatClient(ChatModel chatModel, WeatherTools weatherTools) {
        return ChatClient.builder(chatModel)
            .defaultTools(weatherTools)  // 注册工具
            .build();
    }
}
```

### 3.2 LangChain4j的@Tool注解方案

LangChain4j提供了类似但略有不同的实现：

```java
// LangChain4j的@Tool注解方案
// 方式二：使用LangChain4j的@Tool

public class BookingTools {
    
    @Tool("搜索从出发地到目的地的可用航班")
    public List<Flight> searchFlights(
            @P("出发城市，如'北京'、'上海'") String from,
            @P("目的城市") String to,
            @P("出发日期，格式YYYY-MM-DD") LocalDate date,
            @P(value = "乘客人数", optional = true) Integer passengers) {
        
        int pax = passengers != null ? passengers : 1;
        return flightService.search(from, to, date, pax);
    }
    
    @Tool("预订指定航班")
    public BookingConfirmation bookFlight(
            @P("航班号") String flightNumber,
            @P("乘客姓名列表") List<String> passengerNames,
            @P("座位偏好，WINDOW/AISLE/NO_PREFERENCE") 
            @PEnum({"WINDOW", "AISLE", "NO_PREFERENCE"})
            String seatPreference) {
        
        return bookingService.book(flightNumber, passengerNames, 
            SeatPreference.valueOf(seatPreference));
    }
}

// 使用
ChatLanguageModel model = OpenAiChatModel.builder()
    .apiKey(System.getenv("OPENAI_API_KEY"))
    .modelName("gpt-4")
    .build();

AiServices<Assistant> aiServices = AiServices.builder(Assistant.class)
    .chatLanguageModel(model)
    .tools(new BookingTools())
    .build();

Assistant assistant = aiServices.build();
String result = assistant.chat("帮我订一张明天从北京到上海的机票，靠窗座位");
```

### 3.3 Spring AI vs LangChain4j对比

| 维度 | Spring AI | LangChain4j |
|------|-----------|-------------|
| Spring集成 | 原生深度集成 | 通过Spring Boot Starter |
| 学习曲线 | 低（Spring开发者友好） | 中（需要理解LangChain概念） |
| API稳定性 | 较新，API可能变化 | 更成熟，API相对稳定 |
| 多模型支持 | OpenAI/Azure/Anthropic/Ollama | 更广泛（20+提供商） |
| Function Calling | @Tool注解 + 自动Schema生成 | @Tool注解 + 自动Schema生成 |
| Structured Outputs | 通过ChatOptions配置 | 通过responseFormat配置 |
| 中文文档 | 较少 | 中等 |
| 企业级特性 | Spring生态（监控、配置、安全） | LangSmith集成（可观测性） |

对于大多数Java项目，如果已经使用Spring Boot，Spring AI是更自然的选择。如果需要更灵活的模型切换或更丰富的AI工作流能力，LangChain4j更合适。

## 第四部分：实战——复杂购票操作的Function Calling

### 4.1 场景定义

假设我们要实现一个AI购票助手，支持复杂的多步骤购票流程：

```
用户："帮我订两张下周五从北京出发到上海的高铁票，要靠窗的座位，二等座就行"

需要的Function：
1. searchTrains    - 查询车次
2. checkSeats      - 查询余票
3. calculatePrice  - 计算票价
4. createOrder     - 创建订单
5. payOrder        - 支付订单
```

### 4.2 工具定义

```java
// 购票系统的工具定义和实现
@Component
public class TrainTicketTools {
    
    private final TrainService trainService;
    private final OrderService orderService;
    private final PaymentService paymentService;
    
    @Tool(name = "searchTrains",
          description = "根据出发地、目的地和日期查询可用车次")
    public List<TrainInfo> searchTrains(
            @ToolParam(description = "出发城市，如'北京'、'上海'")
            String fromCity,
            @ToolParam(description = "目的城市")
            String toCity,
            @ToolParam(description = "出发日期，格式YYYY-MM-DD")
            String date,
            @ToolParam(description = "座位类型：SECOND_CLASS(二等座)/FIRST_CLASS(一等座)/BUSINESS(商务座)")
            String seatType) {
        
        // 解析日期
        LocalDate travelDate = LocalDate.parse(date);
        
        // 处理"下周X"这种自然语言表述（由LLM转为具体日期）
        // 实际调用列车查询服务
        return trainService.search(fromCity, toCity, travelDate, 
            SeatType.valueOf(seatType));
    }
    
    @Tool(name = "checkSeats",
          description = "检查指定车次的余票情况")
    public SeatAvailability checkSeats(
            @ToolParam(description = "车次号，如'G123'")
            String trainNumber,
            @ToolParam(description = "出发日期")
            String date,
            @ToolParam(description = "需要查询的座位类型")
            String seatType) {
        
        return trainService.checkAvailability(trainNumber, 
            LocalDate.parse(date), SeatType.valueOf(seatType));
    }
    
    @Tool(name = "calculatePrice",
          description = "计算指定车次和座位类型的票价")
    public PriceInfo calculatePrice(
            @ToolParam(description = "车次号")
            String trainNumber,
            @ToolParam(description = "座位类型")
            String seatType,
            @ToolParam(description = "购票张数")
            Integer ticketCount) {
        
        return trainService.calculatePrice(trainNumber, 
            SeatType.valueOf(seatType), ticketCount);
    }
    
    @Tool(name = "createOrder",
          description = "创建购票订单（不会自动支付）",
          requiresConfirmation = true  // 需要用户确认
    )
    public OrderResult createOrder(
            @ToolParam(description = "车次号")
            String trainNumber,
            @ToolParam(description = "出发日期")
            String date,
            @ToolParam(description = "座位类型")
            String seatType,
            @ToolParam(description = "购票张数")
            Integer ticketCount,
            @ToolParam(description = "乘客姓名列表")
            List<String> passengerNames,
            @ToolParam(description = "座位偏好：WINDOW(靠窗)/AISLE(过道)/NO_PREFERENCE(无偏好)")
            String seatPreference) {
        
        // 创建订单
        OrderRequest request = OrderRequest.builder()
            .trainNumber(trainNumber)
            .date(LocalDate.parse(date))
            .seatType(SeatType.valueOf(seatType))
            .ticketCount(ticketCount)
            .passengerNames(passengerNames)
            .seatPreference(SeatPreference.valueOf(seatPreference))
            .build();
        
        return orderService.createOrder(request);
    }
    
    @Tool(name = "payOrder",
          description = "支付指定订单",
          requiresConfirmation = true  // 需要用户确认
    )
    public PaymentResult payOrder(
            @ToolParam(description = "订单号")
            String orderId,
            @ToolParam(description = "支付方式：ALIPAY/WECHAT/CREDIT_CARD")
            String paymentMethod) {
        
        return paymentService.pay(orderId, PaymentMethod.valueOf(paymentMethod));
    }
}
```

### 4.3 多步骤编排与确认机制

```java
// 多步骤购票流程编排
@Service
public class TicketBookingAgent {
    
    private final ChatClient chatClient;
    private final TrainTicketTools tools;
    
    // 需要用户确认的操作
    private static final Set<String> CONFIRMATION_REQUIRED = Set.of(
        "createOrder", "payOrder"
    );
    
    public BookingResponse handleBooking(String userMessage, String sessionId) {
        // 构建System Prompt
        String systemPrompt = """
            你是一个高铁购票助手。在处理购票请求时，遵循以下流程：
            
            1. 首先搜索可用车次（searchTrains）
            2. 展示结果给用户，让用户选择车次
            3. 查询余票（checkSeats）
            4. 计算价格（calculatePrice）
            5. 展示订单信息给用户确认
            6. 用户确认后创建订单（createOrder）
            7. 引导用户支付（payOrder）
            
            关键规则：
            - 涉及支付和创建订单的操作，必须先向用户确认
            - 当用户信息不完整时，主动询问缺失的信息
            - 价格信息必须来自calculatePrice函数，不要自己估算
            - 如果用户要求靠窗座位但有其他偏好，使用seatPreference参数
            """;
        
        // 调用ChatClient，自动处理工具调用
        ChatResponse response = chatClient.call(
            new Prompt(
                List.of(
                    new SystemMessage(systemPrompt),
                    new UserMessage(userMessage)
                ),
                OpenAiChatOptions.builder()
                    .withFunctions(tools)  // 注册所有工具
                    .build()
            )
        );
        
        return processResponse(response, sessionId);
    }
    
    private BookingResponse processResponse(ChatResponse response, String sessionId) {
        Message output = response.getResult().getOutput();
        
        if (output instanceof AssistantMessage assistantMsg) {
            // 检查是否有工具调用
            if (assistantMsg.hasToolCalls()) {
                
                // 处理工具调用
                for (ToolCall toolCall : assistantMsg.getToolCalls()) {
                    String functionName = toolCall.getName();
                    
                    // 需要确认的操作
                    if (CONFIRMATION_REQUIRED.contains(functionName)) {
                        return BookingResponse.requiresConfirmation(
                            buildConfirmationMessage(functionName, toolCall),
                            toolCall
                        );
                    }
                    
                    // 自动执行的操作（查询类）
                    Object result = executeToolCall(toolCall);
                    
                    // 将结果反馈给LLM，继续对话
                    chatClient.call(new Prompt(
                        new FunctionResultMessage(functionName, 
                            objectMapper.writeValueAsString(result))
                    ));
                }
            }
            
            // 返回LLM的文本回复
            return BookingResponse.success(assistantMsg.getContent());
        }
        
        return BookingResponse.error("无法处理响应");
    }
    
    private Object executeToolCall(ToolCall toolCall) {
        // 反射调用对应的工具方法
        // 实际项目中建议使用Spring AI的ToolCallingManager
        return switch (toolCall.getName()) {
            case "searchTrains" -> tools.searchTrains(
                toolCall.getArgument("fromCity"),
                toolCall.getArgument("toCity"),
                toolCall.getArgument("date"),
                toolCall.getArgument("seatType")
            );
            case "checkSeats" -> tools.checkSeats(
                toolCall.getArgument("trainNumber"),
                toolCall.getArgument("date"),
                toolCall.getArgument("seatType")
            );
            case "calculatePrice" -> tools.calculatePrice(
                toolCall.getArgument("trainNumber"),
                toolCall.getArgument("seatType"),
                toolCall.getArgument("ticketCount")
            );
            // ... 其他工具调用
            default -> throw new IllegalArgumentException(
                "Unknown function: " + toolCall.getName());
        };
    }
    
    private String buildConfirmationMessage(String functionName, ToolCall toolCall) {
        if ("createOrder".equals(functionName)) {
            return String.format(
                "请确认以下订单信息：\n" +
                "车次：%s\n" +
                "日期：%s\n" +
                "座位：%s\n" +
                "张数：%s\n" +
                "乘客：%s\n" +
                "偏好：%s\n\n" +
                "是否确认创建订单？",
                toolCall.getArgument("trainNumber"),
                toolCall.getArgument("date"),
                toolCall.getArgument("seatType"),
                toolCall.getArgument("ticketCount"),
                toolCall.getArgument("passengerNames"),
                toolCall.getArgument("seatPreference")
            );
        }
        
        if ("payOrder".equals(functionName)) {
            return String.format(
                "即将支付订单 %s，支付方式：%s\n\n是否确认支付？",
                toolCall.getArgument("orderId"),
                toolCall.getArgument("paymentMethod")
            );
        }
        
        return "是否确认执行此操作？";
    }
}
```

### 4.4 完整对话流程演示

```
用户："帮我订两张下周五从北京到上海的高铁票，要靠窗的座位，二等座就行"

AI：[调用 searchTrains("北京", "上海", "2025-05-16", "SECOND_CLASS")]
    → 返回5个车次

AI："为您找到以下车次（5月16日 周五）：
    - G1次  09:00-13:28  二等座 ¥553  （余票32张）
    - G3次  10:00-14:28  二等座 ¥553  （余票15张）
    - G7次  12:00-16:28  二等座 ¥553  （余票8张）
    - G13次  15:00-19:28  二等座 ¥553  （余票45张）
    - G17次  19:00-23:28  二等座 ¥553  （余票60张）
    请问您想选择哪个车次？"

用户："G7吧"

AI：[调用 checkSeats("G7", "2025-05-16", "SECOND_CLASS")]
    → 二等座剩余8张

AI：[调用 calculatePrice("G7", "SECOND_CLASS", 2)]
    → 总价 ¥1,106

AI："G7次确认信息：
    车次：G7
    日期：2025年5月16日（周五）
    时间：12:00-16:28（4小时28分）
    座位：二等座 × 2张
    偏好：靠窗
    总价：¥1,106（¥553 × 2）
    
    请问乘客姓名是？"

用户："张三和李四"

AI："订单确认：
    - 车次：G7 12:00-16:28
    - 乘客：张三、李四
    - 座位：二等座，靠窗偏好
    - 金额：¥1,106
    
    是否确认创建订单？"

用户："确认"

AI：[调用 createOrder("G7", "2025-05-16", "SECOND_CLASS", 2,
     ["张三", "李四"], "WINDOW")]
    → 订单号：ORD-20250516-78901

AI："订单已创建！订单号 ORD-20250516-78901。
    请选择支付方式：支付宝/微信/信用卡？"

用户："支付宝"

AI："即将使用支付宝支付 ¥1,106，确认支付？"

用户："确认"

AI：[调用 payOrder("ORD-20250516-78901", "ALIPAY")]
    → 支付成功

AI："支付成功！订单详情：
    订单号：ORD-20250516-78901
    车次：G7 2025-05-16 12:00-16:28
    乘客：张三、李四
    座位：二等座 靠窗
    金额：¥1,106（已支付）
    
    乘车当天请提前1小时到达北京南站，祝您旅途愉快！"
```

## 第五部分：最佳实践总结

### 5.1 Function Calling的使用原则

```java
/**
 * Function Calling 设计原则
 */
public class FunctionCallingBestPractices {
    
    // 原则1：函数描述要精确，"像写给一个新同事的任务说明书"
    // ❌ 不好
    @Tool(name = "search", description = "搜索")
    
    // ✅ 好
    @Tool(name = "searchAvailableTrains",
          description = "根据出发城市、目的城市和出发日期搜索可预订的高铁车次，" +
                       "返回车次号、出发时间、到达时间、历时、座位类型、票价和余票数量")
    
    // 原则2：参数描述要包含格式和示例
    // ❌ 不好
    @ToolParam(description = "日期")
    
    // ✅ 好
    @ToolParam(description = "出发日期，格式必须为YYYY-MM-DD，如'2025-05-16'")
    
    // 原则3：枚举值用大写在参数描述中说明
    // ❌ 不好
    @ToolParam(description = "座位类型")
    
    // ✅ 好
    @ToolParam(description = "座位类型：SECOND_CLASS(二等座)/FIRST_CLASS(一等座)/BUSINESS(商务座)")
    
    // 原则4：区分"只读"和"写操作"，写操作需要确认
    // 查询类函数（search/check/calculate）→ LLM直接调用
    // 写操作类函数（create/pay/delete）→ 需要用户确认后再调用
    
    // 原则5：函数粒度适中
    // ❌ 太粗：一个函数做所有事
    @Tool(name = "bookTrainTicket",
          description = "搜索车次、查余票、算价格、创建订单、支付")
    
    // ✅ 合适：每个函数做一件事
    // searchTrains → checkSeats → calculatePrice → createOrder → payOrder
    
    // 原则6：函数数量控制在合理范围
    // 5-15个函数是比较合适的范围
    // 少于5个 → 功能不够用
    // 多于15个 → LLM可能混淆或选错
}
```

### 5.2 Structured Outputs + Function Calling的结合

```java
// 最佳实践：用Structured Outputs约束工具调用的结果格式
// 这样即使LLM多次调用工具，每次返回的结构都是严格一致的

@Tool(name = "searchTrains", description = "搜索车次")
public TrainSearchResult searchTrains(
        @ToolParam(description = "出发城市") String fromCity,
        @ToolParam(description = "目的城市") String toCity,
        @ToolParam(description = "出发日期，YYYY-MM-DD") String date,
        @ToolParam(description = "座位类型") String seatType) {
    
    // 实际查询逻辑
    List<TrainInfo> trains = trainService.search(fromCity, toCity, 
        LocalDate.parse(date), SeatType.valueOf(seatType));
    
    // 返回强类型的结果（利用Structured Outputs保证格式）
    return new TrainSearchResult(
        fromCity, toCity, LocalDate.parse(date),
        SeatType.valueOf(seatType), trains,
        trains.size()
    );
}

// 结果类——利用record确保不可变性
public record TrainSearchResult(
    @JsonProperty("from_city") String fromCity,
    @JsonProperty("to_city") String toCity,
    @JsonProperty("date") LocalDate date,
    @JsonProperty("seat_type") SeatType seatType,
    @JsonProperty("trains") List<TrainInfo> trains,
    @JsonProperty("total_count") int totalCount
) {
    // 可以添加业务方法
    public boolean hasAvailableTrains() {
        return totalCount > 0;
    }
    
    public List<TrainInfo> getMorningTrains() {
        return trains.stream()
            .filter(t -> t.departureTime().getHour() < 12)
            .collect(Collectors.toList());
    }
}
```

## 结语：从"与AI对话"到"用AI编程"

Function Calling的三代演进反映了一个更大的趋势：**AI正在从"聊天工具"变成"编程接口"**。2023年你还得在prompt中小心翼翼地请求"请返回JSON"，2025年你直接定义一个Schema然后说"按这个来"。

对Java开发者来说，这个演进意味着三件事：

1. **类型安全重新变得重要**。当LLM的输出可以被严格约束为某种类型时，Java的强类型系统不再是障碍，而是优势——利用record、sealed class、泛型等特性可以极大地提升与LLM交互的健壮性
2. **Function Calling是AI应用的核心**。不是"让AI说话"，而是"让AI调用你的系统"。你的业务系统有多强，AI的能力就有多强
3. **抽象层次在提升**。你不需要关心LLM内部的token生成机制，Spring AI和LangChain4j的`@Tool`注解让你可以用熟悉的Java方式定义和使用Function Calling

---

**系列四预告**：Spring AI入门实战——从零搭建一个AI驱动的Java应用。我们将从Hello World开始，逐步构建一个完整的AI应用，涵盖Chat API、Function Calling、RAG向量检索、多模态处理等核心能力。敬请期待！

---

> 本文中OpenAI API版本信息基于截至2025年初的公开文档。Spring AI版本为1.0.x系列。具体API可能随版本更新而变化，请以官方文档为准。
