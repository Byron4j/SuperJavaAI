# 第 6 章 · Positional Encoding 详解

---

## 6.1 问题：Transformer 天生"无秩序"

Self-Attention 是**置换等变**（Permutation Equivariant）的——如果打乱输入词序，输出也会以同样的方式被打乱，但每个 token 的表示不变。

```java
// 下面两句话对 Self-Attention（不加位置编码）来说完全相同！
String sentence1 = "我 爱 你";
String sentence2 = "你 爱 我";

// 因为 Self-Attention 只看 token 之间的内容相似度，
// 不考虑 token 在序列中的位置
// 
// sentence1: "我" → Q_我, K_我, V_我
// sentence2: "我" → Q_我, K_我, V_我  ← 完全一样的投影！
//
// 如果把 6 个 token 的 Q/K/V 放一起看，模型无法区分哪个 "我" 在第一个位置
```

这就是 **Positional Encoding** 要解决的问题——给每个位置打上一个"位置标签"。

---

## 6.2 论文方案：正弦余弦编码

论文的公式（非常优雅）：

\[
PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right)
\]

\[
PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)
\]

其中：
- `pos`：token 在序列中的位置（0, 1, 2, ...）
- `i`：维度索引（0, 1, ..., d_model/2 - 1）
- `d_model`：模型维度（512）

---

## 6.3 Java 实现

```java
public class SinusoidalPositionalEncoding {

    /**
     * 生成位置编码矩阵
     * @param maxSeqLen  最大序列长度
     * @param dModel     模型维度（论文: 512）
     * @return PE[position][dimension]
     */
    public static float[][] encode(int maxSeqLen, int dModel) {
        float[][] pe = new float[maxSeqLen][dModel];

        for (int pos = 0; pos < maxSeqLen; pos++) {
            for (int i = 0; i < dModel; i++) {
                // 角度: pos / 10000^(2i/d_model)
                // 等价于: pos / (10000^(2i/d_model))
                // 其中 floor(i/2) 保证每两个相邻维度共享一个频率
                double frequency = Math.pow(10000.0, -2.0 * (i / 2) / dModel);
                double angle = pos * frequency;

                if (i % 2 == 0) {
                    pe[pos][i] = (float) Math.sin(angle);  // 偶数维度: sin
                } else {
                    pe[pos][i] = (float) Math.cos(angle);  // 奇数维度: cos
                }
            }
        }
        return pe;
    }

    /**
     * 使用: 直接加到 Embedding 上
     */
    public static float[][] addPositionalEncoding(float[][] embeddings, float[][] pe) {
        int seqLen = embeddings.length;
        int dModel = embeddings[0].length;
        float[][] result = new float[seqLen][dModel];

        for (int i = 0; i < seqLen; i++) {
            for (int j = 0; j < dModel; j++) {
                result[i][j] = embeddings[i][j] + pe[i][j];
            }
        }
        return result;
    }
}
```

---

## 6.4 为什么用 sin/cos 而不是简单数字？

### 6.4.1 方案对比

```java
// 方案 A: 直接用位置数字（绝对位置编码）
// pos=0 → [0, 0, 0, ...]
// pos=1 → [1, 1, 1, ...]
// 问题: 归一化到 [0,1] 后，长序列中相邻位置的差异微乎其微

// 方案 B: 学习式位置编码（BERT 的方式）
// position_embedding = nn.Embedding(maxSeqLen, d_model)
// 问题: 外推能力差，训练时 max 512，推理时 1024 位就无法处理

// 方案 C: 正弦余弦（论文方式）
// 优势:
// 1. 不需要学习参数（减少过拟合风险）
// 2. 可以外推到任意长度（训练 512，推理可以直接到 1024+）
// 3. 每个维度的频率不同，形成"多尺度位置码"
```

### 6.4.2 sin/cos 的"二进制"特性

观察 PE 矩阵的前几个位置和前几个维度：

```java
// 打印 PE 矩阵的一部分
float[][] pe = SinusoidalPositionalEncoding.encode(8, 6);

// pos   dim0(sin)    dim1(cos)    dim2(sin)    dim3(cos)    dim4(sin)    dim5(cos)
//  0     0.0000       1.0000       0.0000       1.0000       0.0000       1.0000
//  1     0.8415       0.5403       0.0100       0.9999       0.0001       1.0000
//  2     0.9093      -0.4161       0.0200       0.9998       0.0002       1.0000
//  3     0.1411      -0.9900       0.0300       0.9996       0.0003       1.0000
//  4    -0.7568      -0.6536       0.0400       0.9992       0.0004       1.0000
//  5    -0.9589       0.2837       0.0500       0.9988       0.0005       1.0000
//  6    -0.2794       0.9602       0.0599       0.9982       0.0006       1.0000
//  7     0.6570       0.7539       0.0699       0.9976       0.0007       1.0000
```

**关键观察**：
- 低维度（dim 0-1）变化快（高频）—— 捕捉相邻位置差异
- 高维度（dim 4-5）变化慢（低频）—— 捕捉远距离位置关系
- 这就像一个**连续版的二进制计数器**！

---

## 6.5 相对位置的性质

正弦余弦编码有一个精妙的数学性质：**两个位置的编码可以通过线性变换互相表示**。

```java
/**
 * 对于固定的位置偏移 k，存在一个矩阵 M_k 使得:
 * PE(pos + k) = M_k × PE(pos)
 * 
 * 这意味着模型可以用 Self-Attention 学到相对位置关系，
 * 而不需要显式编码！
 * 
 * 证明（简化）:
 * sin(pos + k) = sin(pos)cos(k) + cos(pos)sin(k)
 * cos(pos + k) = cos(pos)cos(k) - sin(pos)sin(k)
 * 
 * 所以 M_k 是一个旋转矩阵:
 * M_k = [cos(k)  sin(k)]
 *       [-sin(k) cos(k)]
 * 
 * 这个性质后来启发了 RoPE (Rotary Position Embedding)
 */
```

---

## 6.6 现代变体

论文的 Sinusoidal Encoding 是最经典的，但后来的模型发展出了多种变体：

### 6.6.1 比较表

| 方法 | 代表模型 | 特点 | 外推能力 |
|---|---|---|---|
| **Sinusoidal** (论文) | 原始 Transformer | 固定函数，无参数 | ✅ 任意长度 |
| **Learned Absolute** | BERT, GPT-1/2 | 每个位置学一个向量 | ❌ 受训时 seqLen 限制 |
| **Relative Position** | Transformer-XL | Attention 分数加相对偏置 | ✅ 理论上任意 |
| **ALiBi** | BLOOM | Attention 分数加线性衰减 | ✅ 极佳外推 |
| **RoPE** | LLaMA, Qwen, DeepSeek | 对 Q、K 做旋转变换 | ✅ 优秀外推 |

### 6.6.2 RoPE 简介（当前主流）

```java
/**
 * RoPE (Rotary Position Embedding) —— 当前大多数开源模型使用的方法
 * 
 * 核心思想: 不是把位置编码加到输入上，而是对 Q 和 K 做旋转变换
 * 使得 Q_i^T · K_j 自动包含位置差 (i - j) 的信息
 * 
 * 操作:
 * (1) 把 Q 和 K 的每两个维度看作一个二维向量
 * (2) 按位置 i 对该二维向量逆时针旋转 angle_i 度
 * 
 * Q_rot_i[2d, 2d+1] = rotate(Q[2d, 2d+1], pos_i * θ_d)
 * K_rot_j[2d, 2d+1] = rotate(K[2d, 2d+1], pos_j * θ_d)
 * 
 * 此时 dot(Q_rot_i, K_rot_j) 只依赖于 (pos_i - pos_j)!
 */
public static float[] rotate2D(float x, float y, float angle) {
    float cos = (float) Math.cos(angle);
    float sin = (float) Math.sin(angle);
    return new float[]{
        x * cos - y * sin,  // 旋转后的 x
        x * sin + y * cos   // 旋转后的 y
    };
}
```

**RoPE 的优势**：相对位置信息直接编码在 Attention 分数里，不需要先加再算，泛化能力更强。

---

## 6.7 为什么位置编码如此重要？

```java
// 没有位置编码的 Self-Attention:
// "The cat sat on the mat" 和 "The mat sat on the cat"
// → 完全相同！（每个 token 的上下文表示一样）

// 有位置编码的 Self-Attention:
// "The cat sat on the mat" 和 "The mat sat on the cat"
// → 不同！（因为位置不同，PE 不同，导致 Q/K/V 投影都不同）
```

**位置编码是 Transformer 唯一知道"顺序"的地方。** 如果去掉 Positional Encoding（或者 PE 设计不好），模型将变成一个**无序的词袋模型**，性能严重下降。

---

## 6.8 验证：PE 对 Attention 的影响

```java
// 实验: 比较有无 PE 时的 Attention 模式

float[][] X_noPE = embedding.forward(tokens);           // 无位置信息
float[][] X_withPE = add(embedding.forward(tokens), pe); // 有位置信息

// 无 PE:
// Attention("The", "cat") ≈ Attention("cat", "The")
// → 对称，无法区分位置

// 有 PE:
// Attention("The_pos0", "cat_pos1") ≠ Attention("cat_pos1", "The_pos0")
// → 因为 Q_pos0 = X_pos0+PE_pos0 ≠ X_pos1+PE_pos1 = Q_pos1
// → Q 不对称，所以 Attention 也不对称，能保留方向性
```

**关键机制**：PE 让 Q 和 K 的投影对位置敏感，从而在 `QK^T` 计算中自然引入位置信息。

---

> **下一章**：[Feed-Forward & LayerNorm & 残差连接](07-feed-forward-layer-norm.md)
