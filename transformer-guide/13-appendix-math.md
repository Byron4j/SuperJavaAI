# 附录 A · 数学基础

---

> 本章为没有深度学习背景的 Java 开发者提供阅读本书所需的最小数学知识。

---

## A.1 向量

```java
// 向量 = 一维浮点数组
float[] v = {1.5f, -2.3f, 0.7f};

// 向量维度 = 数组长度
int dim = v.length;  // 3
```

在 Transformer 中，每个 token 被表示为一个 512 维向量（d_model=512）。

## A.2 矩阵

```java
// 矩阵 = 二维浮点数组 [行][列]
float[][] M = {
    {1.0f, 2.0f, 3.0f},   // 第 0 行
    {4.0f, 5.0f, 6.0f}    // 第 1 行
};

int rows = M.length;       // 2 行
int cols = M[0].length;    // 3 列

// 矩阵形状: [rows, cols] = [2, 3]
```

## A.3 矩阵乘法

```java
/**
 * 矩阵乘法: C = A × B
 * 
 * 前提: A 的列数 == B 的行数
 * 如果 A=[m, n], B=[n, p], 则 C=[m, p]
 * 
 * C[i][j] = Σ_k A[i][k] * B[k][j]
 */
public static float[][] matMul(float[][] A, float[][] B) {
    int m = A.length;         // A 的行数
    int n = A[0].length;      // A 的列数 = B 的行数
    int p = B[0].length;      // B 的列数

    float[][] C = new float[m][p];

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < p; j++) {
            float sum = 0.0f;
            for (int k = 0; k < n; k++) {
                sum += A[i][k] * B[k][j];
            }
            C[i][j] = sum;
        }
    }
    return C;
}
```

**在 Transformer 中的关键矩阵乘法**：

| 操作 | 形状 | 含义 |
|---|---|---|
| `X × W_Q` | `[n, d] × [d, d_k] = [n, d_k]` | 输入投影到 Query 空间 |
| `Q × K^T` | `[n, d_k] × [d_k, n] = [n, n]` | 计算注意力分数矩阵 |
| `Scores × V` | `[n, n] × [n, d_v] = [n, d_v]` | 加权聚合 |

## A.4 转置（Transpose）

```java
/**
 * 矩阵转置: 行列互换
 * A[i][j] → A^T[j][i]
 */
public static float[][] transpose(float[][] A) {
    int m = A.length;
    int n = A[0].length;
    float[][] AT = new float[n][m];

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            AT[j][i] = A[i][j];
        }
    }
    return AT;
}
```

## A.5 点积（Dot Product）

```java
/**
 * 两个向量的点积
 * 
 * a · b = Σ a[i] * b[i]
 * 
 * 几何意义: |a| × |b| × cos(θ)
 * - 同向: 大正数
 * - 正交: 0
 * - 反向: 大负数
 */
public static float dotProduct(float[] a, float[] b) {
    float sum = 0.0f;
    for (int i = 0; i < a.length; i++) {
        sum += a[i] * b[i];
    }
    return sum;
}
```

在 Attention 中，`Q_i · K_j`（点积）衡量两个 token 的相似度。

## A.6 Softmax

```java
/**
 * Softmax: 把任意实数向量转为概率分布
 * 
 * softmax(x_i) = e^{x_i} / Σ e^{x_j}
 * 
 * 特性:
 * 1. 每个输出在 (0, 1) 之间
 * 2. 所有输出之和 = 1.0
 * 3. 保持大小关系: 输入越大，输出越接近 1
 * 4. 可导（方便反向传播）
 */
public static float[] softmax(float[] x) {
    int n = x.length;

    // 减最大值防溢出
    float max = Float.NEGATIVE_INFINITY;
    for (float v : x) max = Math.max(max, v);

    float sum = 0.0f;
    float[] exp = new float[n];
    for (int i = 0; i < n; i++) {
        exp[i] = (float) Math.exp(x[i] - max);
        sum += exp[i];
    }

    for (int i = 0; i < n; i++) {
        exp[i] /= sum;
    }
    return exp;
}
```

## A.7 梯度与反向传播（最小理解）

不需要推导数学公式，只需要理解概念：

```java
/**
 * 梯度 = 偏导数的向量
 * 
 * 对一个函数 f(param1, param2, ...)，梯度 ∂f/∂param 表示：
 * "param 增加一点点，f 会增加多少"
 * 
 * 梯度下降:
 * param = param - learning_rate × gradient
 * 
 * 类比: 在山顶迷路，通过感受脚下的坡度（梯度），
 *       每次往最陡的下坡方向走一小步（learning_rate），
 *       最终到达山谷（损失最小点）
 */

// 简单一维示例
float param = 0.0f;           // 初始随机值
float learningRate = 0.01f;

for (int step = 0; step < 1000; step++) {
    float loss = (param - 3.0f) * (param - 3.0f);  // f(x) = (x-3)², 最小值为 x=3
    float gradient = 2.0f * (param - 3.0f);          // f'(x) = 2(x-3)
    param = param - learningRate * gradient;         // 梯度下降一步
}
// param → 3.0 (收敛到最优值)
```

## A.8 激活函数

```java
/**
 * ReLU: max(0, x)
 * 作用: 引入非线性
 * 特点: 计算快，梯度简单（>0 时为 1，≤0 时为 0）
 */
public static float relu(float x) {
    return Math.max(0, x);
}

/**
 * GELU: 现代 Transformer 替代 ReLU
 * 在 0 附近更平滑，性能更好
 */
public static float gelu(float x) {
    return 0.5f * x * (1 + (float) Math.tanh(
        Math.sqrt(2 / Math.PI) * (x + 0.044715 * Math.pow(x, 3))
    ));
}
```

## A.9 余弦相似度

```java
/**
 * Cosine Similarity: 衡量两个向量方向的一致性（忽略长度）
 * 
 * cos(θ) = (a · b) / (|a| × |b|)
 * 
 * 范围: [-1, 1]
 * 1 = 完全相同方向
 * 0 = 正交（无关）
 * -1 = 完全相反
 * 
 * 常用于 RAG 中检索最相关的文档 chunk
 */
public static float cosineSimilarity(float[] a, float[] b) {
    float dot = dotProduct(a, b);
    float normA = (float) Math.sqrt(dotProduct(a, a));
    float normB = (float) Math.sqrt(dotProduct(b, b));
    return dot / (normA * normB);
}
```

## A.10 多层堆叠的直觉

```java
// 一层: 简单映射
output_1 = layer1.forward(input);

// 两层: 特征组合
output_2 = layer2.forward(output_1);

// N 层: 逐层抽象
// 低层: 学表面特征（词性、句法）
// 中层: 学短语结构
// 高层: 学语义关系、逻辑推理

// 类比: 编译器的 pass
// pass1: 词法分析 → pass2: 语法分析 → pass3: 语义分析 → pass4: 代码生成
```

---

> 以上是阅读本书和这篇论文所需的最小数学知识。如果你已经理解以上内容，完全足够理解 Transformer 的数学原理。
