# Spring AI Function Calling：让 LLM 直接调用你的 Service 层，AI不仅能聊天还能干活

> 之前AI只能回答问题，现在AI能直接操作你的系统——查询数据库、发送邮件、创建订单。这就是Function Calling。

---

## 一、从"会说话的AI"到"能干活的AI"

回想一下传统的AI对话：用户问"帮我查下订单12345的状态"，AI回答"你可以登录系统查询"。这很鸡肋——用户想要的是AI直接告诉他结果，而不是教他怎么用系统。

**Function Calling 就是这个转折点**：你告诉AI"你有一个查订单的工具可以用"，AI理解用户意图后，自动决定调用哪个函数，提取参数，然后你把执行结果返回给AI，AI再组织成自然语言回复用户。

整个流程就像这样：

```
用户: "帮我取消订单12345"
 ↓
AI: 我需要调用 cancelOrder(orderId="12345")
 ↓
你的系统: 执行 cancelOrder("12345") → 返回 {success: true}
 ↓
AI: "已为您成功取消订单12345，退款将在3个工作日内退回您的账户。"
```

Spring AI 让这个过程优雅到了极致——**一个 `@Tool` 注解就能搞定**。

---

## 二、@Tool 注解详解

### 2.1 最小示例

```java
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;

@Service
public class WeatherService {

    @Tool(name = "getWeather", description = "获取指定城市的实时天气信息")
    public String getWeather(
            @ToolParam(description = "城市名称，如北京、上海") String city) {
        // 实际项目中替换为调用天气API
        Map<String, String> weatherData = Map.of(
            "北京", "晴天，25°C，湿度45%",
            "上海", "多云转小雨，22°C，湿度70%",
            "深圳", "雷阵雨，28°C，湿度85%"
        );
        return weatherData.getOrDefault(city, "暂无该城市天气数据");
    }
}
```

### 2.2 @Tool 注解参数详解

| 参数 | 说明 | 示例 |
|------|------|------|
| `name` | 函数名称，LLM通过此名称识别和调用 | `"getOrderStatus"` |
| `description` | **最重要！** LLM根据此描述决定何时调用这个函数 | `"根据订单号查询订单状态，返回订单的当前处理阶段"` |
| `returnDirect` | 是否直接将返回值作为LLM的回答（跳过二次总结） | `false`（默认） |

> **核心要点**：`description` 写得好不好，直接决定了AI能否在正确的时机调用你的函数。描述要清晰、具体，包含触发条件和返回内容。

### 2.3 @ToolParam 注解详解

| 参数 | 说明 | 示例 |
|------|------|------|
| `description` | 参数说明，帮助LLM从用户话术中提取正确的参数值 | `"用户要查询的11位手机号"` |
| `required` | 是否必填 | `true`（默认） |
| `defaultValue` | 默认值 | `"待处理"` |

### 2.4 复杂参数示例

```java
@Service
public class FlightService {

    @Tool(name = "searchFlights", description = """
        搜索航班信息。当用户想查询某条航线的机票时调用。
        需要提供出发地、目的地和出发日期。
        返回符合条件的航班列表及价格信息。
        """)
    public List<FlightInfo> searchFlights(
            @ToolParam(description = "出发城市，如北京") String from,
            @ToolParam(description = "到达城市，如上海") String to,
            @ToolParam(description = "出发日期，格式yyyy-MM-dd") String date,
            @ToolParam(description = "舱位类型：经济舱/商务舱/头等舱",
                       required = false, defaultValue = "经济舱") String cabinClass) {
        
        // 模拟航班搜索
        return List.of(
            new FlightInfo("MU5101", from, to, date + " 08:00", 
                           cabinClass, 1280.0),
            new FlightInfo("CA1234", from, to, date + " 14:30", 
                           cabinClass, 1450.0)
        );
    }
}
```

---

## 三、注册工具函数给 ChatClient

### 3.1 方式一：Bean自动注册（推荐）

Spring AI 会在启动时自动扫描所有带 `@Tool` 注解的Bean方法，并注册为可调用工具：

```java
@Configuration
public class AiConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder) {
        return builder
            // Spring AI会自动发现所有@Tool方法
            .build();
    }
}
```

### 3.2 方式二：手动注册（精确控制）

```java
@Configuration
public class AiConfig {

    @Bean
    public ChatClient chatClient(ChatClient.Builder builder,
                                  WeatherService weatherService,
                                  FlightService flightService) {
        return builder
            .defaultTools(weatherService, flightService)
            .build();
    }
}
```

### 3.3 方式三：请求级别动态选择工具

不同API端点可能需要不同的工具集：

```java
@RestController
public class ChatController {

    private final ChatClient.Builder builder;
    private final OrderToolService orderTools;
    private final FlightToolService flightTools;

    @PostMapping("/chat/order")
    public String orderChat(@RequestBody String question) {
        // 订单相关的聊天，只注册订单工具
        return builder.build()
            .prompt(question)
            .tools(orderTools)  // 只注入订单相关工具
            .call()
            .content();
    }

    @PostMapping("/chat/travel")
    public String travelChat(@RequestBody String question) {
        // 出行相关的聊天，只注册出行工具
        return builder.build()
            .prompt(question)
            .tools(flightTools)  // 只注入出行相关工具
            .call()
            .content();
    }
}
```

---

## 四、实战一：AI自动查询天气 + 订机票

下面是一个完整示例，AI能理解用户"我想去三亚度假"这句话，先查天气再搜机票：

### 4.1 服务实现

```java
@Service
public class TravelAssistantService {

    @Tool(name = "getWeather", description = "查询指定城市未来7天天气预报")
    public WeatherReport getWeather(
            @ToolParam(description = "城市名称") String city) {
        // 模拟天气API调用
        return new WeatherReport(city, "晴转多云", 
                                 24, 32, "东南风3级");
    }

    @Tool(name = "searchFlights", description = """
        搜索国内航班。当用户需要查询或预订从A城市到B城市的机票时调用。
        返回航班号、出发时间、到达时间、价格等信息。
        """)
    public List<Flight> searchFlights(
            @ToolParam(description = "出发城市") String from,
            @ToolParam(description = "到达城市") String to,
            @ToolParam(description = "出发日期，如2025-06-01") String date) {
        
        return List.of(
            new Flight("CA1234", "中国国航", from, to, 
                       date + "T08:30", date + "T10:30", 980.0),
            new Flight("MU5678", "东方航空", from, to,
                       date + "T13:00", date + "T15:00", 1200.0),
            new Flight("CZ9012", "南方航空", from, to,
                       date + "T18:30", date + "T20:30", 860.0)
        );
    }

    @Tool(name = "bookFlight", description = """
        预订机票。当用户确认要购买某个航班时调用。
        需要提供航班号和乘机人信息。
        返回预订结果和订单号。
        """)
    public BookingResult bookFlight(
            @ToolParam(description = "航班号，如CA1234") String flightNo,
            @ToolParam(description = "乘机人姓名") String passengerName,
            @ToolParam(description = "乘机人身份证号") String idCard) {
        
        // 模拟预订
        String bookingId = "BK" + System.currentTimeMillis();
        return new BookingResult(bookingId, flightNo, 
                                  passengerName, "预订成功");
    }
}
```

### 4.2 Controller

```java
@RestController
@RequestMapping("/api/travel")
public class TravelChatController {

    private final ChatClient chatClient;

    public TravelChatController(ChatClient.Builder builder) {
        this.chatClient = builder.build();
    }

    @PostMapping("/chat")
    public String chat(@RequestBody String userMessage) {
        return chatClient.prompt()
            .user(userMessage)
            .call()
            .content();
    }
}
```

### 4.3 对话效果演示

```
用户: "我下周想去三亚玩，帮我看看天气和机票"

AI内部调用链:
  → 调用 getWeather("三亚")
    → 返回: "三亚 晴，30°C~36°C，适合海滩度假"
  → 调用 searchFlights("北京", "三亚", "2025-06-10")
    → 返回: [3个航班信息]

AI回答:
"三亚下周天气不错，晴天为主，30°C~36°C，非常适合海滩度假！
为您找到3个航班：
1. CA1234 中国国航 08:30-10:30 ¥980
2. MU5678 东方航空 13:00-15:00 ¥1200
3. CZ9012 南方航空 18:30-20:30 ¥860
需要我帮您预订哪个航班呢？"
```

```
用户: "帮我订最便宜的那个，我叫张三，身份证110101199001011234"

AI内部调用链:
  → 调用 bookFlight("CZ9012", "张三", "110101199001011234")
    → 返回: 预订成功，订单号BK1716261000

AI回答:
"已为您预订成功！订单信息如下：
- 订单号：BK1716261000
- 航班：CZ9012 南方航空
- 日期：2025-06-10 18:30→20:30
- 乘机人：张三
- 金额：¥860
祝您旅途愉快！"
```

---

## 五、实战二：AI自动创建Jira任务

企业场景中，让AI直接帮你操作Jira：

```java
@Service
public class JiraToolService {

    private final JiraRestClient jiraClient;

    @Tool(name = "createJiraIssue", description = """
        在Jira中创建新的工作任务或Bug。
        当用户需要记录一个问题、创建一个开发任务或需求时调用。
        需要项目标识、问题摘要和描述。
        返回创建成功后的问题编号和链接。
        """)
    public JiraIssueResult createIssue(
            @ToolParam(description = "Jira项目标识，如PROJ、DEV") String projectKey,
            @ToolParam(description = "问题类型：Task/Bug/Story/Epic") String issueType,
            @ToolParam(description = "问题标题摘要") String summary,
            @ToolParam(description = "详细描述") String description,
            @ToolParam(description = "指派人用户名", required = false, 
                       defaultValue = "") String assignee,
            @ToolParam(description = "优先级：Highest/High/Medium/Low", 
                       required = false, defaultValue = "Medium") String priority) {

        IssueInput input = IssueInput.builder()
            .project(ProjectInput.key(projectKey))
            .issueType(IssueTypeInput.name(issueType))
            .summary(summary)
            .description(description)
            .assignee(assignee.isEmpty() ? null : 
                      AssigneeInput.assignee(assignee))
            .priority(PriorityInput.name(priority))
            .build();

        BasicIssue issue = jiraClient.createIssue(input);
        return new JiraIssueResult(
            issue.getKey(), 
            "https://jira.yourcompany.com/browse/" + issue.getKey());
    }

    @Tool(name = "searchJiraIssues", description = """
        搜索Jira中的问题。根据关键词、状态、指派人等条件过滤。
        当用户想了解某些任务的进展状态时调用。
        """)
    public List<JiraIssue> searchIssues(
            @ToolParam(description = "JQL查询语句或关键词") String query,
            @ToolParam(description = "返回结果最大数量", 
                       required = false, defaultValue = "10") Integer maxResults) {
        
        return jiraClient.search(query, maxResults);
    }
}
```

对话效果：

```
用户: "帮我创建一个Bug：登录页面在Safari浏览器上无法显示验证码"

AI: → createJiraIssue("WEB", "Bug", 
      "登录页面Safari浏览器验证码不显示",
      "问题描述：在Safari 17.2版本中...", "", "High")

AI回答:
"已创建Bug：WEB-2847
链接：https://jira.yourcompany.com/browse/WEB-2847
优先级：High
请安排前端团队排查Safari兼容性问题。"
```

---

## 六、Function Calling 的错误处理

### 6.1 工具执行异常处理

```java
@Tool(name = "cancelOrder", description = "取消指定订单")
public CancelResult cancelOrder(
        @ToolParam(description = "订单号") String orderId) {
    try {
        Order order = orderService.findById(orderId);
        if (order == null) {
            return CancelResult.fail("订单" + orderId + "不存在");
        }
        if (!order.isCancellable()) {
            return CancelResult.fail(
                "订单状态为" + order.getStatus() + "，无法取消，" +
                "请联系人工客服");
        }
        orderService.cancel(orderId);
        return CancelResult.success(order);
    } catch (Exception e) {
        log.error("取消订单失败: orderId={}", orderId, e);
        return CancelResult.fail("系统异常，请稍后重试。如急需，请联系人工客服");
    }
}

// 返回结果封装
public record CancelResult(
    boolean success, 
    String message, 
    String orderId, 
    Double refundAmount
) {
    public static CancelResult fail(String message) {
        return new CancelResult(false, message, null, null);
    }
    public static CancelResult success(Order order) {
        return new CancelResult(true, "取消成功", 
            order.getOrderId(), order.getAmount());
    }
}
```

### 6.2 安全控制——敏感操作二次确认

```java
public class SafetyControl {

    /**
     * 敏感操作检查——在执行前判断是否需要额外确认
     */
    public static boolean requiresConfirmation(String toolName) {
        Set<String> sensitiveOperations = Set.of(
            "cancelOrder",      // 取消订单
            "bookFlight",       // 预订机票
            "deleteAccount",    // 删除账户
            "transferMoney"     // 转账
        );
        return sensitiveOperations.contains(toolName);
    }
}

// 在ChatClient的拦截器中实现
@Component
public class ToolExecutionInterceptor implements FunctionCallback {

    @Override
    public String call(String functionName, String input) {
        if (SafetyControl.requiresConfirmation(functionName)) {
            log.warn("执行敏感操作: {}，参数: {}", functionName, input);
            // 可以发送通知、记录审计日志
            auditService.record(functionName, input, 
                SecurityContextHolder.getContext().getAuthentication());
        }
        return executeWithRetry(functionName, input);
    }
}
```

### 6.3 权限控制

```java
@Tool(name = "deleteAccount", description = "删除用户账户")
public String deleteAccount(
        @ToolParam(description = "要删除的用户ID") String userId) {
    
    // 权限检查
    Authentication auth = SecurityContextHolder.getContext()
        .getAuthentication();
    if (auth == null || !auth.getAuthorities().contains(
            new SimpleGrantedAuthority("ROLE_ADMIN"))) {
        return "权限不足：只有管理员才能删除账户";
    }

    // 不能删除自己
    String currentUser = auth.getName();
    if (currentUser.equals(userId)) {
        return "不能删除自己的账户";
    }

    userService.deleteUser(userId);
    return "用户" + userId + "已被成功删除";
}
```

### 6.4 超时和重试

```java
@Configuration
public class FunctionCallingConfig {

    @Bean
    public FunctionCallbackRetryInterceptor retryInterceptor() {
        return new FunctionCallbackRetryInterceptor() {
            @Override
            public String onFailure(String functionName, 
                                     String input, 
                                     Exception e) {
                log.error("Function call failed: {} with input: {}", 
                          functionName, input, e);
                // 返回给LLM的错误信息，LLM会尝试重新理解并调用
                return "执行失败：" + e.getMessage() + 
                       "。请尝试使用不同的参数或联系人工客服。";
            }
        };
    }
}
```

---

## 七、Function Calling 的设计原则

### 7.1 描述即文档

`@Tool` 的 `description` 是给AI看的"函数使用说明书"，必须：

- **明确触发条件**："当用户询问天气、温度、气候时调用"
- **说明参数含义**："城市名称，如北京、上海，不要带'市'字"
- **描述返回内容**："返回JSON包含温度、湿度、天气状况"

### 7.2 单一职责

每个工具函数只做一件事。避免这种：

```java
// 坏设计——一个函数干太多事
@Tool(description = "查询订单或取消订单或修改订单地址")
public String handleOrder(...) { ... }

// 好设计——拆分成三个独立函数
@Tool(description = "查询订单详情")  public Order getOrder(...);
@Tool(description = "取消订单")      public CancelResult cancelOrder(...);
@Tool(description = "修改订单收货地址") public void updateAddress(...);
```

### 7.3 返回值要语义化

返回给LLM的结果应该清晰可读：

```java
// 坏——返回原始对象，LLM看不懂
public Order queryOrder(String orderId) {
    return orderRepository.findById(orderId);
}

// 好——返回语义化的结构化信息
public String queryOrder(String orderId) {
    Order order = orderRepository.findById(orderId);
    if (order == null) return "未找到订单" + orderId;
    return String.format(
        "订单号：%s，商品：%s，金额：%.2f元，状态：%s，下单时间：%s",
        order.getOrderId(), order.getProductName(),
        order.getAmount(), order.getStatusDisplay(),
        order.getCreateTime());
}
```

---

## 八、多工具协作的对话示例

实际对话中，AI会自动编排多个工具的调用顺序：

```
用户: "我今天必须完成项目PROJ的Jira任务WEB-100，但我要出差去上海，
      帮我查下明天北京的天气，再找个下午的航班"

AI内部编排:
  Step 1: searchJiraIssues("key=WEB-100") 
           → 返回: "WEB-100: 登录页重构 [进行中]"
  Step 2: getWeather("北京") 
           → 返回: "晴，25°C"
  Step 3: searchFlights("北京", "上海", "2025-06-06") 
           → 返回: [下午3个航班]

AI回答:
"好的，帮你梳理一下：
1. Jira任务WEB-100「登录页重构」正在进行中，需要今天完成
2. 明天北京天气晴朗，25°C，适合出行
3. 下午有3个航班：MU5678 13:00-15:00 ¥1200 / 
   CA9012 16:00-18:00 ¥980 / CZ3456 18:30-20:30 ¥860
建议你选CA9012，下午4点出发，可以利用上午完成Jira任务后再出发。"
```

---

## 九、小结

Function Calling 让AI从一个"只会说话的聊天机器人"变成了一个"能执行任务的操作系统"。

核心要点：
1. **`@Tool`注解是唯一入口**——写好 description 决定了AI能否识别调用时机
2. **工具粒度要细**——一个函数只做一件事
3. **返回值要语义化**——AI需要的不是原始对象，而是它能读懂的信息
4. **安全控制不可少**——敏感操作加权限校验和审计日志
5. **异常信息要友好**——返回给AI的错误信息决定了用户体验

当AI能从聊天变成干活，接下来要解决的就是——**如何把AI的文本回答变成Java程序可以直接使用的结构化数据**。这就是下一篇 **BeanOutputParser** 要讲的。

---

> **下一篇预告**：Spring AI 输出解析——BeanOutputParser 自动映射为 Java POJO，AI的文本回答秒变结构化数据，彻底告别手写JSON解析！

---

*如果觉得有帮助，欢迎点赞收藏关注，你的支持是我持续输出的动力！*

---
