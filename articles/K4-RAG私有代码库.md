# RAG私有代码库深度解析：企业级代码检索增强生成的架构与实战

**文章标签：** #ai #rag #代码库 #企业应用 #向量检索 #代码检索 #私有部署 #llm-rag

## 目录

- [引言：RAG的工程本质与代码库的特殊性](#引言rag的工程本质与代码库的特殊性)
- [理论基础：为什么RAG能解决代码库问答](#理论基础为什么rag能解决代码库问答)
- [演进史：从文档RAG到代码RAG的技术跃迁](#演进史从文档rag到代码rag的技术跃迁)
- [深度解析：代码RAG的核心架构与算法原理](#深度解析代码rag的核心架构与算法原理)
- [实战案例：五个企业级代码RAG系统实现](#实战案例五个企业级代码rag系统实现)
- [对比分析：代码RAG vs 文档RAG vs 通用RAG](#对比分析代码rag-vs-文档rag-vs-通用rag)
- [性能分析：检索质量、生成质量与系统性能](#性能分析检索质量生成质量与系统性能)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：RAG的工程本质与代码库的特殊性

RAG（Retrieval-Augmented Generation）不是简单的"先搜索再生成"，而是一门**将外部知识库的概率分布注入到语言模型生成过程**的工程技术。

核心认知：

```
纯生成的本质：P(answer | question, model_knowledge)

RAG的本质：P(answer | question, retrieved_context, model_knowledge)
            = ∫ P(answer | question, context) × P(context | question, corpus) dcontext

代码库RAG的特殊性：
- 知识源是结构化代码，不是自然语言文档
- 需要理解代码语义、调用关系、类型系统
- 检索单元是函数/类/模块，不是段落
- 答案需要精确，不能模糊（代码必须可运行）

质量差异的根源：
- 差RAG：检索到无关代码片段 → 模型基于错误上下文生成 → 答案错误
- 好RAG：精确检索到相关代码 → 模型基于准确上下文生成 → 答案正确
```

**关键洞察**：代码RAG的效果不取决于"向量数据库的选择"，而取决于**代码理解粒度**与**检索语义匹配**的精度。

---

## 理论基础：为什么RAG能解决代码库问答

### 1. 语言模型的知识边界与代码理解

#### 代码知识的两个来源

```python
"""
语言模型的代码知识来源：

来源1：预训练数据中的代码
- GitHub公共仓库
- Stack Overflow问答
- 技术文档

局限性：
- 知识截止于训练数据时间
- 不了解私有代码库
- 不了解企业内部框架
- 不了解项目特定约定

来源2：RAG检索到的代码
- 实时从代码库检索
- 精确到函数/类级别
- 包含最新代码
- 包含私有实现

RAG的价值：
P(correct_answer | RAG) = P(correct_answer | retrieval) × P(retrieval | question)

当 P(retrieval | question) → 1（精确检索）时，
P(correct_answer | RAG) → P(correct_answer | ground_truth)
"""
```

#### 代码语义的数学表示

```python
"""
代码的语义表示：

传统文本表示：
text → tokens → embedding vector
问题：丢失了代码结构信息

代码语义表示：
code → AST（抽象语法树） → graph embedding
     → 控制流图（CFG） → path embedding
     → 数据流图（DFG） → flow embedding

联合表示：
code_embedding = α × text_embedding + β × ast_embedding + γ × graph_embedding

其中：
- text_embedding：代码文本的语义（变量名、注释）
- ast_embedding：语法结构（函数调用、类继承）
- graph_embedding：程序依赖图（调用关系、数据依赖）

最优权重（经验值）：
- 问答任务：α=0.5, β=0.3, γ=0.2
- 代码生成：α=0.3, β=0.4, γ=0.3
- Bug修复：α=0.3, β=0.3, γ=0.4
"""

class CodeSemanticEncoder:
    """代码语义编码器"""
    
    def __init__(self, weights={'text': 0.5, 'ast': 0.3, 'graph': 0.2}):
        self.weights = weights
        self.text_encoder = None  # 如BERT、CodeBERT
        self.ast_encoder = None   # 如Tree-LSTM、Graph Neural Network
    
    def encode(self, code: str, language: str = 'python') -> np.ndarray:
        """编码代码为语义向量"""
        # 1. 文本编码
        text_emb = self._encode_text(code)
        
        # 2. AST编码
        ast_emb = self._encode_ast(code, language)
        
        # 3. 图编码（调用关系等）
        graph_emb = self._encode_graph(code, language)
        
        # 加权融合
        combined = (
            self.weights['text'] * text_emb +
            self.weights['ast'] * ast_emb +
            self.weights['graph'] * graph_emb
        )
        
        return combined
    
    def _encode_text(self, code: str) -> np.ndarray:
        """文本语义编码"""
        # 使用CodeBERT或类似模型
        pass
    
    def _encode_ast(self, code: str, language: str) -> np.ndarray:
        """AST结构编码"""
        # 使用tree-sitter解析AST，然后编码
        pass
    
    def _encode_graph(self, code: str, language: str) -> np.ndarray:
        """程序依赖图编码"""
        # 构建调用图、数据流图
        pass
```

### 2. 检索增强的数学原理

#### 向量检索的概率解释

```python
"""
向量检索的概率解释：

给定查询q和文档集合D = {d_1, d_2, ..., d_n}

传统BM25：
Score(q, d) = Σ_{t∈q} IDF(t) × [f(t,d) × (k1+1)] / [f(t,d) + k1 × (1-b+b×|d|/avgdl)]

稠密向量检索：
Score(q, d) = cosine_similarity(embed(q), embed(d))
            = embed(q) · embed(d) / (||embed(q)|| × ||embed(d)||)

概率解释：
P(d | q) ∝ exp(embed(q) · embed(d) / τ)

其中τ是温度参数：
- τ → 0：硬检索（只返回最相似的）
- τ → ∞：均匀分布（所有文档等概率）

后验概率（贝叶斯视角）：
P(d | q) = P(q | d) × P(d) / P(q)

其中：
- P(q | d) ∝ exp(embed(q) · embed(d))  （似然）
- P(d)：文档先验（可考虑文档重要性、新鲜度）
- P(q)：查询先验（归一化常数）
"""
```

#### 重排序的数学原理

```python
"""
两阶段检索的概率解释：

阶段1：召回（Retrieval）
目标：高召回率，快速过滤
方法：近似最近邻（ANN）
返回：Top-K候选（K=100~1000）

阶段2：精排（Reranking）
目标：高精确率，精确排序
方法：交叉编码器（Cross-Encoder）
返回：Top-k最终答案（k=5~10）

为什么两阶段有效？

计算复杂度：
- 稠密检索：O(N × d) 其中N是文档数，d是向量维度
- 交叉编码器：O(K × L × d) 其中K<<N，L是序列长度

准确率：
- 召回阶段：Recall@100 > 90%（找到相关文档）
- 精排阶段：NDCG@5 > 0.85（精确排序）

联合概率：
P(relevant | q) = Σ_{d∈retrieved} P(relevant | d, q) × P(d | q)

其中：
- P(d | q)：召回阶段计算的相似度
- P(relevant | d, q)：精排阶段计算的相关性
"""

class TwoStageRetriever:
    """两阶段检索器"""
    
    def __init__(self, vector_store, reranker):
        self.vector_store = vector_store  # 向量数据库
        self.reranker = reranker          # 重排序模型
        self.recall_k = 100               # 召回数量
        self.final_k = 5                  # 最终返回数量
    
    def retrieve(self, query: str) -> list:
        """两阶段检索"""
        
        # 阶段1：召回
        candidates = self.vector_store.similarity_search(
            query, 
            k=self.recall_k
        )
        
        # 阶段2：精排
        reranked = self.reranker.rerank(query, candidates)
        
        # 返回Top-k
        return reranked[:self.final_k]
    
    def rerank_with_cross_encoder(self, query: str, candidates: list) -> list:
        """
        使用交叉编码器重排序
        
        交叉编码器 vs 双编码器：
        - 双编码器：分别编码查询和文档，点积计算相似度
          优点：速度快，可预计算文档向量
          缺点：查询和文档没有交互，精度较低
        
        - 交叉编码器：将查询和文档拼接输入模型
          优点：查询和文档深度交互，精度高
          缺点：速度慢，无法预计算
        """
        scores = []
        
        for doc in candidates:
            # 交叉编码：拼接查询和文档
            pair_text = f"Query: {query} Document: {doc.content}"
            
            # 编码（实际应调用模型）
            score = self.reranker.encode(pair_text)
            scores.append((score, doc))
        
        # 按分数排序
        scores.sort(key=lambda x: x[0], reverse=True)
        
        return [doc for _, doc in scores]
```

### 3. 上下文窗口与信息融合

```python
"""
上下文窗口的约束与优化：

模型上下文限制：
- GPT-4: 128K tokens
- Claude 4: 200K tokens
- DeepSeek-V4: 128K tokens
- Kimi K2.6: 1M tokens

代码问答的上下文需求：
- 单个函数：50-200 tokens
- 相关函数（调用链）：500-2000 tokens
- 类/模块：1000-5000 tokens
- 完整上下文（跨文件）：可能超过10K tokens

上下文构造策略：

策略1：截断（Truncation）
- 按重要性排序，保留Top-K
- 简单但可能丢失关键信息

策略2：压缩（Compression）
- 使用小模型摘要长代码
- 保留关键逻辑，去除实现细节

策略3：层次化（Hierarchical）
- 第一层：模块/类级别的摘要
- 第二层：函数级别的详细代码
- 第三层：关键行级别的注释

策略4：动态选择（Dynamic）
- 根据问题类型选择上下文粒度
- "这个函数做什么" → 函数级
- "这个模块如何工作" → 模块级
- "为什么报这个错" → 调用链级

信息融合公式：
context = concat(
    system_prompt,
    retrieved_code_1,
    retrieved_code_2,
    ...,
    user_question
)

目标：maximize P(answer | context) 
subject to: len(tokens(context)) ≤ context_limit
"""

class ContextAssembler:
    """上下文组装器"""
    
    def __init__(self, max_tokens: int = 8000):
        self.max_tokens = max_tokens
        self.tokenizer = None  # 实际应加载tokenizer
    
    def assemble(self, query: str, retrieved_chunks: list) -> str:
        """组装上下文"""
        
        # 系统提示
        system_prompt = """你是一个代码助手。基于以下代码片段回答用户问题。
请引用具体的代码文件和行号。如果信息不足，请明确说明。"""
        
        # 按相关性排序
        sorted_chunks = sorted(
            retrieved_chunks, 
            key=lambda x: x['score'], 
            reverse=True
        )
        
        # 逐步添加代码片段，直到接近token限制
        context_parts = [system_prompt]
        current_tokens = self._count_tokens(system_prompt)
        
        for chunk in sorted_chunks:
            chunk_text = self._format_chunk(chunk)
            chunk_tokens = self._count_tokens(chunk_text)
            
            if current_tokens + chunk_tokens + 100 < self.max_tokens:
                context_parts.append(chunk_text)
                current_tokens += chunk_tokens
            else:
                break
        
        # 添加用户问题
        context_parts.append(f"\n问题：{query}")
        
        return "\n\n".join(context_parts)
    
    def _format_chunk(self, chunk: dict) -> str:
        """格式化代码片段"""
        return f"""文件：{chunk['metadata']['file_path']}
行号：{chunk['metadata']['start_line']}-{chunk['metadata']['end_line']}
类型：{chunk['metadata']['type']}
```
{chunk['content']}
```"""
    
    def _count_tokens(self, text: str) -> int:
        """计算token数"""
        # 简化估算：1 token ≈ 0.75个汉字或4个英文字符
        return len(text) // 3
```

---

## 演进史：从文档RAG到代码RAG的技术跃迁

### 第一阶段：基于关键词的代码搜索（2000-2015）

```python
"""
早期代码搜索：

技术：
- 正则表达式匹配
- 倒排索引（Inverted Index）
- 布尔检索（AND/OR/NOT）

代表工具：
- grep
- ack/ag
- OpenGrok
- Sourcegraph（早期）

局限性：
1. 无法理解语义
   搜索"sort"只能找到包含"sort"的代码
   无法理解"order"、"排列"也是相关概念

2. 无法处理自然语言查询
   用户问"怎么实现排序" → 无法直接回答
   只能返回包含"排序"关键词的代码

3. 缺乏上下文理解
   无法理解调用关系
   无法区分不同模块中的同名函数

示例：
用户查询："订单处理逻辑"
搜索结果：所有包含"订单"或"处理"的文件
问题：返回大量无关结果，需要人工筛选
"""
```

### 第二阶段：基于语义的代码搜索（2015-2020）

```python
"""
语义代码搜索时代：

技术突破：
1. 词嵌入（Word2Vec, GloVe）
   - 将代码token映射到向量空间
   - 相似语义的token距离近

2. 代码表示学习
   - code2vec：将代码结构表示为向量
   - CodeBERT：预训练代码语言模型

3. 神经网络排序
   - 使用深度学习模型排序搜索结果
   - 考虑代码结构和语义

代表工作：
- code2vec（Alon et al., 2019）
- CodeBERT（Feng et al., 2020）
- GraphCodeBERT（Guo et al., 2021）

进步：
- 可以理解"排序"和"order by"语义相关
- 可以处理简单的自然语言查询
- 搜索结果质量提升

局限：
- 仍然缺乏生成能力
- 无法理解复杂的多轮对话
- 无法解释代码逻辑

示例：
用户查询："排序算法实现"
搜索结果：返回quicksort、mergesort等实现
进步：可以理解"排序"和"sort"的语义关联
"""
```

### 第三阶段：LLM+检索的代码问答（2020-2023）

```python
"""
LLM+检索时代：

技术突破：
1. GPT-3/ChatGPT的代码能力
   - 可以理解自然语言问题
   - 可以生成代码解释
   - 可以进行代码对话

2. RAG架构成熟
   - 向量数据库（Pinecone, Milvus, Weaviate）
   - Embedding模型（text-embedding-ada-002）
   - 检索+生成流水线

3. 代码专用工具
   - GitHub Copilot（代码补全）
   - Sourcegraph Cody（代码问答）
   - Replit Ghostwriter（代码生成）

架构：
用户问题 → Embedding → 向量检索 → 代码片段
                                      ↓
                              ┌───────────────┐
                              │  Prompt构造   │
                              │  问题+代码    │
                              └───────┬───────┘
                                      ↓
                              ┌───────────────┐
                              │   LLM生成     │
                              │   答案        │
                              └───────────────┘

进步：
- 可以回答"这个函数做什么"
- 可以解释代码逻辑
- 可以进行多轮对话

局限：
- 检索质量不稳定
- 大模型可能 hallucinate
- 缺乏精确的代码引用
"""
```

### 第四阶段：代码专用RAG（2023-2024）

```python
"""
代码专用RAG时代：

技术突破：
1. 代码专用Embedding
   - code-embedding模型（jina-embeddings-v3）
   - 考虑代码结构和语义
   - 支持多种编程语言

2. 代码专用分块策略
   - AST-based分块
   - 函数级、类级、模块级分块
   - 保留调用关系

3. 代码专用检索
   - 混合检索（关键词+语义）
   - 代码图检索（调用关系）
   - 类型感知的检索

4. Agent-based代码助手
   - 可以执行代码
   - 可以浏览文件系统
   - 可以进行多步推理

代表工具：
- Cursor（AI代码编辑器）
- Continue（开源AI助手）
- Aider（命令行AI助手）
- Devin（AI软件工程师）

架构升级：
用户问题 → 意图识别 → 检索策略选择 → 多路检索
                                              ↓
                              ┌───────────────────────────────┐
                              │  多路检索结果融合              │
                              │  - 语义检索结果               │
                              │  - 关键词检索结果             │
                              │  - 调用图检索结果             │
                              └───────────────┬───────────────┘
                                              ↓
                              ┌───────────────────────────────┐
                              │  上下文组装                   │
                              │  - 去重                       │
                              │  - 排序                       │
                              │  - 截断/压缩                  │
                              └───────────────┬───────────────┘
                                              ↓
                              ┌───────────────────────────────┐
                              │  LLM生成                      │
                              │  - 带引用的答案               │
                              │  - 可验证的代码               │
                              └───────────────────────────────┘
"""
```

### 第五阶段：2026年的企业级代码RAG

```python
"""
2026年企业级代码RAG特征：

1. 多模态代码理解
   - 代码 + 文档 + 图表联合理解
   - GLM-5/Gemini 2.0支持代码+架构图
   - 自然语言+代码+可视化的统一表示

2. 实时代码同步
   - 每次commit自动更新索引
   - 增量索引（只更新变更部分）
   - 秒级延迟

3. 深度代码分析
   - 静态分析集成（SonarQube、CodeQL）
   - 动态分析集成（测试覆盖率、性能分析）
   - 代码质量评分

4. 个性化代码助手
   - 学习个人编码风格
   - 学习团队编码规范
   - 学习项目特定约定

5. 安全与合规
   - 私有化部署（数据不出境）
   - 细粒度权限控制
   - 审计日志

6. 多Agent协作
   - 代码检索Agent
   - 代码分析Agent
   - 代码生成Agent
   - 代码审查Agent
   - 多Agent协作完成复杂任务
"""
```

---

## 深度解析：代码RAG的核心架构与算法原理

### 1. 代码解析与分块策略

#### AST-based代码分块

```python
"""
代码分块的核心挑战：

文本分块的问题：
- 按字符/行分块可能切断函数/类
- 丢失语法结构信息
- 无法理解代码边界

AST-based分块的优势：
- 以语法单元为边界（函数、类、方法）
- 保留完整的语义单元
- 便于后续分析（调用关系、依赖关系）

分块粒度选择：
- 细粒度（函数级）：检索精确，但可能丢失上下文
- 中粒度（类级）：平衡精确性和上下文
- 粗粒度（文件级）：上下文完整，但噪声多

最优策略：多层次索引
- 函数级索引（精确检索）
- 类级索引（中等范围）
- 文件级索引（ broad recall）
"""

import tree_sitter
from tree_sitter import Language, Parser

class ASTCodeChunker:
    """基于AST的代码分块器"""
    
    def __init__(self, language: str = 'python'):
        self.language = language
        self.parser = self._initialize_parser(language)
    
    def _initialize_parser(self, language: str):
        """初始化tree-sitter解析器"""
        # 实际应加载对应语言的so文件
        # LANGUAGE = Language('build/my-languages.so', language)
        # parser = Parser()
        # parser.set_language(LANGUAGE)
        return Parser()
    
    def chunk(self, code: str, file_path: str) -> list:
        """
        将代码分块
        
        返回：
        [
            {
                'content': '代码内容',
                'metadata': {
                    'file_path': '...',
                    'type': 'function'|'class'|'module',
                    'name': '函数/类名',
                    'start_line': 10,
                    'end_line': 50,
                    'dependencies': ['dep1', 'dep2']
                }
            }
        ]
        """
        tree = self.parser.parse(bytes(code, 'utf8'))
        root_node = tree.root_node
        
        chunks = []
        
        # 遍历AST，提取函数和类
        for node in root_node.children:
            if node.type in ['function_definition', 'class_definition']:
                chunk = self._extract_chunk(node, code, file_path)
                if chunk:
                    chunks.append(chunk)
        
        return chunks
    
    def _extract_chunk(self, node, code: str, file_path: str) -> dict:
        """提取单个代码块"""
        start_line = node.start_point[0] + 1
        end_line = node.end_point[0] + 1
        content = code[node.start_byte:node.end_byte]
        
        # 提取名称
        name = self._extract_name(node)
        
        # 提取依赖
        dependencies = self._extract_dependencies(node, code)
        
        # 确定类型
        chunk_type = 'function' if node.type == 'function_definition' else 'class'
        
        return {
            'content': content,
            'metadata': {
                'file_path': file_path,
                'type': chunk_type,
                'name': name,
                'start_line': start_line,
                'end_line': end_line,
                'dependencies': dependencies,
                'language': self.language
            }
        }
    
    def _extract_name(self, node) -> str:
        """提取函数/类名"""
        for child in node.children:
            if child.type == 'identifier':
                return child.text.decode('utf8')
        return 'anonymous'
    
    def _extract_dependencies(self, node, code: str) -> list:
        """提取代码块的依赖（调用的函数、使用的类等）"""
        dependencies = []
        
        # 遍历AST查找函数调用
        def traverse(n):
            if n.type == 'call':
                # 提取被调用的函数名
                func_name = self._extract_call_name(n, code)
                if func_name:
                    dependencies.append(func_name)
            for child in n.children:
                traverse(child)
        
        traverse(node)
        return list(set(dependencies))
    
    def _extract_call_name(self, node, code: str) -> str:
        """提取调用表达式中的函数名"""
        # 简化的提取逻辑
        if node.children:
            first_child = node.children[0]
            if first_child.type == 'identifier':
                return first_child.text.decode('utf8')
        return None
```

#### 多层次索引策略

```python
class HierarchicalCodeIndexer:
    """多层次代码索引器"""
    
    def __init__(self):
        self.indexes = {
            'function': [],   # 函数级索引
            'class': [],      # 类级索引
            'file': [],       # 文件级索引
            'module': []      # 模块级索引
        }
    
    def index_repository(self, repo_path: str):
        """索引整个代码库"""
        
        for root, dirs, files in os.walk(repo_path):
            # 排除不需要的目录
            dirs[:] = [d for d in dirs 
                      if d not in ['.git', 'node_modules', '__pycache__', 'venv']]
            
            for file in files:
                file_path = os.path.join(root, file)
                language = self._detect_language(file)
                
                if language:
                    self._index_file(file_path, language)
    
    def _index_file(self, file_path: str, language: str):
        """索引单个文件"""
        with open(file_path, 'r', encoding='utf-8') as f:
            code = f.read()
        
        # 文件级索引
        self.indexes['file'].append({
            'content': code[:5000],  # 前5000字符作为摘要
            'metadata': {
                'file_path': file_path,
                'type': 'file',
                'language': language,
                'line_count': code.count('\n')
            }
        })
        
        # 函数/类级索引
        chunker = ASTCodeChunker(language)
        chunks = chunker.chunk(code, file_path)
        
        for chunk in chunks:
            level = chunk['metadata']['type']
            self.indexes[level].append(chunk)
    
    def _detect_language(self, filename: str) -> str:
        """检测编程语言"""
        extensions = {
            '.py': 'python',
            '.java': 'java',
            '.js': 'javascript',
            '.ts': 'typescript',
            '.go': 'go',
            '.rs': 'rust',
            '.cpp': 'cpp',
            '.c': 'c',
        }
        
        ext = os.path.splitext(filename)[1]
        return extensions.get(ext)
    
    def search(self, query: str, level: str = 'function', k: int = 5) -> list:
        """在指定层级搜索"""
        if level not in self.indexes:
            return []
        
        # 简化的搜索实现（实际应使用向量检索）
        results = []
        for chunk in self.indexes[level]:
            score = self._calculate_similarity(query, chunk['content'])
            results.append({**chunk, 'score': score})
        
        # 排序并返回Top-K
        results.sort(key=lambda x: x['score'], reverse=True)
        return results[:k]
    
    def _calculate_similarity(self, query: str, content: str) -> float:
        """计算查询和内容的相似度"""
        # 简化的Jaccard相似度
        query_words = set(query.lower().split())
        content_words = set(content.lower().split())
        
        intersection = len(query_words & content_words)
        union = len(query_words | content_words)
        
        return intersection / union if union > 0 else 0

import os
```

### 2. 代码专用Embedding模型

#### 代码语义表示学习

```python
"""
代码Embedding的演进：

第一代：文本化Embedding
- 将代码视为纯文本
- 使用通用NLP模型（BERT、RoBERTa）
- 问题：无法理解代码结构

第二代：代码预训练模型
- CodeBERT：在代码上预训练的BERT
- GraphCodeBERT：加入数据流信息
- CodeT5：编码器-解码器架构
- 优势：理解代码语法和语义

第三代：多语言代码Embedding（2024-2026）
- jina-embeddings-v3：多语言，代码理解强
- Qwen3-Embedding：中文代码优化
- OpenAI text-embedding-3：通用但代码能力一般
- 特点：支持100+编程语言，跨语言检索

代码Embedding的特殊要求：
1. 语法敏感性
   - 理解代码结构（缩进、括号、关键字）
   - 区分不同语言的语法

2. 语义敏感性
   - 理解变量名的含义（self-documenting code）
   - 理解注释和文档字符串

3. 结构敏感性
   - 理解调用关系
   - 理解类继承关系
   - 理解模块依赖

训练数据：
- GitHub公共仓库（经过筛选）
- Stack Overflow代码片段
- 代码-文档对（CodeSearchNet）
- 代码-注释对
"""

class CodeEmbedder:
    """代码Embedding生成器"""
    
    def __init__(self, model_name: str = 'jina-embeddings-v3'):
        self.model_name = model_name
        self.model = None  # 实际应加载模型
        self.max_length = 512
    
    def embed(self, code: str, language: str = None) -> np.ndarray:
        """
        生成代码的Embedding向量
        
        策略：
        1. 代码预处理（标准化、格式化）
        2. 添加语言标识
        3. 截断或分块
        4. 生成Embedding
        """
        # 预处理
        processed_code = self._preprocess(code, language)
        
        # 生成Embedding（实际应调用模型）
        # embedding = self.model.encode(processed_code)
        
        # 模拟返回
        return np.random.randn(768)  # 768维向量
    
    def embed_with_context(self, code: str, context: dict) -> np.ndarray:
        """
        带上下文的代码Embedding
        
        context: {
            'file_path': '...',
            'imports': ['...'],
            'docstring': '...',
            'callers': ['...']
        }
        """
        # 构造增强文本
        enhanced_text = f"""
文件：{context.get('file_path', '')}
导入：{', '.join(context.get('imports', []))}
文档：{context.get('docstring', '')}
调用者：{', '.join(context.get('callers', []))}
代码：
{code}
"""
        
        return self.embed(enhanced_text)
    
    def _preprocess(self, code: str, language: str = None) -> str:
        """代码预处理"""
        # 移除多余空白
        code = ' '.join(code.split())
        
        # 添加语言标识
        if language:
            code = f"Language: {language}\n{code}"
        
        return code
    
    def batch_embed(self, codes: list) -> np.ndarray:
        """批量生成Embedding"""
        embeddings = []
        for code in codes:
            emb = self.embed(code)
            embeddings.append(emb)
        
        return np.array(embeddings)
```

#### 混合检索策略

```python
class HybridCodeRetriever:
    """混合代码检索器"""
    
    def __init__(self):
        self.semantic_weight = 0.7
        self.keyword_weight = 0.3
        self.vector_store = None  # 向量数据库
        self.inverted_index = {}  # 倒排索引
    
    def retrieve(self, query: str, k: int = 10) -> list:
        """
        混合检索
        
        结合语义检索和关键词检索的优势：
        - 语义检索：理解查询意图，找到语义相关代码
        - 关键词检索：精确匹配特定标识符、API名称
        
        融合公式：
        Score_hybrid = w_semantic × Score_semantic + w_keyword × Score_keyword
        """
        # 语义检索
        semantic_results = self._semantic_search(query, k=k*2)
        
        # 关键词检索
        keyword_results = self._keyword_search(query, k=k*2)
        
        # 融合结果
        fused_results = self._fuse_results(
            semantic_results, 
            keyword_results,
            self.semantic_weight,
            self.keyword_weight
        )
        
        return fused_results[:k]
    
    def _semantic_search(self, query: str, k: int) -> list:
        """语义检索"""
        # 生成查询的Embedding
        query_embedding = self._embed_query(query)
        
        # 向量相似度搜索
        results = self.vector_store.similarity_search(
            query_embedding,
            k=k
        )
        
        return [{'doc': r, 'score': s, 'type': 'semantic'} 
                for r, s in results]
    
    def _keyword_search(self, query: str, k: int) -> list:
        """关键词检索"""
        query_tokens = set(query.lower().split())
        
        candidates = []
        for token in query_tokens:
            if token in self.inverted_index:
                candidates.extend(self.inverted_index[token])
        
        # 计算TF-IDF分数
        results = []
        for doc_id in set(candidates):
            doc_tokens = self._get_doc_tokens(doc_id)
            
            # 计算重叠
            overlap = len(query_tokens & doc_tokens)
            score = overlap / len(query_tokens)
            
            results.append({
                'doc_id': doc_id,
                'score': score,
                'type': 'keyword'
            })
        
        # 排序并返回Top-K
        results.sort(key=lambda x: x['score'], reverse=True)
        return results[:k]
    
    def _fuse_results(self, semantic_results: list, keyword_results: list,
                     w_semantic: float, w_keyword: float) -> list:
        """融合语义和关键词检索结果"""
        
        # 归一化分数
        max_semantic = max((r['score'] for r in semantic_results), default=1.0)
        max_keyword = max((r['score'] for r in keyword_results), default=1.0)
        
        # 构建文档ID到分数的映射
        doc_scores = {}
        
        for r in semantic_results:
            doc_id = r['doc']['id']
            score = (r['score'] / max_semantic) * w_semantic
            doc_scores[doc_id] = doc_scores.get(doc_id, 0) + score
        
        for r in keyword_results:
            doc_id = r['doc_id']
            score = (r['score'] / max_keyword) * w_keyword
            doc_scores[doc_id] = doc_scores.get(doc_id, 0) + score
        
        # 排序
        sorted_docs = sorted(doc_scores.items(), key=lambda x: x[1], reverse=True)
        
        return [{'doc_id': doc_id, 'score': score} 
                for doc_id, score in sorted_docs]
    
    def _embed_query(self, query: str) -> np.ndarray:
        """将查询转换为Embedding"""
        # 实际应调用Embedding模型
        return np.random.randn(768)
    
    def _get_doc_tokens(self, doc_id: str) -> set:
        """获取文档的token集合"""
        # 实际应从索引中获取
        return set()
```

### 3. 代码库特定的RAG架构

#### 代码知识图谱增强RAG

```python
"""
代码知识图谱：

节点类型：
- File（文件）
- Class（类）
- Function（函数）
- Variable（变量）
- Module（模块）

边类型：
- CONTAINS（包含）
- CALLS（调用）
- IMPORTS（导入）
- INHERITS（继承）
- USES（使用）

示例图谱：

File: main.py
  └── Function: main()
        └── Calls: process_order()
              └── Defined in: order_service.py
                    └── Class: OrderService
                          └── Method: process_order()
                                └── Calls: validate_order()
                                └── Calls: save_to_db()

RAG增强：
1. 检索到函数process_order
2. 图谱查询找到：
   - 调用者：main()
   - 被调用者：validate_order(), save_to_db()
   - 所属类：OrderService
   - 所在文件：order_service.py
3. 将这些相关代码也加入上下文
4. LLM基于完整上下文生成答案
"""

class CodeKnowledgeGraph:
    """代码知识图谱"""
    
    def __init__(self):
        self.nodes = {}  # 节点集合
        self.edges = []  # 边集合
    
    def build_from_repo(self, repo_path: str):
        """从代码库构建知识图谱"""
        
        for root, dirs, files in os.walk(repo_path):
            dirs[:] = [d for d in dirs 
                      if d not in ['.git', 'node_modules', '__pycache__']]
            
            for file in files:
                file_path = os.path.join(root, file)
                self._parse_file(file_path)
    
    def _parse_file(self, file_path: str):
        """解析单个文件，构建图谱"""
        language = self._detect_language(file_path)
        if not language:
            return
        
        with open(file_path, 'r', encoding='utf-8') as f:
            code = f.read()
        
        # 解析AST
        chunker = ASTCodeChunker(language)
        chunks = chunker.chunk(code, file_path)
        
        # 添加文件节点
        file_node = f"file:{file_path}"
        self.nodes[file_node] = {'type': 'file', 'path': file_path}
        
        # 添加函数/类节点和边
        for chunk in chunks:
            node_id = f"{chunk['metadata']['type']}:{chunk['metadata']['name']}"
            self.nodes[node_id] = {
                'type': chunk['metadata']['type'],
                'name': chunk['metadata']['name'],
                'file': file_path,
                'line': chunk['metadata']['start_line']
            }
            
            # 文件包含关系
            self.edges.append({
                'from': file_node,
                'to': node_id,
                'type': 'CONTAINS'
            })
            
            # 调用关系
            for dep in chunk['metadata'].get('dependencies', []):
                self.edges.append({
                    'from': node_id,
                    'to': f"function:{dep}",
                    'type': 'CALLS'
                })
    
    def get_related_code(self, node_id: str, depth: int = 2) -> list:
        """
        获取相关代码（基于图谱遍历）
        
        返回与指定节点相关的代码片段
        """
        related = []
        visited = set()
        queue = [(node_id, 0)]
        
        while queue:
            current, current_depth = queue.pop(0)
            
            if current in visited or current_depth > depth:
                continue
            
            visited.add(current)
            
            if current in self.nodes:
                related.append(self.nodes[current])
            
            # 查找相邻节点
            for edge in self.edges:
                if edge['from'] == current and edge['to'] not in visited:
                    queue.append((edge['to'], current_depth + 1))
                elif edge['to'] == current and edge['from'] not in visited:
                    queue.append((edge['from'], current_depth + 1))
        
        return related
    
    def enrich_context(self, retrieved_chunks: list) -> list:
        """基于知识图谱增强上下文"""
        enriched = []
        
        for chunk in retrieved_chunks:
            enriched.append(chunk)
            
            # 查找相关代码
            node_id = f"{chunk['metadata']['type']}:{chunk['metadata']['name']}"
            related = self.get_related_code(node_id, depth=1)
            
            # 添加相关代码（去重）
            for node in related:
                if node not in enriched:
                    enriched.append(node)
        
        return enriched
    
    def _detect_language(self, file_path: str) -> str:
        """检测编程语言"""
        extensions = {
            '.py': 'python',
            '.java': 'java',
            '.js': 'javascript',
        }
        ext = os.path.splitext(file_path)[1]
        return extensions.get(ext)
```

#### 增量索引与实时同步

```python
class IncrementalCodeIndexer:
    """增量代码索引器"""
    
    def __init__(self, vector_store, code_graph):
        self.vector_store = vector_store
        self.code_graph = code_graph
        self.indexed_files = {}  # file_path -> last_modified
    
    def index_repository(self, repo_path: str):
        """索引代码库（支持增量更新）"""
        
        current_files = set()
        
        for root, dirs, files in os.walk(repo_path):
            dirs[:] = [d for d in dirs 
                      if d not in ['.git', 'node_modules', '__pycache__']]
            
            for file in files:
                file_path = os.path.join(root, file)
                current_files.add(file_path)
                
                # 检查是否需要更新
                last_modified = os.path.getmtime(file_path)
                
                if file_path not in self.indexed_files:
                    # 新文件
                    self._index_file(file_path)
                    self.indexed_files[file_path] = last_modified
                    
                elif last_modified > self.indexed_files[file_path]:
                    # 文件已修改
                    self._update_file(file_path)
                    self.indexed_files[file_path] = last_modified
        
        # 删除已不存在的文件
        removed_files = set(self.indexed_files.keys()) - current_files
        for file_path in removed_files:
            self._remove_file(file_path)
            del self.indexed_files[file_path]
    
    def _index_file(self, file_path: str):
        """索引新文件"""
        language = self._detect_language(file_path)
        if not language:
            return
        
        with open(file_path, 'r', encoding='utf-8') as f:
            code = f.read()
        
        # 分块
        chunker = ASTCodeChunker(language)
        chunks = chunker.chunk(code, file_path)
        
        # 生成Embedding并存储
        for chunk in chunks:
            embedding = self._generate_embedding(chunk['content'])
            
            self.vector_store.add_document(
                id=f"{file_path}:{chunk['metadata']['name']}",
                embedding=embedding,
                content=chunk['content'],
                metadata=chunk['metadata']
            )
        
        # 更新知识图谱
        self.code_graph._parse_file(file_path)
    
    def _update_file(self, file_path: str):
        """更新已修改的文件"""
        # 删除旧索引
        self._remove_file(file_path)
        
        # 重新索引
        self._index_file(file_path)
    
    def _remove_file(self, file_path: str):
        """删除文件索引"""
        # 删除向量数据库中的相关文档
        self.vector_store.delete_by_filter({'file_path': file_path})
    
    def _generate_embedding(self, code: str) -> np.ndarray:
        """生成代码Embedding"""
        embedder = CodeEmbedder()
        return embedder.embed(code)
    
    def _detect_language(self, file_path: str) -> str:
        """检测编程语言"""
        extensions = {
            '.py': 'python',
            '.java': 'java',
            '.js': 'javascript',
        }
        ext = os.path.splitext(file_path)[1]
        return extensions.get(ext)
    
    def setup_git_hook(self, repo_path: str):
        """设置Git钩子，实现自动索引更新"""
        hook_script = f"""#!/bin/bash
# post-commit钩子
python -c "
from incremental_indexer import IncrementalCodeIndexer
indexer = IncrementalCodeIndexer(...)
indexer.index_repository('{repo_path}')
"
"""
        
        hook_path = os.path.join(repo_path, '.git', 'hooks', 'post-commit')
        with open(hook_path, 'w') as f:
            f.write(hook_script)
        
        os.chmod(hook_path, 0o755)
        print(f"Git钩子已设置: {hook_path}")
```

---

## 实战案例：五个企业级代码RAG系统实现

### 案例1：企业代码问答系统

```python
"""
企业代码问答系统架构：

┌─────────────────────────────────────────┐
│           用户提问                        │
│  "订单模块的退款流程是怎么实现的？"        │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         查询理解与扩展                    │
│  - 意图识别：代码解释/代码生成/Bug修复      │
│  - 关键词提取：订单、退款、流程             │
│  - 查询扩展：refund, order, process       │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         多路检索（并行）                   │
│  ├─ 语义检索（Embedding）                 │
│  ├─ 关键词检索（BM25）                    │
│  ├─ 调用图检索（知识图谱）                 │
│  └─ 文件路径检索（模糊匹配）               │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         结果融合与重排序                   │
│  - 去重                                    │
│  - 重排序（Cross-Encoder）                │
│  - 上下文组装                              │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         大模型生成（DeepSeek-V4）         │
│  - 基于检索到的代码生成答案                │
│  - 引用具体的代码文件和行号                │
│  - 提供代码示例                            │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         答案后处理                        │
│  - 代码语法高亮                            │
│  - 引用链接生成                            │
│  - 答案置信度评分                          │
└─────────────────────────────────────────┘
"""

class EnterpriseCodeQASystem:
    """企业代码问答系统"""
    
    def __init__(self, repo_path: str):
        self.repo_path = repo_path
        
        # 初始化组件
        self.vector_store = ChromaVectorStore()
        self.code_graph = CodeKnowledgeGraph()
        self.embedder = CodeEmbedder()
        self.retriever = HybridCodeRetriever()
        self.context_assembler = ContextAssembler()
        
        # 初始化索引
        self.indexer = IncrementalCodeIndexer(
            self.vector_store, 
            self.code_graph
        )
        self.indexer.index_repository(repo_path)
    
    def answer(self, question: str) -> dict:
        """回答代码相关问题"""
        
        # 步骤1：查询理解
        query_info = self._understand_query(question)
        
        # 步骤2：多路检索
        retrieval_results = self._multi_way_retrieve(question, query_info)
        
        # 步骤3：结果融合
        fused_results = self._fuse_and_rerank(retrieval_results)
        
        # 步骤4：上下文组装
        context = self.context_assembler.assemble(question, fused_results)
        
        # 步骤5：生成答案
        answer = self._generate_answer(context, question)
        
        # 步骤6：后处理
        final_answer = self._post_process(answer, fused_results)
        
        return {
            'question': question,
            'answer': final_answer,
            'sources': self._extract_sources(fused_results),
            'confidence': self._calculate_confidence(fused_results)
        }
    
    def _understand_query(self, question: str) -> dict:
        """理解查询意图"""
        # 简化的规则分类
        intent = 'explanation'  # explanation, generation, bug_fix
        
        if any(kw in question for kw in ['怎么写', '实现', '生成']):
            intent = 'generation'
        elif any(kw in question for kw in ['bug', '错误', '修复']):
            intent = 'bug_fix'
        
        # 提取关键词
        keywords = self._extract_keywords(question)
        
        return {
            'intent': intent,
            'keywords': keywords,
            'expanded_query': question  # 实际应做查询扩展
        }
    
    def _multi_way_retrieve(self, question: str, query_info: dict) -> dict:
        """多路检索"""
        results = {
            'semantic': [],
            'keyword': [],
            'graph': [],
            'path': []
        }
        
        # 语义检索
        results['semantic'] = self.retriever._semantic_search(question, k=10)
        
        # 关键词检索
        results['keyword'] = self.retriever._keyword_search(question, k=10)
        
        # 调用图检索（如果有明确的函数名）
        for kw in query_info['keywords']:
            node_id = f"function:{kw}"
            if node_id in self.code_graph.nodes:
                related = self.code_graph.get_related_code(node_id)
                results['graph'].extend(related)
        
        # 文件路径检索
        results['path'] = self._search_by_path(query_info['keywords'])
        
        return results
    
    def _fuse_and_rerank(self, results: dict) -> list:
        """融合并重排序"""
        # 收集所有结果
        all_results = []
        
        for source_type, source_results in results.items():
            for r in source_results:
                all_results.append({
                    **r,
                    'source_type': source_type,
                    'score': r.get('score', 0.5) * self._get_source_weight(source_type)
                })
        
        # 去重
        seen = set()
        unique_results = []
        for r in all_results:
            doc_id = r.get('doc_id') or r.get('doc', {}).get('id')
            if doc_id and doc_id not in seen:
                seen.add(doc_id)
                unique_results.append(r)
        
        # 排序
        unique_results.sort(key=lambda x: x['score'], reverse=True)
        
        return unique_results[:10]
    
    def _get_source_weight(self, source_type: str) -> float:
        """获取不同来源的权重"""
        weights = {
            'semantic': 1.0,
            'keyword': 0.8,
            'graph': 0.9,
            'path': 0.7
        }
        return weights.get(source_type, 0.5)
    
    def _generate_answer(self, context: str, question: str) -> str:
        """生成答案"""
        # 构造提示词
        prompt = f"""基于以下代码片段回答问题。

{context}

问题：{question}

要求：
1. 直接回答用户问题
2. 引用具体的代码文件和函数名
3. 如果涉及代码逻辑，请解释关键步骤
4. 如果不确定，请明确说明"无法从提供的代码中找到答案"
"""
        
        # 实际应调用DeepSeek-V4或Qwen3
        # response = llm.generate(prompt)
        return f"[基于检索结果的答案...]"
    
    def _post_process(self, answer: str, sources: list) -> str:
        """答案后处理"""
        # 添加引用链接
        for source in sources[:3]:
            file_path = source.get('metadata', {}).get('file_path', '')
            if file_path:
                answer += f"\n- [{file_path}]"
        
        return answer
    
    def _extract_sources(self, results: list) -> list:
        """提取来源信息"""
        sources = []
        for r in results[:5]:
            meta = r.get('metadata', {})
            sources.append({
                'file': meta.get('file_path', ''),
                'lines': f"{meta.get('start_line', '')}-{meta.get('end_line', '')}",
                'type': meta.get('type', ''),
                'name': meta.get('name', '')
            })
        return sources
    
    def _calculate_confidence(self, results: list) -> float:
        """计算答案置信度"""
        if not results:
            return 0.0
        
        # 基于检索结果的质量计算
        top_score = results[0].get('score', 0)
        avg_score = sum(r.get('score', 0) for r in results[:3]) / 3
        
        return (top_score + avg_score) / 2
    
    def _extract_keywords(self, text: str) -> list:
        """提取关键词"""
        # 简化的关键词提取
        words = text.split()
        return [w for w in words if len(w) > 2]
    
    def _search_by_path(self, keywords: list) -> list:
        """通过文件路径搜索"""
        results = []
        for kw in keywords:
            for root, dirs, files in os.walk(self.repo_path):
                for file in files:
                    if kw.lower() in file.lower():
                        results.append({
                            'doc_id': os.path.join(root, file),
                            'score': 0.6,
                            'metadata': {'file_path': os.path.join(root, file)}
                        })
        return results

# 模拟向量数据库
class ChromaVectorStore:
    def add_document(self, id: str, embedding: np.ndarray, content: str, metadata: dict):
        pass
    
    def similarity_search(self, query_embedding: np.ndarray, k: int):
        return []
    
    def delete_by_filter(self, filter_dict: dict):
        pass
```

### 案例2：代码审查RAG助手

```python
"""
代码审查RAG助手：

场景：审查Pull Request时，AI助手帮助审查代码

┌─────────────────────────────────────────┐
│         PR代码变更                        │
│  - 新增/修改/删除的文件                    │
│  - Diff内容                               │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         变更影响分析                       │
│  - 修改了哪些函数                          │
│  - 影响了哪些调用者                        │
│  - 是否有 breaking change                │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         相关知识检索                       │
│  - 检索相关代码（调用者、被调用者）          │
│  - 检索项目编码规范                        │
│  - 检索历史类似修改                        │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         多维度审查                         │
│  ├─ 安全性审查（SQL注入、XSS等）           │
│  ├─ 性能审查（N+1查询、大数据处理）        │
│  ├─ 规范审查（命名、格式、注释）           │
│  └─ 逻辑审查（边界条件、错误处理）         │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         审查报告生成                       │
│  - 问题列表（严重级别、位置、建议）         │
│  - 正面反馈（好的实践）                     │
│  - 学习建议（相关知识点）                   │
└─────────────────────────────────────────┘
"""

class CodeReviewRAGAssistant:
    """代码审查RAG助手"""
    
    def __init__(self, repo_path: str):
        self.repo_path = repo_path
        self.qa_system = EnterpriseCodeQASystem(repo_path)
        self.coding_standards = self._load_coding_standards()
    
    def review_pr(self, pr_diff: str) -> dict:
        """审查PR"""
        
        # 步骤1：解析Diff
        changes = self._parse_diff(pr_diff)
        
        # 步骤2：分析影响
        impact = self._analyze_impact(changes)
        
        # 步骤3：检索相关知识
        knowledge = self._retrieve_review_knowledge(changes, impact)
        
        # 步骤4：多维度审查
        reviews = {
            'security': self._security_review(changes, knowledge),
            'performance': self._performance_review(changes, knowledge),
            'style': self._style_review(changes),
            'logic': self._logic_review(changes, knowledge)
        }
        
        # 步骤5：生成报告
        report = self._generate_report(reviews, changes)
        
        return report
    
    def _parse_diff(self, diff: str) -> list:
        """解析Git Diff"""
        # 简化的Diff解析
        changes = []
        lines = diff.split('\n')
        current_file = None
        
        for line in lines:
            if line.startswith('diff --git'):
                current_file = line.split()[-1]
            elif line.startswith('+') and not line.startswith('+++'):
                changes.append({
                    'file': current_file,
                    'type': 'addition',
                    'content': line[1:]
                })
            elif line.startswith('-') and not line.startswith('---'):
                changes.append({
                    'file': current_file,
                    'type': 'deletion',
                    'content': line[1:]
                })
        
        return changes
    
    def _analyze_impact(self, changes: list) -> dict:
        """分析变更影响"""
        affected_functions = set()
        
        for change in changes:
            # 提取函数名（简化）
            func_match = re.search(r'def\s+(\w+)', change['content'])
            if func_match:
                affected_functions.add(func_match.group(1))
        
        return {
            'affected_functions': list(affected_functions),
            'change_count': len(changes)
        }
    
    def _retrieve_review_knowledge(self, changes: list, impact: dict) -> dict:
        """检索审查相关知识"""
        knowledge = {
            'related_code': [],
            'standards': [],
            'history': []
        }
        
        # 检索相关代码
        for func_name in impact['affected_functions']:
            results = self.qa_system.retriever.retrieve(func_name, k=3)
            knowledge['related_code'].extend(results)
        
        # 检索编码规范
        knowledge['standards'] = self.coding_standards
        
        return knowledge
    
    def _security_review(self, changes: list, knowledge: dict) -> list:
        """安全性审查"""
        issues = []
        
        security_patterns = [
            (r'eval\s*\(', 'CRITICAL', '使用了eval()，存在代码注入风险'),
            (r'exec\s*\(', 'CRITICAL', '使用了exec()，存在代码注入风险'),
            (r'\.format\s*\(.*user', 'HIGH', '字符串格式化可能存在注入风险'),
            (r'select\s+.*\+.*user', 'HIGH', 'SQL拼接可能存在注入风险'),
            (r'password\s*=\s*["\'][^"\']+["\']', 'MEDIUM', '硬编码密码'),
        ]
        
        for change in changes:
            for pattern, severity, message in security_patterns:
                if re.search(pattern, change['content'], re.IGNORECASE):
                    issues.append({
                        'file': change['file'],
                        'severity': severity,
                        'message': message,
                        'line': change['content']
                    })
        
        return issues
    
    def _performance_review(self, changes: list, knowledge: dict) -> list:
        """性能审查"""
        issues = []
        
        # 检查N+1查询
        for change in changes:
            if 'for' in change['content'] and 'query' in change['content']:
                issues.append({
                    'file': change['file'],
                    'severity': 'MEDIUM',
                    'message': '循环内可能存在数据库查询，考虑使用批量查询',
                    'line': change['content']
                })
        
        return issues
    
    def _style_review(self, changes: list) -> list:
        """规范审查"""
        issues = []
        
        for change in changes:
            # 检查命名规范
            if 'def ' in change['content']:
                func_name = re.search(r'def\s+(\w+)', change['content'])
                if func_name:
                    name = func_name.group(1)
                    if not re.match(r'^[a-z_][a-z0-9_]*$', name):
                        issues.append({
                            'file': change['file'],
                            'severity': 'LOW',
                            'message': f'函数名"{name}"不符合snake_case规范',
                            'line': change['content']
                        })
        
        return issues
    
    def _logic_review(self, changes: list, knowledge: dict) -> list:
        """逻辑审查"""
        issues = []
        
        for change in changes:
            # 检查错误处理
            if 'try' in change['content'] and 'except' not in change['content']:
                issues.append({
                    'file': change['file'],
                    'severity': 'MEDIUM',
                    'message': 'try块缺少except处理',
                    'line': change['content']
                })
        
        return issues
    
    def _generate_report(self, reviews: dict, changes: list) -> dict:
        """生成审查报告"""
        all_issues = []
        for category, issues in reviews.items():
            for issue in issues:
                issue['category'] = category
                all_issues.append(issue)
        
        # 按严重级别排序
        severity_order = {'CRITICAL': 0, 'HIGH': 1, 'MEDIUM': 2, 'LOW': 3}
        all_issues.sort(key=lambda x: severity_order.get(x['severity'], 99))
        
        return {
            'summary': {
                'total_issues': len(all_issues),
                'critical': sum(1 for i in all_issues if i['severity'] == 'CRITICAL'),
                'high': sum(1 for i in all_issues if i['severity'] == 'HIGH'),
                'medium': sum(1 for i in all_issues if i['severity'] == 'MEDIUM'),
                'low': sum(1 for i in all_issues if i['severity'] == 'LOW')
            },
            'issues': all_issues,
            'changes_analyzed': len(changes)
        }
    
    def _load_coding_standards(self) -> list:
        """加载编码规范"""
        # 实际应从项目配置加载
        return [
            '使用snake_case命名函数和变量',
            '类名使用PascalCase',
            '常量使用UPPER_CASE',
            '函数长度不超过50行',
            '复杂度不超过10'
        ]

import re
import numpy as np
```

### 案例3：代码生成RAG系统

```python
"""
代码生成RAG系统：

场景：根据需求描述生成代码，参考现有代码库的风格和模式

┌─────────────────────────────────────────┐
│         需求描述                          │
│  "实现一个支持幂等性的订单创建接口"         │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         需求分析（LLM）                   │
│  - 提取功能需求                            │
│  - 识别技术约束                            │
│  - 确定相关领域                            │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         检索参考代码                       │
│  - 检索订单相关代码                        │
│  - 检索幂等性实现模式                      │
│  - 检索接口定义规范                        │
│  - 检索错误处理方式                        │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         代码生成（LLM + RAG）             │
│  - 参考现有代码风格                        │
│  - 遵循项目约定                            │
│  - 使用已有工具类                          │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         代码验证                           │
│  - 语法检查                                │
│  - 类型检查                                │
│  - 单元测试生成                            │
└─────────────────────────────────────────┘
"""

class CodeGenerationRAG:
    """代码生成RAG系统"""
    
    def __init__(self, repo_path: str):
        self.repo_path = repo_path
        self.qa_system = EnterpriseCodeQASystem(repo_path)
    
    def generate(self, requirement: str, language: str = 'python') -> dict:
        """根据需求生成代码"""
        
        # 步骤1：需求分析
        analysis = self._analyze_requirement(requirement)
        
        # 步骤2：检索参考代码
        references = self._retrieve_references(analysis)
        
        # 步骤3：生成代码
        code = self._generate_code(requirement, references, language)
        
        # 步骤4：验证代码
        validation = self._validate_code(code, language)
        
        return {
            'requirement': requirement,
            'analysis': analysis,
            'generated_code': code,
            'references': references,
            'validation': validation
        }
    
    def _analyze_requirement(self, requirement: str) -> dict:
        """分析需求"""
        # 使用LLM分析需求
        prompt = f"""分析以下需求，提取关键信息：

需求：{requirement}

请提取：
1. 功能需求（用户想要什么）
2. 非功能需求（性能、安全、可靠性）
3. 技术约束（使用什么技术栈）
4. 相关领域（订单、支付、用户等）
5. 类似功能（现有系统中可能相关的功能）

请以JSON格式输出。
"""
        
        # 实际应调用LLM
        return {
            'functional': ['创建订单', '支持幂等性'],
            'non_functional': ['高并发', '数据一致性'],
            'tech_stack': ['Spring Boot', 'MySQL', 'Redis'],
            'domain': 'order',
            'related_features': ['订单查询', '订单取消']
        }
    
    def _retrieve_references(self, analysis: dict) -> list:
        """检索参考代码"""
        references = []
        
        # 检索领域相关代码
        domain_queries = [
            analysis['domain'],
            *analysis.get('related_features', [])
        ]
        
        for query in domain_queries:
            results = self.qa_system.retriever.retrieve(query, k=3)
            references.extend(results)
        
        # 检索模式代码
        pattern_queries = ['幂等性', 'idempotent', '接口实现', 'Controller']
        for query in pattern_queries:
            results = self.qa_system.retriever.retrieve(query, k=2)
            references.extend(results)
        
        # 去重
        seen = set()
        unique_refs = []
        for ref in references:
            doc_id = ref.get('doc_id') or ref.get('doc', {}).get('id')
            if doc_id and doc_id not in seen:
                seen.add(doc_id)
                unique_refs.append(ref)
        
        return unique_refs[:10]
    
    def _generate_code(self, requirement: str, references: list, language: str) -> str:
        """生成代码"""
        
        # 构造参考代码上下文
        ref_context = "\n\n".join([
            f"参考代码 {i+1}（{ref.get('metadata', {}).get('file_path', '')}）：\n"
            f"```\n{ref.get('content', '')[:500]}\n```"
            for i, ref in enumerate(references[:5])
        ])
        
        prompt = f"""基于以下参考代码，实现需求。

参考代码：
{ref_context}

需求：{requirement}

要求：
1. 遵循参考代码的风格和约定
2. 使用项目中已有的工具类和模式
3. 包含完整的错误处理
4. 添加必要的注释
5. 生成{language}代码

请只输出代码，不要输出解释。
"""
        
        # 实际应调用DeepSeek-V4或Qwen3
        return f"""// 生成的{language}代码
// 基于参考代码的风格和模式

class OrderController {{
    // 幂等性订单创建接口实现
    // ...
}}
"""
    
    def _validate_code(self, code: str, language: str) -> dict:
        """验证代码"""
        validation = {
            'syntax_valid': True,
            'issues': []
        }
        
        if language == 'python':
            # 语法检查
            try:
                import ast
                ast.parse(code)
                validation['syntax_valid'] = True
            except SyntaxError as e:
                validation['syntax_valid'] = False
                validation['issues'].append(str(e))
        
        elif language in ['java', 'javascript']:
            # 简化的语法检查
            if '{' in code and '}' not in code:
                validation['issues'].append('括号不匹配')
        
        return validation
```

### 案例4：代码库智能导航

```python
"""
代码库智能导航系统：

场景：帮助新成员快速理解代码库

功能：
1. "这个项目是怎么组织的？"
   → 返回项目结构图 + 模块说明

2. "订单模块在哪里？"
   → 返回相关文件列表 + 入口函数

3. "从API接收到订单创建，数据流是怎样的？"
   → 返回调用链图 + 数据流说明

4. "这个函数被哪些地方调用了？"
   → 返回调用者列表 + 调用上下文
"""

class CodebaseNavigator:
    """代码库智能导航"""
    
    def __init__(self, repo_path: str):
        self.repo_path = repo_path
        self.code_graph = CodeKnowledgeGraph()
        self.code_graph.build_from_repo(repo_path)
        self.qa_system = EnterpriseCodeQASystem(repo_path)
    
    def navigate(self, query: str) -> dict:
        """导航查询"""
        
        # 识别查询类型
        query_type = self._classify_navigation_query(query)
        
        if query_type == 'structure':
            return self._get_project_structure()
        elif query_type == 'location':
            return self._find_code_location(query)
        elif query_type == 'dataflow':
            return self._trace_dataflow(query)
        elif query_type == 'callers':
            return self._find_callers(query)
        else:
            return self.qa_system.answer(query)
    
    def _classify_navigation_query(self, query: str) -> str:
        """分类导航查询"""
        structure_keywords = ['结构', '组织', '架构', '怎么组织', '目录']
        location_keywords = ['在哪里', '哪个文件', '位置']
        dataflow_keywords = ['数据流', '流程', '怎么走的', '调用链']
        caller_keywords = ['谁调用', '被谁', '调用者']
        
        if any(kw in query for kw in structure_keywords):
            return 'structure'
        elif any(kw in query for kw in location_keywords):
            return 'location'
        elif any(kw in query for kw in dataflow_keywords):
            return 'dataflow'
        elif any(kw in query for kw in caller_keywords):
            return 'callers'
        else:
            return 'general'
    
    def _get_project_structure(self) -> dict:
        """获取项目结构"""
        structure = {
            'root': self.repo_path,
            'modules': []
        }
        
        for item in os.listdir(self.repo_path):
            item_path = os.path.join(self.repo_path, item)
            if os.path.isdir(item_path) and not item.startswith('.'):
                module_info = {
                    'name': item,
                    'description': self._guess_module_description(item),
                    'file_count': sum(1 for _, _, files in os.walk(item_path) for f in files),
                    'key_files': self._get_key_files(item_path)
                }
                structure['modules'].append(module_info)
        
        return structure
    
    def _guess_module_description(self, module_name: str) -> str:
        """猜测模块描述"""
        descriptions = {
            'order': '订单管理模块',
            'user': '用户管理模块',
            'payment': '支付模块',
            'product': '商品管理模块',
            'api': 'API接口层',
            'service': '业务逻辑层',
            'dao': '数据访问层',
            'model': '数据模型层',
            'utils': '工具类',
            'config': '配置管理'
        }
        return descriptions.get(module_name.lower(), f'{module_name}模块')
    
    def _get_key_files(self, module_path: str) -> list:
        """获取模块的关键文件"""
        key_files = []
        
        for root, dirs, files in os.walk(module_path):
            for file in files:
                if file.endswith(('.py', '.java', '.js')):
                    if any(keyword in file.lower() for keyword in ['main', 'app', 'config', 'service']):
                        rel_path = os.path.relpath(os.path.join(root, file), self.repo_path)
                        key_files.append(rel_path)
        
        return key_files[:5]
    
    def _find_code_location(self, query: str) -> dict:
        """查找代码位置"""
        # 提取可能的函数/类名
        keywords = self._extract_keywords(query)
        
        locations = []
        for keyword in keywords:
            # 在知识图谱中查找
            for node_id, node in self.code_graph.nodes.items():
                if keyword.lower() in node.get('name', '').lower():
                    locations.append({
                        'name': node['name'],
                        'type': node['type'],
                        'file': node.get('file', ''),
                        'line': node.get('line', 0)
                    })
        
        return {
            'query': query,
            'locations': locations[:10]
        }
    
    def _trace_dataflow(self, query: str) -> dict:
        """追踪数据流"""
        # 提取起始点
        keywords = self._extract_keywords(query)
        
        # 查找入口函数
        entry_points = []
        for keyword in keywords:
            for node_id, node in self.code_graph.nodes.items():
                if (node.get('type') == 'function' and 
                    keyword.lower() in node.get('name', '').lower()):
                    entry_points.append(node_id)
        
        # 追踪调用链
        dataflows = []
        for entry in entry_points:
            flow = self._trace_call_chain(entry, depth=3)
            dataflows.append(flow)
        
        return {
            'query': query,
            'entry_points': entry_points,
            'dataflows': dataflows
        }
    
    def _trace_call_chain(self, node_id: str, depth: int = 3) -> list:
        """追踪调用链"""
        chain = []
        current = node_id
        
        for _ in range(depth):
            if current not in self.code_graph.nodes:
                break
            
            node = self.code_graph.nodes[current]
            chain.append({
                'name': node['name'],
                'type': node['type'],
                'file': node.get('file', '')
            })
            
            # 查找被调用的函数
            next_nodes = [
                edge['to'] for edge in self.code_graph.edges
                if edge['from'] == current and edge['type'] == 'CALLS'
            ]
            
            if next_nodes:
                current = next_nodes[0]
            else:
                break
        
        return chain
    
    def _find_callers(self, query: str) -> dict:
        """查找调用者"""
        # 提取函数名
        keywords = self._extract_keywords(query)
        
        callers = []
        for keyword in keywords:
            target_id = f"function:{keyword}"
            
            # 在知识图谱中查找调用关系
            for edge in self.code_graph.edges:
                if edge['to'] == target_id and edge['type'] == 'CALLS':
                    caller_id = edge['from']
                    if caller_id in self.code_graph.nodes:
                        caller = self.code_graph.nodes[caller_id]
                        callers.append({
                            'name': caller['name'],
                            'type': caller['type'],
                            'file': caller.get('file', '')
                        })
        
        return {
            'target': keywords[0] if keywords else '',
            'callers': callers
        }
    
    def _extract_keywords(self, text: str) -> list:
        """提取关键词"""
        words = text.split()
        return [w for w in words if len(w) > 2 and not w.startswith(('怎么', '什么', '哪里'))]
```

### 案例5：多仓库代码检索系统

```python
"""
多仓库代码检索系统：

场景：大型企业有数百个代码仓库，需要统一检索

挑战：
1. 数据量大（TB级代码）
2. 仓库间依赖关系复杂
3. 权限控制（不同团队只能访问特定仓库）
4. 实时性（新代码及时索引）

架构：

┌─────────────────────────────────────────┐
│         统一查询接口                       │
│  - 自然语言查询                            │
│  - 代码片段查询                            │
│  - 结构查询（按语言、仓库、作者）           │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         查询路由层                         │
│  - 根据权限过滤可访问仓库                   │
│  - 根据查询意图选择索引                     │
│  - 负载均衡                                 │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         分布式索引集群                     │
│  ├─ 索引分片1（仓库A-C）                  │
│  ├─ 索引分片2（仓库D-F）                  │
│  ├─ 索引分片3（仓库G-I）                  │
│  └─ ...                                  │
└─────────────────┬───────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│         结果聚合层                         │
│  - 跨仓库结果合并                          │
│  - 去重                                    │
│  - 排序                                    │
└─────────────────────────────────────────┘
"""

class MultiRepoCodeSearch:
    """多仓库代码检索系统"""
    
    def __init__(self, repo_configs: list):
        """
        repo_configs: [
            {'name': 'repo1', 'path': '/path/to/repo1', 'shard': 1},
            {'name': 'repo2', 'path': '/path/to/repo2', 'shard': 1},
            ...
        ]
        """
        self.repo_configs = repo_configs
        self.shards = {}  # shard_id -> [repos]
        self.indexers = {}  # repo_name -> indexer
        self.permissions = {}  # user -> [allowed_repos]
        
        self._initialize_shards()
    
    def _initialize_shards(self):
        """初始化分片"""
        for config in self.repo_configs:
            shard_id = config['shard']
            if shard_id not in self.shards:
                self.shards[shard_id] = []
            self.shards[shard_id].append(config)
            
            # 初始化索引器
            self.indexers[config['name']] = IncrementalCodeIndexer(
                ChromaVectorStore(),
                CodeKnowledgeGraph()
            )
            self.indexers[config['name']].index_repository(config['path'])
    
    def search(self, query: str, user: str, filters: dict = None) -> dict:
        """
        跨仓库搜索
        
        filters: {
            'language': 'python',
            'repo': ['repo1', 'repo2'],
            'author': 'john'
        }
        """
        # 获取用户有权限的仓库
        allowed_repos = self._get_user_permissions(user)
        
        # 应用过滤器
        if filters and filters.get('repo'):
            allowed_repos = [r for r in allowed_repos if r in filters['repo']]
        
        # 并行搜索各仓库
        all_results = []
        for repo_name in allowed_repos:
            repo_results = self._search_repo(repo_name, query, filters)
            for result in repo_results:
                result['repo'] = repo_name
            all_results.extend(repo_results)
        
        # 合并和排序
        merged_results = self._merge_results(all_results)
        
        return {
            'query': query,
            'total_results': len(merged_results),
            'results': merged_results[:20],
            'searched_repos': allowed_repos
        }
    
    def _get_user_permissions(self, user: str) -> list:
        """获取用户权限"""
        return self.permissions.get(user, [config['name'] for config in self.repo_configs])
    
    def _search_repo(self, repo_name: str, query: str, filters: dict) -> list:
        """搜索单个仓库"""
        indexer = self.indexers.get(repo_name)
        if not indexer:
            return []
        
        # 使用混合检索
        retriever = HybridCodeRetriever()
        results = retriever.retrieve(query, k=10)
        
        # 应用过滤器
        if filters:
            if filters.get('language'):
                results = [
                    r for r in results 
                    if r.get('metadata', {}).get('language') == filters['language']
                ]
        
        return results
    
    def _merge_results(self, results: list) -> list:
        """合并多仓库结果"""
        # 按分数排序
        results.sort(key=lambda x: x.get('score', 0), reverse=True)
        
        # 去重（跨仓库可能重复）
        seen_content = set()
        unique_results = []
        
        for result in results:
            content_hash = hash(result.get('content', '')[:100])
            if content_hash not in seen_content:
                seen_content.add(content_hash)
                unique_results.append(result)
        
        return unique_results
    
    def index_new_repo(self, repo_config: dict):
        """添加新仓库到索引"""
        self.repo_configs.append(repo_config)
        
        shard_id = repo_config['shard']
        if shard_id not in self.shards:
            self.shards[shard_id] = []
        self.shards[shard_id].append(repo_config)
        
        # 初始化索引
        self.indexers[repo_config['name']] = IncrementalCodeIndexer(
            ChromaVectorStore(),
            CodeKnowledgeGraph()
        )
        self.indexers[repo_config['name']].index_repository(repo_config['path'])
    
    def set_permissions(self, user: str, allowed_repos: list):
        """设置用户权限"""
        self.permissions[user] = allowed_repos
```

---

## 对比分析：代码RAG vs 文档RAG vs 通用RAG

### 1. 核心差异对比

```
对比维度：

┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│     维度        │    文档RAG      │    代码RAG      │    通用RAG      │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 知识源          │ 自然语言文档    │ 结构化代码      │ 混合            │
│                 │ (Markdown, PDF) │ (Python, Java)  │                 │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 分块单位        │ 段落/章节       │ 函数/类/模块    │ 段落/语义单元   │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 分块策略        │ 按长度/语义     │ 按AST结构       │ 按语义/长度     │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ Embedding模型   │ 通用文本模型    │ 代码专用模型    │ 通用模型        │
│                 │ (BERT, E5)      │ (CodeBERT,      │                 │
│                 │                 │  jina-embeddings│                 │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 检索策略        │ 语义检索        │ 语义+结构+调用图│ 语义+关键词     │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 上下文组装      │ 拼接文档片段    │ 代码+调用链+注释│ 拼接文本片段    │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 答案要求        │ 准确、完整      │ 精确、可运行    │ 准确、相关      │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 评估指标        │ 相关性、完整性  │ 精确性、可执行性│ 相关性、准确性  │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 更新频率        │ 低（文档变更少）│ 高（代码频繁提交│ 中              │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ 权限控制        │ 简单            │ 复杂（行级权限）│ 简单            │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘

关键差异：
1. 代码RAG需要理解程序结构（AST、调用图）
2. 代码RAG的答案需要精确（代码必须可运行）
3. 代码RAG的更新频率高（每次commit都可能需要更新）
4. 代码RAG需要考虑权限（不同开发者访问不同代码）
```

### 2. 技术栈对比

```python
"""
技术栈对比：

文档RAG技术栈：
- 文档解析：PyPDF, python-docx, unstructured
- 分块：RecursiveCharacterTextSplitter
- Embedding：text-embedding-3, BGE, E5
- 向量数据库：Pinecone, Milvus, Weaviate
- 检索：Dense Retrieval
- 生成：GPT-4, Claude

代码RAG技术栈：
- 代码解析：tree-sitter, Pygments, AST
- 分块：AST-based chunker
- Embedding：jina-embeddings-v3, CodeBERT, Qwen3-Embedding
- 向量数据库：Milvus, Pgvector, Chroma
- 检索：Hybrid (Dense + Keyword + Graph)
- 生成：DeepSeek-V4, Qwen3, Claude 4

通用RAG技术栈：
- 解析：多种解析器组合
- 分块：语义分块 + 固定长度
- Embedding：通用Embedding模型
- 向量数据库：通用向量数据库
- 检索：Dense + Sparse
- 生成：通用大模型

代码RAG的特殊组件：
1. 代码解析器（tree-sitter）
2. 知识图谱（调用关系图）
3. 代码专用Embedding
4. 增量索引系统
5. 代码验证器（语法检查、类型检查）
"""
```

### 3. 适用场景对比

```
场景适用性：

文档RAG适用：
✅ 产品文档问答
✅ 技术文档检索
✅ 知识库问答
✅ 政策/法规查询

代码RAG适用：
✅ 代码库问答（"这个函数做什么"）
✅ 代码生成（参考现有代码风格）
✅ Bug修复（检索类似问题的解决方案）
✅ 代码审查（检索相关代码和模式）
✅ 代码导航（"订单模块在哪里"）

通用RAG适用：
✅ 混合类型数据的问答
✅ 企业全面知识检索
✅ 跨领域问题回答

不适用代码RAG的场景：
❌ 纯自然语言问答（不需要代码理解）
❌ 非结构化数据检索（图片、视频）
❌ 实时性要求极高的场景（索引更新延迟）

不适用文档RAG的场景：
❌ 需要精确代码引用的问答
❌ 代码生成和补全
❌ 代码依赖分析
"""
```

---

## 性能分析：检索质量、生成质量与系统性能

### 1. 检索质量评估

```python
"""
检索质量指标：

1. 召回率（Recall）
   Recall@K = |相关文档 ∩ 返回文档| / |相关文档|
   
   代码RAG的目标：
   - Recall@10 > 0.90（Top-10中找到90%的相关代码）
   - Recall@50 > 0.95

2. 精确率（Precision）
   Precision@K = |相关文档 ∩ 返回文档| / |返回文档|
   
   代码RAG的目标：
   - Precision@5 > 0.80（Top-5中80%是相关代码）
   - Precision@10 > 0.70

3. MRR（Mean Reciprocal Rank）
   MRR = (1/|Q|) × Σ(1/rank_i)
   
   代码RAG的目标：MRR > 0.75

4. NDCG（Normalized Discounted Cumulative Gain）
   考虑文档相关度等级的排序质量
   
   代码RAG的目标：NDCG@10 > 0.85

影响检索质量的因素：

因素1：分块质量
- 分块过大：包含噪声，降低精确率
- 分块过小：丢失上下文，降低召回率
- 最优：函数/类级别分块

因素2：Embedding质量
- 通用Embedding vs 代码专用Embedding
- 代码专用Embedding提升15-25%

因素3：检索策略
- 纯语义检索 vs 混合检索
- 混合检索（语义+关键词+图谱）提升20-30%

因素4：重排序
- 无重排序 vs Cross-Encoder重排序
- 重排序提升NDCG@5约10-15%
"""

class RetrievalEvaluator:
    """检索质量评估器"""
    
    def __init__(self):
        self.ground_truth = {}  # query -> [relevant_doc_ids]
    
    def evaluate(self, queries: list, retriever) -> dict:
        """评估检索质量"""
        metrics = {
            'recall@5': [],
            'recall@10': [],
            'precision@5': [],
            'precision@10': [],
            'mrr': [],
            'ndcg@10': []
        }
        
        for query in queries:
            # 获取检索结果
            results = retriever.retrieve(query, k=10)
            retrieved_ids = [r.get('doc_id') or r.get('doc', {}).get('id') 
                           for r in results]
            
            # 获取相关文档
            relevant_ids = self.ground_truth.get(query, [])
            
            if not relevant_ids:
                continue
            
            # 计算指标
            metrics['recall@5'].append(self._recall_at_k(retrieved_ids, relevant_ids, 5))
            metrics['recall@10'].append(self._recall_at_k(retrieved_ids, relevant_ids, 10))
            metrics['precision@5'].append(self._precision_at_k(retrieved_ids, relevant_ids, 5))
            metrics['precision@10'].append(self._precision_at_k(retrieved_ids, relevant_ids, 10))
            metrics['mrr'].append(self._mrr(retrieved_ids, relevant_ids))
            metrics['ndcg@10'].append(self._ndcg_at_k(retrieved_ids, relevant_ids, 10))
        
        # 计算平均值
        return {k: sum(v)/len(v) if v else 0 for k, v in metrics.items()}
    
    def _recall_at_k(self, retrieved: list, relevant: list, k: int) -> float:
        """计算Recall@K"""
        retrieved_k = set(retrieved[:k])
        relevant_set = set(relevant)
        
        if not relevant_set:
            return 0.0
        
        return len(retrieved_k & relevant_set) / len(relevant_set)
    
    def _precision_at_k(self, retrieved: list, relevant: list, k: int) -> float:
        """计算Precision@K"""
        retrieved_k = retrieved[:k]
        if not retrieved_k:
            return 0.0
        
        relevant_set = set(relevant)
        return len(set(retrieved_k) & relevant_set) / len(retrieved_k)
    
    def _mrr(self, retrieved: list, relevant: list) -> float:
        """计算MRR"""
        relevant_set = set(relevant)
        
        for i, doc_id in enumerate(retrieved):
            if doc_id in relevant_set:
                return 1.0 / (i + 1)
        
        return 0.0
    
    def _ndcg_at_k(self, retrieved: list, relevant: list, k: int) -> float:
        """计算NDCG@K"""
        # 简化的DCG计算（假设相关度为1或0）
        dcg = 0.0
        for i, doc_id in enumerate(retrieved[:k]):
            if doc_id in relevant:
                dcg += 1.0 / np.log2(i + 2)
        
        # 理想DCG
        ideal_dcg = sum(1.0 / np.log2(i + 2) for i in range(min(k, len(relevant))))
        
        return dcg / ideal_dcg if ideal_dcg > 0 else 0.0
```

### 2. 生成质量评估

```python
"""
生成质量指标：

1. 答案相关性（Answer Relevance）
   - 答案是否直接回答了用户问题
   - 评估方法：人工评分或LLM-as-a-Judge

2. 代码正确性（Code Correctness）
   - 生成的代码是否可以运行
   - 是否解决了用户问题
   - 评估方法：执行测试用例

3. 引用准确性（Citation Accuracy）
   - 引用的代码文件和行号是否正确
   - 评估方法：人工验证

4. 幻觉率（Hallucination Rate）
   - 答案中是否包含虚构的信息
   - 评估方法：与源代码对比

5. 用户满意度（User Satisfaction）
   - 用户对答案的满意程度
   - 评估方法：用户反馈

提升生成质量的策略：

策略1：提高检索质量
- 更好的分块策略
- 更好的Embedding模型
- 混合检索

策略2：优化上下文组装
- 保留完整的函数定义
- 包含调用关系
- 添加类型信息

策略3：提示词工程
- 明确的指令（"基于提供的代码回答"）
-  Few-shot示例
- 约束输出格式

策略4：后处理验证
- 代码语法检查
- 引用验证
- 逻辑一致性检查

策略5：人机协同
- 低置信度答案转人工
- 用户反馈循环优化
"""
```

### 3. 系统性能评估

```python
"""
系统性能指标：

1. 索引性能
   - 全量索引时间：代码库大小 / 索引速度
   - 增量索引时间：通常 < 1分钟
   - 索引存储大小：通常比原始代码大2-5倍

2. 检索性能
   - 查询延迟：P50 < 100ms, P99 < 500ms
   - 吞吐量：QPS > 1000（单机）
   - 并发能力：取决于向量数据库

3. 生成性能
   - 首token延迟：< 500ms
   - 完整生成时间：取决于输出长度
   - 流式输出延迟：< 100ms/token

4. 资源消耗
   - CPU：索引时高，查询时低
   - 内存：向量存储需要较大内存
   - 存储：Embedding向量 + 原始代码

性能优化策略：

1. 索引优化
   - 增量索引（只更新变更）
   - 批量索引（提高吞吐量）
   - 分层索引（文件/类/函数）

2. 检索优化
   - ANN索引（HNSW, IVF）
   - 缓存热门查询
   - 预过滤（语言、路径）

3. 生成优化
   - 流式输出
   - 模型量化（INT8/INT4）
   - 批处理生成

4. 架构优化
   - 读写分离
   - 分片（按仓库/语言）
   - CDN加速（静态资源）
"""
```

---

## 常见陷阱与最佳实践

### 常见陷阱

```
陷阱1：分块粒度过大或过小
- 过大：包含无关代码，噪声多
- 过小：丢失上下文，无法理解
- 解决：函数/类级别分块，多层次索引

陷阱2：使用通用Embedding模型
- 通用模型不理解代码结构
- 检索质量差
- 解决：使用代码专用Embedding（jina-embeddings-v3, CodeBERT）

陷阱3：忽视代码更新
- 代码频繁变更，索引过期
- 用户得到过时的答案
- 解决：增量索引 + Git钩子自动更新

陷阱4：忽略调用关系
- 只看当前函数，不看调用者/被调用者
- 无法回答"数据流"类问题
- 解决：构建代码知识图谱，检索时包含相关调用链

陷阱5：权限控制缺失
- 所有用户访问所有代码
- 安全风险
- 解决：基于角色的权限控制，索引时过滤无权访问的代码

陷阱6：幻觉问题
- LLM编造不存在的代码
- 引用的行号错误
- 解决：严格的引用验证 + 人机协同审核

陷阱7：性能问题
- 大代码库索引慢
- 查询延迟高
- 解决：分布式索引 + ANN + 缓存

陷阱8：评估不足
- 没有系统的评估体系
- 不知道系统实际效果
- 解决：建立评估数据集，定期评测

陷阱9：上下文窗口溢出
- 检索结果太多，超出LLM上下文限制
- 关键信息被截断
- 解决：智能截断、压缩、层次化上下文

陷阱10：多语言支持不足
- 只支持一种编程语言
- 跨语言检索困难
- 解决：多语言解析器 + 跨语言Embedding
```

### 最佳实践

```python
"""
最佳实践清单：

1. 分块策略
   ✓ 使用AST-based分块
   ✓ 保留函数/类完整性
   ✓ 添加元数据（文件路径、行号、依赖）
   ✓ 多层次索引（文件/类/函数）

2. Embedding选择
   ✓ 代码专用Embedding（jina-embeddings-v3, CodeBERT）
   ✓ 考虑语言和领域
   ✓ 定期更新Embedding模型

3. 检索优化
   ✓ 混合检索（语义+关键词+图谱）
   ✓ 两阶段检索（召回+精排）
   ✓ 查询扩展（同义词、相关概念）

4. 上下文管理
   ✓ 智能组装（去重、排序、截断）
   ✓ 包含调用关系
   ✓ 压缩长代码

5. 索引更新
   ✓ 增量索引
   ✓ Git钩子自动触发
   ✓ 监控索引新鲜度

6. 权限控制
   ✓ 基于角色的访问控制
   ✓ 行级权限（敏感代码）
   ✓ 审计日志

7. 质量保障
   ✓ 引用验证
   ✓ 代码语法检查
   ✓ 人工审核低置信度答案

8. 性能优化
   ✓ 向量索引优化（HNSW）
   ✓ 缓存热门查询
   ✓ 分布式部署

9. 评估体系
   ✓ 建立评估数据集
   ✓ 定期评测检索质量
   ✓ 收集用户反馈

10. 可观测性
    ✓ 全链路追踪
    ✓ 检索质量监控
    ✓ 成本监控
"""

class BestPracticeCodeRAG:
    """遵循最佳实践的代码RAG系统"""
    
    def __init__(self, repo_path: str):
        self.repo_path = repo_path
        
        # 使用AST-based分块
        self.chunker = ASTCodeChunker()
        
        # 使用代码专用Embedding
        self.embedder = CodeEmbedder('jina-embeddings-v3')
        
        # 使用混合检索
        self.retriever = HybridCodeRetriever()
        
        # 使用增量索引
        self.indexer = IncrementalCodeIndexer(
            ChromaVectorStore(),
            CodeKnowledgeGraph()
        )
        
        # 使用知识图谱增强
        self.code_graph = CodeKnowledgeGraph()
        
        # 评估器
        self.evaluator = RetrievalEvaluator()
    
    def setup(self):
        """系统初始化"""
        # 构建知识图谱
        self.code_graph.build_from_repo(self.repo_path)
        
        # 构建索引
        self.indexer.index_repository(self.repo_path)
        
        # 设置Git钩子
        self.indexer.setup_git_hook(self.repo_path)
        
        print("代码RAG系统初始化完成")
    
    def query(self, question: str, user: str = None) -> dict:
        """查询（带最佳实践）"""
        
        # 1. 权限检查
        if user and not self._check_permission(user, question):
            return {'error': '无权访问相关代码'}
        
        # 2. 多路检索
        results = self.retriever.retrieve(question, k=20)
        
        # 3. 知识图谱增强
        enriched_results = self.code_graph.enrich_context(results)
        
        # 4. 上下文组装
        context = self._assemble_context(enriched_results, max_tokens=8000)
        
        # 5. 生成答案
        answer = self._generate(context, question)
        
        # 6. 验证引用
        validated_answer = self._validate_citations(answer, enriched_results)
        
        # 7. 计算置信度
        confidence = self._calculate_confidence(enriched_results)
        
        return {
            'answer': validated_answer,
            'sources': self._extract_sources(enriched_results),
            'confidence': confidence,
            'suggest_human_review': confidence < 0.7
        }
    
    def _check_permission(self, user: str, question: str) -> bool:
        """检查用户权限"""
        # 实际应查询权限系统
        return True
    
    def _assemble_context(self, results: list, max_tokens: int) -> str:
        """组装上下文"""
        assembler = ContextAssembler(max_tokens)
        return assembler.assemble("", results)
    
    def _generate(self, context: str, question: str) -> str:
        """生成答案"""
        # 实际应调用LLM
        return f"基于上下文的答案..."
    
    def _validate_citations(self, answer: str, results: list) -> str:
        """验证引用"""
        # 检查引用的文件是否存在
        for result in results:
            file_path = result.get('metadata', {}).get('file_path', '')
            if file_path and not os.path.exists(file_path):
                # 移除无效引用
                answer = answer.replace(file_path, f"{file_path}(文件已移动或删除)")
        
        return answer
    
    def _calculate_confidence(self, results: list) -> float:
        """计算置信度"""
        if not results:
            return 0.0
        
        top_score = results[0].get('score', 0)
        return min(top_score, 1.0)
    
    def _extract_sources(self, results: list) -> list:
        """提取来源"""
        sources = []
        for r in results[:5]:
            meta = r.get('metadata', {})
            sources.append({
                'file': meta.get('file_path', ''),
                'lines': f"{meta.get('start_line', '')}-{meta.get('end_line', '')}",
                'name': meta.get('name', '')
            })
        return sources
```

---

## 面试题与参考答案

### Q1：代码RAG和普通文档RAG的核心区别是什么？

**参考答案**：

代码RAG和普通文档RAG的核心区别体现在三个层面：

1. **知识表示层面**：
   - 文档RAG处理的是自然语言文本，知识以段落/章节为单位
   - 代码RAG处理的是结构化代码，知识以函数/类/模块为单位
   - 代码RAG需要理解程序结构（AST、调用图、依赖关系），而文档RAG只需要理解语义

2. **检索策略层面**：
   - 文档RAG主要依赖语义检索（Embedding相似度）
   - 代码RAG需要混合检索：语义检索 + 关键词检索（精确匹配函数名） + 图谱检索（调用关系）
   - 代码RAG的分块策略基于AST结构，文档RAG基于文本长度或语义

3. **生成质量层面**：
   - 文档RAG的答案要求准确、完整
   - 代码RAG的答案要求精确、可运行，引用的代码行号必须准确
   - 代码RAG的幻觉问题更严重（编造的代码无法运行），需要更严格的验证

数学上，代码RAG的检索目标函数需要考虑额外的结构约束：
```
Score_code(doc, query) = α × SemanticSimilarity(doc, query) 
                        + β × StructuralMatch(doc, query)
                        + γ × DependencyRelevance(doc, query)
```

### Q2：如何设计代码分块策略？

**参考答案**：

代码分块策略设计需要考虑三个因素：

1. **语法完整性**：
   - 分块边界应该是语法单元的边界（函数、类、方法）
   - 避免将一个函数切分到两个块中
   - 使用tree-sitter等工具基于AST分块

2. **语义完整性**：
   - 每个分块应该包含完整的功能单元
   - 包含必要的上下文（函数签名、关键注释）
   - 对于长函数，可以考虑按逻辑块进一步拆分

3. **检索粒度**：
   - 细粒度（函数级）：检索精确，但可能丢失调用上下文
   - 中粒度（类级）：平衡精确性和上下文
   - 粗粒度（模块级）：上下文完整，但噪声多

**推荐策略：多层次索引**
```python
# 层次1：函数级索引（精确检索）
def process_order(order_id):
    '''处理订单'''
    order = get_order(order_id)
    if not order:
        return None
    # ...

# 层次2：类级索引（中等范围）
class OrderService:
    def create_order(self, ...): ...
    def process_order(self, ...): ...
    def cancel_order(self, ...): ...

# 层次3：文件级索引（broad recall）
# order_service.py - 包含订单相关的所有功能
```

**分块元数据**：
```json
{
  "chunk_id": "uuid",
  "file_path": "src/service/OrderService.java",
  "language": "java",
  "type": "method",
  "name": "processOrder",
  "start_line": 45,
  "end_line": 67,
  "dependencies": ["get_order", "validate_order"],
  "class_name": "OrderService",
  "signature": "public Order processOrder(String orderId)",
  "docstring": "处理订单流程"
}
```

### Q3：如何处理代码RAG中的幻觉问题？

**参考答案**：

代码RAG中的幻觉问题比文档RAG更严重，因为编造的代码无法运行。解决方案包括：

1. **检索增强验证**：
   - 限制LLM只能基于检索到的代码生成答案
   - 在提示词中明确要求"如果信息不足，请明确说明"
   - 使用约束解码（Constrained Decoding）限制输出

2. **引用验证**：
   - 要求LLM引用具体的文件路径和行号
   - 后处理阶段验证引用是否存在
   - 验证引用的代码片段是否与答案一致

3. **代码执行验证**：
   - 对生成的代码进行语法检查（AST解析）
   - 尝试执行生成的代码（在沙箱环境中）
   - 验证编译/解释是否通过

4. **人机协同**：
   - 低置信度答案转人工审核
   - 建立用户反馈机制
   - 定期人工抽查答案质量

5. **模型选择**：
   - 使用代码专用模型（DeepSeek-V4, Qwen3）
   - 这些模型在代码任务上幻觉率更低
   - 温度参数设置为0或接近0，减少随机性

### Q4：代码RAG的索引更新策略有哪些？

**参考答案**：

代码RAG的索引更新策略需要考虑代码变更频繁的特点：

1. **全量索引**：
   - 适用：首次索引、重建索引
   - 优点：简单、一致性好
   - 缺点：耗时、资源占用大
   - 频率：很少（如每周一次）

2. **增量索引**：
   - 适用：日常更新
   - 方法：对比文件修改时间，只更新变更的文件
   - 优点：快速、资源占用少
   - 频率：每次commit或定时（如每5分钟）

3. **实时索引（Git钩子）**：
   - 方法：在Git post-commit钩子中触发索引更新
   - 优点：实时性最好
   - 缺点：可能阻塞commit操作
   - 优化：异步索引（钩子触发消息队列，后台处理）

4. **批量索引**：
   - 适用：大量变更（如分支合并）
   - 方法：收集一段时间内的变更，批量处理
   - 优点：效率高
   - 缺点：有一定延迟

**推荐策略**：
```
日常：增量索引（定时任务，每5分钟）
提交：Git钩子触发消息队列
合并：批量索引（合并后立即执行）
重建：全量索引（每月一次或必要时）
```

**实现要点**：
- 使用文件修改时间（mtime）判断是否需要更新
- 删除的代码需要从索引中移除
- 重命名的文件需要更新索引中的路径
- 保持索引和代码库的一致性

### Q5：如何评估代码RAG系统的效果？

**参考答案**：

代码RAG系统的评估需要从三个维度进行：

1. **检索质量**：
   - 召回率（Recall@K）：相关代码是否被检索到
   - 精确率（Precision@K）：检索结果的相关性
   - MRR（Mean Reciprocal Rank）：第一个相关结果的排名
   - NDCG：考虑相关度等级的排序质量
   - 评估方法：人工标注数据集，或利用代码结构自动生成测试集

2. **生成质量**：
   - 答案相关性：是否直接回答了问题
   - 代码正确性：引用的代码是否准确、可运行
   - 引用准确性：文件路径、行号是否正确
   - 幻觉率：是否包含虚构的信息
   - 评估方法：人工评分、代码执行测试、引用验证

3. **系统性能**：
   - 索引性能：索引构建时间、增量更新延迟
   - 检索性能：查询延迟（P50, P95, P99）、吞吐量（QPS）
   - 生成性能：首token延迟、完整生成时间
   - 资源消耗：CPU、内存、存储

**评估数据集构建**：
```python
# 人工标注数据集
qa_pairs = [
    {
        'question': '订单退款流程是怎么实现的？',
        'relevant_code': [
            {'file': 'src/order/RefundService.java', 'lines': '10-50'},
            {'file': 'src/order/OrderService.java', 'lines': '100-120'}
        ],
        'expected_answer': '退款流程包括...',
        'difficulty': 'medium'
    },
    # ...
]
```

**自动化评估流程**：
1. 构建评估数据集（100-500对QA）
2. 运行系统生成答案
3. 计算检索指标（Recall, Precision, NDCG）
4. 计算生成指标（相关性、正确性）
5. 对比基线系统（如纯关键词搜索）
6. 生成评估报告

### Q6：代码RAG在私有部署时需要考虑哪些安全和合规问题？

**参考答案**：

私有部署代码RAG时的安全和合规考虑：

1. **数据安全**：
   - 代码不出境：如果使用云端Embedding或LLM，需要确保代码数据不离开企业网络
   - 解决方案：私有化部署Embedding模型和LLM，或使用本地向量数据库

2. **权限控制**：
   - 行级权限：不同开发者只能访问其有权限的代码
   - 索引隔离：为不同团队建立独立的索引
   - 审计日志：记录谁访问了什么代码

3. **敏感信息保护**：
   - 代码扫描：索引前扫描并脱敏敏感信息（密码、密钥、API Token）
   - 正则过滤：使用正则表达式检测并替换敏感信息
   - 人工审核：对疑似敏感信息的代码进行人工确认

4. **合规要求**：
   - 开源合规：确保使用的开源组件符合许可证要求
   - 数据保护：遵守GDPR、CCPA等数据保护法规
   - 知识产权：保护企业知识产权，防止泄露

5. **网络安全**：
   - 访问控制：API认证、IP白名单
   - 传输加密：HTTPS/TLS
   - 存储加密：敏感数据加密存储

**实施建议**：
```python
# 敏感信息检测
SENSITIVE_PATTERNS = [
    r'password\s*=\s*["\'][^"\']+["\']',
    r'api_key\s*=\s*["\'][^"\']+["\']',
    r'secret\s*=\s*["\'][^"\']+["\']',
    r'AK[0-9A-Z]{16,}',  # 阿里云AccessKey
    r'sk-[a-zA-Z0-9]{20,}',  # OpenAI API Key
]

def sanitize_code(code: str) -> str:
    """代码脱敏"""
    for pattern in SENSITIVE_PATTERNS:
        code = re.sub(pattern, '[REDACTED]', code)
    return code
```

### Q7：如何优化代码RAG的检索速度？

**参考答案**：

优化代码RAG检索速度的策略：

1. **索引优化**：
   - 使用ANN索引（HNSW、IVF）：将搜索复杂度从O(N)降低到O(logN)
   - 索引分片：按语言、仓库、模块分片，减少单次搜索范围
   - 索引压缩：量化（PQ、SQ）减少索引大小

2. **查询优化**：
   - 查询缓存：缓存热门查询的结果（Redis）
   - 查询预处理：快速过滤明显不相关的结果
   - 查询路由：根据查询类型选择不同的索引

3. **并行化**：
   - 多路检索并行执行（语义、关键词、图谱同时检索）
   - 多线程/多进程处理批量查询
   - 异步I/O（非阻塞API调用）

4. **硬件优化**：
   - GPU加速：Embedding生成和相似度计算
   - 内存优化：将热数据保留在内存中
   - SSD存储：快速读取索引

5. **架构优化**：
   - 读写分离：读多写少的场景
   - CDN加速：静态资源缓存
   - 边缘计算：就近部署索引节点

**性能目标**：
- P50延迟 < 50ms
- P95延迟 < 200ms
- P99延迟 < 500ms
- 单机QPS > 1000

### Q8：2026年代码RAG的发展趋势是什么？

**参考答案**：

2026年代码RAG的发展趋势：

1. **多模态代码理解**：
   - 代码 + 架构图 + 文档的联合理解
   - GLM-5、Gemini 2.0支持代码+图表
   - 自然语言提问，系统理解代码和图表回答

2. **Agent-based代码助手**：
   - 从问答进化到Agent（能执行代码、浏览文件、多步推理）
   - Cursor、Devin等工具的成熟
   - 多Agent协作完成复杂任务

3. **实时协作**：
   - 毫秒级索引更新
   - 实时代码同步和检索
   - 支持多人协作场景

4. **个性化**：
   - 学习个人编码风格
   - 学习团队编码规范
   - 个性化代码推荐

5. **深度分析集成**：
   - 静态分析（SonarQube、CodeQL）
   - 动态分析（性能分析、测试覆盖）
   - 代码质量评分和建议

6. **私有化部署普及**：
   - 企业越来越重视代码安全
   - 开源模型（DeepSeek-V4、Qwen3）能力足够
   - 私有化部署成本降低

7. **跨仓库、跨语言检索**：
   - 支持微服务架构下的跨仓库检索
   - 支持多语言项目（Java + Python + JavaScript）
   - 统一的代码知识图谱

---

*此文原创，转载请注明出处。*
