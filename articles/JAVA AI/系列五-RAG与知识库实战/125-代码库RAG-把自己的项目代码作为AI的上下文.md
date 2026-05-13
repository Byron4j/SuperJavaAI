# 代码库 RAG：把自己的项目代码作为 AI 的上下文，AI 终于知道你项目的接口定义了

> 你有没有这种感觉：让 ChatGPT 写代码，它写得贼溜，但一放到你的项目里就水土不服？因为它根本不知道你项目里 `UserService` 长什么样！

## 一、问题的根源：大模型不知道你的代码

大模型（GPT-4、Claude、通义千问等）的训练数据截止到某个时间点，它们不知道你公司内部的代码库、不知道你项目的架构设计、不知道你自定义的注解和工具类。

于是出现了经典场景：

```java
// 你让 AI 帮你写一个用户查询接口
// AI 写出来的代码：
@RestController
public class UserController {
    @Autowired
    private JdbcTemplate jdbcTemplate;  // 你项目用的是 MyBatis-Plus 啊喂！
    
    @GetMapping("/user/{id}")
    public Map<String, Object> getUser(@PathVariable Long id) {
        return jdbcTemplate.queryForMap("SELECT * FROM t_user WHERE id=?", id);
        // 你项目有统一的 Result<T> 返回体，AI 不知道！
    }
}
```

AI 写的代码功能正确，但和你的项目格格不入。它不知道你项目里有 `BaseController`、`Result<T>`、`AuthUtils`、`BizException`……它像一个新入职的员工，只懂 Java 语法，不懂你们团队的约定。

**解决方案正是代码库 RAG（Retrieval-Augmented Generation）**——把你的项目代码作为上下文喂给 AI。

## 二、代码库 RAG 的核心思路

整个流程分为两个阶段：

**索引阶段（离线）**：扫描项目代码 → 代码切割 → 生成 Embedding → 存入向量数据库

**检索阶段（在线）**：用户提问 → 生成查询 Embedding → 向量检索 → 拼接上下文 → 发给 LLM

```
┌─────────────────────────────────────────────────────────────┐
│                     索引阶段（离线）                          │
│                                                             │
│  Java 项目 ──▶ 代码切割 ──▶ Code Embedding ──▶ 向量数据库     │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                     检索阶段（在线）                          │
│                                                             │
│  用户提问 ──▶ Query Embedding ──▶ 相似度检索 ──▶ Top-K 代码   │
│       │                                                     │
│       └──────▶ Prompt = 问题 + 检索到的代码 ──▶ LLM ──▶ 回答  │
└─────────────────────────────────────────────────────────────┘
```

关键在于第一步：**怎么切割代码**？代码和自然语言不同，它有严格的语法结构和依赖关系，乱切会破坏语义。

## 三、代码切割策略：按函数 / 按类 / 按文件

### 3.1 按函数切割（推荐）

对于 Java 项目，**以方法为粒度**切割是最佳实践。每个方法作为一个 chunk，附带类名、包名、注释等信息作为元数据。

```java
import com.github.javaparser.JavaParser;
import com.github.javaparser.ast.CompilationUnit;
import com.github.javaparser.ast.body.MethodDeclaration;
import com.github.javaparser.ast.body.ClassOrInterfaceDeclaration;

import java.io.IOException;
import java.nio.file.*;
import java.util.*;

public class CodeChunker {

    public record CodeChunk(
        String chunkId,       // 唯一标识
        String filePath,      // 源文件路径
        String className,     // 类名（含包名）
        String methodName,    // 方法名
        String signature,     // 方法签名
        String code,          // 完整方法代码
        String javadoc,       // 文档注释
        List<String> annotations // 注解列表
    ) {}

    public List<CodeChunk> chunkJavaFile(Path filePath) throws IOException {
        List<CodeChunk> chunks = new ArrayList<>();
        String source = Files.readString(filePath);

        try {
            CompilationUnit cu = new JavaParser().parse(source).getResult()
                .orElseThrow(() -> new RuntimeException("解析失败"));

            String packageName = cu.getPackageDeclaration()
                .map(pd -> pd.getNameAsString())
                .orElse("default");

            for (var type : cu.getTypes()) {
                if (type instanceof ClassOrInterfaceDeclaration clazz) {
                    String className = packageName + "." + clazz.getNameAsString();

                    // 获取类的 Javadoc 作为额外上下文
                    String classJavadoc = clazz.getJavadoc()
                        .map(jd -> jd.getDescription().toText())
                        .orElse("");

                    for (var member : clazz.getMembers()) {
                        if (member instanceof MethodDeclaration method) {

                            String methodName = method.getNameAsString();
                            String signature = method.getDeclarationAsString(true, true, true);

                            List<String> annotations = method.getAnnotations().stream()
                                .map(a -> a.getNameAsString())
                                .toList();

                            String javadoc = method.getJavadoc()
                                .map(jd -> jd.getDescription().toText())
                                .orElse("");

                            // 构造包含完整上下文的方法代码块
                            String enrichedCode = String.format(
                                "// Class: %s\n// Package: %s\n%s\npublic class %s {\n    %s\n    %s\n}",
                                className, packageName,
                                classJavadoc.isEmpty() ? "" : "// " + classJavadoc,
                                clazz.getNameAsString(),
                                javadoc.isEmpty() ? "" : "/** " + javadoc + " */",
                                method.toString().replace("\n", "\n    ")
                            );

                            chunks.add(new CodeChunk(
                                className + "#" + methodName,
                                filePath.toString(),
                                className,
                                methodName,
                                signature,
                                enrichedCode,
                                javadoc,
                                annotations
                            ));
                        }
                    }
                }
            }
        } catch (Exception e) {
            // 解析失败时，退化为按文件切割
            chunks.add(fallbackChunk(filePath, source));
        }

        return chunks;
    }

    private CodeChunk fallbackChunk(Path filePath, String source) {
        return new CodeChunk(
            filePath.toString(),
            filePath.toString(),
            filePath.toString(),
            "",
            "",
            source,
            "",
            List.of()
        );
    }
}
```

### 3.2 按类切割（较大粒度）

当方法都很短或者类本身就是一个完整的逻辑单元时，按类切割更合适：

```java
public List<CodeChunk> chunkJavaFileByClass(Path filePath) throws IOException {
    List<CodeChunk> chunks = new ArrayList<>();
    String source = Files.readString(filePath);

    try {
        CompilationUnit cu = new JavaParser().parse(source).getResult().orElseThrow();
        String packageName = cu.getPackageDeclaration()
            .map(pd -> pd.getNameAsString()).orElse("default");

        for (var type : cu.getTypes()) {
            if (type instanceof ClassOrInterfaceDeclaration clazz) {
                String className = packageName + "." + clazz.getNameAsString();
                String javadoc = clazz.getJavadoc()
                    .map(jd -> jd.getDescription().toText()).orElse("");

                // 提取类的关键信息（字段 + 方法签名），不包含方法体
                StringBuilder classSummary = new StringBuilder();
                classSummary.append("// Class: ").append(className).append("\n");
                if (!javadoc.isEmpty()) {
                    classSummary.append("// Description: ").append(javadoc).append("\n");
                }

                // 字段列表
                clazz.getFields().forEach(field ->
                    classSummary.append("// Field: ").append(field).append("\n")
                );

                // 方法签名列表
                clazz.getMethods().forEach(method ->
                    classSummary.append("// Method: ").append(method.getSignature()).append("\n")
                );

                chunks.add(new CodeChunk(
                    className,
                    filePath.toString(),
                    className,
                    "",
                    "",
                    classSummary.toString() + "\n" + clazz.toString(),
                    javadoc,
                    List.of()
                ));
            }
        }
    } catch (Exception e) {
        chunks.add(fallbackChunk(filePath, source));
    }

    return chunks;
}
```

### 3.3 按文件切割（最大粒度）

适用于小型项目或者配置文件：

```java
public List<CodeChunk> chunkJavaFileByFile(Path filePath) throws IOException {
    String source = Files.readString(filePath);
    String relativePath = filePath.toString();

    // 过滤掉过大的文件（超过 8192 个 token → 约 30000 字符）
    if (source.length() > 30000) {
        return chunkJavaFile(filePath); // 退化为按方法切割
    }

    return List.of(new CodeChunk(
        relativePath,
        relativePath,
        relativePath,
        "",
        "",
        "// File: " + relativePath + "\n" + source,
        "",
        List.of()
    ));
}
```

### 3.4 混合策略（工程实践）

实际项目中，我会根据文件大小动态选择策略：

```java
public class AdaptiveCodeChunker {

    private static final int METHOD_CHUNK_THRESHOLD = 500;   // 方法超过 500 行才单独切割
    private static final int FILE_CHUNK_THRESHOLD = 300;     // 文件超过 300 行就按方法切

    public List<CodeChunk> chunkProject(Path projectRoot) throws IOException {
        List<CodeChunk> allChunks = new ArrayList<>();

        try (var stream = Files.walk(projectRoot)) {
            stream.filter(p -> p.toString().endsWith(".java"))
                .filter(p -> !p.toString().contains("/test/"))  // 排除测试代码
                .filter(p -> !p.toString().contains("/target/")) // 排除编译输出
                .forEach(file -> {
                    try {
                        String source = Files.readString(file);
                        int lineCount = source.split("\n").length;

                        if (lineCount < FILE_CHUNK_THRESHOLD) {
                            allChunks.addAll(chunkJavaFileByFile(file));
                        } else if (lineCount < 1000) {
                            allChunks.addAll(chunkJavaFile(file));       // 按方法切
                        } else {
                            allChunks.addAll(chunkJavaFileByClass(file)); // 大文件先按类切
                        }
                    } catch (IOException e) {
                        System.err.println("跳过文件: " + file);
                    }
                });
        }

        System.out.printf("项目中共切割出 %d 个代码块\n", allChunks.size());
        return allChunks;
    }
}
```

## 四、代码 Embedding 的特殊处理

### 4.1 为什么通用 Embedding 模型对代码效果差？

通用 Embedding 模型（如 text-embedding-ada-002）是用自然语言语料训练的。代码和自然语言有本质区别：

| 维度 | 自然语言 | 代码 |
|------|---------|------|
| 语义载体 | 词汇 | 符号、关键字 |
| 结构 | 线性 | 树形（AST） |
| 上下文依赖 | 段落 | 函数调用链 |

例如，`getUserById` 和 `findUserById` 在自然语言中相似，但在代码中它们是两个不同的方法名——一个遵循 `get` 前缀规范，一个遵循 `find` 前缀规范。

### 4.2 使用专门的 Code Embedding 模型

推荐使用专门针对代码训练的 Embedding 模型：

```java
import java.util.List;

public class CodeEmbeddingService {

    // 方案一：使用 OpenAI 的 text-embedding-3-large（通用但效果好）
    // 方案二：使用专门的 Code Embedding 模型，如 CodeBERT、UniXcoder
    // 方案三：本地部署 sentence-transformers/all-MiniLM-L6-v2（轻量够用）

    public interface EmbeddingClient {
        List<Float> embed(String text);
    }

    // OpenAI 方案
    public static class OpenAIEmbeddingClient implements EmbeddingClient {
        private final OpenAIClient client;
        private final String model;

        public OpenAIEmbeddingClient(String apiKey, String model) {
            this.client = new OpenAIClient(apiKey);
            this.model = model != null ? model : "text-embedding-3-large";
        }

        @Override
        public List<Float> embed(String text) {
            // 调用 OpenAI Embedding API
            EmbeddingResponse response = client.embeddings()
                .model(model)
                .input(text)
                .execute();
            return response.data().get(0).embedding();
        }
    }

    // 本地 ONNX Runtime 方案（推荐用于内网部署）
    public static class LocalCodeEmbeddingClient implements EmbeddingClient {
        private final OnnxEmbeddingModel model;
        private final CodeTokenizer tokenizer;

        public LocalCodeEmbeddingClient(String modelPath, String tokenizerPath) {
            this.model = new OnnxEmbeddingModel(modelPath);
            this.tokenizer = new CodeTokenizer(tokenizerPath);
        }

        @Override
        public List<Float> embed(String codeText) {
            // 预处理：保留代码的关键结构信息
            String preprocessed = preprocessCode(codeText);
            long[] tokenIds = tokenizer.encode(preprocessed);
            float[] embedding = model.infer(tokenIds);
            return toFloatList(embedding);
        }

        private String preprocessCode(String code) {
            // 1. 保留注释（Javadoc 包含重要语义）
            // 2. 将方法名、类名提取为附加句子
            // 3. 对注解做特殊处理
            StringBuilder enhanced = new StringBuilder(code);

            // 从代码中提取关键标识符作为增强文本
            Set<String> identifiers = extractIdentifiers(code);
            for (String id : identifiers) {
                // 将驼峰命名拆分为自然语言
                enhanced.append(" ").append(camelCaseToWords(id));
            }

            return enhanced.toString();
        }

        private Set<String> extractIdentifiers(String code) {
            Set<String> identifiers = new HashSet<>();
            // 正则匹配 Java 标识符（类名、方法名、变量名）
            java.util.regex.Matcher m = java.util.regex.Pattern
                .compile("\\b([A-Z][a-z]+)+\\b|\\b([a-z]+[A-Z][a-z]*)+\\b")
                .matcher(code);
            while (m.find()) {
                identifiers.add(m.group());
            }
            return identifiers;
        }

        private String camelCaseToWords(String camelCase) {
            return camelCase.replaceAll("([a-z])([A-Z])", "$1 $2")
                .replaceAll("([A-Z])([A-Z][a-z])", "$1 $2");
        }

        private List<Float> toFloatList(float[] arr) {
            List<Float> list = new ArrayList<>(arr.length);
            for (float f : arr) list.add(f);
            return list;
        }

        private static class OnnxEmbeddingModel {
            OnnxEmbeddingModel(String path) { /* 加载 ONNX 模型 */ }
            float[] infer(long[] inputIds) { return new float[384]; } // 简化
        }

        private static class CodeTokenizer {
            CodeTokenizer(String path) { /* 加载分词器 */ }
            long[] encode(String text) { return new long[512]; } // 简化
        }
    }
}
```

### 4.3 代码的语义增强

为了让 Embedding 更准确，我们在编码前对代码做语义增强：

```java
public class CodeSemanticEnhancer {

    /**
     * 将代码块增强为更丰富的文本表示，提高 Embedding 准确度
     */
    public static String enhance(CodeChunker.CodeChunk chunk) {
        StringBuilder enhanced = new StringBuilder();

        // 1. 前置自然语言描述
        enhanced.append("Java class ").append(chunk.className());
        if (chunk.methodName() != null && !chunk.methodName().isEmpty()) {
            enhanced.append(" method ").append(chunk.methodName());
        }
        enhanced.append("\n");

        // 2. 业务描述（从 Javadoc 提取）
        if (chunk.javadoc() != null && !chunk.javadoc().isEmpty()) {
            enhanced.append("Description: ").append(chunk.javadoc()).append("\n");
        }

        // 3. 技术特征标签
        enhanced.append("Annotations: ");
        for (String ann : chunk.annotations()) {
            enhanced.append(ann).append(" ");
            // 为常见注解生成自然语言标签
            enhanced.append(getAnnotationDescription(ann)).append(" ");
        }
        enhanced.append("\n");

        // 4. 方法签名（强化类型信息）
        enhanced.append("Signature: ").append(chunk.signature()).append("\n");

        // 5. 被检索的代码本身
        enhanced.append("Code:\n").append(chunk.code());

        return enhanced.toString();
    }

    private static String getAnnotationDescription(String annotation) {
        return switch (annotation) {
            case "RestController" -> "REST API controller";
            case "Service" -> "business service layer";
            case "Repository" -> "data access layer";
            case "Transactional" -> "transactional operation";
            case "Async" -> "asynchronous operation";
            case "Cacheable" -> "cached operation";
            case "RequestMapping", "GetMapping", "PostMapping" -> "HTTP endpoint";
            default -> "";
        };
    }
}
```

## 五、检索增强的代码生成：完整实现

### 5.1 项目代码索引器

```java
import java.io.IOException;
import java.nio.file.Path;
import java.util.List;
import java.util.concurrent.Executors;

public class CodebaseIndexer {

    private final AdaptiveCodeChunker chunker;
    private final CodeEmbeddingService.EmbeddingClient embeddingClient;
    private final VectorStore vectorStore;

    public CodebaseIndexer(EmbeddingClient embeddingClient, VectorStore vectorStore) {
        this.chunker = new AdaptiveCodeChunker();
        this.embeddingClient = embeddingClient;
        this.vectorStore = vectorStore;
    }

    public void indexProject(Path projectRoot, String projectName) throws IOException {
        System.out.println("开始索引项目: " + projectName);

        // 1. 代码切割
        List<CodeChunker.CodeChunk> chunks = chunker.chunkProject(projectRoot);

        // 2. 批量生成 Embedding（并发处理）
        int batchSize = 20;
        for (int i = 0; i < chunks.size(); i += batchSize) {
            int end = Math.min(i + batchSize, chunks.size());
            List<CodeChunker.CodeChunk> batch = chunks.subList(i, end);

            // 并发处理每个 batch
            try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
                for (CodeChunker.CodeChunk chunk : batch) {
                    executor.submit(() -> {
                        String enhanced = CodeSemanticEnhancer.enhance(chunk);
                        List<Float> embedding = embeddingClient.embed(enhanced);

                        // 构造元数据
                        Map<String, Object> metadata = Map.of(
                            "project", projectName,
                            "filePath", chunk.filePath(),
                            "className", chunk.className(),
                            "methodName", chunk.methodName(),
                            "signature", chunk.signature()
                        );

                        vectorStore.insert(new VectorDocument(
                            chunk.chunkId(),
                            embedding,
                            chunk.code(),
                            metadata
                        ));
                    });
                }
            }
            System.out.printf("已索引 %d/%d 个代码块\n", end, chunks.size());
        }

        System.out.println("项目索引完成！共 " + chunks.size() + " 个代码块");
    }
}
```

### 5.2 代码检索器

```java
public class CodeRetriever {

    private final CodeEmbeddingService.EmbeddingClient embeddingClient;
    private final VectorStore vectorStore;
    private final int topK;
    private final double similarityThreshold;

    public CodeRetriever(EmbeddingClient client, VectorStore store,
                         int topK, double similarityThreshold) {
        this.embeddingClient = client;
        this.vectorStore = store;
        this.topK = topK;
        this.similarityThreshold = similarityThreshold;
    }

    /**
     * 根据用户的自然语言问题，检索相关代码
     */
    public List<VectorDocument> retrieve(String userQuery) {
        // 1. 查询意图增强
        String enhancedQuery = enhanceQuery(userQuery);

        // 2. 生成查询 Embedding
        List<Float> queryEmbedding = embeddingClient.embed(enhancedQuery);

        // 3. 向量检索
        List<VectorDocument> candidates = vectorStore.search(
            queryEmbedding, topK * 2);  // 取两倍候选，后续做重排序

        // 4. 重排序：综合相似度和代码质量
        return rerank(candidates, userQuery, topK);
    }

    private String enhanceQuery(String query) {
        // 将自然语言查询转化为更贴近代码的描述
        StringBuilder enhanced = new StringBuilder();
        enhanced.append(query).append("\n");

        // 识别查询意图
        if (query.contains("接口") || query.contains("API")) {
            enhanced.append("RestController GetMapping PostMapping RequestMapping");
        }
        if (query.contains("数据库") || query.contains("数据") || query.contains("查询")) {
            enhanced.append("Repository Mapper JPA MyBatis SQL");
        }
        if (query.contains("服务") || query.contains("业务")) {
            enhanced.append("Service Transactional");
        }
        if (query.contains("配置") || query.contains("设置")) {
            enhanced.append("ConfigurationProperties Bean Value Component");
        }

        return enhanced.toString();
    }

    private List<VectorDocument> rerank(
            List<VectorDocument> candidates, String query, int k) {

        // 重排序策略：
        // 1. 代码中包含的 Java 注解与查询意图的匹配度
        // 2. 代码块的完整度（方法级 > 类级 > 文件级）
        // 3. 文件路径的领域相关度

        return candidates.stream()
            .filter(doc -> doc.similarity() >= similarityThreshold)
            .sorted((a, b) -> {
                double scoreA = calculateRelevanceScore(a, query);
                double scoreB = calculateRelevanceScore(b, query);
                return Double.compare(scoreB, scoreA);
            })
            .limit(k)
            .toList();
    }

    private double calculateRelevanceScore(VectorDocument doc, String query) {
        double score = doc.similarity();

        // 方法级代码块加分
        if (doc.metadata().get("methodName") != null
            && !doc.metadata().get("methodName").toString().isEmpty()) {
            score += 0.1;
        }

        // Javadoc 存在加分
        if (doc.content().contains("/**")) {
            score += 0.05;
        }

        // Service/Controller 类加分（更可能是业务核心代码）
        String className = (String) doc.metadata().get("className");
        if (className != null && (className.contains("Service")
            || className.contains("Controller"))) {
            score += 0.1;
        }

        return score;
    }
}
```

### 5.3 RAG 代码生成器

```java
public class CodeRAGGenerator {

    private final CodeRetriever retriever;
    private final LLMClient llmClient;

    // 代码生成的 Prompt 模板
    private static final String CODE_GEN_PROMPT = """
        你是一个 Java 开发助手。请根据以下项目代码上下文，完成用户的编程需求。

        ## 项目代码上下文：
        %s

        ## 用户需求：
        %s

        ## 要求：
        1. 必须使用项目中已有的类和工具方法
        2. 必须遵循项目的代码风格和命名规范
        3. 如果涉及数据库操作，使用项目中已有的 Mapper/Repository
        4. 返回值统一使用项目中的响应体格式
        5. 生成完整可编译的 Java 代码
        6. 只输出代码，不需要额外解释
        """;

    public CodeRAGGenerator(CodeRetriever retriever, LLMClient llmClient) {
        this.retriever = retriever;
        this.llmClient = llmClient;
    }

    /**
     * 对方法进行增强：先检索项目上下文，再让 LLM 生成
     */
    public String generateEnhanced(String userRequest) {
        // 1. 检索相关代码
        List<VectorDocument> relevantCode = retriever.retrieve(userRequest);

        // 2. 构建代码上下文
        String codeContext = buildContext(relevantCode);

        // 3. 拼接 Prompt
        String prompt = String.format(CODE_GEN_PROMPT, codeContext, userRequest);

        // 4. 调用 LLM
        return llmClient.chat(prompt);
    }

    /**
     * 对比：不使用 RAG 直接生成
     */
    public String generateWithoutRAG(String userRequest) {
        String prompt = "请完成以下 Java 编程需求，只输出代码：\n" + userRequest;
        return llmClient.chat(prompt);
    }

    private String buildContext(List<VectorDocument> docs) {
        StringBuilder context = new StringBuilder();

        for (int i = 0; i < docs.size(); i++) {
            VectorDocument doc = docs.get(i);
            context.append("--- 相关代码片段 ")
                .append(i + 1).append(" ---\n");
            context.append("// 文件: ").append(doc.metadata().get("filePath")).append("\n");
            context.append("// 类: ").append(doc.metadata().get("className")).append("\n");
            if (doc.metadata().get("methodName") != null
                && !doc.metadata().get("methodName").toString().isEmpty()) {
                context.append("// 方法: ").append(doc.metadata().get("methodName")).append("\n");
            }
            context.append("// 相似度: ").append(String.format("%.2f", doc.similarity())).append("\n");
            context.append(doc.content()).append("\n\n");
        }

        return context.toString();
    }

    public interface LLMClient {
        String chat(String prompt);
    }
}
```

### 5.4 向量数据库抽象层

```java
import java.util.List;
import java.util.Map;

public interface VectorStore {

    void insert(VectorDocument document);

    void insertBatch(List<VectorDocument> documents);

    List<VectorDocument> search(List<Float> queryVector, int topK);

    void delete(String id);

    void clear();
}

public record VectorDocument(
    String id,
    List<Float> vector,
    String content,
    Map<String, Object> metadata,
    double similarity  // 仅检索结果时使用
) {
    public VectorDocument(String id, List<Float> vector,
                          String content, Map<String, Object> metadata) {
        this(id, vector, content, metadata, 0.0);
    }
}

// Milvus 实现
public class MilvusVectorStore implements VectorStore {
    private final MilvusClient client;
    private final String collectionName;

    public MilvusVectorStore(String host, int port, String collectionName) {
        this.client = new MilvusClient(host, port);
        this.collectionName = collectionName;
    }

    @Override
    public void insert(VectorDocument document) {
        // insertEntity(collectionName, document)
    }

    @Override
    public void insertBatch(List<VectorDocument> documents) {
        // 批量插入，Milvus 官方推荐
    }

    @Override
    public List<VectorDocument> search(List<Float> queryVector, int topK) {
        // 调用 Milvus 搜索 API，返回 Top-K 结果
        return List.of(); // 简化
    }

    @Override public void delete(String id) {}
    @Override public void clear() {}

    private static class MilvusClient {
        MilvusClient(String host, int port) {}
    }
}

// 简单的内存实现（适合 Demo）
public class InMemoryVectorStore implements VectorStore {
    private final Map<String, VectorDocument> store = new ConcurrentHashMap<>();
    private final CosineSimilarity similarity = new CosineSimilarity();

    @Override
    public void insert(VectorDocument document) {
        store.put(document.id(), document);
    }

    @Override
    public void insertBatch(List<VectorDocument> documents) {
        documents.forEach(this::insert);
    }

    @Override
    public List<VectorDocument> search(List<Float> queryVector, int topK) {
        float[] qVec = toArray(queryVector);

        return store.values().stream()
            .map(doc -> {
                float[] dVec = toArray(doc.vector());
                double sim = similarity.compute(qVec, dVec);
                return new VectorDocument(
                    doc.id(), doc.vector(), doc.content(), doc.metadata(), sim);
            })
            .sorted((a, b) -> Double.compare(b.similarity(), a.similarity()))
            .limit(topK)
            .toList();
    }

    @Override public void delete(String id) { store.remove(id); }
    @Override public void clear() { store.clear(); }

    private float[] toArray(List<Float> list) {
        float[] arr = new float[list.size()];
        for (int i = 0; i < list.size(); i++) arr[i] = list.get(i);
        return arr;
    }

    private static class CosineSimilarity {
        double compute(float[] a, float[] b) {
            double dot = 0, normA = 0, normB = 0;
            for (int i = 0; i < a.length; i++) {
                dot += a[i] * b[i];
                normA += a[i] * a[i];
                normB += b[i] * b[i];
            }
            return dot / (Math.sqrt(normA) * Math.sqrt(normB));
        }
    }
}
```

## 六、效果对比：有 RAG vs 无 RAG

### 测试场景

假设项目中有以下代码：

```java
// com.example.common.Result.java
public class Result<T> {
    private int code;
    private String message;
    private T data;

    public static <T> Result<T> success(T data) {
        Result<T> r = new Result<>();
        r.code = 200;
        r.message = "success";
        r.data = data;
        return r;
    }

    public static <T> Result<T> error(int code, String message) {
        Result<T> r = new Result<>();
        r.code = code;
        r.message = message;
        return r;
    }
}

// com.example.mapper.UserMapper.java
@Mapper
public interface UserMapper extends BaseMapper<User> {}

// com.example.entity.User.java
@TableName("t_user")
public class User {
    @TableId private Long id;
    private String username;
    private String email;
    private Integer status;
}
```

**用户需求**："帮我写一个根据邮箱查询用户的接口"

**无 RAG 的生成结果**：

```java
@RestController
@RequestMapping("/api")
public class UserController {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;  // ❌ 项目用的是 MyBatis-Plus
    
    @GetMapping("/users/by-email")
    public User getUserByEmail(@RequestParam String email) {
        return jdbcTemplate.queryForObject(  // ❌ 没使用 BaseMapper
            "SELECT * FROM users WHERE email=?", User.class, email);
    }
    // ❌ 没有统一返回体 Result
}
```

**有 RAG 的生成结果**：

```java
@RestController
@RequestMapping("/api/user")
public class UserController {
    
    @Autowired
    private UserMapper userMapper;  // ✅ 使用了项目已有的 Mapper
    
    @GetMapping("/by-email")
    public Result<User> getUserByEmail(@RequestParam String email) {
        // ✅ 使用了项目的统一返回体
        User user = userMapper.selectOne(
            new LambdaQueryWrapper<User>()
                .eq(User::getEmail, email)
                .eq(User::getStatus, 1)
        );
        
        if (user == null) {
            return Result.error(404, "用户不存在");  // ✅ 使用了项目的 Result
        }
        return Result.success(user);
    }
}
```

高下立判。有 RAG 的代码直接可以在项目中编译运行，不需要二次修改。

## 七、生产环境注意事项

### 7.1 增量索引

项目代码每天都在变，每次全量索引不现实：

```java
public class IncrementalIndexer {

    private final Map<String, String> fileHashCache = new ConcurrentHashMap<>();

    public void incrementalIndex(Path projectRoot) throws IOException {
        try (var stream = Files.walk(projectRoot)) {
            stream.filter(p -> p.toString().endsWith(".java"))
                .forEach(file -> {
                    String newHash = computeFileHash(file);
                    String oldHash = fileHashCache.get(file.toString());

                    if (!newHash.equals(oldHash)) {
                        // 文件有变更，重新索引
                        System.out.println("检测到变更: " + file);
                        reindexFile(file);
                        fileHashCache.put(file.toString(), newHash);
                    }
                });
        }
    }

    private String computeFileHash(Path file) {
        try {
            byte[] bytes = Files.readAllBytes(file);
            java.security.MessageDigest md = java.security.MessageDigest.getInstance("SHA-256");
            byte[] digest = md.digest(bytes);
            return java.util.HexFormat.of().formatHex(digest);
        } catch (Exception e) {
            return "";
        }
    }

    private void reindexFile(Path file) {
        // 删除旧索引 → 重新切割 → 重新生成 Embedding → 重新插入
    }
}
```

### 7.2 权限控制

不同团队的开发者只能检索自己有权限的代码：

```java
public class PermissionAwareRetriever {

    public List<VectorDocument> retrieve(String query, String userId) {
        List<VectorDocument> candidates = basicRetrieve(query);

        // 根据用户的团队/角色过滤代码
        return candidates.stream()
            .filter(doc -> hasPermission(userId, doc))
            .toList();
    }

    private boolean hasPermission(String userId, VectorDocument doc) {
        // 从代码仓库的权限管理系统中查询
        // 例如：通过 GitLab API 查询该文件的可见性
        return true;
    }
}
```

### 7.3 跨仓库检索

大型项目可能跨多个 Git 仓库：

```java
public class MultiRepoRetriever {

    private final Map<String, CodeRetriever> retrievers = new ConcurrentHashMap<>();

    public void registerRepo(String repoName, Path repoPath) throws IOException {
        // 每个仓库独立的索引空间
        CodeRetriever retriever = buildRetriever(repoPath);
        retrievers.put(repoName, retriever);
    }

    public List<VectorDocument> retrieveAcrossRepos(String query, Set<String> repoNames) {
        // 从多个仓库检索
        return repoNames.stream()
            .flatMap(repo -> retrievers.get(repo).retrieve(query).stream())
            .sorted((a, b) -> Double.compare(b.similarity(), a.similarity()))
            .limit(10)
            .toList();
    }
}
```

## 八、总结

代码库 RAG 让 AI 真正"认识"了你的项目。核心要点：

1. **切割粒度**：以方法为单位切割 + 保留类/包上下文信息
2. **语义增强**：在 Embedding 前给代码拼接注解描述、驼峰拆分等自然语言标签
3. **检索重排序**：综合向量相似度 + 代码结构特征 + 领域相关性
4. **增量更新**：基于文件 Hash 的增量索引，适应代码频繁变更

有了代码库 RAG，你再也不用跟 AI 说"请用我们项目的 Result 类封装返回值"——它自己就知道了。

> 下一篇预告：**API 文档 RAG：自动回答"这个接口怎么用"的内部助手**——我们将把 Swagger/OpenAPI 文档向量化，做一个主动回答"这个接口参数是什么？返回值是什么？"的智能助手，从此前后端联调不再吵架！
