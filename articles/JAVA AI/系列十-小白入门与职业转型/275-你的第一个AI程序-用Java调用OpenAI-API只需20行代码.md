# 你的第一个 AI 程序：用 Java 调用 OpenAI API 只需 20 行代码，ChatGPT在你代码里

## 开篇：扔掉所有理论，直接写代码

前面 9 篇文章讲了大量的理论知识。Transformer 的架构、Attention 的原理、Embedding 的妙用...我知道你可能已经有点晕了。

今天不讲了！我们**直接写代码**。

你只需要一个 OpenAI API Key，20 行 Java 代码，就能让 ChatGPT 在你的程序里说话。**不需要学 Python，不需要理解 AI 原理，甚至不需要配置任何复杂的环境**。

坐稳，5 分钟后你的第一个 AI 程序就能跑起来。

## 一、准备工作（2 分钟）

### 1.1 获取 API Key

```bash
# 1. 打开 https://platform.openai.com
# 2. 注册/登录
# 3. 点击右上角头像 → "View API keys"
# 4. 点击 "Create new secret key"
# 5. 复制保存（只显示一次！）

# 样例格式：sk-proj-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

**费用说明**：
- 新注册用户通常有 $5 免费额度
- GPT-4o-mini 非常便宜：$0.15/百万输入 Token，$0.60/百万输出 Token
- 你写 20 篇文章的调用费用也不到 1 美分

### 1.2 创建 Java 项目

```bash
# 方法1：用 Maven 创建
mvn archetype:generate \
  -DgroupId=com.example \
  -DartifactId=my-first-ai \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DinteractiveMode=false

cd my-first-ai
```

**pom.xml**：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example</groupId>
    <artifactId>my-first-ai</artifactId>
    <version>1.0-SNAPSHOT</version>
    
    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>
    
    <dependencies>
        <!-- OpenAI 官方 Java SDK -->
        <dependency>
            <groupId>com.openai</groupId>
            <artifactId>openai-java</artifactId>
            <version>0.18.0</version>
        </dependency>
    </dependencies>
</project>
```

或者用 Gradle：

```groovy
// build.gradle
plugins {
    id 'java'
}

repositories {
    mavenCentral()
}

dependencies {
    implementation 'com.openai:openai-java:0.18.0'
}
```

## 二、第一个程序：让 ChatGPT 说"Hello"（20 行代码）

```java
// src/main/java/com/example/FirstAI.java
package com.example;

import com.openai.client.OpenAIClient;
import com.openai.client.okhttp.OpenAIOkHttpClient;
import com.openai.models.ChatCompletion;
import com.openai.models.ChatCompletionCreateParams;
import com.openai.models.ChatModel;

public class FirstAI {
    public static void main(String[] args) {
        // 1. 从环境变量读取 API Key（不要硬编码！）
        String apiKey = System.getenv("OPENAI_API_KEY");
        
        // 2. 创建 OpenAI 客户端
        OpenAIClient client = OpenAIOkHttpClient.builder()
            .apiKey(apiKey)
            .build();
        
        // 3. 构建请求参数
        ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
            .model(ChatModel.GPT_4O_MINI)   // 最便宜的模型
            .addUserMessage("用一句话介绍你自己")
            .maxTokens(100)
            .build();
        
        // 4. 发送请求，获取回复
        ChatCompletion completion = client.chat().completions().create(params);
        
        // 5. 打印回复
        String reply = completion.choices().get(0).message().content().orElse("无回复");
        System.out.println("AI: " + reply);
    }
}
```

运行：

```bash
# 设置环境变量
export OPENAI_API_KEY="sk-your-key-here"

# 编译运行
mvn compile exec:java -Dexec.mainClass="com.example.FirstAI"
# 或者
javac -cp ... com/example/FirstAI.java
java -cp ... com.example.FirstAI

# 输出：
# AI: 你好！我是由OpenAI开发的人工智能助手，可以帮助回答问题、提供建议等。
```

**20 行代码，就这么简单。** 去掉了大括号和空行，核心逻辑不到 15 行。

## 三、多轮对话：让 AI 记住上下文

上面的程序是一次性的——每次调用互不关联，AI 不记得你上一轮说了什么。

要实现多轮对话，需要把**历史消息**都带上：

```java
// ChatBot.java — 多轮对话机器人
package com.example;

import com.openai.client.OpenAIClient;
import com.openai.client.okhttp.OpenAIOkHttpClient;
import com.openai.models.*;
import java.util.ArrayList;
import java.util.List;
import java.util.Scanner;

public class ChatBot {
    private final OpenAIClient client;
    private final List<ChatCompletionMessageParam> history;
    
    public ChatBot() {
        String apiKey = System.getenv("OPENAI_API_KEY");
        this.client = OpenAIOkHttpClient.builder()
            .apiKey(apiKey)
            .build();
        this.history = new ArrayList<>();
        
        // 设定系统角色
        history.add(ChatCompletionMessageParam.ofSystem(
            "你是一个编程助手，用简洁的语言回答Java相关问题。回答不超过100字。"
        ));
    }
    
    public String chat(String userMessage) {
        // 把用户消息加入历史
        history.add(ChatCompletionMessageParam.ofUser(userMessage));
        
        // 构建请求（带上完整对话历史）
        ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
            .model(ChatModel.GPT_4O_MINI)
            .messages(history)
            .maxTokens(200)
            .build();
        
        // 发送请求
        ChatCompletion completion = client.chat().completions().create(params);
        String reply = completion.choices().get(0).message().content().orElse("");
        
        // 把 AI 回复也加入历史
        history.add(ChatCompletionMessageParam.ofAssistant(reply));
        
        return reply;
    }
    
    public static void main(String[] args) {
        ChatBot bot = new ChatBot();
        Scanner scanner = new Scanner(System.in);
        
        System.out.println("Java 编程助手已启动！(输入 quit 退出)");
        
        while (true) {
            System.out.print("\n你: ");
            String input = scanner.nextLine().trim();
            
            if (input.equalsIgnoreCase("quit")) {
                System.out.println("再见！");
                break;
            }
            if (input.isEmpty()) continue;
            
            String reply = bot.chat(input);
            System.out.println("AI: " + reply);
        }
        
        scanner.close();
    }
}
```

多轮对话测试：

```
Java 编程助手已启动！(输入 quit 退出)

你: 什么是Lambda表达式？
AI: Lambda表达式是Java 8引入的语法糖，用于简化匿名内部类。

你: 给我一个例子
AI: (x, y) -> x + y 表示一个接受两个参数并返回它们之和的函数。

你: 那Stream怎么用？
AI: Stream是Java 8的另一个特性，可与Lambda配合实现函数式编程...
```

看到了吗？AI 记住了你之前问的是 Lambda，所以它的回答是有上下文的。

## 四、流式输出：像 ChatGPT 一样逐字显示

调用 API 后返回完整回复太慢了。真正的 ChatGPT 是**逐字蹦出来**的——这就是流式输出（Streaming）。

```java
// StreamChat.java — 流式输出
package com.example;

import com.openai.client.OpenAIClient;
import com.openai.client.okhttp.OpenAIOkHttpClient;
import com.openai.models.*;

public class StreamChat {
    public static void main(String[] args) {
        String apiKey = System.getenv("OPENAI_API_KEY");
        
        OpenAIClient client = OpenAIOkHttpClient.builder()
            .apiKey(apiKey)
            .build();
        
        ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
            .model(ChatModel.GPT_4O_MINI)
            .addUserMessage("用Java写一个快速排序，包含注释")
            .maxTokens(500)
            .build();
        
        System.out.print("AI: ");
        
        // 流式调用——每收到一个 chunk 就打印
        client.chat().completions().createStreaming(params)
            .stream()
            .forEach(chunk -> {
                chunk.choices().stream()
                    .filter(choice -> choice.delta().content().isPresent())
                    .forEach(choice -> {
                        String text = choice.delta().content().get();
                        System.out.print(text);  // 逐字打印！
                    });
            });
        
        System.out.println("\n\n流式输出完成！");
    }
}
```

运行效果：
```
AI: ```java
public class QuickSort {
    /**
     * 快速排序主方法
     * @param arr 待排序数组
     * @param low 起始索引
     * @param high 结束索引
     */
    public static void sort(int[] arr, int low, int high) {
        ...
    }
}
```
流式输出完成！
```

文字会像打字机一样一个字一个字蹦出来，体验感和 ChatGPT 一模一样。

## 五、System Prompt：给 AI 设定人设

System Prompt 是最容易被新手忽略但最重要的功能。它相当于**给 AI 下达了一个永久的角色设定**。

```java
// SystemPromptDemo.java
package com.example;

public class SystemPromptDemo {
    
    public static void main(String[] args) {
        String apiKey = System.getenv("OPENAI_API_KEY");
        OpenAIClient client = OpenAIOkHttpClient.builder()
            .apiKey(apiKey)
            .build();
        
        // ----- 不同的 System Prompt -----
        
        // Prompt 1: 代码审查专家
        chatWithSystemPrompt(client, 
            "你是一个资深Java代码审查专家。"
            + "对于任何代码，你必须指出："
            + "1. 潜在Bug 2. 性能问题 3. 安全漏洞 4. 可读性改进",
            "请审查这段代码：\n"
            + "public int calc(List<Integer> nums) {\n"
            + "  int r = 0;\n"
            + "  for (int i=0; i<nums.size(); i++) {\n"
            + "    r = r + nums.get(i);\n"
            + "  }\n"
            + "  return r;\n"
            + "}"
        );
        
        // Prompt 2: 段子手
        chatWithSystemPrompt(client,
            "用脱口秀的风格回答所有问题，每句话都要有梗",
            "解释一下什么是递归"
        );
        
        // Prompt 3: JSON 输出机器人
        chatWithSystemPrompt(client,
            "你永远只以JSON格式回复，格式为："
            + "{\"summary\": \"一句话总结\", \"details\": [\"要点1\", \"要点2\"]}",
            "介绍一下 Spring Boot 的自动配置原理"
        );
    }
    
    private static void chatWithSystemPrompt(
        OpenAIClient client, String systemPrompt, String userMessage
    ) {
        ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
            .model(ChatModel.GPT_4O_MINI)
            .addSystemMessage(systemPrompt)
            .addUserMessage(userMessage)
            .maxTokens(300)
            .build();
        
        ChatCompletion completion = client.chat().completions().create(params);
        String reply = completion.choices().get(0).message().content().orElse("");
        
        System.out.println("=== System Prompt ===");
        System.out.println(systemPrompt.substring(0, 80) + "...");
        System.out.println("=== Response ===");
        System.out.println(reply);
        System.out.println();
    }
}
```

## 六、Function Calling：让 AI 调用你的代码

这是 AI 编程中最强大的功能之一。AI 不再只是"说话"，而是可以**调用你的 API**。

```java
// WeatherBot.java — AI 能查天气了！
package com.example;

import com.openai.client.OpenAIClient;
import com.openai.client.okhttp.OpenAIOkHttpClient;
import com.openai.models.*;
import java.util.*;

public class WeatherBot {
    
    // 模拟天气 API（实际项目中你调用真正的天气接口）
    private static final Map<String, String> WEATHER_DB = Map.of(
        "北京", "晴天，25°C，湿度40%",
        "上海", "多云，28°C，湿度65%",
        "广州", "雷阵雨，30°C，湿度85%",
        "深圳", "阴天，29°C，湿度70%"
    );
    
    public static void main(String[] args) {
        String apiKey = System.getenv("OPENAI_API_KEY");
        OpenAIClient client = OpenAIOkHttpClient.builder()
            .apiKey(apiKey)
            .build();
        
        // 查询
        askWeather(client, "北京今天天气怎么样？");
        askWeather(client, "上海和广州哪个城市今天更适合出游？");
    }
    
    private static void askWeather(OpenAIClient client, String question) {
        System.out.println("用户: " + question);
        
        // 构建请求，告诉 AI 它可以调用一个"获取天气"的函数
        ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
            .model(ChatModel.GPT_4O_MINI)
            .addUserMessage(question)
            .addTool(ChatCompletionTool.builder()
                .function(FunctionDefinition.builder()
                    .name("get_weather")
                    .description("获取指定城市的实时天气信息")
                    .parameters(FunctionParameters.builder()
                        .putAdditionalProperty("type", "object")
                        .putAdditionalProperty("properties", Map.of(
                            "city", Map.of(
                                "type", "string",
                                "description", "城市名称，如：北京、上海、广州"
                            )
                        ))
                        .putAdditionalProperty("required", List.of("city"))
                        .build())
                    .build())
                .build())
            .build();
        
        // 发送请求
        ChatCompletion completion = client.chat().completions().create(params);
        
        // 检查 AI 是否想调用函数
        for (ChatCompletion.Choice choice : completion.choices()) {
            ChatCompletionMessage message = choice.message();
            
            if (message.toolCalls().isPresent()) {
                for (ChatCompletionMessageToolCall toolCall : message.toolCalls().get()) {
                    if (toolCall.function().name().equals("get_weather")) {
                        // AI 想查天气！解析参数，调用真正的天气 API
                        String args = toolCall.function().arguments();
                        String city = extractCity(args);
                        String weather = getWeather(city);
                        
                        System.out.println("  [AI 调用了 get_weather('" + city + "') → " + weather + "]");
                        
                        // 把函数调用的结果返回给 AI
                        // 让 AI 根据天气信息给出最终答案
                        // （真实项目中这一步需要额外一次 API 调用，这里简化演示）
                    }
                }
            }
        }
        
        String reply = completion.choices().get(0).message().content().orElse("");
        System.out.println("AI: " + reply);
        System.out.println();
    }
    
    private static String getWeather(String city) {
        return WEATHER_DB.getOrDefault(city, "天气数据未找到");
    }
    
    private static String extractCity(String json) {
        // 简单解析 JSON（实际用 Jackson/Gson）
        return json.replaceAll(".*\"city\"\\s*:\\s*\"([^\"]+)\".*", "$1");
    }
}
```

这就是 Function Calling——AI 不再只是"说话"，而是可以**操作外部世界**。查询天气、查询数据库、发送邮件、操作 IoT 设备...全都变成可能。

## 七、图片识别：让 AI 看图说话

GPT-4o 和 GPT-4o-mini 都支持图片识别：

```java
// ImageAnalyzer.java
package com.example;

import com.openai.client.OpenAIClient;
import com.openai.client.okhttp.OpenAIOkHttpClient;
import com.openai.models.*;
import java.nio.file.*;
import java.util.Base64;

public class ImageAnalyzer {
    public static void main(String[] args) throws Exception {
        String apiKey = System.getenv("OPENAI_API_KEY");
        OpenAIClient client = OpenAIOkHttpClient.builder()
            .apiKey(apiKey)
            .build();
        
        // 读取图片并转 Base64
        byte[] imageBytes = Files.readAllBytes(Path.of("screenshot.png"));
        String base64Image = Base64.getEncoder().encodeToString(imageBytes);
        
        // 构建请求（图片作为 content 的一部分）
        ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
            .model(ChatModel.GPT_4O_MINI)
            .addUserMessage(ChatCompletionUserMessageParam.builder()
                .contentOfString("请描述这张图片的内容，如果有代码，请解释代码的功能和潜在问题")
                .build())
            .addUserMessage(ChatCompletionUserMessageParam.builder()
                .contentOfImageUrl(ImageUrl.builder()
                    .url("data:image/png;base64," + base64Image)
                    .build())
                .build())
            .maxTokens(500)
            .build();
        
        ChatCompletion completion = client.chat().completions().create(params);
        String reply = completion.choices().get(0).message().content().orElse("");
        System.out.println(reply);
    }
}
```

## 八、JSON 模式：让 AI 返回结构化数据

```java
// JsonModeDemo.java
package com.example;

import com.openai.client.OpenAIClient;
import com.openai.client.okhttp.OpenAIOkHttpClient;
import com.openai.models.*;
import com.fasterxml.jackson.databind.ObjectMapper;

public class JsonModeDemo {
    public static void main(String[] args) throws Exception {
        String apiKey = System.getenv("OPENAI_API_KEY");
        OpenAIClient client = OpenAIOkHttpClient.builder()
            .apiKey(apiKey)
            .build();
        
        String text = "张三于2024年1月15日入职阿里巴巴，担任高级Java工程师，月薪35000元。";

        ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
            .model(ChatModel.GPT_4O_MINI)
            .addSystemMessage("你是一个信息提取机器人。从文本中提取信息，只返回JSON，不要任何其他内容。")
            .addUserMessage(String.format(
                "从以下文本中提取信息，返回JSON格式：\n\n%s\n\n"
                + "JSON格式：{\"name\": \"姓名\", \"company\": \"公司\", \"position\": \"职位\", "
                + "\"salary\": 薪资数字, \"startDate\": \"入职日期\"}",
                text
            ))
            .responseFormat(ResponseFormat.builder()
                .type(ResponseFormatType.JSON_OBJECT)
                .build())
            .build();
        
        ChatCompletion completion = client.chat().completions().create(params);
        String jsonStr = completion.choices().get(0).message().content().orElse("{}");
        
        // 直接解析成 Java 对象
        ObjectMapper mapper = new ObjectMapper();
        Map<String, Object> result = mapper.readValue(jsonStr, Map.class);
        
        System.out.println("提取结果：");
        result.forEach((k, v) -> System.out.println("  " + k + ": " + v));
    }
}
```

## 九、一个完整的小项目：AI 代码审查助手

把前面学到的组装成一个实用工具：

```java
// AICodeReviewer.java
package com.example;

import com.openai.client.OpenAIClient;
import com.openai.client.okhttp.OpenAIOkHttpClient;
import com.openai.models.*;
import java.io.IOException;
import java.nio.file.*;

public class AICodeReviewer {
    private static final String SYSTEM_PROMPT = """
        你是一个资深 Java 代码审查专家。对提供的代码进行审查，按以下格式输出：
        
        1. 代码概述（一句话）
        2. 潜在 Bug（如果有，列出）
        3. 性能问题（如果有，列出）
        4. 安全漏洞（如果有，列出）
        5. 可读性改进建议
        6. 综合评分（1-10分）
        
        请使用中文输出，简洁直接。
        """;
    
    private final OpenAIClient client;
    
    public AICodeReviewer() {
        String apiKey = System.getenv("OPENAI_API_KEY");
        this.client = OpenAIOkHttpClient.builder()
            .apiKey(apiKey)
            .build();
    }
    
    public String review(String code) {
        ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
            .model(ChatModel.GPT_4O_MINI)
            .addSystemMessage(SYSTEM_PROMPT)
            .addUserMessage("请审查以下 Java 代码：\n\n```java\n" + code + "\n```")
            .maxTokens(1000)
            .build();
        
        ChatCompletion completion = client.chat().completions().create(params);
        return completion.choices().get(0).message().content().orElse("审查失败");
    }
    
    public String reviewFile(String filePath) throws IOException {
        String code = Files.readString(Path.of(filePath));
        return review(code);
    }
    
    public static void main(String[] args) throws IOException {
        AICodeReviewer reviewer = new AICodeReviewer();
        
        String testCode = """
            public class UserService {
                private Connection conn;
                
                public User findUser(String userId) {
                    String sql = "SELECT * FROM users WHERE id = '" + userId + "'";
                    Statement stmt = conn.createStatement();
                    ResultSet rs = stmt.executeQuery(sql);
                    if (rs.next()) {
                        return new User(rs.getString("id"), rs.getString("name"));
                    }
                    return null;
                }
            }
            """;
        
        System.out.println("===== AI 代码审查报告 =====\n");
        System.out.println(reviewer.review(testCode));
    }
}
```

## 十、成本控制最佳实践

```java
public class CostControl {
    
    // 1. 使用便宜的模型
    private static final ChatModel MODEL = ChatModel.GPT_4O_MINI;
    // GPT-4o-mini 比 GPT-4o 便宜 30-60 倍，日常使用完全足够
    
    // 2. 限制 maxTokens
    // .maxTokens(300)  而不是默认的无限
    
    // 3. 缓存常见问题的回复
    private Map<String, String> cache = new HashMap<>();
    public String chatWithCache(String prompt) {
        if (cache.containsKey(prompt)) {
            return cache.get(prompt);  // 命中缓存，不花钱
        }
        String reply = callAPI(prompt);
        cache.put(prompt, reply);
        return reply;
    }
    
    // 4. 监控花费
    public void logUsage(ChatCompletion completion) {
        var usage = completion.usage();
        if (usage.isPresent()) {
            int promptTokens = usage.get().promptTokens();
            int completionTokens = usage.get().completionTokens();
            double cost = (promptTokens * 0.15 + completionTokens * 0.6) / 1_000_000.0;
            System.out.printf("本次调用: 输入%d Token, 输出%d Token, 费用$%.6f%n",
                promptTokens, completionTokens, cost);
        }
    }
}
```

## 总结：你已经是 AI 开发者了

恭喜！跑完这些代码，你已经：
1. 成功调用了 OpenAI API
2. 实现了多轮对话
3. 实现了流式输出
4. 学会了 System Prompt
5. 了解了 Function Calling
6. 能解析图片内容
7. 能获取结构化 JSON 输出
8. 掌握成本控制

**你不再只是"对 AI 感兴趣的程序员"，你已经是一个 AI 应用开发者了。**

---

**下篇预告**：刚才的 Demo 都是命令行程序。下一篇我们用 Spring Boot 搭建一个完整的智能聊天机器人后端，带 Web 界面，带对话历史，带用户管理。附完整源码，直接 clone 就能跑。
