# 大模型名词解释大全：Token、Embedding、Attention、Softmax 通俗解读，10分钟扫清障碍

## 开篇：别让术语挡住你的路

"今天我们来讲讲 LLM 的 Tokenization 机制，包括 BPE 算法和 SentencePiece 的实现细节，以及 Embedding 在高维空间中的语义表征..."

打住！你是不是也经历过这种场景——兴冲冲点进一篇 AI 教程，第一段就看到一堆陌生名词，然后默默关掉网页？

作为一个从 Java 转行 AI 的过来人，我太懂这种感受了。我在学大模型的前三个月，每天晚上都在 Google 上搜"什么是 Embedding"，第二天又忘了。

今天这篇文章，我用**最接地气的类比**，把大模型领域最核心的 12 个名词全部拆开讲透。不写一个数学公式，只用人话+代码。

读完这篇文章，你再看到这些词，脑子里浮现的将不再是黑人问号，而是清晰的概念映射。

## 一、Token（词元/令牌）

### 是什么？

**Token 是大模型理解文本的最小单位**。就像你写 Java 时的 `int`、`String` 是基本单位，大模型处理文本时用的是 Token。

### 不是按字切的，也不是按词切的

很多人以为一个 Token = 一个汉字，或者一个 Token = 一个英文单词。**都不对！**

```python
# 用 OpenAI 的 tiktoken 库试试
import tiktoken

enc = tiktoken.get_encoding("cl100k_base")

text = "我爱吃苹果"
tokens = enc.encode(text)
print(tokens)        # [57668, 244, 163, 233, 254, 102]
print(len(tokens))   # 6 个 Token！

text_en = "I love apples"
tokens_en = enc.encode(text_en)
print(tokens_en)     # [40, 3021, 19550]  
print(len(tokens_en)) # 3 个 Token
```

你会发现规律：
- **英文**：常见单词约 1 个 Token，罕见词可能被拆分
- **中文**：约 1-2 个汉字算 1 个 Token
- **代码**：关键字可能被单独切成一个 Token

### 为什么用 Token 而不是直接用字？

因为**效率**。如果每个汉字是一个 Token，那 2000 字的文章就有 2000 个 Token，模型的注意力矩阵就是 2000×2000，计算量爆炸。

用 Token 压缩后，可能只有 800 个 Token，计算量直接减少 6 倍多。

### Token 的 Java 类比

```java
// 你可以把 Token 理解成编译原理里的 Lexeme（词法单元）
// 编译器先做词法分析，把代码切分成 Token
// 大模型也是先做分词，把文本切分成 Token

String code = "int x = 10 + 20;";
// 传统编译器的词法分析：
// Token 1: "int" (关键字)
// Token 2: "x"   (标识符)
// Token 3: "="   (运算符)
// Token 4: "10"  (数字)
// Token 5: "+"   (运算符)
// Token 6: "20"  (数字)
// Token 7: ";"   (分隔符)

// 大模型的 Tokenization 类似，但方法不同（用BPE/WordPiece）
```

### Token 的计费意义

你调用 API 是按 Token 计费的：

| 模型 | 输入价格（每百万Token） | 输出价格（每百万Token） |
|------|----------------------|----------------------|
| GPT-4o | $2.50 | $10.00 |
| GPT-4o-mini | $0.15 | $0.60 |
| Claude 3.5 Sonnet | $3.00 | $15.00 |
| DeepSeek V3 | $0.27 | $1.10 |

这就是为什么大家的 Prompt 前面经常加一句"请简练回答"——省钱。

## 二、Embedding（词嵌入/向量化）

### 是什么？

**Embedding 就是把一个 Token 变成一个固定长度的浮点数数组（向量）**。

比如 Token "猫" → [0.021, 0.834, -0.417, 0.556, ...]（通常是 768 维或 1536 维）。

### 为什么需要 Embedding？

计算机只能用数字计算。你要让计算机理解"猫"和"狗"语义相近，那就只能用数字来表示这种"相近"关系——**语义相近的词，它们的向量在空间中距离也近**。

### 一个直观的例子

```python
# 用 OpenAI 的 Embedding API 做个小实验
from openai import OpenAI
import numpy as np

client = OpenAI()

def get_embedding(text):
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text
    )
    return response.data[0].embedding

# 获取几个词的向量
cat_vec = get_embedding("猫")
dog_vec = get_embedding("狗")
car_vec = get_embedding("汽车")
kitten_vec = get_embedding("小猫")

# 计算余弦相似度
def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

print(f"猫 vs 狗:   {cosine_similarity(cat_vec, dog_vec):.4f}")    # 0.85+
print(f"猫 vs 汽车: {cosine_similarity(cat_vec, car_vec):.4f}")    # 0.45-
print(f"猫 vs 小猫: {cosine_similarity(cat_vec, kitten_vec):.4f}") # 0.92+
```

你会发现：**"猫"和"狗"的相似度远高于"猫"和"汽车"**。模型就这样"理解"了语义。

### 一个冷知识

Embedding 向量还可以做加减法：

```python
# 词向量的神奇算术
king = get_embedding("国王")
man = get_embedding("男人")
woman = get_embedding("女人")

# 国王 - 男人 + 女人 ≈ 女王
queen_vec = king - man + woman

# 这个向量最接近的词就是"女王"！
```

这就是 Embedding 的魔力——**语义关系被编码进了向量空间的距离和方向中**。

### Java 中使用 Embedding

```java
// 你可以把 Embedding 存到向量数据库里做语义搜索
public class DocumentSearch {
    private Map<String, float[]> docEmbeddings = new HashMap<>();
    
    public void addDocument(String docId, String content) {
        float[] embedding = callEmbeddingAPI(content); // 调用API获取向量
        docEmbeddings.put(docId, embedding);
    }
    
    public List<String> search(String query, int topK) {
        float[] queryEmbedding = callEmbeddingAPI(query);
        
        // 计算相似度并排序
        return docEmbeddings.entrySet().stream()
            .sorted((a, b) -> Double.compare(
                cosineSimilarity(queryEmbedding, b.getValue()),
                cosineSimilarity(queryEmbedding, a.getValue())
            ))
            .limit(topK)
            .map(Map.Entry::getKey)
            .collect(Collectors.toList());
    }
    
    private float cosineSimilarity(float[] a, float[] b) {
        float dot = 0, normA = 0, normB = 0;
        for (int i = 0; i < a.length; i++) {
            dot += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }
        return (float) (dot / (Math.sqrt(normA) * Math.sqrt(normB)));
    }
}
```

## 三、Attention（注意力机制）

### 是什么？

**Attention 回答的问题是：处理当前位置时，应该"看"输入中的哪些位置，以及看出多少"力气"**。

### 用开会来类比

想象你参加一个 10 人的会议，你是其中一个发言者。轮到你发言时：
- 你的"注意力"主要放在刚才提问的那个人身上（高权重）
- 你也分一部分注意力给老板，看他脸色（中等权重）
- 你基本不管角落里玩手机的同学（低权重）

**Transformer 的 Self-Attention 就是让每个词决定自己要"听"其他哪些词的话。**

### Java 代码类比

```java
// 数据库查询的类比
// Q (Query) = 你要查什么
// K (Key)   = 数据的索引标签
// V (Value) = 数据的实际内容

// 类比：在 Map<String, Person> 中查找名字含"张"的人
// Q = "张"（查询条件）
// K = 每个人的名字（用来匹配的键）
// V = Person 对象的完整信息（要取出的值）

// Attention 的实质就是：
// similarity = dotProduct(Q, K)  → 算出匹配度
// weight = softmax(similarity)   → 归一化为权重
// output = sum(weight * V)       → 加权求和
```

### 自注意力 vs 交叉注意力

| 类型 | 说明 | 类比 |
|------|------|------|
| Self-Attention | Q、K、V 都来自同一个序列 | 自己在心里反复推敲 |
| Cross-Attention | Q来自Decoder，K/V来自Encoder | 翻译时参考原文 |

### 代码实现

```python
import numpy as np

def attention(Q, K, V):
    """
    Q: Query矩阵 (seq_len_q, d_k)
    K: Key矩阵   (seq_len_k, d_k)
    V: Value矩阵 (seq_len_k, d_v)
    """
    d_k = Q.shape[-1]
    
    # 1. 计算相似度分数
    scores = np.dot(Q, K.T)  # (seq_len_q, seq_len_k)
    
    # 2. 缩放（防止点积过大）
    scores = scores / np.sqrt(d_k)
    
    # 3. Softmax 归一化
    weights = np.exp(scores) / np.sum(np.exp(scores), axis=-1, keepdims=True)
    
    # 4. 加权求和
    output = np.dot(weights, V)  # (seq_len_q, d_v)
    
    return output, weights
```

## 四、Softmax

### 是什么？

**Softmax 把一组任意实数转换成一组"概率"，所有概率加起来等于 1。**

### 最简单的例子

```python
# 假设模型算出三个候选词的分值
scores = [2.0, 1.0, 0.1]  # 可能是任何实数

import math

def softmax(scores):
    exp_scores = [math.exp(s) for s in scores]
    total = sum(exp_scores)
    return [e / total for e in exp_scores]

probs = softmax(scores)
print(probs)  # [0.659, 0.242, 0.099]
# 解释：第一个词有65.9%的概率被选中
```

### 为什么需要 Softmax？

1. **把任意值映射到 0~1 之间**：不管输入多大多少，输出一定在合理范围
2. **所有值加起来等于 1**：天然符合"概率分布"的定义
3. **放大差异**：`exp()` 函数会让大的更大、小的更小，让模型更"自信"

### Softmax 在 Attention 里的作用

```python
# 以"我爱你"为例
# 第一个字"我"对三个字的注意力原始分数
raw_scores = [0.8, 0.3, 0.1]

# 经过 Softmax 变成：
attention_weights = [0.60, 0.30, 0.10]
# "我"60%关注自己，30%关注"爱"，10%关注"你"
```

### Temperature 参数对 Softmax 的影响

这就是为什么你可以调节 AI 的"创意程度"：

```python
def softmax_with_temperature(scores, temperature=1.0):
    """
    temperature < 1: 更确定，输出更保守
    temperature > 1: 更随机，输出更创意
    """
    scores = [s / temperature for s in scores]
    exp_scores = [math.exp(s) for s in scores]
    total = sum(exp_scores)
    return [e / total for e in exp_scores]

# 原始分数
scores = [2.0, 1.8, 1.6]

print(f"Temperature 1.0: {softmax_with_temperature(scores, 1.0)}")
# [0.42, 0.34, 0.24] — 差异适中

print(f"Temperature 0.5: {softmax_with_temperature(scores, 0.5)}")
# [0.58, 0.28, 0.14] — 差异拉大，模型更"确定"

print(f"Temperature 2.0: {softmax_with_temperature(scores, 2.0)}")
# [0.38, 0.34, 0.28] — 差异缩小，模型更"随机"
```

## 五、Transformer

前面已经详细讲过，这里简单概括：

> **Transformer = Embedding + Positional Encoding + N层×(Self-Attention + FeedForward + LayerNorm) + 解码输出**

核心思想：**并行的注意力机制取代了串行的RNN，让训练可以充分利用GPU并行计算**。

## 六、LLM（大语言模型）

**LLM = Large Language Model。就是用海量文本数据训练出来的、能理解和生成文本的巨型神经网络。**

常见 LLM：

| 模型 | 参数规模 | 开发者 | 是否开源 |
|------|---------|--------|---------|
| GPT-4o | 未公布(~1.7T) | OpenAI | 闭源 |
| Claude 3.5 | 未公布 | Anthropic | 闭源 |
| LLaMA 3 405B | 4050亿 | Meta | 开源 |
| Qwen 2.5 72B | 720亿 | 阿里 | 开源 |
| DeepSeek V3 | 6710亿(MoE) | 深度求索 | 开源 |

参数规模越大≠越聪明，但**趋势上越大越好**。

## 七、Fine-tuning（微调）

**在预训练模型的基础上，用特定领域的数据再训练一小会儿**。

```python
# 类比Java程序员的理解
# 预训练模型 = JDK（通用的基础能力）
# Fine-tuning = 导入特定业务的Jar包（专有领域的知识）

# 比如你把一个通用模型Fine-tune成：
# - 法律文书生成器
# - Java代码审查助手
# - 医疗报告摘要工具
```

微调不是从头训练，只是"校准"：

```python
from transformers import AutoModelForCausalLM, AutoTokenizer, Trainer

# 加载预训练模型（这步就像 import java.util.*）
model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-7B")

# 在你的数据上微调（这步就像写自己的业务代码）
trainer = Trainer(
    model=model,
    train_dataset=my_legal_dataset,
    ...
)
trainer.train()
```

## 八、RAG（检索增强生成）

**让模型在回答前先去"翻书"（检索外部知识库），把找到的内容附在 Prompt 里再回答**。

```python
# RAG 的四步流程
def rag_pipeline(user_question):
    # 1. 把用户问题转成向量
    question_embedding = embedding_model.encode(user_question)
    
    # 2. 在向量数据库中检索相似内容
    relevant_docs = vector_db.search(question_embedding, top_k=5)
    
    # 3. 把检索到的文档拼进 Prompt
    prompt = f"""基于以下参考资料回答问题：
    
参考资料：
{relevant_docs}

用户问题：{user_question}

请基于参考资料回答，如果参考资料中没有相关信息，请明确说明。"""
    
    # 4. 发给大模型
    answer = llm.chat(prompt)
    return answer
```

**RAG 的价值**：
- 不需要重新训练模型就能更新知识
- 回答有据可查，减少幻觉
- 企业数据不会泄露给模型训练

## 九、Prompt Engineering（提示词工程）

**如何写 Prompt 以获得更好的输出**。

几个核心技巧：

```python
# 1. 角色设定
prompt = "你是一个资深Java架构师，请审查以下代码..."

# 2. 结构化输出
prompt = "请以JSON格式返回结果：{\"问题\": \"...\", \"修复建议\": \"...\", \"风险等级\": \"...\"}"

# 3. Few-shot（给示例）
prompt = """
将以下中文翻译成英文：
中文：你好 → 英文：Hello
中文：谢谢 → 英文：Thank you
中文：再见 → 英文："""
# 模型会自然地输出 "Goodbye"

# 4. Chain-of-Thought（思维链）
prompt = "请逐步推理：如果 x + 2 = 5，那么 x = ?\n第一步：\n第二步：\n答案："
```

## 十、Hallucination（幻觉）

**模型一本正经地胡说八道**。

```
用户：你知道李白写过什么关于Java的诗吗？
模型：李白的《将进酒·Java版》中写道："君不见 import 之水天上来..."
```

别笑，类似的事情真发生过。模型不是在"撒谎"，它只是在做"下一个字预测"时，根据统计规律生成了似模似样的内容。

防范方法：
- 使用 RAG（上文讲过）
- 降低 Temperature
- 让模型先说"不确定"再回答
- 要求模型引用来源

## 十一、Context Window（上下文窗口）

**模型一次能"看到"的最大 Token 数量**。

| 模型 | 上下文窗口 |
|------|-----------|
| GPT-4o | 128K |
| Claude 3.5 Sonnet | 200K |
| Gemini 1.5 Pro | 2M |
| Qwen 2.5 | 128K |

128K Token ≈ 一本中篇小说。所以现在可以让模型完整读完一本书再回答问题。

## 十二、Inference（推理）

**用训练好的模型做预测（回答问题、生成文本）的过程**。

- **Training（训练）**：学知识，耗时数周到数月，烧电费
- **Inference（推理）**：用知识，每次几毫秒到几秒，相对便宜

就是你调用一次 `/v1/chat/completions` API 的过程。

## 总结：一图记住所有名词

```
用户输入文本
    ↓
[Tokenization] → 切成 Token 们
    ↓
[Embedding]    → 每个 Token 变成一个向量
    ↓
[Attention]    → 各 Token 互相"看"彼此，计算关系
    ↓
[Transformer]  → 反复×N层，逐步理解+生成
    ↓
[Softmax]      → 把分数变成概率
    ↓
[Sampling]     → 按概率选下一个 Token
    ↓
重复直到生成完整回答 → 返回给用户
```

可选增强：
- **RAG**：生成前去知识库"翻资料"
- **Fine-tuning**：用特定领域数据"特训"过
- **Prompt Engineering**：写好"提示词"获得更好输出

这就是大模型的全部核心概念。看到这里，你已经超越了 90% 对 AI 一知半解的人。

---

**下篇预告**：从 1958 年的感知机聊到 2024 年的 GPT-4，我把 AI 70 多年的历史浓缩成 5 个里程碑故事，保证比听评书还有意思。敬请期待《AI 发展简史：从感知机到 GPT-4，程序员必知的 5 个里程碑》。
