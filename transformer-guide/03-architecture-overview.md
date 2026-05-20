# 第 3 章 · 架构全景：Encoder-Decoder

---

## 3.1 整体架构一张图

论文提出的 Transformer 是一个 **Encoder-Decoder** 结构：

```
输入:   "I   love   you"
          │     │     │
    ┌─────┴─────┴─────┴──────┐
    │  Input Embedding       │  ← 把 token 映射为向量 [seqLen, d_model]
    │  + Positional Encoding │  ← 加上位置信息
    └──────────┬─────────────┘
               │
    ┌──────────▼─────────────┐
    │  ENCODER (×N=6 层)     │
    │  ┌───────────────────┐ │
    │  │ Multi-Head Attn   │ │  ← 每个 token 看所有其他 token
    │  │ + Add & Norm      │ │
    │  ├───────────────────┤ │
    │  │ Feed-Forward      │ │  ← 每个 token 独立做非线性变换
    │  │ + Add & Norm      │ │
    │  └───────────────────┘ │
    └──────────┬─────────────┘
               │  encoder output: [seqLen, d_model]
               │
    ┌──────────▼─────────────┐
    │  DECODER (×N=6 层)     │
    │  ┌───────────────────┐ │
    │  │ Masked Multi-Head │ │  ← 只看"已生成的"token（Causal）
    │  │ + Add & Norm      │ │
    │  ├───────────────────┤ │
    │  │ Cross-Attention   │ │  ← 看 Encoder 的输出
    │  │ + Add & Norm      │ │
    │  ├───────────────────┤ │
    │  │ Feed-Forward      │ │
    │  │ + Add & Norm      │ │
    │  └───────────────────┘ │
    └──────────┬─────────────┘
               │
    ┌──────────▼─────────────┐
    │  Linear + Softmax      │  ← 输出每个词的概率分布
    └──────────┬─────────────┘
               │
输出:    "我   爱   你"
```

---

## 3.2 两大模块：Encoder 与 Decoder 的分工

```java
/**
 * Encoder 的职责：把源语言序列编码成"语义表示"
 * 
 * 输入："I love you" → [101, 45, 2073, 2017, 102]  (token IDs)
 * 输出：[seqLen=5, d_model=512] 的浮点矩阵
 *       ┌─────────────────────────┐
 *       │ 0.12 -0.34  0.78  ...   │  ← "I" 的上下文向量
 *       │ 0.45  0.23 -0.11  ...   │  ← "love" 的上下文向量
 *       │ ...                     │
 *       └─────────────────────────┘
 * 这个矩阵已经包含了整个句子的语法和语义信息
 */
public class Encoder {
    private final Embedding embedding;
    private final PositionalEncoding posEncoding;
    private final List<EncoderLayer> layers;

    public float[][] encode(int[] tokenIds) {
        float[][] x = embedding.forward(tokenIds);           // [n, d_model]
        float[][] pe = posEncoding.encode(tokenIds.length);  // [n, d_model]
        x = add(x, pe);

        for (EncoderLayer layer : layers) {
            x = layer.forward(x);  // 逐层传递
        }
        return x;  // 这就是"源语言的语义表示"
    }
}

/**
 * Decoder 的职责：根据 Encoder 的输出，自回归地生成目标序列
 * 
 * 第 1 步：输入 "<s>" → 预测 "我"
 * 第 2 步：输入 "<s> 我" → 预测 "爱"
 * 第 3 步：输入 "<s> 我 爱" → 预测 "你"
 * 第 4 步：输入 "<s> 我 爱 你" → 预测 "</s>" (结束)
 */
public class Decoder {
    private final Embedding embedding;
    private final PositionalEncoding posEncoding;
    private final List<DecoderLayer> layers;
    private final Linear outputProjection;
    private final Softmax softmax;

    public int[] generate(float[][] encoderOutput, int maxLen) {
        int[] generated = {START_TOKEN};

        while (generated[generated.length - 1] != END_TOKEN && generated.length < maxLen) {
            float[][] decoded = forward(encoderOutput, generated);
            // 取最后一个位置的 logits
            float[] lastLogits = decoded[decoded.length - 1];
            // 转换为概率分布，采样/贪心选下一个 token
            int nextToken = argmax(softmax.forward(lastLogits));
            generated = append(generated, nextToken);
        }
        return generated;
    }
}
```

---

## 3.3 Encoder Layer 拆解

每一层 Encoder 包含两个子层：

```java
public class EncoderLayer {
    private final MultiHeadAttention selfAttention;
    private final FeedForward feedForward;
    private final LayerNorm norm1, norm2;

    /**
     * Sublayer 1: Self-Attention + 残差 + LayerNorm
     * Sublayer 2: Feed-Forward  + 残差 + LayerNorm
     */
    public float[][] forward(float[][] x) {
        // Sublayer 1: 每个 token 关注所有其他 token（双向，无遮罩）
        float[][] attnOutput = selfAttention.forward(x, x, x); // Q=K=V 都来自 x
        x = norm1.forward(add(x, attnOutput));   // 残差连接

        // Sublayer 2: 每个位置独立做非线性变换
        float[][] ffnOutput = feedForward.forward(x);
        x = norm2.forward(add(x, ffnOutput));    // 残差连接

        return x;
    }
}
```

**关键点**：Encoder 的 Self-Attention 是**双向**的——处理"love"时，它能看到前面的"I"和后面的"you"。

---

## 3.4 Decoder Layer 拆解

Decoder 比 Encoder 多一个子层：

```java
public class DecoderLayer {
    private final MultiHeadAttention maskedSelfAttention;  // 只看已生成的
    private final MultiHeadAttention crossAttention;       // 看 Encoder 的输出
    private final FeedForward feedForward;
    private final LayerNorm norm1, norm2, norm3;

    public float[][] forward(float[][] x, float[][] encoderOutput) {
        // Sublayer 1: Masked Self-Attention（只看过去，不能看未来）
        float[][] maskedOutput = maskedSelfAttention.forward(x, x, x, causalMask(x.length));
        x = norm1.forward(add(x, maskedOutput));

        // Sublayer 2: Cross-Attention（Q 来自 Decoder，K/V 来自 Encoder）
        float[][] crossOutput = crossAttention.forward(x, encoderOutput, encoderOutput);
        x = norm2.forward(add(x, crossOutput));

        // Sublayer 3: Feed-Forward
        float[][] ffnOutput = feedForward.forward(x);
        x = norm3.forward(add(x, ffnOutput));

        return x;
    }
}
```

**三个子层的分工**：

| 子层 | Decoder 的 Q 来自 | K 和 V 来自 | 目的 |
|---|---|---|---|
| Masked Self-Attn | Decoder 自己的输入 | Decoder 自己的输入（加 mask） | 理解已生成部分的上下文 |
| Cross-Attention | Decoder 当前的表示 | Encoder 的输出 | 找到当前生成词对应的源语言词汇 |
| Feed-Forward | Decoder 当前表示 | 无（独立处理每个位置） | 非线性特征变换 |

---

## 3.5 三种主流变体

原始论文是 Encoder-Decoder，但后来的模型衍生出三种范式：

### 3.5.1 Encoder-Decoder（原始）

```
用途：序列到序列任务（翻译、摘要）
代表：原始 Transformer, T5, BART
结构：Encoder(源) → Decoder(目标)
```

### 3.5.2 Encoder-Only

```
用途：理解任务（分类、实体识别、问答）
代表：BERT, RoBERTa
结构：Encoder(N 层) → Pooling → 分类头

特点：双向 Self-Attention，能看到完整上下文
```

### 3.5.3 Decoder-Only（最重要！）

```
用途：生成任务（对话、代码生成、续写）
代表：GPT系列, LLaMA, Claude, Gemini, DeepSeek
结构：Decoder(N 层) → 自回归生成

特点：Masked Self-Attention + 去掉 Cross-Attention（因为没有外部 Encoder）
```

```java
// Decoder-Only 的简化结构（GPT 类模型）
public class GPTBlock {
    private final MultiHeadAttention maskedSelfAttention; // 只看过去
    private final FeedForward feedForward;
    private final LayerNorm norm1, norm2;

    public float[][] forward(float[][] x) {
        // 只有 Masked Self-Attention + FFN，没有 Cross-Attention
        float[][] attnOutput = maskedSelfAttention.forward(x, x, x, causalMask(x.length));
        x = norm1.forward(add(x, attnOutput));
        float[][] ffnOutput = feedForward.forward(x);
        x = norm2.forward(add(x, ffnOutput));
        return x;
    }
}
```

**这就是今天所有大语言模型的本质：堆叠 N 层仅含 Masked Self-Attention + FFN 的 Block。**

---

## 3.6 数据流全景

```
                      Transformer 数据维度变化
                      
  Encoder:                                Decoder:
                                          
  [seqLen_src, vocabSize]                [seqLen_tgt, vocabSize]
         │ Embedding                              │ Embedding
  [seqLen_src, d_model]                   [seqLen_tgt, d_model]
         │ PE                                   │ PE
  [seqLen_src, d_model]                   [seqLen_tgt, d_model]
         │                                      │
  ┌──────▼──────┐ ×6                    ┌────────▼────────┐ ×6
  │ Self-Attn   │                       │ Masked Self-Attn│
  │   [不变]     │                       │ Cross-Attn      │ ←───── encoder output
  │ Feed-Forward│                       │ Feed-Forward    │     [seqLen_src, d_model]
  │   [不变]     │                       │   [不变]         │
  └──────┬──────┘                       └────────┬────────┘
         │                                      │
  [seqLen_src, d_model]                   [seqLen_tgt, d_model]
                                                │ Linear + Softmax
                                           [seqLen_tgt, vocabSize]
```

**维度不变量**：在整个 Encoder/Decoder 内部，矩阵形状 `[seqLen, d_model]` 始终保持不变。

---

> **下一章**：[Self-Attention 深度数学推导](04-self-attention-deep-dive.md)
