# 第 8 章 · 完整前向传播：Putting It All Together

---

前面 7 章拆解了每个组件，本章把它们串成完整的 Transformer 前向传播——就像把 `@Component` 拼成一个 Spring Boot 应用。

---

## 8.1 完整 Encoder 实现

```java
/**
 * Transformer Encoder
 * 输入: token IDs [seqLen]
 * 输出: 上下文表示 [seqLen, d_model]
 */
public class TransformerEncoder {
    private final Embedding tokenEmbedding;
    private final SinusoidalPositionalEncoding posEncoding;
    private final Dropout dropout;
    private final List<EncoderLayer> layers;

    public TransformerEncoder(int vocabSize, int dModel, int numLayers, int numHeads, int dFf) {
        this.tokenEmbedding = new Embedding(vocabSize, dModel);
        this.posEncoding = new SinusoidalPositionalEncoding();
        this.dropout = new Dropout(0.1f);
        this.layers = IntStream.range(0, numLayers)
            .mapToObj(i -> new EncoderLayer(dModel, numHeads, dFf))
            .toList();
    }

    public float[][] forward(int[] tokenIds, boolean training) {
        int seqLen = tokenIds.length;

        // Step 1: Token Embedding + Positional Encoding
        float[][] x = tokenEmbedding.forward(tokenIds);  // [seqLen, d_model]
        float[][] pe = posEncoding.encode(seqLen);
        x = add(x, pe);

        // Step 2: Dropout (训练时)
        x = dropout.forward(x, training);

        // Step 3: 逐层传递
        for (EncoderLayer layer : layers) {
            x = layer.forward(x, training);
        }

        return x;  // [seqLen, d_model] — 这就是源语言的"语义表示"
    }
}

/**
 * Encoder 的一层
 */
class EncoderLayer {
    private final MultiHeadAttention selfAttention;
    private final FeedForward feedForward;
    private final LayerNorm norm1, norm2;
    private final Dropout dropout1, dropout2;

    public EncoderLayer(int dModel, int numHeads, int dFf) {
        this.selfAttention = new MultiHeadAttention(dModel, numHeads);
        this.feedForward = new FeedForward(dModel, dFf);
        this.norm1 = new LayerNorm(dModel);
        this.norm2 = new LayerNorm(dModel);
        this.dropout1 = new Dropout(0.1f);
        this.dropout2 = new Dropout(0.1f);
    }

    public float[][] forward(float[][] x, boolean training) {
        // Sublayer 1: Self-Attention
        float[][] attnOutput = selfAttention.forward(x);
        attnOutput = dropout1.forward(attnOutput, training);
        x = norm1.forward(add(x, attnOutput));  // x = LayerNorm(x + Dropout(Attention(x)))

        // Sublayer 2: Feed-Forward
        float[][] ffnOutput = feedForward.forward(x);
        ffnOutput = dropout2.forward(ffnOutput, training);
        x = norm2.forward(add(x, ffnOutput));  // x = LayerNorm(x + Dropout(FFN(x)))

        return x;
    }
}
```

---

## 8.2 完整 Decoder 实现

```java
/**
 * Transformer Decoder（自回归）
 * 输入: 已生成的 token IDs [genLen]
 *       Encoder 输出 [srcLen, d_model]
 * 输出: 每个位置的下一个 token 概率分布 [genLen, vocabSize]
 */
public class TransformerDecoder {
    private final Embedding tokenEmbedding;
    private final SinusoidalPositionalEncoding posEncoding;
    private final Dropout dropout;
    private final List<DecoderLayer> layers;
    private final Linear outputProjection;

    public TransformerDecoder(int vocabSize, int dModel, int numLayers, int numHeads, int dFf) {
        this.tokenEmbedding = new Embedding(vocabSize, dModel);
        this.posEncoding = new SinusoidalPositionalEncoding();
        this.dropout = new Dropout(0.1f);
        this.layers = IntStream.range(0, numLayers)
            .mapToObj(i -> new DecoderLayer(dModel, numHeads, dFf))
            .toList();
        this.outputProjection = new Linear(dModel, vocabSize);
    }

    public float[][] forward(int[] generatedTokenIds, float[][] encoderOutput, boolean training) {
        int genLen = generatedTokenIds.length;

        // Step 1: Token Embedding + Positional Encoding
        float[][] x = tokenEmbedding.forward(generatedTokenIds);
        float[][] pe = posEncoding.encode(genLen);
        x = add(x, pe);
        x = dropout.forward(x, training);

        // Step 2: 生成 Causal Mask
        float[][] causalMask = createCausalMask(genLen);  // 上三角为 -∞

        // Step 3: 逐层传递
        for (DecoderLayer layer : layers) {
            x = layer.forward(x, encoderOutput, causalMask, training);
        }

        // Step 4: 输出投影 → 词表概率分布
        return outputProjection.forward(x);  // [genLen, vocabSize]
    }

    /**
     * 生成因果遮罩矩阵
     * 保证第 t 个位置只能看到 0..t 的信息
     */
    private float[][] createCausalMask(int len) {
        float[][] mask = new float[len][len];
        for (int i = 0; i < len; i++) {
            for (int j = 0; j < len; j++) {
                mask[i][j] = (j > i) ? Float.NEGATIVE_INFINITY : 0.0f;
                //   [ 0    -∞   -∞ ]    pos=0 只能看自己
                //   [ 0     0   -∞ ]    pos=1 可以看 0 和 1
                //   [ 0     0    0 ]    pos=2 可以看 0, 1, 2
            }
        }
        return mask;
    }
}

/**
 * Decoder 的一层
 */
class DecoderLayer {
    private final MultiHeadAttention maskedSelfAttention;
    private final MultiHeadAttention crossAttention;
    private final FeedForward feedForward;
    private final LayerNorm norm1, norm2, norm3;
    private final Dropout dropout1, dropout2, dropout3;

    public DecoderLayer(int dModel, int numHeads, int dFf) {
        this.maskedSelfAttention = new MultiHeadAttention(dModel, numHeads);
        this.crossAttention = new MultiHeadAttention(dModel, numHeads);
        this.feedForward = new FeedForward(dModel, dFf);
        this.norm1 = new LayerNorm(dModel);
        this.norm2 = new LayerNorm(dModel);
        this.norm3 = new LayerNorm(dModel);
        this.dropout1 = new Dropout(0.1f);
        this.dropout2 = new Dropout(0.1f);
        this.dropout3 = new Dropout(0.1f);
    }

    public float[][] forward(float[][] x, float[][] encoderOutput, 
                              float[][] causalMask, boolean training) {
        // Sublayer 1: Masked Self-Attention
        float[][] maskedOutput = maskedSelfAttention.forward(x, causalMask);
        maskedOutput = dropout1.forward(maskedOutput, training);
        x = norm1.forward(add(x, maskedOutput));

        // Sublayer 2: Cross-Attention (Q from Decoder, K/V from Encoder)
        float[][] crossOutput = crossAttention.forward(x, encoderOutput, encoderOutput);
        crossOutput = dropout2.forward(crossOutput, training);
        x = norm2.forward(add(x, crossOutput));

        // Sublayer 3: Feed-Forward
        float[][] ffnOutput = feedForward.forward(x);
        ffnOutput = dropout3.forward(ffnOutput, training);
        x = norm3.forward(add(x, ffnOutput));

        return x;
    }
}
```

---

## 8.3 完整 Transformer（Encoder + Decoder）

```java
/**
 * 完整 Transformer 模型
 * 
 * 论文 Base 版本参数:
 *   N = 6 层
 *   d_model = 512
 *   h = 8 个头
 *   d_ff = 2048
 *   d_k = d_v = 64
 */
public class Transformer {
    private final TransformerEncoder encoder;
    private final TransformerDecoder decoder;
    private final Softmax softmax;

    public Transformer(int srcVocabSize, int tgtVocabSize) {
        int dModel = 512;
        int numLayers = 6;
        int numHeads = 8;
        int dFf = 2048;

        this.encoder = new TransformerEncoder(srcVocabSize, dModel, numLayers, numHeads, dFf);
        this.decoder = new TransformerDecoder(tgtVocabSize, dModel, numLayers, numHeads, dFf);
        this.softmax = new Softmax();
    }

    /**
     * 训练时的前向传播
     */
    public TrainingOutput forward(int[] srcTokenIds, int[] tgtTokenIds) {
        float[][] encoderOutput = encoder.forward(srcTokenIds, true);
        float[][] logits = decoder.forward(tgtTokenIds, encoderOutput, true);
        // logits: [seqLen_tgt, vocabSize]
        return new TrainingOutput(logits);
    }

    /**
     * 推理时的自回归生成（贪心解码）
     */
    public int[] generate(int[] srcTokenIds, int maxLen) {
        float[][] encoderOutput = encoder.forward(srcTokenIds, false);
        int[] generated = {START_TOKEN};

        while (generated[generated.length - 1] != END_TOKEN && generated.length < maxLen) {
            float[][] logits = decoder.forward(generated, encoderOutput, false);
            // 取最后一个位置的预测
            float[] lastLogits = logits[logits.length - 1];
            float[] probs = softmax.forward(lastLogits);
            int nextToken = argmax(probs);
            generated = append(generated, nextToken);
        }
        return generated;
    }
}
```

---

## 8.4 参数总量统计（Base 版本）

```
                    参数量计算
                    
Token Embedding:     srcVocab × d_model                   (≈ 37K × 512)
                        + tgtVocab × d_model              (≈ 37K × 512)
                        ≈ 37.4M

Positional Encoding:  0 (固定函数，无参数)                 0

每层 Encoder:
  Multi-Head Attn:     4 × (d_model × d_model)  = 4 × 512²  ≈ 1.05M
  Feed-Forward:        d_model × d_ff × 2      = 2 × 512 × 2048 ≈ 2.10M
  LayerNorm:           d_model × 4             = 4 × 512  ≈ 2K
  每层小计:                                             ≈ 3.15M

每层 Decoder:
  Masked Attn:         4 × 512²                             ≈ 1.05M
  Cross-Attn:          4 × 512²                             ≈ 1.05M
  Feed-Forward:        2 × 512 × 2048                       ≈ 2.10M
  LayerNorm:           4 × 512 × 1.5(due to 3 norms)       ≈ 3K
  每层小计:                                             ≈ 4.20M

6 层 Encoder × 3.15M  = 18.9M
6 层 Decoder × 4.20M  = 25.2M
Embedding              = 37.4M
────────────────────────────────
总参数 ≈ 81.5M （论文报告 65M，差异来自 vocab 大小等细节）
```

---

## 8.5 计算复杂度分析

```java
/**
 * 单层的计算复杂度
 * 
 * 记:
 *   n = 序列长度 (seqLen)
 *   d = d_model
 * 
 * (1) Self-Attention: QK^T × V
 *     - Q = XW_Q:     O(n · d²)
 *     - K = XW_K:     O(n · d²)
 *     - V = XW_V:     O(n · d²)
 *     - QK^T:         O(n² · d)      ← n² 项！序列长度的瓶颈
 *     - Scores × V:   O(n² · d)
 *     合计: O(n·d² + n²·d)
 * 
 * (2) Feed-Forward: XW_1W_2
 *     - XW_1:         O(n · d · d_ff) = O(n · d · 4d) = O(4n·d²)
 *     - hidden W_2:   O(n · d_ff · d) = O(4n·d²)
 *     合计: O(n·d²)
 * 
 * 总复杂度: O(n·d² + n²·d)
 * 
 * 当 n < d 时（短序列）:  瓶颈是 d² 项 → O(n·d²)
 * 当 n > d 时（长序列）:  瓶颈是 n² 项 → O(n²·d)  ← 这就是为什么长上下文很贵
 */
```

**这就是为什么长上下文（128K/1M）需要 Flash Attention、稀疏 Attention 等优化——因为 `n²` 项在 n=128K 时极其巨大。**

---

## 8.6 内存占用估算

```java
/**
 * 推理时的一次前向传播内存占用（fp16, Base 模型）
 * 
 * 模型权重: 65M × 2 bytes = 130 MB
 * 
 * 激活值 (n=512): 
 *   - Q, K, V:      3 × n × d = 3 × 512 × 512 × 2B ≈ 1.5 MB
 *   - Scores:        n × n × h = 512 × 512 × 8 × 2B ≈ 4 MB
 *   - FFN hidden:    n × d_ff = 512 × 2048 × 2B   ≈ 2 MB
 *   - 其他:         ≈ 2 MB
 *   每层激活: ≈ 9.5 MB
 *   6 层 Encoder:  ≈ 57 MB
 *   6 层 Decoder:  ≈ 57 MB
 *   ─────────────────
 *   总内存 ≈ 130 MB (权重) + 114 MB (激活) ≈ 244 MB
 * 
 * 对于 n=2048: 激活 ≈ 60 MB/层 × 12 层 ≈ 720 MB
 * 对于 n=4096: 激活 ≈ 200 MB/层 × 12 层 ≈ 2.4 GB
 * 
 * n² 的增长是限制上下文长度的根本原因！
 */
```

---

## 8.7 与后续大模型的关系

| 模型 | 基于论文的变动 | 参数量 | 层数 N | d_model | 头数 h |
|---|---|---|---|---|---|
| Transformer Base | 原始 | 65M | 6+6 | 512 | 8 |
| Transformer Big | 原始 | 213M | 6+6 | 1024 | 16 |
| BERT-Large | Encoder-Only | 340M | 24 | 1024 | 16 |
| GPT-3 | Decoder-Only, Pre-LN | 175B | 96 | 12288 | 96 |
| LLaMA 2 7B | Decoder-Only, RoPE, Pre-LN, SwiGLU | 7B | 32 | 4096 | 32 |
| GPT-4 | 架构未公开，专家混合 (MoE) | ~1.8T | 未知 | 未知 | 未知 |

> 注意：GPT-3 已经 175B 参数，但核心 Block 仍然只是 Masked Self-Attention + FFN + 残差 + LayerNorm——和 2017 年的论文几乎一样，只是"放大"了。

---

> **下一章**：[训练与推理](09-training-inference.md)
