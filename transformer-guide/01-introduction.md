# 第 1 章 · 引言：为什么 Java 开发者需要读懂这篇论文

---

## 1.1 一篇论文引发的 AI 革命

2017 年 6 月，Google Brain 团队的 Ashish Vaswani 等八位研究者向 NeurIPS 会议提交了一篇标题大胆的论文——《Attention Is All You Need》。论文只有 15 页，没有复杂的公式推导，核心思想极其简洁：

> **扔掉 RNN/LSTM，仅用 Self-Attention 机制就能完成所有序列建模任务。**

当时没人能预料到，这篇论文会催生出一个数万亿美元的产业。下图展示了这篇论文的影响力路径：

```
Attention Is All You Need (2017.06)
        │
        ├──► BERT (2018.10, Google) ── 横扫 NLP 基准
        │       └──► Encoder-Only 范式
        │
        ├──► GPT-1 (2018.06, OpenAI) ── 验证 Decoder-Only
        │       ├──► GPT-2 (2019.02) ── "Too dangerous to release"
        │       ├──► GPT-3 (2020.05) ── 175B 参数，Few-shot 能力涌现
        │       ├──► GPT-3.5 / ChatGPT (2022.11) ── 2 个月破亿用户
        │       └──► GPT-4 (2023.03) ── 多模态、Agent 能力
        │
        ├──► T5 (2019.10) ── Text-to-Text 统一框架
        ├──► ViT (2020.10) ── Transformer 进入计算机视觉
        ├──► CLIP / DALL-E ── 多模态融合
        └──► LLaMA / Claude / Gemini / DeepSeek ── 开源生态
```

---

## 1.2 为什么是 Java 开发者？

你可能在想：*"我是个写 Spring Boot 的，为什么要看一篇 2017 年的深度学习论文？"*

答案很直白——**因为你的项目里已经到处都是 Transformer 了**。

### 你已经在用 Transformer 的场景

```java
// 场景 1：调用 OpenAI API 做文本生成
@PostMapping("/chat")
public String chat(@RequestBody String prompt) {
    ChatCompletionRequest request = ChatCompletionRequest.builder()
        .model("gpt-4")
        .maxTokens(2048)     // ← 这里的 maxTokens 就是 Decoder 循环次数上限
        .temperature(0.7)    // ← Temperature 控制 Softmax 的"陡峭度"
        .messages(List.of(new Message("user", prompt)))
        .build();
    return openAiService.createChatCompletion(request)
        .getChoices().get(0).getMessage().getContent();
}

// 场景 2：RAG（检索增强生成）
List<Document> chunks = textSplitter.split(document, 512); // ← 512 是论文的原始 seqLen
float[][] embeddings = embeddingModel.embed(chunks);       // ← 1536 维 = d_model

// 场景 3：本地推理
OnnxModel model = OnnxModel.load("llama-7b.onnx");        // ← ONNX 导出的 Transformer
```

**如果你不理解 Token、Context、Temperature、Embedding Dimension 的底层逻辑，你永远只能"调 API"而无法"做设计"。**

---

## 1.3 本书的核心理念：用 Java 思维理解 Transformer

传统 AI 教材的路径是：

```
线性代数 → 概率论 → 机器学习基础 → RNN/LSTM → Attention → Transformer
```

这个过程需要半年以上。本书采用完全不同的路径：

```
你已知的 Java 概念 → Transformer 对应概念 → 论文公式 → Java 伪代码直译
```

### 一个例子：Attention 就是带权重的 SELECT JOIN

```sql
-- 传统 SQL
SELECT value, SUM(similarity * value) AS weighted_output
FROM (
    SELECT t1.value, 
           COSINE_SIMILARITY(t1.query, t2.key) AS similarity
    FROM tokens t1, tokens t2
) GROUP BY t1.position;

-- 这就是 Self-Attention 的 SQL 版！
```

```java
// Java 流式版本
tokens.parallelStream()
    .map(token -> {
        double[] weights = tokens.stream()
            .mapToDouble(other -> dotProduct(token.query(), other.key()))
            .toArray();
        weights = softmax(weights);
        return weightedSum(tokens, weights);
    })
    .toList();
```

---

## 1.4 本书结构导航

本书分为四个部分：

| 部分 | 章节 | 目标 |
|---|---|---|
| **基础篇** | 第 1-3 章 | 理解 Transformer 是什么、为什么要取代 RNN |
| **核心篇** | 第 4-6 章 | 彻底搞懂 Self-Attention、Multi-Head、位置编码 |
| **系统篇** | 第 7-9 章 | FFN、LayerNorm、训练与推理完整流程 |
| **实战篇** | 第 10-15 章 | KV Cache、Spring Boot 集成、FAQ |

---

## 1.5 学习前提

你需要：
- 熟悉 Java 基本语法（`int[][]`, `for`, `List`, `Stream`）
- 了解矩阵乘法的基本概念（行 × 列）
- 知道什么是概率归一化（如 `Math.exp` 后的 `sum=1.0`）

你**不需要**：
- Python 知识
- 深度学习框架经验（PyTorch / TensorFlow）
- 微积分、凸优化等高阶数学
- 任何 RNN/LSTM 的先验知识

---

## 1.6 论文原始信息

| 项目 | 内容 |
|---|---|
| 标题 | *Attention Is All You Need* |
| 作者 | Ashish Vaswani, Noam Shazeer, Niki Parmar, Jakob Uszkoreit, Llion Jones, Aidan N. Gomez, Łukasz Kaiser, Illia Polosukhin |
| 机构 | Google Brain, Google Research |
| 发表 | NeurIPS 2017 |
| 页数 | 15 页 |
| 引用次数 | 140,000+（截至 2025 年） |
| 核心贡献 | 提出 Transformer 架构，仅用 Self-Attention 替代 RNN |

---

> **下一章**：[核心思想：从 `for` 循环到 `parallelStream`](02-core-idea.md)
