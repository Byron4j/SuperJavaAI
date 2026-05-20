# 第 12 章 · Java 类比速查表

---

> 一页纸对照 Transformer 论文中的每个概念和 Java 世界的对应物。

---

## 12.1 架构总览

| Transformer 组件 | Java 世界类比 | 一句话解释 |
|---|---|---|
| **整体架构** | `Spring Boot Application` | Encoder = `@Service` 处理输入，Decoder = `@Controller` 输出 |
| **Token** | `int` (词汇表索引) | 文本的最基本单位，相当于分词后的 `String` |
| **Embedding** | `Map<Integer, float[]>` | 把 token ID 映射为稠密向量，类似 `indexOf` 之后查表 |
| **d_model** | `int EMBEDDING_DIM = 512` | 向量的维度，所有内部运算的统一维度 |
| **vocabSize** | `Map.size()` | 词汇表大小，确定输出分类数 |

## 12.2 位置编码

| Transformer 组件 | Java 世界类比 | 一句话解释 |
|---|---|---|
| **Positional Encoding** | `IntStream.range(0, n).boxed()` | 给每个 token 打上"第几个位置"的标签 |
| **sin/cos 编码** | `Math.sin(pos * freq)` | 多尺度正弦波编码，天然支持外推 |
| **RoPE** | `Collections.rotate(list, offset)` | 用旋转变换替代加法，相对位置编码 |

## 12.3 注意力机制

| Transformer 组件 | Java 世界类比 | 一句话解释 |
|---|---|---|
| **Query (Q)** | `SQL WHERE 子句` | "我想查什么" |
| **Key (K)** | `SQL 索引列` | "我可以被怎样找到" |
| **Value (V)** | `SELECT 返回的列` | 实际要提取的信息内容 |
| **QK^T** | `dotProduct(vecA, vecB)` | 计算两个向量相似度的点积 |
| **÷√d_k** | `scores / Math.sqrt(d_k)` | 方差归一化，防 Softmax 梯度消失 |
| **Softmax** | `sortDescending + limit(K) + normalize` | 大数变概率，小概率压到接近 0 |
| **Output = Softmax(QK^T)×V** | `weighted average of V by attention weights` | 按"相关性"对信息做加权混合 |
| **Self-Attention** | `sentence.stream().flatMap(token -> allTokens)` | 自己和自己做全连接关注 |

## 12.4 多头注意力

| Transformer 组件 | Java 世界类比 | 一句话解释 |
|---|---|---|
| **Multi-Head (h=8)** | `CompletableFuture.allOf(8个任务)` | 8 个独立线程各自计算，最后合并 |
| **每个头** | `不同视角的 Code Reviewer` | 各自关注不同维度（语法、指代、语义、位置） |
| **Head维度: d_k = d_model/h** | `拆分任务给多个 worker` | 512 → 8×64，总参数不变但允许多视角 |
| **W_O 投影** | `flatMap(结果) + 最终汇总` | 把 8 个 64 维结果拼回 1 个 512 维 |

## 12.5 全连接层与正则化

| Transformer 组件 | Java 世界类比 | 一句话解释 |
|---|---|---|
| **Feed-Forward** | `单一职责的小型 Processor` | 升维到 4× → ReLU → 降维回来 |
| **ReLU** | `Math.max(0, x)` | 非线性激活函数，引入表达能力 |
| **残差连接** | `try-finally { return input + enhanced; }` | 保证原始信息不丢失 |
| **LayerNorm** | `normalize(row) → mean=0, std=1` | 每行独立归一化，稳定训练 |
| **Dropout** | `if (random < 0.1) skip neuron` | 随机丢弃 10% 神经元，防止过拟合 |

## 12.6 推理优化

| Transformer 组件 | Java 世界类比 | 一句话解释 |
|---|---|---|
| **自回归生成** | `while (!end && i < max) append(predict())` | 循环生成直到结束标记 |
| **Causal Mask** | `Stream.limit(currentStep)` | 只能看已经生成的 token |
| **KV Cache** | `Redis / Caffeine Cache` | 已算过的 K、V 存起来，下次复用 |
| **Temperature** | `logits / T` → 控制随机性 | T→0: 确定性, T→∞: 完全随机 |
| **Top-K Sampling** | `Stream.sorted(TOP_K).random()` | 从概率最高的 K 个候选中随机选 |
| **Beam Search** | `Dijkstra 扩展 K 条路径` | 不是贪心一步，而是保留 K 条候选路径 |
| **Flash Attention** | `map-reduce 分批处理` | 不在显存写完整注意力矩阵，减少内存带宽瓶颈 |

## 12.7 训练

| Transformer 组件 | Java 世界类比 | 一句话解释 |
|---|---|---|
| **Teacher Forcing** | `手把手教学（用正确答案而非预测）` | 训练时直接给正确答案做下一步输入 |
| **Cross-Entropy Loss** | `-log(P(正确答案))` | 预测概率和真实标签的 KL 散度 |
| **Label Smoothing** | `不把鸡蛋放一个篮子里` | 正确标签给 0.9 而非 1.0，其余均分 ε |
| **Adam Optimizer** | `自适应学习率的 SGD` | 每个参数有自己的学习率，动量和二阶矩 |
| **Warmup** | `JVM 预热阶段` | 学习率先从小变大，再逐步衰减 |

## 12.8 模型变体速查

| 变体 | 结构 | Java 类比 | 代表 |
|---|---|---|---|
| **Encoder-Decoder** | 两个模块 | `Service + Controller` | 原始 Transformer, T5 |
| **Encoder-Only** | 只用 Encoder | 纯后端 `@Service` (分析数据) | BERT |
| **Decoder-Only** | 只用 Decoder | `@RestController` (直接面向用户) | GPT, LLaMA, Claude |
| **MoE** | FFN 变多专家 | `懒加载的多个 Service 实现` | Mixtral, GPT-4 |

## 12.9 一句话总结

```
Transformer = Embedding + PositionalEncoding
            + N × (MultiHeadAttention + FeedForward + 残差 + LayerNorm)
            + Linear + Softmax
```

**这 15 页论文的架构，支撑了今天所有的 ChatGPT、Claude、Gemini、DeepSeek。**

---

> **下一章**：[附录 A：数学基础](13-appendix-math.md)
