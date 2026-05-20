# 附录 C · 常见问题 FAQ

---

## Q1: Transformer 为什么叫 "Transformer"？

论文作者从未明确解释，但业界共识是：它把输入序列"变换"（transform）为输出序列，而且核心操作是"注意力变换"——通过 Q/K/V 的线性变换和加权聚合来重新表示每个 token。

---

## Q2: 为什么 Self-Attention 需要 Q, K, V 三个矩阵？一个不行吗？

如果只有一个矩阵（Q=K=V=X），那么：

```java
Attention(X, X, X) = softmax(X·X^T) · X
```

这会强制模型在同一空间里既做"查询"又做"内容"，限制了表达能力。三个独立矩阵让模型能在不同空间中做查询、匹配和聚合，类似于数据库的"查询字段≠索引字段≠返回字段"。

---

## Q3: 多头注意力的 8 个头能减少吗？

可以。实验证明很多头是冗余的——AlphaFold 只用 4 个头，某些蒸馏模型只用 1-2 个头。但头数减少会降低模型的表达能力上限。**性价比最高的是 8-16 个头。**

---

## Q4: Positional Encoding 为什么用 sin/cos 而不用学习式？

论文选择了固定 sin/cos 编码，原因：
1. 不需学习参数（更简洁）
2. 外推性好（训练 512，推理可直接 1024+）
3. 数学性质优美（相对位置可通过线性变换表示）

但 BERT 和 GPT-1/2 用的是学习式位置编码，RoPE 是现代主流方案。

---

## Q5: 为什么 FFN 要升维到 4 倍？（d_ff = 4 × d_model）

这是一个经验值。升维给模型更多"思考空间"，降维回来保持了维度一致性。实验表明 4 倍在速度/效果上达到了最佳平衡。太小（2 倍）表达能力不够，太大（8 倍）参数量过大。

---

## Q6: LayerNorm 和 BatchNorm 到底有什么区别？

```
BatchNorm: 同一维度，跨所有样本 → 需要大 batch size
LayerNorm: 同一样本，跨所有维度 → 与 batch size 无关

文本的 batch 中句子长度不一（需要 padding），
BatchNorm 对变长序列极不友好，所以 NLP 选用 LayerNorm。
```

---

## Q7: Transformer 能处理的最长序列是多少？

**理论上**：无限。sin/cos 位置编码可以计算任意位置。

**实际上**：
- 论文原版：512（受限于训练数据长度和显存）
- GPT-3：2048
- LLaMA 2：4096
- GPT-4 Turbo：128K
- Claude 3：200K
- Gemini 1.5 Pro：1M+

限制来自显存（Attention 的 O(n²) 计算）而非架构本身。

---

## Q8: 为什么 Decoder-Only 成了主流？

原始论文是 Encoder-Decoder，但今天的 LLM 几乎都是 Decoder-Only。原因：
1. **统一性**：所有任务都可以统一为"续写文本"
2. **效率**：Encoder 不是必需的（没有跨语言翻译的场景）
3. **扩展性**：Decoder-Only 结构更简单，更容易堆大
4. **实证**：Scaling Law 证明 Decoder-Only 效果与 Encoder-Decoder 相当但更简单

---

## Q9: KV Cache 为什么只缓存 K 和 V，不缓存 Q？

因为 Q 是"查询向量"——每一步只生成 1 个新 token，所以只需要 1 个新 Q；但新 token 需要和**所有**历史 token 做注意力计算，所以需要所有历史的 K 和 V。

```java
// Step t 的 Attention:
// Q_new: [1, d_k]         ← 只有当前 token，不需要缓存
// K_all: [t+1, d_k]       ← 需要所有历史 K，要缓存
// V_all: [t+1, d_v]       ← 需要所有历史 V，要缓存
// output = softmax(Q_new · K_all^T / √d_k) · V_all
```

---

## Q10: GPT-4 有多少参数？用了几层？

OpenAI 尚未公开 GPT-4 的具体架构。业界推测：
- 参数量：~1.8 万亿（MoE，8 个专家，每次激活约 280B）
- 层数：约 120 层
- 训练数据：~13 万亿 token

但这些是推测，未被官方确认。

---

## Q11: Transformer 和 CNN 有什么区别？

| 维度 | CNN | Transformer |
|---|---|---|
| 感受野 | 局部（卷积核尺寸限制） | 全局（一次注意所有位置） |
| 参数共享 | 是（卷积核滑动） | 否（每个位置独立投影） |
| 并行度 | 高 | 高（但 Attention 是 O(n²)） |
| 数据偏好 | 图像（空间局部性） | 文本（长距离依赖） |
| 位置信息 | 隐式（卷积核位置） | 显式（Positional Encoding） |
| 代表作 | ResNet, VGG | GPT, BERT, ViT |

有趣的是，ViT (Vision Transformer) 证明了 Transformer 做图像也能打平甚至超越 CNN。

---

## Q12: 为什么训练要用大量 GPU，推理只需要一个？

- **训练**：需要存储梯度、优化器状态、中间激活值。一个 7B 模型训练需要 ~56GB 显存（仅模型+优化器状态）。
- **推理**：只需要存储模型权重和 KV Cache，一个 7B 模型推理在 fp16 下只需 ~14GB 显存。

此外，训练需要全 batch 同时计算，推理只需 1 个样本。

---

## Q13: Temperature=0 会发生什么？

Temperature=0 时相当于**贪心解码**：每次都选概率最高的 token，输出完全确定。优点是不随机，缺点是容易陷入重复循环、缺乏多样性。实际中通常用 0.1-0.3 作为"近似确定"设置。

---

## Q14: 论文的 "Big" 版本和 "Base" 版本有什么区别？

| 参数 | Base | Big |
|---|---|---|
| N（层数） | 6 | 6 |
| d_model | 512 | 1024 |
| d_ff | 2048 | 4096 |
| h（头数） | 8 | 16 |
| 参数量 | 65M | 213M |
| Dropout | 0.1 | 0.3 |

---

## Q15: 能用纯 Java 实现一个 Transformer 吗？

可以，但不实用。纯 Java 实现的推理速度可能只有 Python/C++ 的 1-10%。推荐方案：
1. **调 API**：OpenAI / 云服务商 API（第 11 章方案一）
2. **ONNX Runtime**：Java 调用 C++ ONNX Runtime（第 11 章方案二）
3. **TensorFlow Java / DJL**：Java 深度学习框架
4. **Llama.cpp Java 绑定**：用 JNI 调 C++ 推理引擎

纯 Java 实现更适合学习和理解原理，不适合生产部署。

---

## Q16: Attention 的 O(n²) 有办法优化吗？

有！以下是主流优化方案：

| 方案 | 复杂度 | 思想 |
|---|---|---|
| Sparse Attention | O(n√n) | 每个 token 只关注 √n 个其他 token |
| Local Attention | O(n×w) | 只关注窗口内的邻居 |
| Linformer | O(n) | 用低秩矩阵近似 Self-Attention |
| Flash Attention | O(n²) 但 I/O 减少 | 分块计算，利用 GPU 缓存层级 |
| Ring Attention | O(n²/k) | 多 GPU 环形分布序列 |

---

## Q17: 为什么需要 "Causal Mask"（因果遮罩）？

因为 Decoder 在生成时不能"偷看未来"——这和现实世界一致，你只能说你知道的。如果训练时不加 Mask，模型会直接作弊（通过看正确答案来预测）。

```java
// 训练时如果不用 Causal Mask:
// 输入: "<s> 我 爱"  ← 看到 "你" 在后面
// → 直接记住了答案，失去了学习推理的能力

// 加了 Causal Mask 后不能看到后面
// → 模型学会了真正的条件生成
```

---

## Q18: 论文中提到的 BLEU 分数是什么？

BLEU (Bilingual Evaluation Understudy) 是机器翻译的评价指标，衡量生成译文和参考译文的 n-gram 重叠程度。

- 范围：0-100（通常用 0-1 表示）
- 论文 WMT 2014 英德翻译：28.4 (Base) / 29.3 (Big)
- 论文 WMT 2014 英法翻译：41.0 (Base) / 43.3 (Big)

高于 30 通常被认为是"好用"的翻译质量。现代 LLM 翻译在各种基准上已远超这个数字。

---

> **返回**：[目录](README.md)
