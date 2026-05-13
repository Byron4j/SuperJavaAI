# Agent 与 RAG 的协同：让 Agent 学会"查资料+做决策"，既有知识又能行动的 AI

## 一、引言：两种 AI 范式的握手

如果你关注 AI 应用开发，一定知道两个很火的概念：**Agent 和 RAG**。

- **Agent**：能自主决策、调用工具、执行多步骤任务的智能体
- **RAG**：检索增强生成，让 LLM 在回答时先查知识库

但它们大部分时间在各自玩各自的。RAG 的应用通常是「用户提问→检索文档→LLM 回答」的单轮模式；Agent 的应用通常是「感知→决策→行动」的工具调用模式。

有没有想过——**让 Agent 和 RAG 握手**？

想象一个客服 Agent：

1. 用户：「我买的手机屏幕坏了，能退吗？」
2. Agent **决策**：这是一个售后问题，我需要知道退货政策 → **调用 RAG 检索商品售后政策文档**
3. RAG 检索到：『非人为损坏，7 天内支持退货，15 天内支持换货』
4. Agent **再决策**：结合订单信息判断——用户买了 5 天，屏幕是自己摔的 → 「您好，人为损坏不在保修范围，建议您到官方维修点检测。」
5. Agent **行动**：创建售后工单，发送维修点地址。

这就是 **Agent + RAG 协同**：Agent 负责决策和行动，RAG 负责在 Agent 需要的时候提供知识。

> 全文约 5000 字。读完你会掌握 Agent + RAG 协同的三种模式及其 Java 实现。

---

## 二、Agent + RAG 的三种协同模式

```
模式一：前置检索（Pre-Retrieval）
  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
  │ 用户   │ →  │ RAG  │ →  │Agent │ →  │ 结果  │
  │ 输入   │    │ 检索  │    │ 决策  │    │ 输出  │
  └──────┘    └──────┘    └──────┘    └──────┘
  适用：客服问答、规章制度解释、FAQ 型场景

模式二：按需检索（On-Demand Retrieval）
  ┌──────┐    ┌──────┐    ┌──────┐    ┌──────┐
  │ 用户   │ →  │Agent │ ⇄  │ RAG  │ →  │ 结果  │
  │ 输入   │    │ 决策  │    │ 工具  │    │ 输出  │
  └──────┘    └──────┘    └──────┘    └──────┘
  适用：复杂决策场景，Agent 动态决定何时查资料

模式三：多路检索（Multi-Source RAG）
  ┌──────┐    ┌──────┐    ┌──────────┐
  │ 用户   │ →  │Agent │ ⇄  │RAG工具1  │ 知识库A（政策文档）
  │ 输入   │    │ 决策  │    │RAG工具2  │ 知识库B（产品手册）
  └──────┘    └──────┘    │RAG工具3  │ 知识库C（FAQ）
                          └──────────┘
  适用：企业级场景，Agent 根据问题类型选择不同的知识源
```

今天重点讲模式二和模式三——Agent 按需检索，这是最高效的组合。

---

## 三、RAG 工具化：把检索能力封装成 Tool

在 Agent 架构中，「知识检索」和其他工具（代码执行、API 调用、文件操作）一样，都是一种 Tool。关键设计是：

```java
/**
 * RAG Tool——将检索能力封装为 Agent 可调用的工具
 */
public class RAGTool implements Tool {
    
    private final String name;
    private final String description;
    private final RAGEngine ragEngine;
    
    public RAGTool(String name, String description, RAGEngine ragEngine) {
        this.name = name;
        this.description = description;
        this.ragEngine = ragEngine;
    }
    
    @Override
    public String getName() { return name; }
    
    @Override
    public String getDescription() { return description; }
    
    @Override
    public String getParameterSchema() {
        return """
            {
              "query": "用户要查询的问题或关键词",
              "topK": "返回的文档数量（默认3）",
              "filter": "可选的过滤条件（如文档分类、时间范围）"
            }""";
    }
    
    @Override
    public String execute(Map<String, Object> params) throws Exception {
        String query = (String) params.getOrDefault("query", "");
        int topK = (int) params.getOrDefault("topK", 3);
        Map<String, String> filters = (Map<String, String>) params.get("filter");
        
        // 调用 RAG 引擎执行检索
        List<RetrievedDocument> docs = ragEngine.search(query, topK, filters);
        
        if (docs.isEmpty()) {
            return "未找到与「" + query + "」相关的文档。";
        }
        
        // 格式化检索结果为 Agent 可理解的文本
        return formatRetrievalResults(docs, query);
    }
    
    private String formatRetrievalResults(List<RetrievedDocument> docs, String query) {
        StringBuilder sb = new StringBuilder();
        sb.append("查询「").append(query).append("」找到 ").append(docs.size())
                .append(" 条相关文档：\n\n");
        
        for (int i = 0; i < docs.size(); i++) {
            RetrievedDocument doc = docs.get(i);
            sb.append("--- 文档 ").append(i + 1).append(" (相关度: ")
                    .append(String.format("%.2f", doc.getRelevanceScore())).append(") ---\n");
            sb.append("标题: ").append(doc.getTitle()).append("\n");
            sb.append("分类: ").append(doc.getCategory()).append("\n");
            sb.append("内容: ").append(doc.getContent()).append("\n");
            if (doc.getSource() != null) {
                sb.append("来源: ").append(doc.getSource()).append("\n");
            }
            sb.append("\n");
        }
        
        return sb.toString();
    }
}

/**
 * 检索文档
 */
@Data
@Builder
public class RetrievedDocument implements Serializable {
    private String id;
    private String title;
    private String content;
    private String category;
    private String source;
    private double relevanceScore;
    private Map<String, Object> metadata;
}
```

---

## 四、RAG Engine 实现：基于向量数据库的检索

```java
/**
 * RAG 引擎核心——文档索引 + 向量检索
 */
@Service
@Slf4j
public class RAGEngine {
    
    private final EmbeddingService embeddingService;
    private final VectorStore vectorStore;
    private final DocumentStore documentStore;
    private final int defaultTopK;
    
    public RAGEngine(EmbeddingService embeddingService,
                     VectorStore vectorStore,
                     DocumentStore documentStore,
                     @Value("${rag.default-top-k:5}") int defaultTopK) {
        this.embeddingService = embeddingService;
        this.vectorStore = vectorStore;
        this.documentStore = documentStore;
        this.defaultTopK = defaultTopK;
    }
    
    /**
     * 文档入库——将文档分块后向量化并存储
     */
    public void indexDocument(Document doc) {
        log.info("Indexing document: {}", doc.getTitle());
        
        // 1. 文档分块（按段落、语义边界切分）
        List<DocumentChunk> chunks = splitDocument(doc);
        
        // 2. 向量化每个文档块
        List<String> chunkTexts = chunks.stream()
                .map(DocumentChunk::getText)
                .collect(Collectors.toList());
        
        List<float[]> embeddings = embeddingService.embed(chunkTexts);
        
        // 3. 存入向量数据库
        List<String> chunkIds = new ArrayList<>();
        for (int i = 0; i < chunks.size(); i++) {
            DocumentChunk chunk = chunks.get(i);
            chunk.setDocId(doc.getId());
            chunk.setEmbedding(embeddings.get(i));
            
            String chunkId = vectorStore.insert(chunk.getEmbedding(), 
                    Map.of(
                            "docId", doc.getId(),
                            "title", doc.getTitle(),
                            "chunkIndex", String.valueOf(i),
                            "category", doc.getCategory() != null ? doc.getCategory() : ""
                    ));
            chunkIds.add(chunkId);
            chunk.setId(chunkId);
        }
        
        // 4. 存储文档元数据和分块信息
        documentStore.save(doc);
        documentStore.saveChunks(doc.getId(), chunks);
        
        log.info("Indexed {} chunks for document: {}", chunks.size(), doc.getTitle());
    }
    
    /**
     * 检索相关文档上下文
     */
    public List<RetrievedDocument> search(String query, int topK, Map<String, String> filters) {
        // 1. 对查询进行向量化
        float[] queryEmbedding = embeddingService.embedSingle(query);
        
        // 2. 向量相似度搜索
        List<VectorStore.SearchResult> results = vectorStore.search(
                queryEmbedding, topK, filters);
        
        if (results.isEmpty()) {
            // 备用方案：关键词搜索
            results = fallbackKeywordSearch(query, topK);
        }
        
        // 3. 获取完整的文档内容
        List<RetrievedDocument> docs = new ArrayList<>();
        for (VectorStore.SearchResult result : results) {
            Map<String, String> meta = result.getMetadata();
            String docId = meta.get("docId");
            
            Document doc = documentStore.getById(docId);
            if (doc == null) continue;
            
            docs.add(RetrievedDocument.builder()
                    .id(docId)
                    .title(doc.getTitle())
                    .content(result.getContent())
                    .category(meta.getOrDefault("category", ""))
                    .source(doc.getSource())
                    .relevanceScore(result.getScore())
                    .metadata(new HashMap<>())
                    .build());
        }
        
        return docs;
    }
    
    /**
     * 智能搜索——Agent 调用时的大包装方法
     * 先做向量搜索，如果结果相关性不够，再用关键词搜索补充
     */
    public String smartSearch(String query, int topK) {
        List<RetrievedDocument> docs = search(query, topK, null);
        
        StringBuilder result = new StringBuilder();
        result.append("基于查询「").append(query).append("」的检索结果：\n\n");
        
        for (int i = 0; i < docs.size(); i++) {
            RetrievedDocument doc = docs.get(i);
            result.append(i + 1).append(". 【").append(doc.getTitle()).append("】");
            if (!doc.getCategory().isEmpty()) {
                result.append(" [").append(doc.getCategory()).append("]");
            }
            result.append(" (相关度: ").append(String.format("%.2f", doc.getRelevanceScore()))
                    .append(")\n");
            result.append("   ").append(doc.getContent()).append("\n\n");
        }
        
        return result.toString();
    }
    
    /**
     * 文档分块策略
     */
    private List<DocumentChunk> splitDocument(Document doc) {
        String text = doc.getContent();
        List<DocumentChunk> chunks = new ArrayList<>();
        
        // 按段落分割
        String[] paragraphs = text.split("\n\n");
        StringBuilder currentChunk = new StringBuilder();
        int chunkIndex = 0;
        int maxChunkSize = 1000; // 每块最大字符数
        
        for (String para : paragraphs) {
            if (currentChunk.length() + para.length() > maxChunkSize 
                    && !currentChunk.isEmpty()) {
                chunks.add(DocumentChunk.builder()
                        .index(chunkIndex++)
                        .text(currentChunk.toString().trim())
                        .build());
                currentChunk = new StringBuilder();
            }
            currentChunk.append(para).append("\n\n");
        }
        
        if (!currentChunk.isEmpty()) {
            chunks.add(DocumentChunk.builder()
                    .index(chunkIndex)
                    .text(currentChunk.toString().trim())
                    .build());
        }
        
        return chunks;
    }
    
    private List<VectorStore.SearchResult> fallbackKeywordSearch(String query, int topK) {
        // 基于关键词的兜底搜索
        return documentStore.keywordSearch(query, topK).stream()
                .map(doc -> new VectorStore.SearchResult(
                        doc.getContent(), 0.5, Map.of("docId", doc.getId())))
                .collect(Collectors.toList());
    }
}

/**
 * 嵌入服务
 */
public interface EmbeddingService {
    /** 批量向量化 */
    List<float[]> embed(List<String> texts);
    
    /** 单文本向量化 */
    float[] embedSingle(String text);
}

/**
 * 向量数据库操作接口
 */
public interface VectorStore {
    
    /** 插入向量，返回 ID */
    String insert(float[] embedding, Map<String, String> metadata);
    
    /** 向量相似度搜索 */
    List<SearchResult> search(float[] queryEmbedding, int topK, Map<String, String> filters);
    
    /** 删除 */
    void delete(String id);
    
    @Data
    @AllArgsConstructor
    class SearchResult {
        private String content;
        private double score;
        private Map<String, String> metadata;
    }
}

@Data
@Builder
class Document {
    private String id;
    private String title;
    private String content;
    private String category;
    private String source;
    private LocalDateTime createdAt;
}

@Data
@Builder
class DocumentChunk {
    private String id;
    private String docId;
    private int index;
    private String text;
    private float[] embedding;
}
```

---

## 五、Agent + RAG 协同：On-Demand 模式实现

```java
/**
 * RAG 增强型 Agent——在思考过程中可以调用 RAG 工具获取知识
 */
@Service
@Slf4j
public class RAGAugmentedAgent {
    
    private final OpenAiService llmService;
    private final RAGEngine ragEngine;
    private final Map<String, RAGTool> ragTools;
    
    public RAGAugmentedAgent(OpenAiService llmService, RAGEngine ragEngine) {
        this.llmService = llmService;
        this.ragEngine = ragEngine;
        this.ragTools = new HashMap<>();
    }
    
    /**
     * 注册一个 RAG Tool（绑定特定知识库）
     */
    public void registerRAGTool(String name, String description, RAGEngine engine) {
        ragTools.put(name, new RAGTool(name, description, engine));
    }
    
    /**
     * 带 RAG 能力的 Agent 执行主循环
     */
    public String executeWithRAG(String userQuery, List<Tool> otherTools) {
        List<ChatMessage> conversation = new ArrayList<>();
        conversation.add(new ChatMessage(ChatMessageRole.SYSTEM.value(), 
                buildSystemPrompt(ragTools, otherTools)));
        conversation.add(new ChatMessage(ChatMessageRole.USER.value(), userQuery));
        
        int iteration = 0;
        int maxIteration = 8;
        StringBuilder finalAnswer = new StringBuilder();
        
        while (iteration < maxIteration) {
            iteration++;
            
            // 调用 LLM
            ChatCompletionResult response = llmService.createChatCompletion(
                    ChatCompletionRequest.builder()
                            .model("gpt-4-turbo")
                            .messages(conversation)
                            .temperature(0.3)
                            .build());
            
            String assistantMessage = response.getChoices().get(0).getMessage().getContent();
            conversation.add(new ChatMessage(ChatMessageRole.ASSISTANT.value(), 
                    assistantMessage));
            
            // 解析是否含有工具调用
            ToolCall toolCall = parseToolCall(assistantMessage);
            
            if (toolCall == null) {
                // 没有工具调用，Agent 认为任务完成
                finalAnswer.append(assistantMessage);
                break;
            }
            
            // 执行工具调用（RAG 检索或其他工具）
            String toolResult = executeToolCall(toolCall);
            
            // 将工具结果反馈给 Agent
            conversation.add(new ChatMessage(ChatMessageRole.USER.value(), 
                    "工具执行结果（" + toolCall.getToolName() + "）：\n" + toolResult));
        }
        
        return finalAnswer.toString();
    }
    
    private String buildSystemPrompt(Map<String, RAGTool> ragTools, 
                                      List<Tool> otherTools) {
        StringBuilder sb = new StringBuilder();
        sb.append("""
            你是一个智能助手，结合知识库检索和工具调用来完成任务。
            
            ## 工作流程
            1. 理解用户的问题或任务
            2. 判断是否需要从知识库中检索信息
            3. 如果需要检索，使用 RAG 工具获取相关文档
            4. 基于检索结果进行推理和决策
            5. 如果还需要执行其他操作，调用对应的工具
            6. 给出最终答案
            
            ## 可用工具
            """);
        
        for (RAGTool tool : ragTools.values()) {
            sb.append("- **").append(tool.getName()).append("**: ")
                    .append(tool.getDescription()).append("\n");
        }
        
        if (otherTools != null) {
            for (Tool tool : otherTools) {
                sb.append("- **").append(tool.getName()).append("**: ")
                        .append(tool.getDescription()).append("\n");
            }
        }
        
        sb.append("""
            
            ## 工具调用格式
            当需要使用工具时，按以下格式输出：
            ```tool_call
            {
              "tool": "工具名称",
              "params": {
                "query": "查询内容",
                "topK": 3,
                "filter": {"category": "政策文档"}
              }
            }
            ```
            """);
        
        return sb.toString();
    }
    
    private ToolCall parseToolCall(String message) {
        Pattern pattern = Pattern.compile(
                "```tool_call\\s*\\n(\\{.*?\\})\\s*```", Pattern.DOTALL);
        Matcher matcher = pattern.matcher(message);
        
        if (matcher.find()) {
            try {
                ObjectMapper mapper = new ObjectMapper();
                JsonNode node = mapper.readTree(matcher.group(1));
                
                return new ToolCall(
                        node.get("tool").asText(),
                        parseParams(node.get("params")));
            } catch (Exception e) {
                log.warn("Failed to parse tool call: {}", e.getMessage());
            }
        }
        return null;
    }
    
    private Map<String, Object> parseParams(JsonNode paramsNode) {
        Map<String, Object> params = new HashMap<>();
        paramsNode.fields().forEachRemaining(entry -> {
            JsonNode value = entry.getValue();
            if (value.isInt()) {
                params.put(entry.getKey(), value.asInt());
            } else if (value.isDouble()) {
                params.put(entry.getKey(), value.asDouble());
            } else if (value.isTextual()) {
                params.put(entry.getKey(), value.asText());
            } else if (value.isObject()) {
                Map<String, String> subMap = new HashMap<>();
                value.fields().forEachRemaining(e -> 
                        subMap.put(e.getKey(), e.getValue().asText()));
                params.put(entry.getKey(), subMap);
            }
        });
        return params;
    }
    
    private String executeToolCall(ToolCall toolCall) {
        try {
            RAGTool ragTool = ragTools.get(toolCall.getToolName());
            if (ragTool != null) {
                return ragTool.execute(toolCall.getParams());
            }
            return "未知工具: " + toolCall.getToolName();
        } catch (Exception e) {
            return "工具执行失败: " + e.getMessage();
        }
    }
    
    @AllArgsConstructor
    @Data
    private static class ToolCall {
        private String toolName;
        private Map<String, Object> params;
    }
}
```

---

## 六、实战：智能客服 + 知识库

```java
@RestController
@RequestMapping("/api/smart-cs")
public class SmartCustomerServiceController {
    
    private final RAGAugmentedAgent agent;
    private final RAGEngine policiesEngine;
    private final RAGEngine productsEngine;
    private final RAGEngine faqEngine;
    
    public SmartCustomerServiceController(RAGAugmentedAgent agent,
                                           @Qualifier("policies") RAGEngine policiesEngine,
                                           @Qualifier("products") RAGEngine productsEngine,
                                           @Qualifier("faq") RAGEngine faqEngine) {
        this.agent = agent;
        this.policiesEngine = policiesEngine;
        this.productsEngine = productsEngine;
        this.faqEngine = faqEngine;
        
        // 注册多个知识域的 RAG Tool
        agent.registerRAGTool("PolicySearch", "检索退货、换货、保修等售后政策", policiesEngine);
        agent.registerRAGTool("ProductSearch", "检索产品参数、使用说明、规格信息", productsEngine);
        agent.registerRAGTool("FAQSearch", "检索常见问题解答", faqEngine);
    }
    
    @PostMapping("/chat")
    public ResponseEntity<CustomerServiceResponse> chat(@RequestBody ChatRequest request) {
        
        // 定义业务工具（创建工单、发送邮件等）
        List<Tool> businessTools = List.of(
                Tool.of("CreateTicket", "创建售后工单", params -> {
                    String issue = (String) params.get("issue");
                    return "工单已创建，工单号: TK-" + System.currentTimeMillis() 
                            % 100000 + "，问题: " + issue;
                }),
                Tool.of("SendEmail", "发送邮件给客户", params -> {
                    return "已向客户发送通知邮件";
                }),
                Tool.of("RefundCheck", "查询退款处理状态", params -> {
                    String orderNo = (String) params.getOrDefault("orderNo", "");
                    return "订单 " + orderNo + " 退款状态: 处理中，预计 3 个工作日内到账";
                })
        );
        
        String answer = agent.executeWithRAG(request.getQuestion(), businessTools);
        
        return ResponseEntity.ok(CustomerServiceResponse.builder()
                .question(request.getQuestion())
                .answer(answer)
                .build());
    }
}

@Data
class ChatRequest {
    private String question;
    private String userId;
}

@Data
@Builder
class CustomerServiceResponse {
    private String question;
    private String answer;
}
```

**一个真实对话流程**：

```
用户: "我刚买的耳机左耳没声音，能换货吗？"

Agent 思考链:
  Step 1: 判断 → 需要查售后政策 → 调用 PolicySearch("耳机 换货 故障")
  
  RAG 返回: 
    文档1: 《电子产品售后政策》"非人为故障，7天内换货，15天内维修"
    文档2: 《耳机保修条款》"保修期1年，凭购买凭证享受保修服务"
  
  Step 2: 判断 → 需要确认订单信息 → 调用 OrderQuery(userId)
  
  订单系统返回:
    订单 #88421，购买时间: 3天前，状态: 已签收
  
  Step 3: 综合决策 →
    "您好！根据您的订单信息（购买3天），符合7天换货政策。
     我已为您创建换货工单 TK-45231，请您将耳机寄回至...
     预计换货周期为5个工作日。"
  
  Step 4: 行动 → CreateTicket + SendEmail → 完成
```

---

## 七、关键设计模式总结

**Agent + RAG 协同的三个关键设计**：

### 1. RAG as Tool
把 RAG 检索封装为 Tool，Agent 像调用其他工具一样调用它。

### 2. Context Window 管理
检索结果可能很长。需要在 Prompt 中精确控制，避免 Token 溢出：
- **Top-K 截断**：只返回最相关的 K 条结果
- **内容压缩**：将检索到的内容让 LLM 摘要后再传给 Agent
- **Relevance Threshold**：过滤相关性低于阈值的文档

### 3. 多知识域路由
```
if (用户问题包含"退货"|"换货"|"保修") → PolicySearch
if (用户问题包含"怎么用"|"参数"|"规格") → ProductSearch
if (用户问题包含"为什么"|"怎么办") → FAQSearch
```

---

## 八、总结

Agent + RAG 的组合让 AI 应用从「会回答」进化到「会行动」：

- **纯 RAG**：用户问→检索→回答（被动）
- **纯 Agent**：用户说→决策→行动（没有知识来源）
- **Agent + RAG**：用户说→Agent 决策→按需检索知识→基于知识再决策→行动（主动 + 有知识）

这是企业级 AI 应用的标准范式——**既有知识，又能行动**。

---

**下篇预告**：《构建你的第一个 AI Agent 团队：从需求分析到上线》——3 个 Agent 组成一个开发小组！需求分析师 + 开发工程师 + 测试工程师协作完成一个完整项目。完整可运行代码，一键启动你的第一个 AI 开发团队！
