# 第 5 章 · Multi-Head Attention

---

## 5.1 单头的局限性

单头注意力只有一个 Q、K、V 投影，模型只能学习**一种类型**的关系模式：

```
单头可能学到: "主谓关系"（主语 → 谓语）
但学不到: "指代关系"（代词 → 实体）
也学不到: "修饰关系"（形容词 → 名词）
更学不到: "语义相似性"（同义词/近义词）
```

论文的解决方案：**Multi-Head Attention**——让模型同时拥有多个"视角"。

---

## 5.2 概念：多个注意力头并行运行

```java
public class MultiHeadAttention {
    // 论文参数: h = 8 个头, d_model = 512, d_k = d_v = 64
    private final int numHeads;    // h = 8
    private final int d_model;     // 512
    private final int d_k;         // d_model / h = 64 (每个头的 Q/K 维度)
    private final int d_v;         // d_model / h = 64 (每个头的 V 维度)

    // 每个头有自己独立的 4 个投影矩阵
    // 总共 4 × 8 = 32 个矩阵，但每个很小（512×64 = 32K 参数）
    private final float[][][] W_Q_heads;  // [h][d_model][d_k]
    private final float[][][] W_K_heads;  // [h][d_model][d_k]
    private final float[][][] W_V_heads;  // [h][d_model][d_v]
    
    // 输出投影（拼接后还原维度）: [h*d_v, d_model] = [512, 512]
    private final float[][] W_O;

    // ===== 每个头的计算完全相同 =====
    // head_i = Attention(Q·W_Qi, K·W_Ki, V·W_Vi)
    
    public float[][] forward(float[][] X) {
        int seqLen = X.length;

        // 所有头的结果
        float[][][] headOutputs = new float[numHeads][seqLen][d_v];

        // 每个头独立计算
        for (int h = 0; h < numHeads; h++) {
            // 投影到低维空间
            float[][] Q = matMul(X, W_Q_heads[h]);  // [seqLen, d_k]
            float[][] K = matMul(X, W_K_heads[h]);  // [seqLen, d_k]
            float[][] V = matMul(X, W_V_heads[h]);  // [seqLen, d_v]

            // 标准 Scaled Dot-Product Attention
            headOutputs[h] = scaledDotProductAttention(Q, K, V);
        }

        // 拼接所有头: [numHeads][seqLen][d_v] → [seqLen, numHeads*d_v]
        float[][] concat = concatHeads(headOutputs);  // [seqLen, 512]

        // 最终线性投影
        return matMul(concat, W_O);  // [seqLen, d_model]
    }

    private float[][] scaledDotProductAttention(float[][] Q, float[][] K, float[][] V) {
        int seqLen = Q.length;
        int d_k = Q[0].length;

        float[][] scores = matMul(Q, transpose(K));
        float scale = (float) Math.sqrt(d_k);
        for (int i = 0; i < seqLen; i++)
            for (int j = 0; j < seqLen; j++)
                scores[i][j] /= scale;
        
        return matMul(rowSoftmax(scores), V);
    }
}
```

---

## 5.3 维度变化图

```
输入 X: [seqLen=10, d_model=512]
                │
    ┌───────────┼───────────┬───────────┬   ...   ┐  (8 个头并行)
    ▼           ▼           ▼           ▼         ▼
  Head 0      Head 1      Head 2      Head 3   Head 7
    │           │           │           │         │
    │ Q0×K0^T   │ Q1×K1^T   │ Q2×K2^T   │         │
    │ softmax   │ softmax   │ softmax   │         │
    │ ×V0       │ ×V1       │ ×V2       │         │
    │           │           │           │         │
    ▼           ▼           ▼           ▼         ▼
[seqLen,64] [seqLen,64] [seqLen,64] [seqLen,64] [seqLen,64]

    └───────────┴───────────┴───────────┴─── ... ──┘
                        │ 拼接
                [seqLen, 8×64 = 512]
                        │ × W_O
                [seqLen, d_model = 512]
```

**关键设计**：每个头看到的维度是 `d_model / h = 512/8 = 64`，使得总计算量**与单头全文 512 维注意力基本相同**。

---

## 5.4 用 `CompletableFuture` 理解并行性

```java
/**
 * Multi-Head Attention 是完全可并行的
 * 8 个头互相独立，没有数据依赖
 * 
 * 类比: Java 的 CompletableFuture.allOf()
 */
public float[][] forwardParallel(float[][] X) {
    List<CompletableFuture<float[][]>> futures = IntStream.range(0, numHeads)
        .mapToObj(h -> CompletableFuture.supplyAsync(() -> {
            // 每个头在自己的线程里计算
            float[][] Q = matMul(X, W_Q_heads[h]);
            float[][] K = matMul(X, W_K_heads[h]);
            float[][] V = matMul(X, W_V_heads[h]);
            return scaledDotProductAttention(Q, K, V);
        }))
        .toList();

    // 等待所有头完成
    float[][][] headOutputs = CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
        .thenApply(v -> futures.stream().map(CompletableFuture::join).toArray(float[][][]::new))
        .join();

    // 拼接 + 投影
    float[][] concat = concatHeads(headOutputs);
    return matMul(concat, W_O);
}
```

**实际上在 GPU 上是如此实现的**：把 8 个头的 Q、K、V 矩阵放入一个批次（batch），用一次大矩阵乘法同时完成 8 个头的计算。

---

## 5.5 每个头学到了什么？

虽然我们无法精确解读每个头的功能，但研究者通过可视化发现了一些规律：

| 头的类型（通过模式归纳） | 可能的功能 | 典型注意力模式 |
|---|---|---|
| **位置头** | 关注相邻/相近的 token | 对角线附近权重高 |
| **句法头** | 关注语法关系（主语→谓语，形容词→名词） | 特定语法依赖边权重高 |
| **指代头** | 代词解析（"it"→"cat"） | 跨长距离的单点高权重 |
| **分隔头** | 关注句子边界、标点符号 | `[SEP]`、`[CLS]` 权重高 |
| **自关注头** | token 主要关注自己 | 对角线权重大 |
| **语义头** | 关注语义相似的词 | 同义词/相关词权重高 |

**类比**：8 个对同一段代码有不同视角的 Code Reviewer：
- 一个人检查线程安全
- 一个人检查空指针
- 一个人检查性能
- 一个人检查命名规范
- ...

最后合并他们的意见，形成完整的 Code Review。

---

## 5.6 参数统计

论文原版 Transformer Base 的 Multi-Head Attention 参数：

```
h = 8 个头
d_model = 512
d_k = d_v = 64

每个头的 Q 矩阵: d_model × d_k = 512 × 64 = 32,768
每个头的 K 矩阵: 512 × 64 = 32,768
每个头的 V 矩阵: 512 × 64 = 32,768
8 个头的 Q/K/V:  8 × 3 × 32,768 = 786,432  ← 但实际实现通常合并

输出投影 W_O: h × d_v × d_model = 8 × 64 × 512 = 262,144

总计: ~1,048,576 ≈ 1M 参数 / 层
```

**为什么合并实现更好？**

```
实践中，所有头的 Q/K/V 矩阵被合并为一个大矩阵：
  W_Q_merged: [d_model, d_model] = [512, 512]  ← 而不是 8 个小矩阵

然后在一次矩阵乘法后按头切分：
  Q_merged = X × W_Q_merged         → [seqLen, 512]
  Q_heads   = reshape(Q_merged, [seqLen, 8, 64])  → 切分成 8 个头

这样做一次大矩阵乘法比 8 次小矩阵乘法快得多（GPU 更友好）。
```

---

## 5.7 论文原话

> **"Multi-head attention allows the model to jointly attend to information from different representation subspaces at different positions. With a single attention head, averaging inhibits this."**

翻译：多头注意力让模型能在不同位置**同时关注**来自不同表示子空间的信息。单头注意力的平均化会抑制这种能力。

---

## 5.8 常见问题：头越多越好吗？

| 头的数量 | 优点 | 缺点 |
|---|---|---|
| 1 | 计算快 | 表达能力差 |
| 4-8 | 论文建议范围，性价比高 | — |
| 16+ | 表达能力更强 | 某些头可能冗余，计算开销大 |
| 64+ | 理论上可以更细粒度 | d_k 太小（如 512/64=8），每个头的表示空间受限 |

**实验表明**：大部分 Transformer 模型用 8-16 个头即可，超过 32 个头的收益递减明显。2019 年有论文证明：**剪掉大部分头后模型性能下降很小**——说明很多头是冗余的。

---

> **下一章**：[Positional Encoding 详解](06-positional-encoding.md)
