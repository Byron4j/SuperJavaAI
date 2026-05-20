# 第 4 章 · Self-Attention 深度数学推导

---

> 这一章是本书最重要的章节。如果你只能读一章，就读这一章。

---

## 4.1 论文核心公式

\[
\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right) V
\]

这就是整篇论文的灵魂。15 页论文的价值浓缩在这 8 个符号里。

```
分解为 4 步：
 (1)  QK^T       → 计算相关性分数矩阵
 (2)  /√d_k      → 缩放，防梯度消失
 (3)  softmax    → 归一化为注意力权重
 (4)  ×V         → 按权重聚合信息
```

---

## 4.2 第一步：Q、K、V 从哪来？

Q、K、V 都是输入 X 经过**不同的**线性变换得到的：

```java
public class SelfAttentionProjections {
    // 可学习的权重矩阵（训练时更新）
    private final float[][] W_Q;  // [d_model, d_k] — 论文: d_k = d_model / h = 64
    private final float[][] W_K;  // [d_model, d_k]
    private final float[][] W_V;  // [d_model, d_v] — 论文: d_v = d_k = 64

    /**
     * 输入 X: [seqLen, d_model]
     * 输出: Q, K, V 三个矩阵
     */
    public ProjectionResult project(float[][] X) {
        float[][] Q = matMul(X, W_Q);  // [seqLen, d_k]
        float[][] K = matMul(X, W_K);  // [seqLen, d_k]
        float[][] V = matMul(X, W_V);  // [seqLen, d_v]
        return new ProjectionResult(Q, K, V);
    }
}
```

### 为什么需要三个矩阵？

```
Q (Query)   = "我想找什么？"     — 每个 token 的"搜索意图"
K (Key)     = "我有什么可被搜的？" — 每个 token 的"索引标签"
V (Value)   = "我的实际内容是什么？" — 每个 token 的"载荷数据"
```

**类比：数据库查询**

```sql
-- 在 Transformer 中，所有 token 既是查询方，也是被查方
SELECT v.content, 
       SUM(similarity(q, k) * v.content) AS weighted_content
FROM vocabulary AS q        -- Query: 当前 token 想找什么
CROSS JOIN vocabulary AS k  -- Key:   所有 token 的索引
JOIN vocabulary AS v ON k.id = v.id  -- Value: 实际内容
GROUP BY q.id;
```

```java
// 每个 token 生成自己的 Q, K, V
record TokenView(float[] query, float[] key, float[] value) {}

List<TokenView> views = tokens.stream()
    .map(x -> {
        float[] q = matVecMul(W_Q, x);  // 64 维查询向量
        float[] k = matVecMul(W_K, x);  // 64 维索引向量
        float[] v = matVecMul(W_V, x);  // 64 维内容向量
        return new TokenView(q, k, v);
    })
    .toList();
```

---

## 4.3 第二步：QK^T —— 计算"注意力分数"矩阵

```
Q: [seqLen, d_k]    ×    K^T: [d_k, seqLen]    =    Scores: [seqLen, seqLen]

        ┌─────────┐         ┌──────────┐              ┌───────────┐
        │ q_1 ·   │         │ k_1 k_2..│              │ 1.2 -0.3..│
        │ q_2 ·   │    ×    │   ..     │    =         │ 0.7  2.1..│
        │ q_3 ·   │         └──────────┘              │ 0.1  0.8..│
        └─────────┘                                   └───────────┘
         "每个 token    "每个 token                     "score[i][j] =
          想知道什么"     有什么特征"                     token_i 对 token_j
                                                       的关注度"
```

```java
/**
 * 计算注意力分数矩阵
 * Scores[i][j] = 第 i 个 token 的 Query 与第 j 个 token 的 Key 的点积
 */
public static float[][] computeScores(float[][] Q, float[][] K) {
    int seqLen = Q.length;
    int d_k = Q[0].length;
    float[][] scores = new float[seqLen][seqLen];

    for (int i = 0; i < seqLen; i++) {
        for (int j = 0; j < seqLen; j++) {
            // dotProduct(q_i, k_j) = Σ q_i[t] · k_j[t]
            float dot = 0.0f;
            for (int t = 0; t < d_k; t++) {
                dot += Q[i][t] * K[j][t];
            }
            scores[i][j] = dot;
        }
    }
    return scores;
}
```

**直觉**：
- `scores[i][j]` 越大 → token i 和 token j 越相关
- `scores[i][j]` 可以是**任意实数**（正或负）
- 点积衡量两个向量的**方向相似度**：同向 → 大正数，正交 → 0，反向 → 大负数

---

## 4.4 第三步：÷ √d_k —— 为什么要缩放？

```java
// 不加缩放的危险
int d_k = 64;   // 每个头的 Query/Key 维度
// Q、K 的每个元素假设服从 N(0,1)，那么：
// E[q·k] = 0
// Var(q·k) = d_k = 64  ← 方差随维度线性增长！

// 当 d_k = 64, Var = 64 → 标准差 = 8
// 有些点积值可能达到 ±30，导致 softmax 后梯度接近 0

// 缩放后:
// Var(q·k / √d_k) = d_k / d_k = 1  ← 完美！保持方差为 1
// 保证了梯度的稳定传播
```

```java
// 数学原理
// 假设 Q 和 K 的每个元素独立同分布，均值为 0，方差为 1
// dot(Q_i, K_j) = Σ q_it * k_jt
// E[dot] = Σ E[q_it] * E[k_jt] = Σ 0 * 0 = 0
// Var[dot] = Σ Var[q_it * k_jt] = Σ Var[q_it] * Var[k_jt] = Σ 1 * 1 = d_k

// 缩放: dot / √d_k
// Var[dot / √d_k] = d_k / d_k = 1  ← 无论 d_k 多大，方差恒为 1
```

**如果不用 `√d_k` 缩放会怎样？** Softmax 在大数值上会退化：

```java
// 假设未缩放的点积值为: [50, 2, 10, -5]
float[] scores = {50, 2, 10, -5};
float[] probs = softmax(scores);
// 结果: [1.000, 0.000, 0.000, 0.000]  ← 几乎完全集中到最大值
// 梯度: [≈0, ≈0, ≈0, ≈0]              ← 所有梯度接近 0，无法学习

// 缩放后: scores/8 = [6.25, 0.25, 1.25, -0.625]
// 结果: [0.97, 0.002, 0.024, 0.004]   ← 仍然有区分度，但相对平缓
// 梯度: [0.03, 0.002, 0.023, 0.004]   ← 有意义的梯度，可学习
```

---

## 4.5 第四步：Softmax —— 从分数到概率权重

```java
/**
 * Softmax: 把任意实数向量转为概率分布（和为 1）
 * 
 * softmax(x_i) = e^{x_i} / Σ e^{x_j}
 */
public static float[] softmax(float[] x) {
    int n = x.length;

    // Step 1: 减去最大值（数值稳定性技巧）
    // e^50 会溢出 float 范围！减去最大值后保证输入 ≤ 0
    float max = Float.NEGATIVE_INFINITY;
    for (float v : x) max = Math.max(max, v);

    // Step 2: e^x
    float sum = 0.0f;
    float[] exp = new float[n];
    for (int i = 0; i < n; i++) {
        exp[i] = (float) Math.exp(x[i] - max);
        sum += exp[i];
    }

    // Step 3: 归一化
    for (int i = 0; i < n; i++) {
        exp[i] /= sum;
    }
    return exp;
}

// 矩阵版本：对 scores 的每一行做 softmax
public static float[][] rowSoftmax(float[][] scores) {
    float[][] weights = new float[scores.length][];
    for (int i = 0; i < scores.length; i++) {
        weights[i] = softmax(scores[i]);
    }
    return weights;
}
```

**Softmax 的直观理解**：

```java
// 输入:  ["cat", "dog", "the", "sat"]
// 处理 "sat" 时的注意力分数（未经缩放）:
// scores = [3.2, 1.1, 0.3, 5.0]

// Softmax 后:
// weights = [0.14, 0.02, 0.01, 0.83]
//            ↑     ↑     ↑     ↑
//           cat   dog   the   "sat" 自己（自己对自己关注最高）

// 含义: "sat" 在理解自己时，83% 注意力放在自己身上，
//       14% 放在 "cat"（因为"猫在坐"），
//       其它词几乎忽略。
```

---

## 4.6 第五步：× V —— 加权聚合

```java
/**
 * 最终输出: 按注意力权重对 Value 向量做加权求和
 * 
 * Output[i] = Σ_j weights[i][j] · V[j]
 * 
 * 权重矩阵 × Value矩阵 = 新表示
 * [seqLen, seqLen] × [seqLen, d_v] = [seqLen, d_v]
 */
public static float[][] aggregate(float[][] weights, float[][] V) {
    int seqLen = weights.length;
    int d_v = V[0].length;
    float[][] output = new float[seqLen][d_v];

    for (int i = 0; i < seqLen; i++) {
        // 第 i 个 token 的新表示 = 所有 token 的 Value 的加权和
        for (int j = 0; j < seqLen; j++) {
            float w = weights[i][j]; // token i 对 token j 的注意力权重
            for (int d = 0; d < d_v; d++) {
                output[i][d] += w * V[j][d];
            }
        }
    }
    return output;
}
```

---

## 4.7 完整代码：四步合一

```java
/**
 * Self-Attention 完整实现
 * 
 * @param X 输入矩阵 [seqLen, d_model]
 * @param W_Q, W_K, W_V 可学习的投影矩阵
 * @return 注意力输出 [seqLen, d_v]
 */
public class ScaledDotProductAttention {

    private final float[][] W_Q, W_K, W_V;

    public float[][] forward(float[][] X) {
        int seqLen = X.length;
        int d_k = W_K[0].length;

        // 1. 投影得到 Q, K, V
        float[][] Q = matMul(X, W_Q);  // [seqLen, d_k]
        float[][] K = matMul(X, W_K);  // [seqLen, d_k]
        float[][] V = matMul(X, W_V);  // [seqLen, d_v]

        // 2. 计算分数: Q × K^T
        float[][] scores = matMul(Q, transpose(K));  // [seqLen, seqLen]

        // 3. 缩放: / √d_k
        float scale = (float) Math.sqrt(d_k);
        for (int i = 0; i < seqLen; i++) {
            for (int j = 0; j < seqLen; j++) {
                scores[i][j] /= scale;
            }
        }

        // 4. Softmax 行归一化 → 注意力权重
        float[][] weights = rowSoftmax(scores);  // [seqLen, seqLen]

        // 5. 加权聚合: weights × V
        float[][] output = matMul(weights, V);  // [seqLen, d_v]

        return output;
    }
}
```

---

## 4.8 数值示例：用手算理解 Attention

假设一个极端简化的场景：
- `seqLen = 3`（句子："I love you"）
- `d_model = 2`（2 维 Embedding，只为演示）
- `d_k = d_v = 2`

```java
// 输入 Embedding（已包含 Positional Encoding）
float[][] X = {
    {0.8,  0.2},   // "I"
    {0.5,  0.9},   // "love"
    {0.1,  0.7}    // "you"
};

// 假设投影矩阵 W_Q = W_K = W_V = I（单位矩阵，简化）
float[][] Q = X;  // 与实际一样
float[][] K = X;  // 与实际一样
float[][] V = X;  // 与实际一样

// ===== Step 2: Q × K^T =====
//           Q            K^T              scores
//     [0.8  0.2]   [0.8  0.5  0.1 ]   [0.68  0.58  0.22]
//     [0.5  0.9] × [0.2  0.9  0.7 ] = [0.58  1.06  0.68]
//     [0.1  0.7]                     [0.22  0.68  0.50]
//
// 其中 scores[1][1] = 0.5×0.5 + 0.9×0.9 = 1.06  ← "love"与自己的相似度

// ===== Step 3: 缩放（d_k = 2, sqrt(2) ≈ 1.414）=====
// scores[0] = [0.48, 0.41, 0.16]
// scores[1] = [0.41, 0.75, 0.48]
// scores[2] = [0.16, 0.48, 0.35]

// ===== Step 4: Softmax 行归一化 =====
// weights[0] = [0.39, 0.36, 0.25]  ← "I" 看 "I":39%, "love":36%, "you":25%
// weights[1] = [0.32, 0.45, 0.23]  ← "love" 看自己最多
// weights[2] = [0.28, 0.38, 0.34]

// ===== Step 5: weights × V =====
// output[0] = 0.39×[0.8,0.2] + 0.36×[0.5,0.9] + 0.25×[0.1,0.7]
//           = [0.31+0.18+0.025, 0.08+0.32+0.175]
//           = [0.52, 0.58]
//
// output[1] = 0.32×[0.8,0.2] + 0.45×[0.5,0.9] + 0.23×[0.1,0.7]
//           = [0.26+0.23+0.02, 0.06+0.41+0.16]
//           = [0.51, 0.63]
//
// output[2] = 0.28×[0.8,0.2] + 0.38×[0.5,0.9] + 0.34×[0.1,0.7]
//           = [0.22+0.19+0.03, 0.06+0.34+0.24]
//           = [0.44, 0.64]

// 观察: 每个输出向量都是所有输入向量的"融合"，融合比例由注意力权重决定
```

**关键洞察**：经过 Self-Attention 后，"love" 的表示从 `[0.5, 0.9]` 变成了 `[0.51, 0.63]`——它融合了 "I" 和 "you" 的信息变得不同。这正是 **"Contextualized Representation"（上下文相关表示）** 的含义。

---

## 4.9 Self-Attention 的梯度流

反向传播时的关键梯度：

```java
/**
 * 以 Q 为例的梯度计算
 * 
 * output = softmax(QK^T / √d_k) × V
 * 
 * ∂L/∂Q = 1/√d_k · [(∂L/∂output) · V^T - output · ((∂L/∂output)^T · output)] · K
 * 
 * 完整推导较复杂，核心要点：
 * 1. Softmax 的梯度是 Jacobian: ∂softmax(x)_i / ∂x_j = softmax_i·(δ_ij - softmax_j)
 * 2. 因为缩放了 1/√d_k，梯度也相应放大 √d_k 倍（所以需要缩放！）
 * 3. 每个 token 的梯度都流经了与所有其他 token 的注意力权重
 */
```

---

## 4.10 为什么叫做"Self"-Attention？

因为 **Q、K、V 都来自同一个输入**——序列在"自己注意自己"。

```
普通 Attention:  Q 来自序列A, K/V 来自序列B  （如 Encoder-Decoder 之间的 Cross-Attention）
Self-Attention:   Q、K、V 都来自同一个序列    （如 Encoder 内部、Decoder 的 Masked Self-Attention）
```

---

> **下一章**：[Multi-Head Attention](05-multi-head-attention.md)
