# Ollama Java SDK 开发：自定义 Model 拉取与管理，用 Java 代码掌控本地模型的全生命周期

> 上一篇文章我们聊了 Ollama 的快速上手和 Spring Boot 集成。那层封装很好用，但如果你是做 AI 平台的，你需要的是直接用 Java 代码操控 Ollama 的每一寸细节——拉模型、删模型、看进度、切模型，全部程序化。

---

## 一、Ollama REST API 全解析

Ollama 的 REST API 极其简洁，所有接口都基于 `http://localhost:11434/api/`，返回 JSON 或流式 NDJSON。以下是核心接口一览：

| 接口 | 方法 | 用途 | 流式 |
|------|------|------|------|
| `/api/generate` | POST | 文本生成（补全模式） | ✅ |
| `/api/chat` | POST | 对话生成（Chat 模式） | ✅ |
| `/api/pull` | POST | 拉取模型 | ✅ |
| `/api/push` | POST | 推送模型到仓库 | ✅ |
| `/api/list` | GET | 列出本地模型 | ❌ |
| `/api/show` | POST | 查看模型详情 | ❌ |
| `/api/delete` | DELETE | 删除模型 | ❌ |
| `/api/copy` | POST | 复制模型 | ❌ |
| `/api/tags` | GET | 列出模型版本 | ❌ |

**/api/chat 请求体示例：**

```json
{
  "model": "qwen2.5-coder:7b",
  "messages": [
    {"role": "system", "content": "你是一个 Java 开发助手"},
    {"role": "user", "content": "写一个线程安全的单例模式"}
  ],
  "stream": true,
  "options": {
    "temperature": 0.7,
    "top_p": 0.9,
    "max_tokens": 2048
  }
}
```

**/api/pull 请求体：**

```json
{
  "name": "qwen2.5-coder:7b",
  "stream": true,
  "insecure": false
}
```

---

## 二、用 Java 封装 OllamaClient 工具类

下面我们实现一个完整的 `OllamaClient`，支持所有核心操作。不依赖 Spring，纯 JDK 11+ 即可运行。

### 2.1 基础 HTTP 客户端

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.Map;

public class OllamaClient {

    private final HttpClient httpClient;
    private final ObjectMapper objectMapper;
    private final String baseUrl;

    public OllamaClient(String baseUrl) {
        this.baseUrl = baseUrl;
        this.httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(30))
            .build();
        this.objectMapper = new ObjectMapper();
    }

    public OllamaClient() {
        this("http://localhost:11434");
    }

    // --- 模型列表 ---
    public ListResponse listModels() throws Exception {
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(baseUrl + "/api/tags"))
            .GET()
            .build();

        HttpResponse<String> response = httpClient.send(
            request, HttpResponse.BodyHandlers.ofString()
        );
        return objectMapper.readValue(response.body(), ListResponse.class);
    }

    // --- 模型详情 ---
    public ShowResponse showModel(String modelName) throws Exception {
        Map<String, String> body = Map.of("name", modelName);
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(baseUrl + "/api/show"))
            .POST(HttpRequest.BodyPublishers.ofString(
                objectMapper.writeValueAsString(body)))
            .header("Content-Type", "application/json")
            .build();

        HttpResponse<String> response = httpClient.send(
            request, HttpResponse.BodyHandlers.ofString()
        );
        return objectMapper.readValue(response.body(), ShowResponse.class);
    }

    // --- 删除模型 ---
    public void deleteModel(String modelName) throws Exception {
        Map<String, String> body = Map.of("name", modelName);
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(baseUrl + "/api/delete"))
            .method("DELETE", HttpRequest.BodyPublishers.ofString(
                objectMapper.writeValueAsString(body)))
            .header("Content-Type", "application/json")
            .build();

        httpClient.send(request, HttpResponse.BodyHandlers.discarding());
    }

    // --- 复制模型 ---
    public void copyModel(String source, String destination) throws Exception {
        Map<String, String> body = Map.of(
            "source", source,
            "destination", destination
        );
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(baseUrl + "/api/copy"))
            .POST(HttpRequest.BodyPublishers.ofString(
                objectMapper.writeValueAsString(body)))
            .header("Content-Type", "application/json")
            .build();

        httpClient.send(request, HttpResponse.BodyHandlers.discarding());
    }
}
```

### 2.2 同步 Chat 请求

```java
// 在 OllamaClient 类中补充
public ChatResponse chat(ChatRequest chatRequest) throws Exception {
    String json = objectMapper.writeValueAsString(chatRequest);
    HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create(baseUrl + "/api/chat"))
        .POST(HttpRequest.BodyPublishers.ofString(json))
        .header("Content-Type", "application/json")
        .build();

    HttpResponse<String> response = httpClient.send(
        request, HttpResponse.BodyHandlers.ofString()
    );
    return objectMapper.readValue(response.body(), ChatResponse.class);
}
```

### 2.3 同步 Generate 请求

```java
public GenerateResponse generate(String model, String prompt,
                                  Map<String, Object> options) throws Exception {
    Map<String, Object> body = new HashMap<>();
    body.put("model", model);
    body.put("prompt", prompt);
    body.put("stream", false);
    if (options != null) {
        body.put("options", options);
    }

    String json = objectMapper.writeValueAsString(body);
    HttpRequest request = HttpRequest.newBuilder()
        .uri(URI.create(baseUrl + "/api/generate"))
        .POST(HttpRequest.BodyPublishers.ofString(json))
        .header("Content-Type", "application/json")
        .build();

    HttpResponse<String> response = httpClient.send(
        request, HttpResponse.BodyHandlers.ofString()
    );
    return objectMapper.readValue(response.body(), GenerateResponse.class);
}
```

---

## 三、模型下载进度监控（SSE 流式读取）

Ollama 的 `/api/pull` 接口在 `stream: true` 时会持续推送 NDJSON 格式的进度数据。我们用 Java 的 `InputStream` 逐行读取实现进度监控。

### 3.1 流式拉取模型（带进度回调）

```java
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.util.function.Consumer;

public void pullModel(String modelName, Consumer<PullProgress> onProgress,
                       Consumer<String> onComplete, Consumer<Throwable> onError) {
    Map<String, Object> body = Map.of(
        "name", modelName,
        "stream", true
    );

    try {
        String json = objectMapper.writeValueAsString(body);
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(baseUrl + "/api/pull"))
            .POST(HttpRequest.BodyPublishers.ofString(json))
            .header("Content-Type", "application/json")
            .timeout(Duration.ofHours(2)) // 大模型下载可能很久
            .build();

        HttpResponse<InputStream> response = httpClient.send(
            request, HttpResponse.BodyHandlers.ofInputStream()
        );

        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(response.body()))) {
            String line;
            while ((line = reader.readLine()) != null) {
                if (line.isBlank()) continue;
                PullProgress progress = objectMapper
                    .readValue(line, PullProgress.class);
                onProgress.accept(progress);

                if ("success".equals(progress.getStatus())) {
                    onComplete.accept("模型 " + modelName + " 下载完成");
                    break;
                }
            }
        }
    } catch (Exception e) {
        onError.accept(e);
    }
}
```

### 3.2 进度数据模型

```java
import com.fasterxml.jackson.annotation.JsonProperty;

public class PullProgress {
    private String status;
    private String digest;

    @JsonProperty("total")
    private Long totalBytes;

    @JsonProperty("completed")
    private Long completedBytes;

    // getters & setters 省略

    public double getPercent() {
        if (totalBytes == null || totalBytes == 0) return 0;
        return completedBytes * 100.0 / totalBytes;
    }

    public String getReadableProgress() {
        return String.format("%.1f%% (%s / %s)",
            getPercent(),
            formatBytes(completedBytes),
            formatBytes(totalBytes));
    }

    private String formatBytes(Long bytes) {
        if (bytes == null) return "0 B";
        if (bytes < 1024) return bytes + " B";
        double kb = bytes / 1024.0;
        if (kb < 1024) return String.format("%.1f KB", kb);
        double mb = kb / 1024.0;
        if (mb < 1024) return String.format("%.1f MB", mb);
        return String.format("%.2f GB", mb / 1024.0);
    }
}
```

### 3.3 使用示例

```java
public static void main(String[] args) {
    OllamaClient client = new OllamaClient();

    System.out.println("开始下载模型...");
    client.pullModel("qwen2.5-coder:7b",
        progress -> {
            System.out.print("\r" + progress.getStatus()
                + " -> " + progress.getReadableProgress());
        },
        msg -> System.out.println("\n" + msg),
        err -> System.err.println("下载失败: " + err.getMessage())
    );
}
```

---

## 四、流式 Chat（Server-Sent Events 风格）

### 4.1 流式对话方法

```java
public void chatStream(ChatRequest chatRequest,
                       Consumer<String> onToken,
                       Consumer<Throwable> onError) {
    // 确保 stream 为 true
    chatRequest.setStream(true);

    try {
        String json = objectMapper.writeValueAsString(chatRequest);
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(baseUrl + "/api/chat"))
            .POST(HttpRequest.BodyPublishers.ofString(json))
            .header("Content-Type", "application/json")
            .build();

        HttpResponse<InputStream> response = httpClient.send(
            request, HttpResponse.BodyHandlers.ofInputStream()
        );

        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(response.body()))) {
            String line;
            while ((line = reader.readLine()) != null) {
                if (line.isBlank()) continue;

                StreamChunk chunk = objectMapper
                    .readValue(line, StreamChunk.class);

                if (chunk.getMessage() != null
                    && chunk.getMessage().getContent() != null) {
                    onToken.accept(chunk.getMessage().getContent());
                }

                if (chunk.getDone() != null && chunk.getDone()) {
                    break; // 流结束
                }
            }
        }
    } catch (Exception e) {
        onError.accept(e);
    }
}
```

### 4.2 使用 Spring MVC 的 SSE 返回

```java
@RestController
@RequestMapping("/api/ollama")
public class OllamaController {

    private final OllamaClient ollamaClient = new OllamaClient();

    @GetMapping(value = "/chat/stream", produces = "text/event-stream")
    public Flux<String> chatStream(@RequestParam String message) {
        return Flux.create(sink -> {
            ChatRequest req = new ChatRequest();
            req.setModel("qwen2.5-coder:7b");
            req.setMessages(List.of(
                new Message("user", message)
            ));

            ollamaClient.chatStream(req,
                token -> sink.next(token),
                error -> {
                    sink.error(error);
                    sink.complete();
                }
            );
        });
    }
}
```

---

## 五、模型管理后台（Spring Boot + 前端简单页面）

### 5.1 后端管理接口

```java
@RestController
@RequestMapping("/admin/models")
public class ModelManagementController {

    private final OllamaClient ollamaClient = new OllamaClient();

    // 列出所有模型
    @GetMapping
    public ResponseEntity<?> listModels() {
        try {
            return ResponseEntity.ok(ollamaClient.listModels());
        } catch (Exception e) {
            return ResponseEntity.status(500)
                .body(Map.of("error", e.getMessage()));
        }
    }

    // 查看模型详情
    @GetMapping("/{name}")
    public ResponseEntity<?> showModel(@PathVariable String name) {
        try {
            return ResponseEntity.ok(ollamaClient.showModel(name));
        } catch (Exception e) {
            return ResponseEntity.status(500)
                .body(Map.of("error", e.getMessage()));
        }
    }

    // 删除模型
    @DeleteMapping("/{name}")
    public ResponseEntity<?> deleteModel(@PathVariable String name) {
        try {
            ollamaClient.deleteModel(name);
            return ResponseEntity.ok(Map.of("success", true));
        } catch (Exception e) {
            return ResponseEntity.status(500)
                .body(Map.of("error", e.getMessage()));
        }
    }

    // 拉取模型（SSE 进度推送）
    @GetMapping("/pull/{name}")
    public SseEmitter pullModel(@PathVariable String name) {
        SseEmitter emitter = new SseEmitter(0L); // 不超时

        new Thread(() -> {
            try {
                ollamaClient.pullModel(name,
                    progress -> {
                        try {
                            emitter.send(SseEmitter.event()
                                .name("progress")
                                .data(progress));
                        } catch (IOException e) {
                            emitter.completeWithError(e);
                        }
                    },
                    msg -> {
                        try {
                            emitter.send(SseEmitter.event()
                                .name("complete")
                                .data(msg));
                            emitter.complete();
                        } catch (IOException e) {
                            emitter.completeWithError(e);
                        }
                    },
                    err -> emitter.completeWithError(err)
                );
            } catch (Exception e) {
                emitter.completeWithError(e);
            }
        }).start();

        return emitter;
    }
}
```

### 5.2 前端简单页面（HTML + SSE）

```html
<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <title>Ollama 模型管理</title>
    <style>
        body { font-family: system-ui; max-width: 800px; margin: 50px auto; }
        .model-card { border: 1px solid #ddd; padding: 15px; margin: 10px 0;
                      border-radius: 8px; }
        .progress-bar { background: #e0e0e0; height: 20px; border-radius: 10px;
                        overflow: hidden; }
        .progress-fill { background: #4caf50; height: 100%; width: 0%;
                         transition: width 0.3s; }
    </style>
</head>
<body>
    <h1>Ollama 模型管理中心</h1>

    <div>
        <input type="text" id="modelName"
               placeholder="输入模型名称，如 qwen2.5-coder:7b">
        <button onclick="pullModel()">拉取模型</button>
    </div>

    <div id="progress" style="margin-top: 20px; display:none;">
        <div class="progress-bar">
            <div class="progress-fill" id="progressFill"></div>
        </div>
        <p id="progressText"></p>
    </div>

    <h2>本地模型列表</h2>
    <div id="modelList"></div>

    <script>
        async function loadModels() {
            const res = await fetch('/admin/models');
            const data = await res.json();
            const list = document.getElementById('modelList');
            list.innerHTML = data.models.map(m => `
                <div class="model-card">
                    <strong>${m.name}</strong>
                    <span>${(m.size / 1024 / 1024 / 1024).toFixed(2)} GB</span>
                    <button onclick="deleteModel('${m.name}')">删除</button>
                </div>
            `).join('');
        }

        function pullModel() {
            const name = document.getElementById('modelName').value;
            document.getElementById('progress').style.display = 'block';

            const eventSource = new EventSource(`/admin/models/pull/${name}`);

            eventSource.addEventListener('progress', (e) => {
                const data = JSON.parse(e.data);
                document.getElementById('progressFill').style.width
                    = data.percent + '%';
                document.getElementById('progressText').innerText
                    = data.readableProgress;
            });

            eventSource.addEventListener('complete', () => {
                eventSource.close();
                loadModels();
            });
        }

        async function deleteModel(name) {
            await fetch(`/admin/models/${name}`, { method: 'DELETE' });
            loadModels();
        }

        loadModels();
    </script>
</body>
</html>
```

---

## 六、多模型切换策略

在复杂业务场景中，不同任务需要调用不同模型。例如：代码生成用 DeepSeek-Coder，文档摘要用 Qwen，翻译用 Llama。我们需要一套灵活的多模型切换机制。

### 6.1 模型注册表

```java
@Component
public class ModelRegistry {

    private final Map<String, ModelConfig> modelMap = new ConcurrentHashMap<>();

    @PostConstruct
    public void init() {
        // 从配置文件或数据库加载
        modelMap.put("code", new ModelConfig("deepseek-coder-v2:16b", 0.3));
        modelMap.put("chat", new ModelConfig("qwen2.5:14b", 0.7));
        modelMap.put("translate", new ModelConfig("llama3.1:8b", 0.2));
        modelMap.put("summary", new ModelConfig("qwen2.5:7b", 0.5));
    }

    public ModelConfig getConfig(String taskType) {
        return modelMap.getOrDefault(taskType,
            modelMap.get("chat")); // 默认模型
    }

    public void register(String taskType, ModelConfig config) {
        modelMap.put(taskType, config);
    }

    public record ModelConfig(String modelName, double temperature) {}
}
```

### 6.2 路由服务

```java
@Service
public class ModelRoutingService {

    private final OllamaClient ollamaClient;
    private final ModelRegistry registry;

    public ModelRoutingService(OllamaClient ollamaClient,
                                ModelRegistry registry) {
        this.ollamaClient = ollamaClient;
        this.registry = registry;
    }

    public String executeForTask(String taskType, String userMessage) {
        ModelRegistry.ModelConfig config = registry.getConfig(taskType);

        ChatRequest request = new ChatRequest();
        request.setModel(config.modelName());
        request.setMessages(List.of(new Message("user", userMessage)));

        Map<String, Object> options = new HashMap<>();
        options.put("temperature", config.temperature());
        request.setOptions(options);

        return ollamaClient.chatSync(request)
            .getMessage().getContent();
    }
}
```

### 6.3 使用示例

```java
// 生成代码
String code = routingService.executeForTask("code",
    "用 Java 实现一个线程池");

// 文档摘要
String summary = routingService.executeForTask("summary",
    documentContent);

// 翻译
String english = routingService.executeForTask("translate",
    "请将以下内容翻译成英文：" + chineseText);
```

---

## 七、总结

本文我们从零实现了一个完整的 Ollama Java SDK，涵盖：

- **REST API 全封装**：generate / chat / pull / list / show / delete / copy
- **流式进度监控**：基于 SSE/NDJSON 的模型下载进度实时推送
- **模型管理后台**：Spring Boot + SseEmitter 实现的前后端联动
- **多模型切换**：任务类型路由 + 模型注册表

这套 SDK 可以作为一个独立的 Maven 库发布，团队中任何 Java 项目直接引入即可调用本地大模型，完全摆脱对 Spring AI 等上层框架的依赖，让你对模型的生命周期有 100% 的控制力。

---

**下篇预告**：《Open WebUI + Ollama：搭建企业内部 ChatGPT 替代平台，2 小时上线公司专属 AI 门户》——如何用 Docker Compose 一键部署带用户管理、权限控制、公司 AD/LDAP 对接的 WebUI，再配合 Java 后端打造企业级 AI 平台。敬请期待！
