# 用 Spring Boot 搭建一个智能聊天机器人后端（附完整源码），1小时搞定

## 开篇：从 Demo 到产品

上一篇我们写了 20 行 Java 代码调用 OpenAI API。但那是命令行程序——离真正的产品还差得远。

今天我们把那个 Demo 升级成一个**可以上线的聊天机器人后端**。完整的 Spring Boot 项目，带 Web 界面、对话历史、流式输出、会话管理。你读完这篇文章就能 Clone 到本地跑起来。

话不多说，开始写代码。

## 一、项目架构概览

```
ai-chatbot/
├── pom.xml
├── src/main/java/com/example/chatbot/
│   ├── ChatbotApplication.java          # 启动类
│   ├── config/
│   │   └── OpenAIConfig.java            # OpenAI 配置
│   ├── controller/
│   │   └── ChatController.java          # REST API
│   ├── service/
│   │   └── ChatService.java             # 核心业务逻辑
│   ├── model/
│   │   ├── ChatRequest.java             # 请求 DTO
│   │   └── ChatResponse.java            # 响应 DTO
│   └── config/
│       └── CorsConfig.java              # 跨域配置
└── src/main/resources/
    ├── application.yml
    └── static/
        └── index.html                   # 前端聊天界面
```

## 二、pom.xml：最小依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.3.0</version>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>ai-chatbot</artifactId>
    <version>1.0.0</version>
    <name>AI Chatbot</name>
    
    <properties>
        <java.version>17</java.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- OpenAI Java SDK -->
        <dependency>
            <groupId>com.openai</groupId>
            <artifactId>openai-java</artifactId>
            <version>0.18.0</version>
        </dependency>
        
        <!-- Lombok（减少样板代码） -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

## 三、application.yml：配置

```yaml
server:
  port: 8080

spring:
  application:
    name: ai-chatbot

# OpenAI 配置
openai:
  api-key: ${OPENAI_API_KEY:your-key-here}
  model: gpt-4o-mini
  max-tokens: 2000
  temperature: 0.7

# 日志
logging:
  level:
    com.example.chatbot: DEBUG
```

## 四、启动类和配置

```java
// ChatbotApplication.java
package com.example.chatbot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ChatbotApplication {
    public static void main(String[] args) {
        SpringApplication.run(ChatbotApplication.class, args);
    }
}
```

```java
// config/OpenAIConfig.java
package com.example.chatbot.config;

import com.openai.client.OpenAIClient;
import com.openai.client.okhttp.OpenAIOkHttpClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenAIConfig {
    
    @Value("${openai.api-key}")
    private String apiKey;
    
    @Bean
    public OpenAIClient openAIClient() {
        return OpenAIOkHttpClient.builder()
            .apiKey(apiKey)
            .build();
    }
}
```

```java
// config/CorsConfig.java
package com.example.chatbot.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;
import org.springframework.web.filter.CorsFilter;

import java.util.List;

@Configuration
public class CorsConfig {
    @Bean
    public CorsFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOriginPatterns(List.of("*"));
        config.setAllowedMethods(List.of("GET", "POST", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return new CorsFilter(source);
    }
}
```

## 五、DTO 模型

```java
// model/ChatRequest.java
package com.example.chatbot.model;

import lombok.Data;

@Data
public class ChatRequest {
    /** 会话ID（第一次为空，后续传入以保持上下文） */
    private String sessionId;
    
    /** 用户消息 */
    private String message;
}
```

```java
// model/ChatResponse.java
package com.example.chatbot.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ChatResponse {
    /** 会话ID（用于后续对话） */
    private String sessionId;
    
    /** AI 回复 */
    private String content;
    
    /** Token 使用统计 */
    private int promptTokens;
    private int completionTokens;
}
```

```java
// model/StreamChunk.java
package com.example.chatbot.model;

import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class StreamChunk {
    /** 流式输出的文本片段 */
    private String content;
    
    /** 是否已经结束 */
    private boolean done;
}
```

## 六、核心业务逻辑

```java
// service/ChatService.java
package com.example.chatbot.service;

import com.openai.client.OpenAIClient;
import com.openai.models.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class ChatService {
    
    private final OpenAIClient openAIClient;
    private final String model;
    private final int maxTokens;
    private final double temperature;
    
    // 内存中的对话历史（生产环境请用 Redis）
    private final Map<String, List<ChatCompletionMessageParam>> sessions = new ConcurrentHashMap<>();
    
    public ChatService(
        OpenAIClient openAIClient,
        @Value("${openai.model}") String model,
        @Value("${openai.max-tokens}") int maxTokens,
        @Value("${openai.temperature}") double temperature
    ) {
        this.openAIClient = openAIClient;
        this.model = model;
        this.maxTokens = maxTokens;
        this.temperature = temperature;
    }
    
    /**
     * 普通对话（非流式）
     */
    public Map<String, Object> chat(String sessionId, String userMessage) {
        // 获取或创建会话历史
        if (sessionId == null || sessionId.isEmpty()) {
            sessionId = UUID.randomUUID().toString();
        }
        
        List<ChatCompletionMessageParam> history = sessions
            .computeIfAbsent(sessionId, k -> new ArrayList<>());
        
        // 如果是新会话，加上系统提示
        if (history.isEmpty()) {
            history.add(ChatCompletionMessageParam.ofSystem(
                "你是一个乐于助人的AI助手。请用中文回答，简洁清晰。"
            ));
        }
        
        // 添加用户消息
        history.add(ChatCompletionMessageParam.ofUser(userMessage));
        
        // 调用 OpenAI
        ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
            .model(ChatModel.of(model))
            .messages(history)
            .maxTokens(maxTokens)
            .temperature(temperature)
            .build();
        
        ChatCompletion completion = openAIClient.chat().completions().create(params);
        
        String reply = completion.choices().get(0)
            .message().content().orElse("抱歉，我没有生成回复。");
        
        // 添加 AI 回复到历史
        history.add(ChatCompletionMessageParam.ofAssistant(reply));
        
        // 限制历史长度（防止 Context Window 溢出）
        while (history.size() > 40) {
            history.remove(1); // 保留系统提示
        }
        
        // 构建返回结果
        Map<String, Object> result = new HashMap<>();
        result.put("sessionId", sessionId);
        result.put("content", reply);
        
        // 添加用量统计
        completion.usage().ifPresent(usage -> {
            result.put("promptTokens", usage.promptTokens());
            result.put("completionTokens", usage.completionTokens());
        });
        
        return result;
    }
    
    /**
     * 流式对话（SSE）
     */
    public SseEmitter chatStream(String sessionId, String userMessage) {
        SseEmitter emitter = new SseEmitter(300_000L); // 5分钟超时
        
        if (sessionId == null || sessionId.isEmpty()) {
            sessionId = UUID.randomUUID().toString();
        }
        
        final String finalSessionId = sessionId;
        
        List<ChatCompletionMessageParam> history = sessions
            .computeIfAbsent(sessionId, k -> new ArrayList<>());
        
        if (history.isEmpty()) {
            history.add(ChatCompletionMessageParam.ofSystem(
                "你是一个乐于助人的AI助手。请用中文回答，简洁清晰。"
            ));
        }
        
        history.add(ChatCompletionMessageParam.ofUser(userMessage));
        
        // 异步处理流式响应
        new Thread(() -> {
            StringBuilder fullReply = new StringBuilder();
            
            try {
                ChatCompletionCreateParams params = ChatCompletionCreateParams.builder()
                    .model(ChatModel.of(model))
                    .messages(history)
                    .maxTokens(maxTokens)
                    .temperature(temperature)
                    .build();
                
                // 先发送 sessionId
                emitter.send(SseEmitter.event()
                    .name("session")
                    .data(Map.of("sessionId", finalSessionId)));
                
                // 流式发送文本
                openAIClient.chat().completions().createStreaming(params)
                    .stream()
                    .forEach(chunk -> {
                        chunk.choices().stream()
                            .filter(c -> c.delta().content().isPresent())
                            .forEach(c -> {
                                String text = c.delta().content().get();
                                fullReply.append(text);
                                try {
                                    emitter.send(SseEmitter.event()
                                        .name("message")
                                        .data(text));
                                } catch (Exception e) {
                                    // 客户端断开了
                                }
                            });
                    });
                
                // 发送完成事件
                history.add(ChatCompletionMessageParam.ofAssistant(fullReply.toString()));
                
                // 限制历史长度
                while (history.size() > 40) {
                    history.remove(1);
                }
                
                emitter.send(SseEmitter.event()
                    .name("done")
                    .data(Map.of("content", fullReply.toString())));
                emitter.complete();
                
            } catch (Exception e) {
                try {
                    emitter.send(SseEmitter.event()
                        .name("error")
                        .data(Map.of("message", e.getMessage())));
                    emitter.completeWithError(e);
                } catch (Exception ex) {
                    emitter.completeWithError(ex);
                }
            }
        }).start();
        
        return emitter;
    }
    
    /**
     * 清除会话
     */
    public void clearSession(String sessionId) {
        sessions.remove(sessionId);
    }
}
```

## 七、REST Controller

```java
// controller/ChatController.java
package com.example.chatbot.controller;

import com.example.chatbot.model.ChatRequest;
import com.example.chatbot.service.ChatService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.MediaType;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;

import java.util.Map;

@RestController
@RequestMapping("/api/chat")
public class ChatController {
    
    @Autowired
    private ChatService chatService;
    
    /**
     * 普通对话接口
     */
    @PostMapping
    public Map<String, Object> chat(@RequestBody ChatRequest request) {
        return chatService.chat(request.getSessionId(), request.getMessage());
    }
    
    /**
     * 流式对话接口（SSE - Server-Sent Events）
     * 浏览器可以用 EventSource 接收
     */
    @PostMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter chatStream(@RequestBody ChatRequest request) {
        return chatService.chatStream(request.getSessionId(), request.getMessage());
    }
    
    /**
     * 清除会话
     */
    @DeleteMapping("/session/{sessionId}")
    public Map<String, String> clearSession(@PathVariable String sessionId) {
        chatService.clearSession(sessionId);
        return Map.of("message", "会话已清除", "sessionId", sessionId);
    }
    
    /**
     * 健康检查
     */
    @GetMapping("/health")
    public Map<String, String> health() {
        return Map.of("status", "UP");
    }
}
```

## 八、前端聊天界面

在 `src/main/resources/static/index.html` 创建一个漂亮的聊天界面：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI 聊天助手</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { 
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            background: #1a1a2e; color: #eee; height: 100vh; display: flex; 
        }
        .sidebar {
            width: 260px; background: #16213e; padding: 20px;
            display: flex; flex-direction: column;
        }
        .sidebar h2 { margin-bottom: 20px; color: #0f3460; font-size: 18px; }
        .new-chat-btn {
            background: #0f3460; color: white; border: none; padding: 12px;
            border-radius: 8px; cursor: pointer; font-size: 14px; margin-bottom: 20px;
        }
        .new-chat-btn:hover { background: #1a4a7a; }
        .chat-container {
            flex: 1; display: flex; flex-direction: column; max-width: 900px;
            margin: 0 auto; padding: 20px;
        }
        .messages {
            flex: 1; overflow-y: auto; padding: 20px 0;
            display: flex; flex-direction: column; gap: 16px;
        }
        .message {
            display: flex; gap: 12px; max-width: 85%;
        }
        .message.user { align-self: flex-end; flex-direction: row-reverse; }
        .message.assistant { align-self: flex-start; }
        .avatar {
            width: 36px; height: 36px; border-radius: 50%;
            display: flex; align-items: center; justify-content: center;
            font-size: 16px; flex-shrink: 0;
        }
        .message.user .avatar { background: #0f3460; }
        .message.assistant .avatar { background: #533483; }
        .content {
            background: #16213e; padding: 12px 16px; border-radius: 12px;
            line-height: 1.6; white-space: pre-wrap; word-break: break-word;
        }
        .message.user .content { background: #0f3460; }
        .input-area {
            display: flex; gap: 12px; padding: 16px 0;
            border-top: 1px solid #16213e;
        }
        .input-area textarea {
            flex: 1; background: #16213e; border: 1px solid #0f3460;
            color: white; padding: 12px; border-radius: 8px; resize: none;
            font-family: inherit; font-size: 14px; outline: none;
        }
        .input-area textarea:focus { border-color: #533483; }
        .input-area button {
            background: #533483; color: white; border: none; padding: 0 24px;
            border-radius: 8px; cursor: pointer; font-size: 14px; font-weight: 600;
        }
        .input-area button:hover { background: #6b44a0; }
        .input-area button:disabled { opacity: 0.5; cursor: not-allowed; }
        .typing-indicator {
            display: flex; gap: 4px; padding: 8px 0;
        }
        .typing-indicator span {
            width: 8px; height: 8px; background: #533483; border-radius: 50%;
            animation: typing 1.4s infinite;
        }
        .typing-indicator span:nth-child(2) { animation-delay: 0.2s; }
        .typing-indicator span:nth-child(3) { animation-delay: 0.4s; }
        @keyframes typing {
            0%, 60%, 100% { opacity: 0.3; transform: translateY(0); }
            30% { opacity: 1; transform: translateY(-4px); }
        }
    </style>
</head>
<body>
    <div class="sidebar">
        <h2>AI 聊天助手</h2>
        <button class="new-chat-btn" onclick="newChat()">+ 新建对话</button>
        <div id="session-info" style="font-size:12px; color:#666; margin-top: auto;">
            Session: -
        </div>
    </div>
    <div class="chat-container">
        <div class="messages" id="messages">
            <div class="message assistant">
                <div class="avatar">AI</div>
                <div class="content">你好！我是 AI 聊天助手，有什么可以帮你的？</div>
            </div>
        </div>
        <div class="input-area">
            <textarea id="userInput" rows="1" placeholder="输入消息..."
                onkeydown="if(event.key==='Enter'&&!event.shiftKey){event.preventDefault();sendMessage()}"></textarea>
            <button id="sendBtn" onclick="sendMessage()">发送</button>
        </div>
    </div>

    <script>
        let sessionId = null;

        function newChat() {
            sessionId = null;
            document.getElementById('messages').innerHTML = `
                <div class="message assistant">
                    <div class="avatar">AI</div>
                    <div class="content">你好！我是 AI 聊天助手，有什么可以帮你的？</div>
                </div>`;
            document.getElementById('session-info').textContent = 'Session: -';
        }

        async function sendMessage() {
            const input = document.getElementById('userInput');
            const message = input.value.trim();
            if (!message) return;

            // 显示用户消息
            addMessage('user', message);
            input.value = '';
            document.getElementById('sendBtn').disabled = true;

            // 显示加载动画
            const loadingMsg = addMessage('assistant', '', true);

            try {
                const response = await fetch('/api/chat/stream', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ sessionId, message })
                });

                const reader = response.body.getReader();
                const decoder = new TextDecoder();
                let buffer = '';
                let aiContent = '';

                // 移除加载动画
                loadingMsg.querySelector('.content').innerHTML = '';

                while (true) {
                    const { done, value } = await reader.read();
                    if (done) break;
                    
                    buffer += decoder.decode(value, { stream: true });
                    const lines = buffer.split('\n');
                    buffer = lines.pop() || '';

                    for (const line of lines) {
                        if (line.startsWith('data:')) {
                            const data = line.slice(5).trim();
                            if (!data) continue;
                            
                            try {
                                const parsed = JSON.parse(data);
                                if (parsed.sessionId) {
                                    sessionId = parsed.sessionId;
                                    document.getElementById('session-info')
                                        .textContent = `Session: ${sessionId.substring(0, 8)}...`;
                                } else if (typeof parsed === 'string') {
                                    aiContent += parsed;
                                    loadingMsg.querySelector('.content').textContent = aiContent;
                                }
                            } catch(e) {
                                // 非 JSON 数据
                            }
                        } else if (line.startsWith('event:message')) {
                            continue; // sse event name, skip
                        }
                    }
                }
            } catch (error) {
                addMessage('assistant', '抱歉，发生了网络错误：' + error.message);
            } finally {
                document.getElementById('sendBtn').disabled = false;
            }
        }

        function addMessage(role, content, isLoading = false) {
            const msgDiv = document.createElement('div');
            msgDiv.className = `message ${role}`;
            msgDiv.innerHTML = `
                <div class="avatar">${role === 'user' ? '我' : 'AI'}</div>
                <div class="content">
                    ${isLoading ? '<div class="typing-indicator"><span></span><span></span><span></span></div>' : content}
                </div>`;
            document.getElementById('messages').appendChild(msgDiv);
            document.getElementById('messages').scrollTop = 
                document.getElementById('messages').scrollHeight;
            return msgDiv;
        }
    </script>
</body>
</html>
```

## 九、运行项目

```bash
# 1. 设置 API Key
export OPENAI_API_KEY="sk-your-key-here"

# 2. 启动
mvn spring-boot:run

# 3. 打开浏览器访问
# http://localhost:8080
```

你会看到一个暗黑风格的聊天界面，和 ChatGPT 长得差不多。输入消息，AI 会流式回复。

## 十、API 测试

```bash
# 普通对话
curl -X POST http://localhost:8080/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "用Java写一个单例模式"}'

# 流式对话
curl -X POST http://localhost:8080/api/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"sessionId":"test-123","message":"解释一下JVM内存模型"}' \
  --no-buffer

# 健康检查
curl http://localhost:8080/api/chat/health
```

## 十一、生产环境改进建议

当前版本是**最小可用产品（MVP）**，生产环境还需要：

```java
// 1. Redis 存储会话历史（当前是内存存储，重启就没了）
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, List<ChatCompletionMessageParam>> redisTemplate(
        RedisConnectionFactory factory
    ) {
        // 配置 Redis 存储会话
    }
}

// 2. 对话历史持久化到数据库（MySQL/PostgreSQL）
@Entity
@Table(name = "chat_history")
public class ChatMessage {
    @Id private String id;
    private String sessionId;
    private String role;      // user / assistant / system
    private String content;
    private LocalDateTime createdAt;
}

// 3. 用户认证和会话隔离
@PostMapping
public Map<String, Object> chat(
    @RequestBody ChatRequest request,
    @AuthenticationPrincipal UserDetails user  // 获取当前用户
) {
    return chatService.chat(user.getUsername(), request);
}

// 4. 流控（Rate Limiting）
// 用 Bucket4j 或 Guava RateLimiter 防止滥用

// 5. 多模型支持
// 支持切换 GPT-4o / Claude / 本地模型
```

## 十二、总结

| 模块 | 技术选型 | 代码行数 |
|------|---------|---------|
| Web 框架 | Spring Boot 3 | ~20 |
| AI SDK | openai-java | ~30 |
| 核心服务 | ChatService | ~120 |
| REST API | ChatController | ~30 |
| 前端界面 | HTML + CSS + JS | ~150 |
| 配置 | application.yml | ~10 |
| **总计** | | **~360** |

360 行代码，你拥有了一个功能完整的 AI 聊天机器人。接下来可以基于这个基础，继续添加 RAG（知识库）、Function Calling（工具调用）、多模态（图片识别）等功能。

---

**下篇预告**：普通聊天机器人只会"说话"，下一篇我们让 AI 长出手和脚——通过 Function Calling 让 AI 查询真实天气、调用数据库、发送邮件。给 AI 装上插件系统！
