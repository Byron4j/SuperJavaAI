# API 文档 RAG：自动回答"这个接口怎么用"的内部助手，从此前后端联调不再吵架

> "你这个接口返回的 `data` 到底是数组还是对象？""为什么我传了 `userId` 还是报 400？""v2 版本的这个接口和 v1 有什么变化？"——这些对话，每天在前端和后端之间上演无数次。如果有一个 AI 助手能自动回答这些问题呢？

## 一、前后端联调的"永恒痛点"

来看一个经典的联调对话：

```
前端：这个 /api/order/create 接口，我传了 userId 还是报 400？
后端：你看文档啊，参数名是 user_id，下划线！
前端：文档里写的不是 userId 吗？
后端：你看的是 v1 文档吧，v2 改了……
前端：那你倒是更新文档啊！
后端：我代码都写了，文档晚点更新……
```

这种对话的根源在于：API 文档和代码不同步、文档分散在多个地方、查文档效率低。**API 文档 RAG** 就是解决这个问题的。

## 二、API 文档 RAG 的系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     API 文档 RAG 系统架构                         │
│                                                                 │
│  ┌─────────────────┐    ┌──────────────────┐    ┌────────────┐ │
│  │ Swagger/OpenAPI │───▶│  文档解析引擎     │───▶│ 向量数据库  │ │
│  │    文档导入      │    │ - 结构化解析      │    │            │ │
│  └─────────────────┘    │ - 参数提取        │    └─────┬──────┘ │
│                         │ - 响应体拆解      │          │        │
│  ┌─────────────────┐    │ - 多版本管理      │          │        │
│  │   手写 Markdown  │───▶│                  │          │        │
│  │     文档导入      │    └──────────────────┘          │        │
│  └─────────────────┘                                   │        │
│                                                        │        │
│  ┌─────────────────┐                          ┌────────▼──────┐ │
│  │  用户提问        │──────▶  Chat 界面  ◀─────│  LLM 生成回答  │ │
│  │ "创建订单接口    │                          │               │ │
│  │  需要哪些参数？" │                          └───────────────┘ │
│  └─────────────────┘                                            │
└─────────────────────────────────────────────────────────────────┘
```

## 三、OpenAPI/Swagger 文档的向量化策略

### 3.1 为什么不能直接把整个 JSON 扔进去？

OpenAPI 文档是一个巨大的 JSON/YAML，直接 Embedding 会导致：
- 关键信息被稀释（真正的接口参数淹没在 paths/components/servers 中）
- 版本差异无法区分
- 查询"xx 接口的参数"时，检索到的是不相干的 schema 定义

**正确做法**：按接口 + 参数拆解为多个 chunk。

### 3.2 Swagger 文档解析器

```java
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;

import java.util.*;
import java.util.stream.Collectors;

public class OpenAPIParser {

    private static final ObjectMapper JSON_MAPPER = new ObjectMapper();
    private static final ObjectMapper YAML_MAPPER = new ObjectMapper(new YAMLFactory());

    public record ApiEndpoint(
        String path,                    // /api/user/{id}
        String method,                  // GET / POST / PUT / DELETE
        String summary,                 // 接口摘要
        String description,             // 详细描述
        String tag,                     // 分组标签
        List<ApiParameter> parameters,  // 参数列表
        ApiResponseBody requestBody,    // 请求体
        Map<String, ApiResponseBody> responses,  // 响应体
        String version,                 // API 版本
        List<String> deprecated         // 废弃标记
    ) {}

    public record ApiParameter(
        String name,        // 参数名
        String in,          // query / path / header / cookie
        boolean required,   // 是否必填
        String type,        // string / integer / boolean
        String description, // 参数描述
        String example,     // 示例值
        String defaultValue,// 默认值
        List<String> enumValues // 可选值
    ) {}

    public record ApiResponseBody(
        String contentType, // application/json
        String schema,      // JSON Schema 结构
        String description, // 描述
        String example      // 响应示例
    ) {}

    /**
     * 解析 OpenAPI 3.x 文档
     */
    public List<ApiEndpoint> parse(byte[] content, String fileType) throws Exception {
        JsonNode root = fileType.equals("yaml") || fileType.equals("yml")
            ? YAML_MAPPER.readTree(content)
            : JSON_MAPPER.readTree(content);

        String apiVersion = root.path("info").path("version").asText();
        String apiTitle = root.path("info").path("title").asText();

        List<ApiEndpoint> endpoints = new ArrayList<>();
        JsonNode paths = root.path("paths");

        Iterator<String> pathNames = paths.fieldNames();
        while (pathNames.hasNext()) {
            String path = pathNames.next();
            JsonNode pathItem = paths.path(path);

            // 遍历每个 HTTP 方法
            for (String method : List.of("get", "post", "put", "delete", "patch")) {
                JsonNode operation = pathItem.path(method);
                if (operation.isMissingNode()) continue;

                endpoints.add(extractEndpoint(path, method.toUpperCase(),
                    operation, apiVersion));
            }
        }

        System.out.printf("解析完成：%s v%s，共 %d 个接口\n",
            apiTitle, apiVersion, endpoints.size());
        return endpoints;
    }

    private ApiEndpoint extractEndpoint(String path, String method,
                                         JsonNode operation, String version) {
        // 解析参数
        List<ApiParameter> parameters = new ArrayList<>();
        JsonNode paramsNode = operation.path("parameters");
        if (paramsNode.isArray()) {
            for (JsonNode param : paramsNode) {
                parameters.add(new ApiParameter(
                    param.path("name").asText(),
                    param.path("in").asText(),
                    param.path("required").asBoolean(false),
                    param.path("schema").path("type").asText("string"),
                    param.path("description").asText(""),
                    param.path("example").asText(""),
                    param.path("schema").path("default").asText(""),
                    extractEnumValues(param.path("schema").path("enum"))
                ));
            }
        }

        // 解析请求体
        ApiResponseBody requestBody = null;
        JsonNode reqBody = operation.path("requestBody");
        if (!reqBody.isMissingNode()) {
            JsonNode content = reqBody.path("content").path("application/json");
            requestBody = new ApiResponseBody(
                "application/json",
                content.path("schema").toString(),
                reqBody.path("description").asText(""),
                content.path("example").toString()
            );
        }

        // 解析响应体
        Map<String, ApiResponseBody> responses = new LinkedHashMap<>();
        JsonNode resps = operation.path("responses");
        Iterator<String> statusCodes = resps.fieldNames();
        while (statusCodes.hasNext()) {
            String code = statusCodes.next();
            JsonNode resp = resps.path(code);
            JsonNode respContent = resp.path("content").path("application/json");
            responses.put(code, new ApiResponseBody(
                "application/json",
                respContent.path("schema").toString(),
                resp.path("description").asText(""),
                respContent.path("example").toString()
            ));
        }

        return new ApiEndpoint(
            path,
            method,
            operation.path("summary").asText(""),
            operation.path("description").asText(""),
            operation.path("tags").get(0).asText(""),
            parameters,
            requestBody,
            responses,
            version,
            operation.has("deprecated") ? List.of("deprecated") : List.of()
        );
    }

    private List<String> extractEnumValues(JsonNode enumNode) {
        if (enumNode.isArray()) {
            List<String> values = new ArrayList<>();
            for (JsonNode val : enumNode) {
                values.add(val.asText());
            }
            return values;
        }
        return List.of();
    }
}
```

### 3.3 接口参数和返回值的结构化解析

解析出接口后，我们需要将每个接口拆成多个有语义的 chunk：

```java
public class ApiDocumentChunker {

    public record ApiChunk(
        String chunkId,
        String chunkType,       // OVERVIEW / PARAMETER / REQUEST_BODY / RESPONSE / EXAMPLE
        String textContent,     // 自然语言描述（用于 Embedding）
        String structuredData,  // 结构化数据（JSON，供 LLM 精确解析）
        Map<String, Object> metadata
    ) {}

    /**
     * 将一个接口拆解为多个语义 Chunk
     */
    public List<ApiChunk> chunkEndpoint(OpenAPIParser.ApiEndpoint endpoint) {
        List<ApiChunk> chunks = new ArrayList<>();
        String baseId = endpoint.method() + " " + endpoint.path();

        // 1. 接口概览 Chunk（高维度概览，用于模糊搜索）
        chunks.add(new ApiChunk(
            baseId + "#overview",
            "OVERVIEW",
            buildOverviewText(endpoint),
            buildOverviewJson(endpoint),
            Map.of("path", endpoint.path(), "method", endpoint.method(),
                "version", endpoint.version(), "tag", endpoint.tag())
        ));

        // 2. 参数 Chunk（每个参数一个 Chunk）
        for (int i = 0; i < endpoint.parameters().size(); i++) {
            OpenAPIParser.ApiParameter param = endpoint.parameters().get(i);
            chunks.add(new ApiChunk(
                baseId + "#param-" + i,
                "PARAMETER",
                buildParameterText(endpoint, param),
                buildParameterJson(param),
                Map.of("path", endpoint.path(), "method", endpoint.method(),
                    "paramName", param.name(), "version", endpoint.version())
            ));
        }

        // 3. 请求体 Chunk
        if (endpoint.requestBody() != null) {
            chunks.add(new ApiChunk(
                baseId + "#request",
                "REQUEST_BODY",
                buildRequestBodyText(endpoint),
                buildRequestBodyJson(endpoint),
                Map.of("path", endpoint.path(), "method", endpoint.method(),
                    "version", endpoint.version())
            ));
        }

        // 4. 响应体 Chunk（每个状态码一个 Chunk）
        for (var entry : endpoint.responses().entrySet()) {
            String statusCode = entry.getKey();
            OpenAPIParser.ApiResponseBody resp = entry.getValue();
            chunks.add(new ApiChunk(
                baseId + "#response-" + statusCode,
                "RESPONSE",
                buildResponseText(endpoint, statusCode, resp),
                buildResponseJson(statusCode, resp),
                Map.of("path", endpoint.path(), "method", endpoint.method(),
                    "statusCode", statusCode, "version", endpoint.version())
            ));
        }

        return chunks;
    }

    // --- 构建自然语言文本（用于 Embedding）---

    private String buildOverviewText(OpenAPIParser.ApiEndpoint ep) {
        StringBuilder sb = new StringBuilder();
        sb.append("API 接口: ").append(ep.method()).append(" ").append(ep.path()).append("\n");
        sb.append("分组: ").append(ep.tag()).append("\n");
        sb.append("版本: ").append(ep.version()).append("\n");
        if (!ep.summary().isEmpty()) {
            sb.append("摘要: ").append(ep.summary()).append("\n");
        }
        if (!ep.description().isEmpty()) {
            sb.append("详细描述: ").append(ep.description()).append("\n");
        }
        sb.append("参数个数: ").append(ep.parameters().size()).append("\n");
        // 列出参数名称，增强可检索性
        sb.append("参数列表: ");
        for (var p : ep.parameters()) {
            sb.append(p.name()).append("(").append(p.required() ? "必填" : "可选")
                .append(", ").append(p.type()).append(") ");
        }
        return sb.toString();
    }

    private String buildParameterText(OpenAPIParser.ApiEndpoint ep,
                                       OpenAPIParser.ApiParameter param) {
        StringBuilder sb = new StringBuilder();
        sb.append("接口 ").append(ep.method()).append(" ").append(ep.path())
            .append(" 的参数 ").append(param.name()).append("\n");
        sb.append("位置: ").append(param.in()).append("\n");
        sb.append("类型: ").append(param.type()).append("\n");
        sb.append("是否必填: ").append(param.required() ? "是" : "否").append("\n");
        if (!param.description().isEmpty()) {
            sb.append("描述: ").append(param.description()).append("\n");
        }
        if (!param.example().isEmpty()) {
            sb.append("示例值: ").append(param.example()).append("\n");
        }
        if (!param.defaultValue().isEmpty()) {
            sb.append("默认值: ").append(param.defaultValue()).append("\n");
        }
        if (!param.enumValues().isEmpty()) {
            sb.append("可选值: ").append(String.join(", ", param.enumValues())).append("\n");
        }
        return sb.toString();
    }

    private String buildRequestBodyText(OpenAPIParser.ApiEndpoint ep) {
        StringBuilder sb = new StringBuilder();
        sb.append("接口 ").append(ep.method()).append(" ").append(ep.path())
            .append(" 的请求体\n");
        sb.append("Content-Type: ").append(ep.requestBody().contentType()).append("\n");
        if (!ep.requestBody().description().isEmpty()) {
            sb.append("描述: ").append(ep.requestBody().description()).append("\n");
        }
        // 将 JSON Schema 转化为自然语言描述
        sb.append("Schema 结构: ").append(schemaToPlainText(ep.requestBody().schema())).append("\n");
        return sb.toString();
    }

    private String buildResponseText(OpenAPIParser.ApiEndpoint ep,
                                      String statusCode,
                                      OpenAPIParser.ApiResponseBody resp) {
        StringBuilder sb = new StringBuilder();
        sb.append("接口 ").append(ep.method()).append(" ").append(ep.path())
            .append(" 的 ").append(statusCode).append(" 响应\n");
        // HTTP 状态码友好描述
        String statusDesc = switch (statusCode) {
            case "200" -> "请求成功";
            case "201" -> "创建成功";
            case "204" -> "无内容";
            case "400" -> "请求参数错误";
            case "401" -> "未认证";
            case "403" -> "无权限";
            case "404" -> "资源不存在";
            case "500" -> "服务器内部错误";
            default -> statusCode;
        };
        sb.append("状态: ").append(statusDesc).append("\n");
        if (!resp.description().isEmpty()) {
            sb.append("描述: ").append(resp.description()).append("\n");
        }
        sb.append("响应结构: ").append(schemaToPlainText(resp.schema())).append("\n");
        return sb.toString();
    }

    // --- 构建结构化数据（供 LLM 精确解析）---

    private String buildOverviewJson(OpenAPIParser.ApiEndpoint ep) {
        return String.format("""
            {"path":"%s","method":"%s","summary":"%s","tag":"%s","version":"%s",
            "paramCount":%d}
            """, ep.path(), ep.method(), ep.summary(), ep.tag(),
            ep.version(), ep.parameters().size());
    }

    private String buildParameterJson(OpenAPIParser.ApiParameter param) {
        return String.format("""
            {"name":"%s","in":"%s","required":%b,"type":"%s","description":"%s",
            "example":"%s","defaultValue":"%s"}
            """, param.name(), param.in(), param.required(), param.type(),
            param.description(), param.example(), param.defaultValue());
    }

    private String buildRequestBodyJson(OpenAPIParser.ApiEndpoint ep) {
        return String.format("""
            {"contentType":"%s","description":"%s","schema":%s}
            """, ep.requestBody().contentType(),
            ep.requestBody().description(), ep.requestBody().schema());
    }

    private String buildResponseJson(String statusCode,
                                      OpenAPIParser.ApiResponseBody resp) {
        return String.format("""
            {"statusCode":"%s","contentType":"%s","description":"%s","schema":%s}
            """, statusCode, resp.contentType(), resp.description(), resp.schema());
    }

    /**
     * 将 JSON Schema 转换为自然语言描述
     */
    private String schemaToPlainText(String schemaJson) {
        try {
            JsonNode schema = JSON_MAPPER.readTree(schemaJson);
            return jsonSchemaToText(schema, "");
        } catch (Exception e) {
            return schemaJson;
        }
    }

    private String jsonSchemaToText(JsonNode node, String indent) {
        if (node.isMissingNode() || node.isNull()) return "无";

        String type = node.path("type").asText();
        if (type.equals("object")) {
            StringBuilder sb = new StringBuilder("对象 {");
            JsonNode props = node.path("properties");
            if (props.isObject()) {
                Iterator<String> fieldNames = props.fieldNames();
                List<String> fields = new ArrayList<>();
                while (fieldNames.hasNext()) {
                    String name = fieldNames.next();
                    JsonNode prop = props.path(name);
                    String propType = prop.path("type").asText("unknown");
                    String desc = prop.path("description").asText("");
                    fields.add(name + ": " + propType
                        + (desc.isEmpty() ? "" : "(" + desc + ")"));
                }
                sb.append(String.join(", ", fields));
            }
            sb.append("}");
            return sb.toString();
        } else if (type.equals("array")) {
            JsonNode items = node.path("items");
            return "数组[" + jsonSchemaToText(items, indent) + "]";
        } else if (type.equals("string")) {
            return "字符串";
        } else if (type.equals("integer") || type.equals("number")) {
            return "数字";
        } else if (type.equals("boolean")) {
            return "布尔值";
        }
        return type;
    }

    private static final ObjectMapper JSON_MAPPER = new ObjectMapper();
}
```

## 四、多版本 API 文档管理

### 4.1 版本隔离策略

```java
public class MultiVersionApiManager {

    private final Map<String, ApiVersionRegistry> registries = new ConcurrentHashMap<>();
    private final VectorStore vectorStore;

    public MultiVersionApiManager(VectorStore vectorStore) {
        this.vectorStore = vectorStore;
    }

    /**
     * 注册一个服务及其所有版本的 API 文档
     */
    public void registerService(String serviceName, List<ApiDocVersion> versions) {
        ApiVersionRegistry registry = new ApiVersionRegistry(serviceName);
        for (ApiDocVersion version : versions) {
            registry.addVersion(version);
            // 索引时在元数据中标记版本号
            indexApiDocuments(serviceName, version);
        }
        registries.put(serviceName, registry);
    }

    public record ApiDocVersion(
        String version,     // v1.0.0
        byte[] content,     // OpenAPI 文档内容
        String format,      // json / yaml
        String changelog    // 版本变更说明
    ) {}

    private void indexApiDocuments(String serviceName, ApiDocVersion version) {
        try {
            OpenAPIParser parser = new OpenAPIParser();
            List<OpenAPIParser.ApiEndpoint> endpoints =
                parser.parse(version.content(), version.format());

            ApiDocumentChunker chunker = new ApiDocumentChunker();
            List<VectorDocument> documents = new ArrayList<>();

            for (var endpoint : endpoints) {
                List<ApiDocumentChunker.ApiChunk> chunks = chunker.chunkEndpoint(endpoint);
                for (var chunk : chunks) {
                    Map<String, Object> metadata = new HashMap<>(chunk.metadata());
                    metadata.put("serviceName", serviceName);
                    metadata.put("apiVersion", version.version());
                    metadata.put("chunkType", chunk.chunkType());

                    documents.add(new VectorDocument(
                        serviceName + ":" + version.version() + ":" + chunk.chunkId(),
                        null, // Embedding 稍后批量生成
                        chunk.textContent() + "\n---\n" + chunk.structuredData(),
                        metadata
                    ));
                }
            }

            // 批量生成 Embedding 并写入向量数据库
            embedAndStore(documents);
            System.out.printf("服务 %s 版本 %s 已索引，共 %d 个文档块\n",
                serviceName, version.version(), documents.size());

        } catch (Exception e) {
            throw new RuntimeException("API 文档索引失败: " + serviceName, e);
        }
    }

    private void embedAndStore(List<VectorDocument> documents) {
        // 调用 Embedding 服务 + 向量数据库插入
        // 具体实现见第一篇的 EmbeddingService
    }
}
```

### 4.2 跨版本差异查询

用户常问："v2 和 v1 有什么区别？"

```java
public class ApiVersionDiffService {

    /**
     * 对比两个版本的指定接口
     */
    public String diffEndpoint(String serviceName, String versionA,
                                String versionB, String path, String method) {
        // 1. 从向量数据库检索两个版本的接口
        List<VectorDocument> docsA = searchApiDocs(serviceName, versionA, path, method);
        List<VectorDocument> docsB = searchApiDocs(serviceName, versionB, path, method);

        if (docsA.isEmpty() && docsB.isEmpty()) {
            return "未找到该接口";
        }
        if (docsA.isEmpty()) return versionB + " 新增了此接口";
        if (docsB.isEmpty()) return "该接口在 " + versionB + " 中被移除";

        // 2. 构建对比 Prompt
        StringBuilder diffContext = new StringBuilder();
        diffContext.append("## 版本 ").append(versionA).append("：\n");
        for (var doc : docsA) {
            diffContext.append(doc.content()).append("\n");
        }
        diffContext.append("\n## 版本 ").append(versionB).append("：\n");
        for (var doc : docsB) {
            diffContext.append(doc.content()).append("\n");
        }

        String prompt = """
            以下是两个版本的 API 接口文档，请对比差异：
            
            %s
            
            请列出：
            1. 新增的参数
            2. 删除的参数
            3. 修改的参数（包括类型变化、必填性变化）
            4. 返回值的变化
            5. 接口路径或方法的变化
            """.formatted(diffContext.toString());

        return llmClient.chat(prompt);
    }

    private List<VectorDocument> searchApiDocs(String serviceName,
                                                String version, String path, String method) {
        // 利用向量检索 + 元数据过滤
        Map<String, String> filters = Map.of(
            "serviceName", serviceName,
            "apiVersion", version,
            "path", path,
            "method", method
        );
        return vectorStore.searchWithFilters(path + " " + method, 5, filters);
    }

    // 依赖注入
    private final VectorStore vectorStore;
    private final LLMClient llmClient;

    public ApiVersionDiffService(VectorStore vectorStore, LLMClient llmClient) {
        this.vectorStore = vectorStore;
        this.llmClient = llmClient;
    }

    public interface LLMClient {
        String chat(String prompt);
    }
}
```

### 4.3 版本变更自动通知

当 API 文档有更新时，主动通知关注者：

```java
public class ApiChangeNotifier {

    public record ApiChange(
        String serviceName,
        String oldVersion,
        String newVersion,
        String changeType,    // ADDED / MODIFIED / DEPRECATED / REMOVED
        String path,
        String method,
        String description
    ) {}

    /**
     * 版本对比，生成变更通知列表
     */
    public List<ApiChange> detectChanges(String serviceName,
                                          String oldVersion, String newVersion) {
        // 获取两个版本的接口列表
        Set<String> oldEndpoints = getEndpointSet(serviceName, oldVersion);
        Set<String> newEndpoints = getEndpointSet(serviceName, newVersion);

        List<ApiChange> changes = new ArrayList<>();

        // 新增接口
        Set<String> added = new HashSet<>(newEndpoints);
        added.removeAll(oldEndpoints);
        for (String ep : added) {
            changes.add(new ApiChange(serviceName, oldVersion, newVersion,
                "ADDED", extractPath(ep), extractMethod(ep),
                "新增接口 " + ep));
        }

        // 删除接口
        Set<String> removed = new HashSet<>(oldEndpoints);
        removed.removeAll(newEndpoints);
        for (String ep : removed) {
            changes.add(new ApiChange(serviceName, oldVersion, newVersion,
                "REMOVED", extractPath(ep), extractMethod(ep),
                "接口 " + ep + " 已被移除"));
        }

        // 修改的接口（存在于两边但内容不同）
        Set<String> common = new HashSet<>(oldEndpoints);
        common.retainAll(newEndpoints);
        for (String ep : common) {
            // 更细粒度的参数对比
            List<ApiChange> paramChanges = diffParameters(
                serviceName, oldVersion, newVersion,
                extractPath(ep), extractMethod(ep));
            changes.addAll(paramChanges);
        }

        return changes;
    }

    private List<ApiChange> diffParameters(String serviceName,
                                            String v1, String v2,
                                            String path, String method) {
        // 对比两个版本同一接口的参数差异
        // 实现省略：从向量数据库读取两边接口的全部 chunk，按参数名做 diff
        return List.of();
    }

    private Set<String> getEndpointSet(String serviceName, String version) {
        // 从向量数据库查询该版本的所有接口唯一标识
        return Set.of();
    }

    private String extractPath(String endpoint) { return ""; }
    private String extractMethod(String endpoint) { return ""; }
}
```

## 五、API 助手问答引擎

```java
public class ApiAssistantEngine {

    private final VectorStore vectorStore;
    private final EmbeddingService embeddingService;
    private final LLMClient llmClient;

    // 意图识别的 Prompt
    private static final String INTENT_PROMPT = """
        分析用户关于 API 的提问，返回 JSON 格式的意图分析：
        {"intent":"PARAM/REQUEST/RESPONSE/ERROR/DIFF/USAGE/OTHER",
         "keywords":["关键词1","关键词2"],
         "preferVersion":"v2"}
        
        用户提问：
        """;

    // 回答生成的 Prompt
    private static final String ANSWER_PROMPT = """
        你是一个 API 文档助手。根据以下 API 文档片段，回答用户的问题。
        
        ## API 文档：
        %s
        
        ## 用户问题：
        %s
        
        ## 回答要求：
        1. 先给出直接答案
        2. 再给出详细的参数/字段说明
        3. 如果有示例，提供示例代码
        4. 如果有多个版本，明确指出当前回答基于哪个版本
        5. 如果文档中信息不足以回答，请明确说明
        """;

    public ApiAssistantEngine(VectorStore vectorStore,
                               EmbeddingService embeddingService,
                               LLMClient llmClient) {
        this.vectorStore = vectorStore;
        this.embeddingService = embeddingService;
        this.llmClient = llmClient;
    }

    /**
     * 用户提问入口
     */
    public String ask(String userQuestion) {
        // 1. 意图识别
        String intent = analyzeIntent(userQuestion);

        // 2. 根据意图做针对性检索
        List<VectorDocument> relevantDocs = retrieveByIntent(userQuestion, intent);

        // 3. 如果相关文档太少，做一次更宽泛的检索
        if (relevantDocs.size() < 3) {
            relevantDocs = broadRetrieve(userQuestion);
        }

        // 4. 生成回答
        return generateAnswer(userQuestion, relevantDocs);
    }

    private String analyzeIntent(String userQuestion) {
        return llmClient.chat(INTENT_PROMPT + userQuestion);
    }

    private List<VectorDocument> retrieveByIntent(String question, String intent) {
        // 根据意图过滤 chunkType
        Map<String, String> filters = new HashMap<>();

        // 解析 intent JSON（简化处理）
        if (intent.contains("PARAM")) {
            filters.put("chunkType", "PARAMETER");
        } else if (intent.contains("REQUEST")) {
            filters.put("chunkType", "REQUEST_BODY");
        } else if (intent.contains("RESPONSE")) {
            filters.put("chunkType", "RESPONSE");
        } else {
            // 不确定意图，检索所有类型
            filters.put("chunkType", "OVERVIEW");
        }

        return vectorStore.searchWithFilters(question, 10, filters);
    }

    private List<VectorDocument> broadRetrieve(String question) {
        return vectorStore.search(question, 10);
    }

    private String generateAnswer(String question, List<VectorDocument> docs) {
        String context = docs.stream()
            .map(VectorDocument::content)
            .collect(Collectors.joining("\n---\n"));

        return llmClient.chat(String.format(ANSWER_PROMPT, context, question));
    }

    public interface EmbeddingService {
        List<Float> embed(String text);
    }

    public interface LLMClient {
        String chat(String prompt);
    }
}
```

## 六、实际效果演示

**场景一：后端刚改了接口**

```
用户：order/create 接口返回的 data 是什么结构？

AI 助手：
根据 v2.3.0 版本的文档，POST /api/order/create 接口的 200 响应结构如下：

{
  "code": 200,
  "message": "success",
  "data": {
    "orderId": "ORD20260105001",  // 字符串，订单编号
    "orderStatus": "PENDING",      // 枚举: PENDING/PAID/SHIPPED/CANCELLED
    "totalAmount": 299.00,         // 数字，订单总额
    "createdAt": "2026-01-05T10:30:00Z"  // ISO 8601 时间戳
  }
}

注意：v2.3.0 版本中将 order_id（v2.2.0）改为 orderId，请求参数中也做了同样的调整。
```

**场景二：对比版本差异**

```
用户：user/login 接口 v1 和 v2 有什么变化？

AI 助手：
POST /api/user/login 接口的版本差异：

1. 参数变化：
   - v1：password（明文密码方式传递）
   - v2：加密后的 password（需先调用 /api/crypto/encrypt 接口获取公钥加密后传递）

2. 返回值变化：
   - v1 返回：{"token": "xxx", "expireIn": 3600}
   - v2 返回：{"accessToken": "xxx", "refreshToken": "yyy", "expireIn": 7200}

3. 新增：v2 支持 refreshToken 刷新机制，新增 POST /api/user/refresh 接口
```

## 七、生产环境部署建议

```java
public class ApiRAGDeploymentConfig {

    /**
     * 定时从 Swagger UI 地址拉取最新的 OpenAPI 文档
     */
    @Scheduled(cron = "0 0 2 * * ?")  // 每天凌晨2点
    public void syncApiDocs() {
        List<ServiceConfig> services = loadServiceConfigs();

        for (ServiceConfig config : services) {
            try {
                // 从 Swagger UI 拉取 OpenAPI JSON
                String swaggerUrl = config.baseUrl() + "/v3/api-docs";
                byte[] content = httpClient.get(swaggerUrl);

                // 检查是否有变化
                String newHash = computeHash(content);
                String oldHash = getStoredHash(config.serviceName());

                if (!newHash.equals(oldHash)) {
                    // 文档有更新，重新索引
                    multiVersionManager.registerService(
                        config.serviceName(),
                        List.of(new ApiDocVersion(
                            extractVersion(content), content, "json", ""))
                    );
                    updateStoredHash(config.serviceName(), newHash);
                }
            } catch (Exception e) {
                log.error("同步 API 文档失败: {}", config.serviceName(), e);
            }
        }
    }

    /**
     * 接入企业微信/钉钉/飞书机器人
     */
    public String handleBotMessage(String userId, String message) {
        // 用户可以 @机器人 直接问 API 问题
        return apiAssistant.ask(message);
    }

    public record ServiceConfig(String serviceName, String baseUrl) {}
    private List<ServiceConfig> loadServiceConfigs() { return List.of(); }
    private String computeHash(byte[] content) { return ""; }
    private String getStoredHash(String serviceName) { return ""; }
    private void updateStoredHash(String name, String hash) {}
    private String extractVersion(byte[] content) { return ""; }

    // 注入
    private final HttpClient httpClient;
    private final MultiVersionApiManager multiVersionManager;
    private final ApiAssistantEngine apiAssistant;
}
```

## 八、总结

API 文档 RAG 让团队告别"这个接口怎么用"的低效沟通：

1. **结构化拆分**：按 OVERVIEW/PARAMETER/REQUEST_BODY/RESPONSE 拆分接口文档
2. **多版本管理**：版本隔离 + 自动 diff + 变更通知
3. **意图识别**：根据用户问题类型定向检索对应的 chunk
4. **机器人集成**：接入 IM 工具，随时 @机器人 问接口

核心价值：前端不需要翻 Confluence、找 Swagger、翻聊天记录、等后端回复——直接问 AI 助手，秒级获取答案。

> 下一篇预告：**客服知识库 RAG：FAQ 文档 + 历史工单的双重检索策略**——我们将构建一个智能客服系统，新客服 3 天变老手，从 FAQ 和 10 万历史工单中秒级检索最佳答案！
