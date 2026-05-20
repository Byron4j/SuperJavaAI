# 第 2 章 · 核心思想：从 `for` 循环到 `parallelStream`

---

## 2.1 先理解 RNN 为什么"慢"

在 Transformer 之前，处理序列数据（文本、语音、时间序列）的标准方案是 **RNN**（循环神经网络）及其变体 LSTM、GRU。

### RNN 的 Java 版

```java
/**
 * RNN 处理序列 —— 必须按时间步顺序执行
 * 就像用 for 循环遍历 List，第 i 步依赖第 i-1 步的结果
 */
public class RNNCell {
    private final float[][] Wxh;  // 输入权重   [hiddenDim, inputDim]
    private final float[][] Whh;  // 隐藏层权重 [hiddenDim, hiddenDim]
    private final float[] bh;     // 偏置

    /**
     * @param xt    当前输入向量  [inputDim]
     * @param hPrev 上一时刻隐藏状态 [hiddenDim]
     * @return 当前时刻隐藏状态 [hiddenDim]
     */
    public float[] forward(float[] xt, float[] hPrev) {
        // h_t = tanh(W_xh · x_t + W_hh · h_{t-1} + b_h)
        float[] whh = matVecMul(Whh, hPrev);
        float[] wxh = matVecMul(Wxh, xt);
        float[] sum = add(add(wxh, whh), bh);
        return tanh(sum);
    }
}

// 处理一个句子：必须串行
float[] h = new float[hiddenDim]; // 初始隐藏状态为全零
for (int t = 0; t < tokens.length; t++) {
    h = rnnCell.forward(tokens[t], h);       // 第 t 步必须等第 t-1 步完成
    outputs[t] = outputLayer.forward(h);      // 基于当前隐藏状态预测
}
// 对于 1000 个 token 的句子，必须执行 1000 次 sequential 的最内层循环
```

### RNN 的三大致命缺陷

| 缺陷 | 解释 | Java 类比 |
|---|---|---|
| **无法并行** | 第 t 步依赖第 t-1 步的 `h` | 退化为普通 `for` 循环，不能用 `parallelStream` |
| **长距离遗忘** | 梯度在反向传播中指数衰减 | 第 0 个 token 对第 100 个 token 的梯度 ≈ 0 |
| **串行推理慢** | 生成时必须一个一个 token 地算 | 相当于每生成一个词就得从头跑一遍循环 |

---

## 2.2 Transformer 的解法：扔掉隐藏状态，直接算关系

Transformer 的核心思想可以用一句话概括：

> **不再维护一个"记忆向量" h 在时间步间传递，而是让每个 token 直接去看整个序列中的其他所有 token，计算它们之间的"注意力分数"。**

```java
// ==================== RNN 思路 ====================
// 维护一个不断更新的"记忆" h，像传接力棒
float[] h = initHiddenState();
for (Token token : tokens) {
    h = updateMemory(h, token);    // h 是过去所有信息的压缩
    output(computeOutput(h));       // 基于压缩的记忆做决策
}

// ==================== Transformer 思路 ====================
// 不维护记忆，直接让每个 token 和所有 token 做"全连接比较"
Token[] tokens = tokenize(input);
for (int i = 0; i < tokens.length; i++) {
    float[] weightedSum = new float[dModel];
    for (int j = 0; j < tokens.length; j++) {
        float relevance = howRelevant(tokens[i], tokens[j]);  // 计算 i 和 j 的相关性
        weightedSum = add(weightedSum, scale(tokens[j], relevance)); // 加权累积
    }
    output[i] = weightedSum;
}
// 关键：内层循环不依赖于顺序，每个 i 可以并行计算！
```

---

## 2.3 时间复杂度对比：O(n) 串行 → O(n²) 并行

这是最关键的理解：

```
                    RNN                    Transformer
每个 token 计算量     O(d²)                  O(n·d²)
时间步依赖           有（串行 O(n)）         无（完全并行 O(1)）
总时间复杂度         O(n·d²) 串行           O(n²·d) 并行
GPU 利用率          低（等待依赖链）         高（一次矩阵乘法做完所有工作）
```

其中：
- `n` = 序列长度（Sequence Length）
- `d` = 隐藏维度（d_model）

**直观解释**：RNN 像一条高速公路只能一次过一辆车；Transformer 像所有车同时过 n 条并行车道，但每条车道的过路费是 n（因为要和所有其他车协商优先级）。

---

## 2.4 一张图告别旧时代

```
                    序列建模的进化
                    
    RNN               LSTM              Transformer
    ┌─┐               ┌─┐               ┌─┬─┬─┬─┐
    │h│→              │c│ 记忆轨道      │ │ │ │ │  ← 每个 token
    └─┘               ├─┤              ├─┼─┼─┼─┤    同时看到
     ↓                 │h│ 输出轨道     │←─→│←─→│   所有其他
    ┌─┐               └─┘              ├─┼─┼─┼─┤    token
    │h│→               ↓               │ │ │ │ │
    └─┘               ┌─┐              └─┴─┴─┴─┘
     ↓                │c│                 ↑   ↑
    ...               ├─┤             完全并行  无记忆
                      │h│
                      └─┘
                       ↓
    ──────────► 时间  ──────────►       ──────────►
    串行传递隐藏状态   带门控的串行       全连接注意力
```

---

## 2.5 论文中的一句话概括

> **"We propose a new simple network architecture, the Transformer, based solely on attention mechanisms, dispensing with recurrence and convolutions entirely."**

翻译：**我们的 Transformer 架构只依赖注意力机制，完全抛弃了循环和卷积。**

---

## 2.6 动手验证直觉

用一个简单例子感受 Self-Attention 的威力：

### 输入："The cat sat on the mat because it was soft."

问题：**"it" 指代什么？**

| 模型 | 判断方式 | 结果 |
|---|---|---|
| RNN | 顺序读，读到 "it" 时 h 里混入了前面所有词的信息 | 容易混淆（可能认为是 mat） |
| LSTM | 有遗忘门，但 6 个单词距离仍然容易出错 | 可能正确，但不稳定 |
| Self-Attention | "it" 直接计算和所有前面词的相似度 | ✅ 精确——"it" 和 "cat" 的注意力分数最高 |

```java
// Self-Attention 处理指代消解
Token it = tokens.get(7); // "it"
for (Token candidate : tokens.subList(0, 7)) { // 只看前面的词
    float relevance = dotProduct(it.query(), candidate.key());
    log("relevance(\"it\", \"{}\") = {}", candidate.text, relevance);
}
// 输出:
// relevance("it", "The")   = 0.02
// relevance("it", "cat")   = 0.73  ← 最高！
// relevance("it", "sat")   = 0.15
// relevance("it", "on")    = 0.03
// relevance("it", "the")   = 0.01
// relevance("it", "mat")   = 0.05
// relevance("it", "because") = 0.01
```

这就是 Transformer 相比 RNN 的本质优势——**不管两个词距离多远，注意力直接连接，没有信息在"传输"中被稀释。**

---

> **下一章**：[架构全景：Encoder-Decoder](03-architecture-overview.md)
