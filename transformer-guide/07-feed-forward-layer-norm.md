# 第 7 章 · Feed-Forward & LayerNorm & 残差连接

---

Transformer 的每个 Block 包含两个主要子层：**Attention** 和 **Feed-Forward**，然后每个子层后面都跟着**残差连接**和**LayerNorm**。本章逐一讲解这三个"配角"——它们不是论文的创新，但没有它们 Transformer 无法工作。

---

## 7.1 Feed-Forward Network (FFN)

### 7.1.1 公式

\[
\text{FFN}(x) = \max(0, xW_1 + b_1) W_2 + b_2
\]

或者等价写成：

\[
\text{FFN}(x) = \text{ReLU}(xW_1 + b_1) W_2 + b_2
\]

### 7.1.2 Java 实现

```java
public class FeedForward {
    // 论文参数: d_model = 512, d_ff = 2048（4 倍）
    private final int d_model;
    private final int d_ff;

    private final float[][] W1;  // [d_model, d_ff]   形状: 512 × 2048
    private final float[] b1;    // [d_ff]             形状: 2048
    private final float[][] W2;  // [d_ff, d_model]    形状: 2048 × 512
    private final float[] b2;    // [d_model]          形状: 512

    /**
     * FFN 对输入的每个位置（token）独立处理
     * @param x 输入 [seqLen, d_model]
     * @return 输出 [seqLen, d_model]
     */
    public float[][] forward(float[][] x) {
        int seqLen = x.length;
        float[][] output = new float[seqLen][d_model];

        for (int pos = 0; pos < seqLen; pos++) {
            // Step 1: 线性投影到高维 + ReLU 激活
            float[] hidden = new float[d_ff];
            for (int j = 0; j < d_ff; j++) {
                float sum = b1[j];
                for (int i = 0; i < d_model; i++) {
                    sum += x[pos][i] * W1[i][j];
                }
                hidden[j] = Math.max(0, sum);  // ReLU: max(0, x)
            }

            // Step 2: 投影回低维
            for (int j = 0; j < d_model; j++) {
                float sum = b2[j];
                for (int i = 0; i < d_ff; i++) {
                    sum += hidden[i] * W2[i][j];
                }
                output[pos][j] = sum;
            }
        }
        return output;
    }
}
```

### 7.1.3 直观理解

```
FFN 的结构像一个"沙漏"：

    [512 维]  ──►  W1  ──►  [2048 维]  ──►  ReLU  ──►  W2  ──►  [512 维]
    输入              升维        高维表示       激活      降维        输出
    
    升维到 4 倍：给模型更大的"思考空间"
    ReLU：引入非线性（没有非线性，多层线性变换等价于一层）
    降维回来：保持维度一致，方便堆叠
```

### 7.1.4 为什么需要 FFN？

Attention 是**线性**操作（加权求和），虽然 Multi-Head 和投影矩阵引入了一些可学习参数，但本质上 Attention 只是对 Value 向量做**凸组合**（每个头的输出是 V 的加权和）。

FFN 引入了**非线性**（ReLU），让模型有能力学习更复杂的特征。

```java
// 没有 FFN 的情况:
// output = Attention(x)  ← 本质上是 V 的加权平均
// → 无论堆多少层 Attention，都只是反复做加权平均

// 加上 FFN:
// output = FFN(Attention(x))  ← Attention 提取关系，FFN 做非线性变换
// → 每层都能学习更抽象的特征
```

### 7.1.5 参数统计

```
W1: 512 × 2048 = 1,048,576
b1:               2,048
W2: 2048 × 512 = 1,048,576
b2:                 512
───────────────────────────
总计:        ≈ 2,099,712 ≈ 2.1M 参数 / 层
```

**FFN 是 Transformer 中参数量最大的部分**——约占每层参数量的 2/3。

---

## 7.2 残差连接（Residual Connection）

### 7.2.1 公式

\[
\text{Output} = \text{LayerNorm}(x + \text{Sublayer}(x))
\]

### 7.2.2 Java 实现

```java
public class ResidualConnection {

    /**
     * 残差连接：把子层的输出加到输入上
     * 
     * 类比: @Around AOP 切面 — 在原有输入基础上叠加处理结果
     */
    public static float[][] residualAdd(float[][] input, float[][] sublayerOutput) {
        int n = input.length;
        int d = input[0].length;
        float[][] result = new float[n][d];

        for (int i = 0; i < n; i++) {
            for (int j = 0; j < d; j++) {
                result[i][j] = input[i][j] + sublayerOutput[i][j];
            }
        }
        return result;
    }
}
```

### 7.2.3 为什么需要残差连接？

残差连接解决了深层网络的**梯度消失**问题：

```java
// 没有残差的情况:
// h = f(x)     → ∂L/∂x = ∂L/∂h · ∂h/∂x  (可能很小)
// 经过 6 层后: ∂L/∂x = ∏ (很小的梯度) → 0

// 有残差的情况:
// h = x + f(x) → ∂L/∂x = ∂L/∂h · (1 + ∂h/∂x)
// "1" 确保了梯度至少有 1 的底子，不会消失到 0
// 这条"高速公路"让梯度可以直接流回最初的层
```

```java
// 直观理解: try-finally 模式
// try {
//     enhanced = attentionLayer(original);  ← 可能信息丢失
// } finally {
//     result = original + enhanced;         ← 保证原始信息不丢
// }
```

---

## 7.3 Layer Normalization

### 7.3.1 公式

\[
\text{LayerNorm}(x) = \gamma \cdot \frac{x - \mu}{\sigma} + \beta
\]

其中：
- `μ` = 每个样本**所有维度**的均值
- `σ²` = 每个样本**所有维度**的方差
- `γ`, `β` = 可学习的缩放和偏移参数

### 7.3.2 Java 实现

```java
public class LayerNorm {
    private final float eps = 1e-5f;  // 防止除零

    // 可学习参数
    private final float[] gamma;   // [d_model] — 缩放
    private final float[] beta;    // [d_model] — 偏移

    public LayerNorm(int dModel) {
        this.gamma = new float[dModel];
        this.beta = new float[dModel];
        // 初始: gamma = 1, beta = 0
        Arrays.fill(gamma, 1.0f);
        Arrays.fill(beta, 0.0f);
    }

    /**
     * 对输入的每一个位置（每一行）独立做 LayerNorm
     * @param x 输入 [seqLen, d_model] 或 [batchSize, d_model]
     * @return 归一化后的输出，形状不变
     */
    public float[][] forward(float[][] x) {
        int n = x.length;
        int d = x[0].length;
        float[][] output = new float[n][d];

        for (int i = 0; i < n; i++) {
            // Step 1: 计算均值 μ
            float mean = 0.0f;
            for (int j = 0; j < d; j++) {
                mean += x[i][j];
            }
            mean /= d;

            // Step 2: 计算方差 σ²
            float variance = 0.0f;
            for (int j = 0; j < d; j++) {
                float diff = x[i][j] - mean;
                variance += diff * diff;
            }
            variance /= d;

            // Step 3: 归一化 + 缩放偏移
            float std = (float) Math.sqrt(variance + eps);
            for (int j = 0; j < d; j++) {
                output[i][j] = gamma[j] * ((x[i][j] - mean) / std) + beta[j];
            }
        }
        return output;
    }
}
```

### 7.3.3 LayerNorm vs BatchNorm

这是 Java 后端开发者容易混淆的概念：

| | BatchNorm | LayerNorm |
|---|---|---|
| **归一化方向** | 同一特征，跨所有样本 | 同一样本，跨所有特征 |
| **Java 类比** | `SELECT AVG(age) FROM users` | `AVG(all_fields) FOR one user` |
| **依赖 batch size** | 是（batch 太小不稳定） | 否（每个样本独立归一化） |
| **NLP 首选** | ❌（句子长度不同） | ✅（每个 token 独立） |
| **推理/训练一致性** | 差（推理时用训练统计量） | 好（行为完全一致） |

```java
// 直观例子：3 个 token 的 LayerNorm
float[][] x = {
    {2.0, 4.0, 6.0},    // token 0
    {1.0, 3.0, 5.0},    // token 1
    {7.0, 8.0, 9.0}     // token 2
};

// token 0 的均值 = 4.0，方差 = 2.67
// 归一化后: [-1.22, 0, 1.22] → 均值为0，方差为1

// token 1 的均值 = 3.0，方差 = 2.67
// 归一化后: [-1.22, 0, 1.22]

// LayerNorm 对每个 token 独立做，互不影响
```

### 7.3.4 为什么选 LayerNorm 而不是 Post-LN？

论文最初用的是 **Post-LN**（LayerNorm 放在残差之后）：

```
原始: x → Sublayer(x) → Add(x, Sublayer(x)) → LayerNorm
```

后来研究发现 **Pre-LN**（LayerNorm 放在残差之前）训练更稳定，梯度流更顺畅：

```
改进: x → LayerNorm → Sublayer(x) → Add(x, Sublayer(x))
```

**现代的 LLaMA、GPT-3+ 等模型都使用 Pre-LN 方案。**

---

## 7.4 三者如何协作：一个 Block 的完整流程

```java
public class TransformerBlock {
    private final MultiHeadAttention attention;
    private final FeedForward feedForward;
    private final LayerNorm norm1, norm2;

    public float[][] forward(float[][] x) {
        // === Sublayer 1: Attention + 残差 + LayerNorm ===
        float[] attnOutput = attention.forward(x);
        x = add(x, attnOutput);         // 残差: x = x + Attention(x)
        x = norm1.forward(x);           // LayerNorm: 稳定数值分布

        // === Sublayer 2: FFN + 残差 + LayerNorm ===
        float[] ffnOutput = feedForward.forward(x);
        x = add(x, ffnOutput);          // 残差: x = x + FFN(x)
        x = norm2.forward(x);           // LayerNorm

        return x;
    }
}
```

**这 3 行模式就是整个 Transformer 的"心脏"**：

```java
output = LayerNorm(input + Sublayer(input));
```

---

## 7.5 Dropout（论文提了但常被忽略）

论文在三个地方用了 Dropout：

```java
// 1. 每个子层输出后（Attention 和 FFN 之后）
attnOutput = dropout(attention.forward(x), rate=0.1);

// 2. Embedding 和 Positional Encoding 相加后
input = dropout(embedding + positionalEncoding, rate=0.1);

// 3. 训练时随机丢弃一些神经元连接，防止过拟合
// 推理时: dropout 关闭（或等价于 rate=0）
```

```java
public class Dropout {
    private final float rate;  // 论文: 0.1 (丢弃 10%)

    public float[][] forward(float[][] x, boolean training) {
        if (!training) return x;  // 推理时不做 dropout

        float[][] output = new float[x.length][x[0].length];
        for (int i = 0; i < x.length; i++) {
            for (int j = 0; j < x[0].length; j++) {
                if (Math.random() < rate) {
                    output[i][j] = 0;              // 丢弃
                } else {
                    output[i][j] = x[i][j] / (1 - rate);  // 缩放剩余神经元
                }
            }
        }
        return output;
    }
}
```

---

> **下一章**：[完整前向传播：Putting It All Together](08-complete-forward-pass.md)
