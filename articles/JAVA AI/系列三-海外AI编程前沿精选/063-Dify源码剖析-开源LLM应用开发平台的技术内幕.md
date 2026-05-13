# Dify 源码剖析：开源 LLM 应用开发平台的技术内幕，60K Star 背后的架构智慧

> 如果你关注 AI 应用开发，Dify 这个名字大概率不会陌生。GitHub 60K+ Star，国内最成功的开源 LLM 应用平台之一，被大量企业用于搭建智能客服、知识库问答、AI 工作流等场景。但你有没有想过：**一个让你拖拽几下就能跑起来的 AI 应用，背后到底藏着怎样的架构智慧？**

今天我们就来庖丁解牛，从源码层面拆解 Dify 的四大核心设计。

---

## 一、工作流引擎：DAG 图编排设计

Dify 的工作流引擎是整个平台的"大脑"。它的本质是一个 **DAG（有向无环图）编排系统**，让你用拖拽的方式把 LLM、代码执行、知识检索、条件判断等节点串联成一条完整的业务流水线。

### 1.1 核心数据模型

在源码中，工作流定义被序列化为以下结构：

```yaml
app:
  mode: advanced-chat  # 或 workflow
  workflow:
    graph:
      nodes:
        - id: "llm_1"
          type: "llm"
          data:
            model: { provider: "openai", name: "gpt-4" }
            prompt_template: [{ role: "system", text: "你是一个专业助手" }]
          position: { x: 100, y: 200 }
        - id: "knowledge_retrieval_1"
          type: "knowledge-retrieval"
          data:
            dataset_ids: ["dataset-uuid-1"]
            retrieval_mode: "multiple"
          position: { x: 100, y: 400 }
        - id: "code_1"
          type: "code"
          data:
            code_language: "python3"
            code: |
              def main(arg1: str) -> dict:
                  return {"result": arg1.upper()}
          position: { x: 400, y: 200 }
      edges:
        - id: "edge_1"
          source: "start"
          target: "llm_1"
        - id: "edge_2"
          source: "llm_1"
          target: "code_1"
          sourceHandle: "source"
          targetHandle: "target"
```

每个节点本质上是一个 **待执行的函数单元**，边定义了数据的流向。Dify 前端用 ReactFlow 渲染这个 DAG，后端用 Python 解析并执行。

### 1.2 执行引擎源码解析

核心执行逻辑在 `core/workflow/workflow_entry.py` 中：

```python
class WorkflowEntry:
    def __init__(self, tenant_id: str, app_id: str, workflow: Workflow):
        self.graph = Graph.init(workflow.graph)  # 构建 DAG
        self.graph_engine = GraphEngine(
            self.graph,
            self._get_node_runner  # 节点执行器工厂
        )

    def run(self, inputs: dict) -> Generator[GraphEvent, None, None]:
        """流式执行整个工作流 DAG"""
        for event in self.graph_engine.run():
            if event.type == "node_started":
                yield self._wrap_node_start(event)
            elif event.type == "node_finished":
                yield self._wrap_node_finished(event)
            elif event.type == "text_chunk":
                yield self._wrap_text_chunk(event)
```

关键设计点：

1. **拓扑排序执行**：`GraphEngine` 内部先对节点做拓扑排序，确定执行顺序，然后按批次调度。同一批次内无依赖关系的节点可以并行执行。

2. **变量上下文传递**：每个节点的输出自动写入 `WorkflowRuntimeState`，下游节点通过 `${node_id.output_field}` 语法引用上游的输出：

```python
class VariablePool:
    """工作流变量池，类似 Spring 的 BeanFactory"""
    def __init__(self):
        self._variables: dict[str, Any] = {}

    def add(self, node_id: str, outputs: dict):
        for key, value in outputs.items():
            self._variables[f"{node_id}.{key}"] = value

    def resolve(self, template: str) -> str:
        """将 ${node_id.key} 模板替换为实际值"""
        return re.sub(
            r'\$\{(\w+)\.(\w+)\}',
            lambda m: str(self._variables.get(f"{m.group(1)}.{m.group(2)}", m.group(0))),
            template
        )
```

3. **节点类型插件化**：每种节点（LLM、代码、知识检索、HTTP 请求等）都实现了统一的 `NodeRunner` 接口：

```python
class NodeRunner(ABC):
    @abstractmethod
    def run(self, node_data: NodeData, inputs: dict) -> NodeRunResult:
        """执行节点逻辑，返回结果"""
        pass
```

### 1.3 对 Java 开发者的启示

这套设计本质上就是一个 **流程编排引擎**。如果你用 Java 实现类似功能，完全可以参考 Spring State Machine 或自定义 DAG 框架：

```java
// Java 开发者视角的工作流节点抽象
public interface WorkflowNode {
    String getId();
    String getType();
    Map<String, Object> execute(Map<String, Object> inputs, VariableContext ctx);
}

// DAG 执行器
public class DagExecutor {
    public void execute(DagGraph graph, Map<String, Object> initialInputs) {
        List<WorkflowNode> sorted = TopologicalSort.sort(graph);
        Map<String, Map<String, Object>> nodeOutputs = new ConcurrentHashMap<>();
        for (WorkflowNode node : sorted) {
            Map<String, Object> inputs = graph.getIncomingEdges(node.getId())
                .stream()
                .flatMap(e -> nodeOutputs.getOrDefault(e.getSource(), Map.of()).entrySet().stream())
                .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
            Map<String, Object> result = node.execute(inputs, new VariableContext(nodeOutputs));
            nodeOutputs.put(node.getId(), result);
        }
    }
}
```

---

## 二、多模型适配层：统一接入方案

Dify 支持接入 OpenAI、Anthropic、Azure、百川、文心一言、通义千问等 100+ 模型——这背后是一套精巧的 **Provider-Model 两层抽象**。

### 2.1 架构设计

```
┌──────────────────────────────────────┐
│          ModelRuntime (统一入口)       │
├──────────────────────────────────────┤
│  Provider A (OpenAI)                 │
│  ├── LLM: gpt-4, gpt-3.5-turbo      │
│  ├── TextEmbedding: text-embedding   │
│  └── Rerank: (none)                  │
├──────────────────────────────────────┤
│  Provider B (Anthropic)              │
│  ├── LLM: claude-3-opus             │
│  └── ...                             │
├──────────────────────────────────────┤
│  Provider C (Tongyi / 通义千问)      │
│  ├── LLM: qwen-max, qwen-turbo      │
│  └── TextEmbedding: text-embedding   │
└──────────────────────────────────────┘
```

### 2.2 核心接口定义

```python
class LargeLanguageModel(BaseModelInstance):
    """所有 LLM 的统一抽象"""
    model_type: str = "llm"

    def invoke(
        self,
        model: str,
        credentials: dict,
        prompt_messages: list[PromptMessage],
        model_parameters: dict | None = None,
        tools: list[PromptMessageTool] | None = None,
        stop: list[str] | None = None,
        stream: bool = False,
    ) -> LLMResult | Generator:
        pass

    def get_num_tokens(self, model: str, credentials: dict,
                       prompt_messages: list[PromptMessage]) -> int:
        pass
```

每个 Provider 实现这一套接口。以 OpenAI 为例：

```python
class OpenAILargeLanguageModel(LargeLanguageModel):
    def _invoke(self, model: str, credentials: dict,
                prompt_messages: list[PromptMessage], ...) -> LLMResult:
        client = OpenAI(api_key=credentials["openai_api_key"])
        response = client.chat.completions.create(
            model=model,
            messages=[self._convert_prompt_message(m) for m in prompt_messages],
            **model_parameters
        )
        return self._convert_response(response)
```

### 2.3 模型参数标准化

Dify 把各家模型的温度、top_p、max_tokens 等参数做了统一映射：

```python
class ModelParameterRule:
    name: str = "temperature"
    type: str = "float"
    default: float = 0.7
    min: float = 0.0
    max: float = 2.0
    precision: int = 2
```

每家 Provider 可以覆盖这些参数的名称和范围，系统运行时自动转换为该模型的真实参数名。

### 2.4 对 Java 开发者的启示

这就是典型的 **策略模式 + 适配器模式**。Java 实现可以用接口 + SPI（Service Provider Interface），让模型厂商以插件方式接入：

```java
public interface LlmProvider {
    String getProviderName();
    List<ModelType> supportedModelTypes();
    CompletionResult invoke(String model, List<Message> messages,
                            ModelParameters params);
    Stream<CompletionChunk> stream(String model, List<Message> messages,
                                   ModelParameters params);
}
```

通过 SPI 机制，新增模型只需引入一个 JAR 包并在 `META-INF/services` 注册即可，无需修改核心代码。

---

## 三、RAG 管道：全链路设计

Dify 的知识库问答能力依赖完整的 RAG（检索增强生成）管道。这一条链路涉及四个核心阶段：

### 3.1 文档处理管线

```
原始文档 → 解析(PDF/DOCX/MD/Web) → 清洗 → 分块 → 向量化 → 入库
```

```python
class IndexingRunner:
    """文档索引入口"""
    def run(self, dataset_documents: list[Document]):
        for doc in dataset_documents:
            # 1. 解析文档
            text = self._extract_text(doc)

            # 2. 智能分块 — 兼顾语义和长度
            chunks = self._split_text(
                text,
                chunk_size=500,         # 默认 500 token
                chunk_overlap=50,        # 重叠 50 token
                separator="\n\n"         # 优先按段落分割
            )

            # 3. 向量化 Embedding
            embeddings = self._embed_chunks(chunks)

            # 4. 写入向量数据库
            self.vector_store.add(chunks, embeddings)
```

分块策略是 RAG 效果的关键。Dify 支持自定义分隔符、块大小、重叠长度，甚至可以按 Markdown 标题、代码函数边界做智能切分。

### 3.2 检索与重排序

```python
class RetrievalService:
    def retrieve(self, query: str, dataset_ids: list[str],
                 top_k: int = 5) -> list[RetrievedDocument]:
        # 1. 查询向量化
        query_embedding = self.embedding_model.embed(query)

        # 2. 向量检索（支持 Milvus / Weaviate / Qdrant / Pinecone 等）
        candidates = self.vector_store.search(
            query_embedding, dataset_ids, top_k * 3
        )

        # 3. 重排序（Rerank）- 提升相关性
        if self.rerank_model:
            candidates = self.rerank_model.rerank(query, candidates)[:top_k]

        # 4. 返回最终结果
        return candidates
```

这里有一个巧妙的设计——**混合检索**。向量搜索抓语义，BM25 关键词搜索抓精确匹配，两者结果融合后送入 Rerank 模型排序：

```python
def hybrid_search(self, query: str, alpha: float = 0.7):
    vector_results = self.vector_search(query)
    keyword_results = self.bm25_search(query)
    # RRF（Reciprocal Rank Fusion）融合
    return self.rrf_fusion(vector_results, keyword_results, alpha)
```

### 3.3 对 Java 开发者的启示

Java 生态中，Spring AI 已经内置了类似的 RAG 抽象（`DocumentReader`、`DocumentTransformer`、`VectorStore` 等）。但 Dify 的 `chunk_overlap` + `separator` + `rerank` 组合拳值得所有做知识库的开发者借鉴。

---

## 四、多租户隔离方案

Dify 作为 SaaS 平台，多租户隔离是硬需求。它的方案简洁务实：

### 4.1 数据库层隔离

采用 **租户 ID 软隔离**——所有业务表都带 `tenant_id` 字段：

```sql
CREATE TABLE apps (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    mode VARCHAR(50) NOT NULL,
    -- ...
    INDEX idx_tenant_id (tenant_id)
);

CREATE TABLE datasets (
    id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    -- ...
    INDEX idx_tenant_id (tenant_id)
);
```

### 4.2 中间件自动注入

Dify 用 Flask 中间件从 JWT Token 解析当前租户上下文，并在每次数据库查询时自动带上 `tenant_id` 过滤条件：

```python
class TenantContextFilter:
    """自动为所有数据库查询添加 tenant_id 过滤"""
    def filter_query(self, query, tenant_id):
        # 利用 SQLAlchemy 的事件机制
        # 在每次查询前注入 WHERE tenant_id = ?
        return query.filter_by(tenant_id=tenant_id)
```

### 4.3 对 Java 开发者的启示

Java 版的实现思路：

```java
// 1. 用 ThreadLocal 传递租户上下文
public class TenantContext {
    private static final ThreadLocal<String> CURRENT_TENANT = new ThreadLocal<>();
    public static void set(String tenantId) { CURRENT_TENANT.set(tenantId); }
    public static String get() { return CURRENT_TENANT.get(); }
    public static void clear() { CURRENT_TENANT.remove(); }
}

// 2. MyBatis-Plus 拦截器自动注入租户 ID
@Component
public class TenantInterceptor implements InnerInterceptor {
    @Override
    public void beforeQuery(Executor executor, MappedStatement ms,
                            Object parameter, RowBounds rowBounds,
                            ResultHandler resultHandler, BoundSql boundSql) {
        String tenantId = TenantContext.get();
        if (tenantId != null) {
            boundSql.setSql(boundSql.getSql() + " AND tenant_id = '" + tenantId + "'");
        }
    }
}
```

---

## 五、架构总结：Java 开发者能学到什么

回顾 Dify 的整体架构，五条核心思想值得每个后端开发者深思：

| 设计要点 | Dify 的做法 | Java 世界的对应 |
|---------|------------|----------------|
| 工作流编排 | DAG + 拓扑排序 + 事件驱动 | Spring State Machine / Camunda |
| 模型适配 | Provider-Model 两层策略模式 | SPI + 策略模式 |
| RAG 管道 | 分段-向量化-检索-重排全链路 | Spring AI Document Pipeline |
| 多租户 | DB 软隔离 + 中间件自动注入 | MyBatis TenantInterceptor |
| 插件化 | 节点/模型/工具均可插拔 | OSGI / Spring Plugin |

**最大的感悟**：Dify 没有发明任何新技术，它的牛X之处在于把 **LLM 调用、向量检索、工作流编排、多租户权限** 这些成熟技术做了教科书级别的集成。每一个模块抽出来都不复杂，但组合在一起形成了一个真正可用的产品。

---

## 下篇预告

理解了 Dify 的 LLM 编排层，你可能会好奇：**这些大模型到底是怎么在本地跑起来的？** 下一篇文章，我们深入拆解 **Ollama 源码**——它是如何让上万开发者在自己的 MacBook 上流畅运行 Llama 3、DeepSeek 等主流开源模型的。点关注，不错过！

---

*本文基于 Dify v0.6.x 源码分析，部分代码经过简化以突出核心设计。欢迎在评论区讨论你的看法。*
