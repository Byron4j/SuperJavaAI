# 第 14 章 · MCP (Model Context Protocol) 协议详解

---

> Anthropic 于 2024 年 11 月开源的 MCP，被誉为 **"AI 的 USB-C 接口"**——用统一协议连接 LLM 和外部数据/工具。

---

## 14.1 痛点：工具调用的"意大利面条式集成"

第 13 章的 Function Calling 解决了"让模型调用工具"，但留下了治理问题：

```
每个 LLM App 都在重复造轮子:
 ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
 │ 聊天 App A  │    │ 代码助手 B  │    │ 数据分析 C   │
 ├─────────────┤    ├─────────────┤    ├─────────────┤
 │ 自写天气API  │    │ 自写天气API  │    │ 自写天气API  │  ← 重复！
 │ 自写数据库    │    │ 自写数据库    │    │ 自写搜索API  │
 │ 自写搜索     │    │ 自写Git操作  │    │ 自写文件系统  │
 └─────────────┘    └─────────────┘    └─────────────┘

MCP 之后:
 ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
 │ 聊天 App A  │   │ 代码助手 B  │   │ 数据分析 C   │
 └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
        └──────────────────┼──────────────────┘
                           │ MCP 协议
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         天气服务器    数据库服务器   文件服务器
         (MCP Server) (MCP Server) (MCP Server)
```

---

## 14.2 MCP 架构

```
┌─────────────────────────────────────────────────────────────┐
│                     MCP 架构图                              │
│                                                             │
│  ┌──────────────┐      JSON-RPC 2.0        ┌──────────────┐│
│  │              │ ◄──────────────────────► │              ││
│  │  MCP Client  │   (stdio / HTTP SSE)     │  MCP Server  ││
│  │  (Host App)  │                          │  (Tool)      ││
│  │              │                          │              ││
│  │  ┌────────┐  │                          │  ┌────────┐  ││
│  │  │ LLM    │  │                          │  │Tools   │  ││
│  │  │(Claude)│  │                          │  │Resources│  ││
│  │  └────────┘  │                          │  │Prompts │  ││
│  └──────────────┘                          │  └────────┘  ││
│                                            └──────────────┘│
│                                                             │
│  MCP Client (Host) = LLM 应用 (Claude Desktop, IDE, etc.)   │
│  MCP Server       = 提供工具/资源/提示的进程                  │
│                                                             │
│  协议: JSON-RPC 2.0                                          │
│  传输: stdio (推荐) 或 HTTP + SSE (远程)                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 14.3 Java 实现 MCP Server

```java
/**
 * MCP Server —— 提供"天气查询"工具的服务器
 * 
 * 使用 spring-ai-mcp 或直接基于 JSON-RPC 实现
 */
@Component
public class WeatherMcpServer {

    private final McpServer server;

    public WeatherMcpServer() {
        this.server = McpServer.builder()
            .name("weather-server")
            .version("1.0.0")
            .build();
    }

    @PostConstruct
    public void init() {
        // ===== 注册 Tool =====
        server.addTool(Tool.builder()
            .name("get_weather")
            .description("获取指定城市的当前天气信息")
            .inputSchema(JsonSchema.builder()
                .property("city", JsonSchemaType.STRING, "城市名称")
                .property("unit", JsonSchemaType.STRING, "温度单位: celsius/fahrenheit")
                .required("city")
                .build())
            .handler(this::handleWeather)
            .build());

        // ===== 注册 Resource =====
        server.addResource(Resource.builder()
            .uri("weather://forecast/beijing")
            .name("北京天气预报")
            .mimeType("application/json")
            .handler(uri -> fetchForecast("Beijing"))
            .build());

        // ===== 注册 Prompt Template =====
        server.addPrompt(Prompt.builder()
            .name("compare_weather")
            .description("比较两个城市的天气")
            .argument("city1", "第一个城市")
            .argument("city2", "第二个城市")
            .template("""
                比较以下两个城市的天气情况:
                1. {{city1}}
                2. {{city2}}
                
                请从温度、湿度、天气状况三个方面进行对比。
                """)
            .build());
    }

    /**
     * Tool 处理函数
     * 
     * 输入: Tool Call Request (JSON)
     * 输出: Tool Call Result (JSON)
     */
    private ToolResult handleWeather(Map<String, Object> args) {
        String city = (String) args.get("city");
        String unit = (String) args.getOrDefault("unit", "celsius");

        // 真实的天气 API 调用
        WeatherData data = weatherService.getCurrent(city, unit);

        return ToolResult.builder()
            .content(Content.text(formatWeather(data)))
            .build();
    }

    /**
     * 启动 MCP Server（stdio 传输）
     */
    public void startStdio() {
        server.start(new StdioServerTransport());
    }

    /**
     * 启动 MCP Server（HTTP SSE 传输，用于远程访问）
     */
    public void startHttp(int port) {
        server.start(new HttpSseServerTransport(port));
    }
}
```

---

## 14.4 Java 实现 MCP Client

```java
/**
 * MCP Client —— 在 Spring Boot 应用中连接 MCP Server
 */
@Service
public class McpClientService {

    private final McpClient client;

    public McpClientService(@Value("${mcp.server.command:npx}") String command,
                            @Value("${mcp.server.args}") List<String> args) {
        // 通过 stdio 连接本地 MCP Server
        McpServerConfig config = McpServerConfig.builder()
            .command(command)
            .args(args)
            .build();

        this.client = McpClient.sync(config);
        this.client.initialize();
    }

    /**
     * 列出服务器上的所有可用工具
     */
    public List<ToolDefinition> listTools() {
        ListToolsResult result = client.listTools();
        return result.tools();
    }

    /**
     * 调用 MCP 工具
     */
    public ToolResult callTool(String toolName, Map<String, Object> args) {
        CallToolResult result = client.callTool(
            new CallToolRequest(toolName, args)
        );
        return result;
    }

    /**
     * 列出服务器资源
     */
    public List<ResourceDefinition> listResources() {
        return client.listResources().resources();
    }

    /**
     * 读取资源内容
     */
    public ResourceContent readResource(String uri) {
        return client.readResource(new ReadResourceRequest(uri));
    }

    /**
     * 获取提示模板列表
     */
    public List<PromptDefinition> listPrompts() {
        return client.listPrompts().prompts();
    }
}
```

---

## 14.5 MCP 的三个核心原语

| 原语 | 作用 | Java 类比 | 示例 |
|---|---|---|---|
| **Tool** | 让 LLM 执行的函数 | `@PostMapping` Controller 方法 | `get_weather`, `search_docs` |
| **Resource** | LLM 可读取的数据 | `GET /api/resource` REST 接口 | 文件内容、数据库记录、API 数据 |
| **Prompt** | 预定义的提示模板 | 参数化的 `String.format()` | "用{{language}}写一个{{algorithm}}" |

### 14.5.1 Tools

```java
// MCP Tool 的 JSON-RPC 通信格式

// → 请求
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
        "name": "get_weather",
        "arguments": {
            "city": "Beijing",
            "unit": "celsius"
        }
    }
}

// ← 响应
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            {
                "type": "text",
                "text": "北京当前: 晴, 25°C, 湿度 45%"
            }
        ]
    }
}
```

### 14.5.2 Resources

```java
// Resources 用 URI 标识，类似 RESTful API
// weather://current/beijing
// file:///home/user/document.txt
// postgres://database/table/users

// LLM 可以通过 resource/list 发现可用资源
// 通过 resource/read 读取具体内容
```

### 14.5.3 Prompts

```java
// Prompts 是预定义的会话模板，可参数化
// LLM 通过 prompt/list 发现，通过 prompt/get 获取填充后的 prompt

// 定义: "用{{language}}解释{{concept}}"
// 调用: prompt/get("explain", {language: "Java", concept: "MCP"})
// 返回: "用Java解释MCP"
```

---

## 14.6 MCP 传输层

```java
/**
 * MCP 支持两种传输方式
 */
public class McpTransportComparison {

    // ===== 方式 1: stdio（推荐，用于本地进程）=====
    // Client 通过 stdin/stdout 与 Server 进程通信
    McpClient client = McpClient.sync(McpServerConfig.builder()
        .command("npx")
        .args("-y", "@anthropic/mcp-server-weather")
        .build());

    // 优势: 零配置、标准 Unix 风格、安全（无网络暴露）
    // 场景: Claude Desktop、本地 IDE 插件

    // ===== 方式 2: HTTP + SSE（用于远程服务）=====
    // Server 监听 HTTP，Client 通过 HTTP POST 发请求
    // Server 通过 Server-Sent Events 推送通知
    McpClient remoteClient = McpClient.sync(McpServerConfig.builder()
        .url("https://mcp.example.com/weather-server")
        .build());

    // 优势: 远程访问、负载均衡、标准 HTTP 基础设施
    // 场景: 企业内部 MCP 服务、SaaS MCP 提供者
}
```

---

## 14.7 Spring AI 集成 MCP

Spring AI 1.0+ 对 MCP 提供了原生支持：

```java
@SpringBootApplication
public class McpApplication {

    public static void main(String[] args) {
        SpringApplication.run(McpApplication.class, args);
    }

    @Bean
    public ToolCallbackProvider weatherTools(McpClient mcpClient) {
        // 将 MCP Server 的工具自动注册为 Spring AI 的 Tool
        return McpToolCallbackProvider.builder()
            .mcpClient(mcpClient)
            .build();
    }

    @Bean
    public McpSyncClient mcpClient() {
        return McpClient.sync(McpServerConfig.builder()
            .command("java")
            .args("-jar", "weather-mcp-server.jar")
            .build());
    }
}

// 使用时和普通 Function Calling 完全一样:
@RestController
public class ChatController {
    private final ChatClient chatClient;

    @PostMapping("/chat")
    public String chat(@RequestBody String message) {
        return chatClient.prompt()
            .user(message)
            .tools(toolCallbackProvider)  // MCP 工具自动注入
            .call()
            .content();
    }
}
```

---

## 14.8 MCP 生态

| 类型 | 示例 |
|---|---|
| **官方 Server** | Filesystem, GitHub, PostgreSQL, Slack, Google Drive |
| **社区 Server** | Jira, Confluence, Notion, Linear, Figma |
| **企业 Server** | SAP, Salesforce, 内部 API 网关 |
| **Client** | Claude Desktop, Continue.dev, Zed, VS Code 插件 |

**MCP 正在成为 AI 工具互联的标准协议**——类似于 HTTP 成为 Web 的标准协议。

---

## 14.9 MCP vs 传统 Function Calling

| 维度 | 传统 Function Calling | MCP |
|---|---|---|
| **定义方式** | 每个 App 自己定义工具 | 统一协议，Server 一次性定义 |
| **发现机制** | 手动注册 | 自动发现 (`list_tools`) |
| **数据访问** | 无标准方式 | Resource 原语 (URI 寻址) |
| **Prompt 复用** | 每次手写 | Prompt 模板 (参数化) |
| **多应用共享** | 需重复集成 | 一个 Server 服务多个 Client |
| **传输** | 进程内函数调用 | stdio / HTTP (跨进程、跨网络) |
| **安全性** | App 自己负责 | 传输隔离，权限在 Server 端 |

---

> **下一章**：[Skills 机制详解](18-skills-mechanism.md)
