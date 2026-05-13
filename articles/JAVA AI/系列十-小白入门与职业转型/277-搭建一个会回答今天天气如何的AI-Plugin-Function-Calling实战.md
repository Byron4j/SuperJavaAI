# 搭建一个会回答"今天天气如何"的 AI Plugin：Function Calling 实战，给AI装上手和脚

## 开篇：AI 被困在对话框里

你问 ChatGPT："今天北京天气怎么样？"

ChatGPT 回答："很抱歉，我无法获取实时天气信息，我的知识截止于..."

这就是大模型最大的局限——**它只能基于训练数据回答，无法与外部世界交互**。

它不知道今天星期几，不知道你的银行余额，不知道你家附近哪家餐厅在营业。它就像一个知识渊博但被锁在地下室里的教授。

**Function Calling（函数调用）就是给这位教授配一部手机**——教授可以打电话查询天气、转账、预订餐厅了。

今天我们来给聊天机器人装上"手和脚"，让它能和真实世界交互。完整可运行的代码，跟着做就能用。

## 一、Function Calling 的原理（30 秒理解）

```
用户："北京今天天气怎么样？"
    ↓
大模型识别意图："这个用户想查天气，我应该调用 get_weather 函数"
    ↓
大模型输出函数调用请求：
{
  "function": "get_weather",
  "arguments": {"city": "北京"}
}
    ↓
你的代码执行这个函数：调用天气API，获取真实数据
    ↓
把结果返回给大模型："北京今天晴天，25°C"
    ↓
大模型组织语言："北京今天天气很不错！晴空万里，温度25°C..."
    ↓
返回给用户
```

**关键点**：大模型不执行函数，它只是"决定"调用哪个函数。**你的代码**负责执行函数并把结果告诉大模型。

## 二、整体项目结构

基于上一篇文章的 Spring Boot 项目，新增以下文件：

```
src/main/java/com/example/chatbot/
├── function/                          # 新增：函数实现
│   ├── FunctionRegistry.java          # 函数注册中心
│   ├── WeatherFunction.java           # 天气查询
│   ├── StockFunction.java             # 股票查询
│   ├── CalculatorFunction.java        # 数学计算
│   └── DatabaseQueryFunction.java     # 数据库查询
├── service/
│   └── ChatService.java               # 修改：支持 Function Calling
└── model/
    └── FunctionResult.java            # 新增：函数调用结果
```

## 三、定义函数接口

```java
// function/FunctionRegistry.java
package com.example.chatbot.function;

import com.openai.models.*;
import java.util.*;

/**
 * 函数注册中心 —— 管理所有 AI 可以调用的函数
 */
@Component
public class FunctionRegistry {
    
    private final Map<String, CallableFunction> functions = new HashMap<>();
    private final List<ChatCompletionTool> tools = new ArrayList<>();
    
    /**
     * 注册一个函数
     */
    public void register(CallableFunction function) {
        functions.put(function.getName(), function);
        tools.add(buildTool(function));
    }
    
    /**
     * 获取 AI 可用的所有工具定义
     */
    public List<ChatCompletionTool> getTools() {
        return tools;
    }
    
    /**
     * 执行函数调用
     */
    public String execute(String functionName, String argumentsJson) {
        CallableFunction func = functions.get(functionName);
        if (func == null) {
            return "{\"error\": \"未知函数: " + functionName + "\"}";
        }
        try {
            return func.call(argumentsJson);
        } catch (Exception e) {
            return "{\"error\": \"" + e.getMessage() + "\"}";
        }
    }
    
    /**
     * 构建 OpenAI 格式的函数定义
     */
    private ChatCompletionTool buildTool(CallableFunction function) {
        return ChatCompletionTool.builder()
            .function(FunctionDefinition.builder()
                .name(function.getName())
                .description(function.getDescription())
                .parameters(function.getParameters())
                .build())
            .build();
    }
}

/**
 * 可被 AI 调用的函数接口
 */
interface CallableFunction {
    String getName();
    String getDescription();
    FunctionParameters getParameters();
    String call(String argumentsJson) throws Exception;
}
```

## 四、实现具体函数

### 4.1 天气查询

```java
// function/WeatherFunction.java
package com.example.chatbot.function;

import com.openai.models.FunctionParameters;
import org.springframework.stereotype.Component;
import java.util.*;

@Component
public class WeatherFunction implements CallableFunction {
    
    // 模拟天气数据库（实际调天气API）
    private static final Map<String, WeatherInfo> WEATHER_DB = new HashMap<>();
    
    static {
        WEATHER_DB.put("北京", new WeatherInfo("北京", "晴", 25, 40, "东北风3级"));
        WEATHER_DB.put("上海", new WeatherInfo("上海", "多云", 28, 65, "东南风2级"));
        WEATHER_DB.put("广州", new WeatherInfo("广州", "雷阵雨", 30, 85, "南风4级"));
        WEATHER_DB.put("深圳", new WeatherInfo("深圳", "阴", 29, 70, "西南风3级"));
        WEATHER_DB.put("杭州", new WeatherInfo("杭州", "小雨", 22, 80, "北风2级"));
        WEATHER_DB.put("成都", new WeatherInfo("成都", "晴转多云", 26, 55, "微风1级"));
    }
    
    @Override
    public String getName() {
        return "get_weather";
    }
    
    @Override
    public String getDescription() {
        return "获取指定城市的实时天气信息，包括天气状况、温度、湿度、风力等";
    }
    
    @Override
    public FunctionParameters getParameters() {
        return FunctionParameters.builder()
            .putAdditionalProperty("type", "object")
            .putAdditionalProperty("properties", Map.of(
                "city", Map.of(
                    "type", "string",
                    "description", "城市名称，如：北京、上海、广州"
                )
            ))
            .putAdditionalProperty("required", List.of("city"))
            .build();
    }
    
    @Override
    public String call(String argumentsJson) {
        String city = parseArgument(argumentsJson, "city");
        WeatherInfo info = WEATHER_DB.get(city);
        
        if (info == null) {
            return String.format("{\"error\": \"未找到城市 %s 的天气数据\"}", city);
        }
        
        return String.format(
            "{\"city\": \"%s\", \"weather\": \"%s\", \"temperature\": %d, " +
            "\"humidity\": %d, \"wind\": \"%s\"}",
            info.city, info.weather, info.temperature, info.humidity, info.wind
        );
    }
    
    private String parseArgument(String json, String key) {
        // 简单 JSON 解析（生产环境用 Jackson）
        String search = "\"" + key + "\": \"";
        int start = json.indexOf(search) + search.length();
        int end = json.indexOf("\"", start);
        return json.substring(start, end);
    }
    
    static class WeatherInfo {
        String city, weather, wind;
        int temperature, humidity;
        
        WeatherInfo(String city, String weather, int temperature, int humidity, String wind) {
            this.city = city;
            this.weather = weather;
            this.temperature = temperature;
            this.humidity = humidity;
            this.wind = wind;
        }
    }
}
```

### 4.2 股票查询

```java
// function/StockFunction.java
package com.example.chatbot.function;

import com.openai.models.FunctionParameters;
import org.springframework.stereotype.Component;
import java.util.*;

@Component
public class StockFunction implements CallableFunction {
    
    private static final Map<String, StockInfo> STOCK_DB = new HashMap<>();
    
    static {
        STOCK_DB.put("AAPL", new StockInfo("Apple Inc.", 189.30, 2.45, "+1.31%"));
        STOCK_DB.put("GOOGL", new StockInfo("Alphabet Inc.", 142.56, -0.78, "-0.54%"));
        STOCK_DB.put("TSLA", new StockInfo("Tesla Inc.", 248.50, 5.20, "+2.14%"));
        STOCK_DB.put("BABA", new StockInfo("阿里巴巴", 85.40, -1.32, "-1.52%"));
        STOCK_DB.put("0700.HK", new StockInfo("腾讯控股", 368.20, 3.80, "+1.04%"));
    }
    
    @Override
    public String getName() {
        return "get_stock_price";
    }
    
    @Override
    public String getDescription() {
        return "获取指定股票的实时价格、涨跌幅等信息。支持美股代码和港股代码。";
    }
    
    @Override
    public FunctionParameters getParameters() {
        return FunctionParameters.builder()
            .putAdditionalProperty("type", "object")
            .putAdditionalProperty("properties", Map.of(
                "symbol", Map.of(
                    "type", "string",
                    "description", "股票代码，如：AAPL（苹果）、BABA（阿里巴巴）、0700.HK（腾讯）"
                )
            ))
            .putAdditionalProperty("required", List.of("symbol"))
            .build();
    }
    
    @Override
    public String call(String argumentsJson) {
        String symbol = parseJsonValue(argumentsJson, "symbol");
        StockInfo info = STOCK_DB.get(symbol);
        
        if (info == null) {
            return String.format("{\"error\": \"未找到股票代码 %s 的数据\"}", symbol);
        }
        
        return String.format(
            "{\"symbol\": \"%s\", \"name\": \"%s\", \"price\": %.2f, " +
            "\"change\": %.2f, \"changePercent\": \"%s\"}",
            symbol, info.name, info.price, info.change, info.changePercent
        );
    }
    
    private String parseJsonValue(String json, String key) {
        String search = "\"" + key + "\": \"";
        int start = json.indexOf(search) + search.length();
        int end = json.indexOf("\"", start);
        return json.substring(start, end);
    }
    
    static class StockInfo {
        String name, changePercent;
        double price, change;
        StockInfo(String name, double price, double change, String changePercent) {
            this.name = name; this.price = price;
            this.change = change; this.changePercent = changePercent;
        }
    }
}
```

### 4.3 计算器

```java
// function/CalculatorFunction.java
package com.example.chatbot.function;

import com.openai.models.FunctionParameters;
import org.springframework.stereotype.Component;
import java.util.*;

@Component
public class CalculatorFunction implements CallableFunction {
    
    @Override
    public String getName() {
        return "calculate";
    }
    
    @Override
    public String getDescription() {
        return "执行数学计算。支持加减乘除、幂运算等。当用户提出数学计算需求时调用此函数。";
    }
    
    @Override
    public FunctionParameters getParameters() {
        return FunctionParameters.builder()
            .putAdditionalProperty("type", "object")
            .putAdditionalProperty("properties", Map.of(
                "expression", Map.of(
                    "type", "string",
                    "description", "数学表达式，如：'2 + 3 * 4'、'sqrt(16)'、'2 ** 10'"
                )
            ))
            .putAdditionalProperty("required", List.of("expression"))
            .build();
    }
    
    @Override
    public String call(String argumentsJson) {
        String expression = parseJsonValue(argumentsJson, "expression");
        
        try {
            // 安全警告：生产环境不要用 ScriptEngine，用专门的数学表达式解析库
            double result = evaluate(expression);
            return String.format("{\"expression\": \"%s\", \"result\": %s}", 
                expression, formatResult(result));
        } catch (Exception e) {
            return String.format("{\"error\": \"无法计算: %s\"}", e.getMessage());
        }
    }
    
    private double evaluate(String expression) {
        // 简化版计算器（只支持基础运算）
        // 生产环境推荐：exp4j 或 EvalEx
        expression = expression.replaceAll("\\s+", "");
        
        // 处理幂运算
        if (expression.contains("**")) {
            String[] parts = expression.split("\\*\\*");
            return Math.pow(Double.parseDouble(parts[0]), Double.parseDouble(parts[1]));
        }
        
        // 处理乘法
        if (expression.contains("*")) {
            String[] parts = expression.split("\\*");
            return Double.parseDouble(parts[0]) * Double.parseDouble(parts[1]);
        }
        
        // 处理除法
        if (expression.contains("/")) {
            String[] parts = expression.split("/");
            return Double.parseDouble(parts[0]) / Double.parseDouble(parts[1]);
        }
        
        // 处理加减
        if (expression.contains("+")) {
            String[] parts = expression.split("\\+");
            return Double.parseDouble(parts[0]) + Double.parseDouble(parts[1]);
        }
        if (expression.contains("-")) {
            String[] parts = expression.split("-");
            return Double.parseDouble(parts[0]) - Double.parseDouble(parts[1]);
        }
        
        return Double.parseDouble(expression);
    }
    
    private String formatResult(double result) {
        if (result == (long) result) {
            return String.valueOf((long) result);
        }
        return String.format("%.4f", result);
    }
    
    private String parseJsonValue(String json, String key) {
        String search = "\"" + key + "\": \"";
        int start = json.indexOf(search) + search.length();
        int end = json.indexOf("\"", start);
        return json.substring(start, end);
    }
}
```

## 五、修改 ChatService 支持 Function Calling

```java
// service/ChatService.java（核心修改部分）
@Service
public class ChatService {
    
    private final OpenAIClient openAIClient;
    private final FunctionRegistry functionRegistry;
    private final String model;
    
    // 构造函数注入
    public ChatService(
        OpenAIClient openAIClient,
        FunctionRegistry functionRegistry,
        @Value("${openai.model}") String model
    ) {
        this.openAIClient = openAIClient;
        this.functionRegistry = functionRegistry;
        this.model = model;
    }
    
    /**
     * 带 Function Calling 的对话
     * AI 可以调用注册的函数来回答用户问题
     */
    public Map<String, Object> chatWithFunctions(String sessionId, String userMessage) {
        if (sessionId == null || sessionId.isEmpty()) {
            sessionId = UUID.randomUUID().toString();
        }
        
        List<ChatCompletionMessageParam> messages = getOrCreateHistory(sessionId);
        messages.add(ChatCompletionMessageParam.ofUser(userMessage));
        
        // 第一轮：AI 可能会请求调用函数
        ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
            .model(ChatModel.of(model))
            .messages(messages)
            .tools(functionRegistry.getTools())  // 告诉 AI 可以调用哪些函数
            .toolChoice(ToolChoiceOption.AUTO)   // AI 自动决定是否调用函数
            .build();
        
        ChatCompletion completion = openAIClient.chat().completions().create(params);
        ChatCompletionMessage responseMessage = completion.choices().get(0).message();
        
        // 检查 AI 是否想调用函数
        if (responseMessage.toolCalls().isPresent() 
            && !responseMessage.toolCalls().get().isEmpty()) {
            
            // AI 想调用函数！执行函数并获取结果
            messages.add(ChatCompletionMessageParam.ofAssistant(responseMessage));
            
            for (ChatCompletionMessageToolCall toolCall : responseMessage.toolCalls().get()) {
                String functionName = toolCall.function().name();
                String arguments = toolCall.function().arguments();
                
                System.out.printf("[Function Call] %s(%s)%n", functionName, arguments);
                
                // 执行函数
                String result = functionRegistry.execute(functionName, arguments);
                
                System.out.printf("[Function Result] %s%n", result);
                
                // 把函数调用结果添加到对话历史
                messages.add(ChatCompletionMessageParam.ofTool(
                    ChatCompletionToolMessageParam.builder()
                        .toolCallId(toolCall.id())
                        .content(result)
                        .build()
                ));
            }
            
            // 第二轮：把函数结果发给 AI，让它生成最终回复
            ChatCompletionCreateParams secondParams = ChatCompletionCreateParams.builder()
                .model(ChatModel.of(model))
                .messages(messages)
                .build();
            
            ChatCompletion secondCompletion = openAIClient.chat()
                .completions().create(secondParams);
            
            String finalReply = secondCompletion.choices().get(0)
                .message().content().orElse("抱歉，无法生成回复。");
            
            messages.add(ChatCompletionMessageParam.ofAssistant(finalReply));
            
            return Map.of(
                "sessionId", sessionId,
                "content", finalReply,
                "functionsCalled", true
            );
        }
        
        // AI 不需要调用函数，直接返回回复
        String reply = responseMessage.content().orElse("抱歉，无法生成回复。");
        messages.add(ChatCompletionMessageParam.ofAssistant(reply));
        
        return Map.of(
            "sessionId", sessionId,
            "content", reply,
            "functionsCalled", false
        );
    }
    
    private List<ChatCompletionMessageParam> getOrCreateHistory(String sessionId) {
        return sessions.computeIfAbsent(sessionId, k -> {
            List<ChatCompletionMessageParam> history = new ArrayList<>();
            history.add(ChatCompletionMessageParam.ofSystem(
                "你是一个AI助手，可以调用函数来查询天气、股票、进行计算等。"
                + "当用户的问题需要实时数据时，请调用相应的函数。"
                + "获取到函数结果后，请用自然语言回答用户。"
            ));
            return history;
        });
    }
}
```

## 六、在启动时注册所有函数

```java
// ChatbotApplication.java（修改启动类）
@SpringBootApplication
public class ChatbotApplication implements CommandLineRunner {
    
    @Autowired
    private FunctionRegistry functionRegistry;
    
    @Autowired
    private WeatherFunction weatherFunction;
    
    @Autowired
    private StockFunction stockFunction;
    
    @Autowired
    private CalculatorFunction calculatorFunction;
    
    public static void main(String[] args) {
        SpringApplication.run(ChatbotApplication.class, args);
    }
    
    @Override
    public void run(String... args) {
        // 注册所有 AI 可调用的函数
        functionRegistry.register(weatherFunction);
        functionRegistry.register(stockFunction);
        functionRegistry.register(calculatorFunction);
        
        System.out.println("已注册 " + functionRegistry.getTools().size() + " 个函数");
        functionRegistry.getTools().forEach(tool -> 
            System.out.println("  - " + tool.function().name() + ": " 
                + tool.function().description())
        );
    }
}
```

## 七、新增 API 接口

```java
// controller/ChatController.java（新增接口）
@RestController
@RequestMapping("/api/chat")
public class ChatController {
    
    @Autowired
    private ChatService chatService;
    
    // 已有的接口...
    
    /**
     * 带 Function Calling 的对话接口
     */
    @PostMapping("/functions")
    public Map<String, Object> chatWithFunctions(@RequestBody ChatRequest request) {
        return chatService.chatWithFunctions(
            request.getSessionId(), 
            request.getMessage()
        );
    }
}
```

## 八、测试你的 AI Plugin

```bash
# 启动项目
export OPENAI_API_KEY="sk-your-key"
mvn spring-boot:run

# 测试天气查询
curl -X POST http://localhost:8080/api/chat/functions \
  -H "Content-Type: application/json" \
  -d '{"message": "北京今天天气怎么样？适合出去玩吗？"}'

# 测试股票查询
curl -X POST http://localhost:8080/api/chat/functions \
  -H "Content-Type: application/json" \
  -d '{"message": "帮我看看腾讯和阿里巴巴的股价"}'

# 测试计算
curl -X POST http://localhost:8080/api/chat/functions \
  -H "Content-Type: application/json" \
  -d '{"message": "算一下 256 乘以 1024 等于多少"}'

# 测试多轮对话
curl -X POST http://localhost:8080/api/chat/functions \
  -H "Content-Type: application/json" \
  -d '{"message": "上海天气如何？"}'

curl -X POST http://localhost:8080/api/chat/functions \
  -H "Content-Type: application/json" \
  -d '{"sessionId": "返回的sessionId", "message": "那杭州呢？"}'

# 测试不需要函数的普通对话
curl -X POST http://localhost:8080/api/chat/functions \
  -H "Content-Type: application/json" \
  -d '{"message": "什么是Java的垃圾回收机制？"}'
# 不需要函数，AI 直接用知识回答
```

## 九、进阶：让 AI 查询数据库

```java
// function/DatabaseQueryFunction.java
@Component
public class DatabaseQueryFunction implements CallableFunction {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Override
    public String getName() {
        return "query_database";
    }
    
    @Override
    public String getDescription() {
        return "查询公司内部数据库。可以查询员工信息、销售数据、项目进度等。";
    }
    
    @Override
    public FunctionParameters getParameters() {
        return FunctionParameters.builder()
            .putAdditionalProperty("type", "object")
            .putAdditionalProperty("properties", Map.of(
                "query_type", Map.of(
                    "type", "string",
                    "enum", List.of("employee", "sales", "projects"),
                    "description", "查询类型：employee(员工)、sales(销售)、projects(项目)"
                ),
                "filters", Map.of(
                    "type", "object",
                    "description", "筛选条件，如 {\"department\": \"技术部\"}"
                )
            ))
            .putAdditionalProperty("required", List.of("query_type"))
            .build();
    }
    
    @Override
    public String call(String argumentsJson) {
        // 解析参数并执行对应的 SQL 查询
        // 注意：生产环境要用参数化查询，不要拼接 SQL
        String queryType = parseJsonValue(argumentsJson, "query_type");
        
        switch (queryType) {
            case "employee":
                List<Map<String, Object>> employees = jdbcTemplate
                    .queryForList("SELECT name, department, position FROM employees LIMIT 5");
                return toJson(employees);
            case "sales":
                List<Map<String, Object>> sales = jdbcTemplate
                    .queryForList("SELECT month, revenue, profit FROM sales_summary ORDER BY month DESC LIMIT 12");
                return toJson(sales);
            default:
                return "{\"error\": \"不支持的查询类型\"}";
        }
    }
}
```

## 十、最佳实践和注意事项

### 1. 函数描述要精确

```java
// 不好的描述
@Override
public String getDescription() {
    return "查询数据"; // AI 不知道什么时候该调这个函数
}

// 好的描述
@Override
public String getDescription() {
    return "查询公司内部数据库。可以查询："
        + "1. 员工信息（按部门/职位筛选）"
        + "2. 月度销售数据"
        + "3. 项目进度和状态。"
        + "当用户询问公司内部数据时调用此函数。";
}
```

### 2. 参数定义要明确

```java
// 使用枚举约束参数值
"city", Map.of(
    "type", "string",
    "enum", List.of("北京", "上海", "广州", "深圳", "杭州", "成都"),
    "description", "城市名称"
)
```

### 3. 函数返回值要结构化

```java
// 让 AI 容易理解的结构化 JSON
return """
    {
        "success": true,
        "data": {
            "city": "北京",
            "temperature": 25,
            "weather": "晴天"
        },
        "message": "查询成功"
    }
    """;
```

### 4. 安全注意事项

```java
// 1. 限制函数调用频率（防止滥用）
// 2. 验证参数（防止 SQL 注入等攻击）
@Override
public String call(String argumentsJson) {
    // 白名单校验
    if (!ALLOWED_CITIES.contains(city)) {
        return "{\"error\": \"不支持的城市\"}";
    }
    // 参数化查询
    String sql = "SELECT * FROM weather WHERE city = ?";
    return jdbcTemplate.queryForObject(sql, String.class, city);
}

// 3. 不要给 AI 危险的函数权限（删除、退款等敏感操作需要人工确认）
```

## 总结

通过 Function Calling，你的 AI 助手可以：
- 查询实时天气
- 获取股票价格
- 执行数学计算
- 查询数据库
- 调用任何你封装的 API

它的本质是：**AI 做"大脑"（决策调用哪个函数），你的代码做"手脚"（执行具体操作）**。

这个模式是构建 AI Agent（智能体）的基石。后面你搭建企业知识库、AI 客服系统、智能运维助手，都是基于这个模式。

---

**下篇预告**：企业里最常见的需求——上传文档、自动问答。下一篇 30 分钟手把手教你搭建一个企业知识库系统，上传 PDF/Word → 自动分段 → 向量检索 → AI 问答，全程录屏式教程！
