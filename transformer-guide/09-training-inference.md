# 第 9 章 · 训练与推理

---

## 9.1 核心差异：训练 vs 推理

| | 训练 (Training) | 推理 (Inference) |
|---|---|---|
| **数据** | 完整源-目标句对（已知标签） | 只有源句子（逐词生成） |
| **前向传播** | 一次，给完整序列 | N 次，每次多一个 token |
| **反向传播** | 有（更新权重） | 无（权重冻结） |
| **Dropout** | 开启 | 关闭 |
| **Batch Size** | 大（几千到几百万 token） | 1（逐个请求） |
| **输出** | 损失值 | 生成的文本 |
| **位置** | GPU 集群，天到周级 | 单 GPU/CPU，秒级 |

---

## 9.2 训练过程

### 9.2.1 Teacher Forcing

训练时不用自回归循环——直接给整个目标序列，让模型并行预测所有位置的下一个 token：

```java
public class Trainer {

    /**
     * 一个训练步骤
     * 
     * @param srcTokens  源序列 token IDs，如 "I love you" → [37, 1024, 88]
     * @param tgtTokens  目标序列 token IDs，如 "<s> 我 爱 你 </s>" → [1, 56, 243, 72, 2]
     */
    public TrainingStepResult trainStep(int[] srcTokens, int[] tgtTokens) {
        int tgtLen = tgtTokens.length;

        // ===== Teacher Forcing =====
        // 输入目标序列的前 n-1 个 token，预测后 n-1 个 token
        // 
        // Input:    [<s>, 我,  爱,  你]        ← 不含最后一个 token
        // Target:   [我,   爱,  你,  </s>]     ← 不含第一个 token
        //              ↑     ↑    ↑     ↑
        // 并行预测这 4 个位置的下一个 token

        int[] decoderInput = Arrays.copyOf(tgtTokens, tgtLen - 1);  // [<s>, 我, 爱, 你]
        int[] labels = Arrays.copyOfRange(tgtTokens, 1, tgtLen);    // [我, 爱, 你, </s>]

        // 前向传播
        float[][] encoderOutput = encoder.forward(srcTokens, true);
        float[][] logits = decoder.forward(decoderInput, encoderOutput, true);
        // logits: [4, vocabSize] — 每个位置对所有词的打分

        // 计算损失
        float loss = crossEntropyLoss(logits, labels);

        // 反向传播 + 优化器更新（省略）
        return new TrainingStepResult(loss);
    }
}
```

**Teacher Forcing 的本质**：教模型时，直接用**正确答案**作为下一步的输入，而不是用模型自己的预测。就像教小孩写字时握着他的手，而不是让他自己乱写。

### 9.2.2 损失函数：Cross-Entropy Loss

```java
/**
 * Cross-Entropy Loss（交叉熵损失）
 * 
 * 对于每个位置 t:
 *   L_t = -log(P(正确token | 前t个token))
 * 
 * 总损失 = 所有位置的平均:
 *   L = (1/N) Σ_t L_t
 */
public static float crossEntropyLoss(float[][] logits, int[] labels) {
    int seqLen = logits.length;  // 预测的 token 数量
    float totalLoss = 0.0f;

    for (int t = 0; t < seqLen; t++) {
        // logits[t] 是所有词的"能量分数"
        // labels[t] 是正确答案的 index

        // Step 1: Softmax → 概率分布
        float[] probs = softmax(logits[t]);

        // Step 2: -log(P(正确词))
        int correctIndex = labels[t];
        float correctProb = probs[correctIndex];

        // 防止 log(0) → -∞
        float eps = 1e-9f;
        totalLoss += -Math.log(Math.max(correctProb, eps));
    }

    return totalLoss / seqLen;  // 平均损失
}

// 例子：
// 输入: "<s> 我 爱"
// logits[2] = [..., "你": 5.2, "他": 3.1, "她": 2.8, ...]  ← 第3个位置预测"你"分数最高
// softmax →   [..., "你": 0.85, "他": 0.10, "她": 0.05, ...]
// correct = "你" (index=72)
// loss = -log(0.85) ≈ 0.16  ← 预测越准，loss 越小
```

### 9.2.3 Label Smoothing（标签平滑）

论文用了 Label Smoothing 技巧，防止模型过度自信：

```java
/**
 * Label Smoothing
 * 不要"非黑即白"的标签，而是给正确 token 分配 1-ε 的概率，
 * 剩余 ε 均分给所有其他 token
 * 
 * 论文 ε = 0.1
 */
public static float labelSmoothedCrossEntropy(float[][] logits, int[] labels, float epsilon) {
    int vocabSize = logits[0].length;
    float totalLoss = 0.0f;

    for (int t = 0; t < logits.length; t++) {
        float[] probs = softmax(logits[t]);

        // 构造平滑后的目标分布
        // q(y) = (1-ε) + ε/V  if y == correct
        //        ε/V          otherwise
        float correctProb = probs[labels[t]];
        float smoothedLoss = -(1 - epsilon) * Math.log(correctProb + 1e-9f);

        // 所有其他词的平滑损失
        for (int y = 0; y < vocabSize; y++) {
            if (y != labels[t]) {
                smoothedLoss -= (epsilon / vocabSize) * Math.log(probs[y] + 1e-9f);
            }
        }

        totalLoss += smoothedLoss;
    }

    return totalLoss / logits.length;
}
```

### 9.2.4 优化器：Adam + 特殊学习率调度

```java
/**
 * 论文的学习率调度（warmup + decay）
 * 
 * lrate = d_model^(-0.5) × min(step^(-0.5), step × warmup^(-1.5))
 */
public class TransformerLearningRateScheduler {
    private final int dModel;
    private final int warmupSteps;

    public TransformerLearningRateScheduler(int dModel, int warmupSteps) {
        this.dModel = dModel;
        this.warmupSteps = warmupSteps;
    }

    public float getLearningRate(int step) {
        float scale = (float) Math.pow(dModel, -0.5);
        float stepFactor = Math.min(
            (float) Math.pow(step, -0.5),
            step * (float) Math.pow(warmupSteps, -1.5)
        );
        return scale * stepFactor;
    }
}

// 学习率曲线:
//   Warmup 阶段: lr 线性增长 (step < warmupSteps)
//   Decay 阶段:  lr = 1/√step   (step > warmupSteps)
//
//   lr ▲
//     |      ╱╲
//     |     ╱   ╲___
//     |    ╱        ╲___
//     |   ╱             ╲___
//     |  ╱                  ╲
//     └────────────────────────► steps
//        warmup        decay
```

---

## 9.3 推理过程（自回归生成）

### 9.3.1 贪心解码（Greedy Decoding）

```java
/**
 * 贪心解码：每步选概率最高的 token
 * 优点: 速度快，确定性
 * 缺点: 容易陷入重复循环，缺乏多样性
 */
public int[] greedyDecode(int[] srcTokens, int maxLen) {
    float[][] encoderOutput = encoder.forward(srcTokens, false);
    int[] generated = {START_TOKEN};

    while (true) {
        float[][] logits = decoder.forward(generated, encoderOutput, false);
        float[] lastLogits = logits[logits.length - 1];
        float[] probs = softmax(lastLogits);
        
        int nextToken = argmax(probs);
        generated = append(generated, nextToken);

        if (nextToken == END_TOKEN || generated.length >= maxLen) {
            break;
        }
    }
    return generated;
}

// 问题示例:
// 贪心: "I think this is good. I think this is good. I think this is good. ..."
// → 陷入重复循环！
```

### 9.3.2 Top-K 采样

```java
/**
 * Top-K 采样：只从概率最高的 K 个 token 中随机选
 * 论文用了 beam search（束搜索），但现代模型多用采样
 */
public int topKSample(float[] probs, int k) {
    // 1. 找到 K 个最大概率的索引
    int[] topKIndices = topKIndices(probs, k);

    // 2. 提取 K 个概率并重新归一化
    float[] filteredProbs = new float[k];
    float sum = 0;
    for (int i = 0; i < k; i++) {
        filteredProbs[i] = probs[topKIndices[i]];
        sum += filteredProbs[i];
    }
    for (int i = 0; i < k; i++) {
        filteredProbs[i] /= sum;
    }

    // 3. 按概率采样
    double r = Math.random();
    double cumulative = 0;
    for (int i = 0; i < k; i++) {
        cumulative += filteredProbs[i];
        if (r <= cumulative) {
            return topKIndices[i];
        }
    }
    return topKIndices[k - 1];  // fallback
}
```

### 9.3.3 Temperature 的作用

```java
/**
 * Temperature 控制采样的"随机性"
 * 
 * probs_temp[i] = softmax(logits[i] / T)
 * 
 * T → 0:   确定性最大（接近贪心）
 * T = 1:   标准 Softmax
 * T → ∞:   完全随机（均匀分布）
 * 
 * 这是在 API 调用中你设置的 "temperature" 参数
 */
public float[] temperatureScale(float[] logits, float temperature) {
    float[] scaled = new float[logits.length];
    for (int i = 0; i < logits.length; i++) {
        scaled[i] = logits[i] / temperature;
    }
    return softmax(scaled);
}

// T=0.1: 概率集中 → "今天天气很好"
// T=0.7: 正常随机 → "今天天气不错"
// T=1.5: 高度随机 → "今天天色很棒极"
// T=2.0: 近乎乱来 → "今天空气海洋美丽"
```

### 9.3.4 Beam Search（论文使用的方案）

```java
/**
 * Beam Search: 不是只保留一个最优序列，而是保留 beam_size 个候选
 * 论文 beam_size = 4, α = 0.6 (长度惩罚)
 * 
 * 类比: 不是只看当前最优的下一步，而是看"K 条路径"中哪条最终得分最高
 * 类似于 Dijkstra 最短路径算法扩展 K 条路径
 */
public List<Beam> beamSearch(int[] srcTokens, int beamSize, int maxLen, float alpha) {
    float[][] encoderOutput = encoder.forward(srcTokens, false);
    
    // 初始 beam: 只有一个 <s>
    PriorityQueue<Beam> beams = new PriorityQueue<>(
        (a, b) -> Float.compare(b.score, a.score)  // 最大堆
    );
    beams.add(new Beam(new int[]{START_TOKEN}, 0.0f));

    List<Beam> finished = new ArrayList<>();

    for (int step = 0; step < maxLen; step++) {
        List<Beam> candidates = new ArrayList<>();

        for (Beam beam : beams) {
            if (beam.isFinished()) {
                finished.add(beam);
                continue;
            }

            // 前向传播
            float[][] logits = decoder.forward(beam.tokens, encoderOutput, false);
            float[] probs = softmax(logits[logits.length - 1]);

            // 取 top-k 候选
            int[] topIndices = topKIndices(probs, beamSize);
            for (int idx : topIndices) {
                int[] newTokens = append(beam.tokens, idx);
                float logProb = beam.score + (float) Math.log(probs[idx] + 1e-9f);
                candidates.add(new Beam(newTokens, logProb));
            }
        }

        // 保留 beamSize 个最好的候选
        candidates.sort((a, b) -> Float.compare(b.score, a.score));
        beams.clear();
        beams.addAll(candidates.subList(0, Math.min(beamSize, candidates.size())));
    }

    // 选择最优（考虑长度惩罚）
    finished.addAll(beams);
    for (Beam b : finished) {
        b.score /= (float) Math.pow(b.tokens.length, alpha);  // 长度惩罚
    }
    finished.sort((a, b) -> Float.compare(b.score, a.score));
    return finished;
}

class Beam {
    int[] tokens;
    float score;

    boolean isFinished() {
        return tokens[tokens.length - 1] == END_TOKEN;
    }
}
```

---

## 9.4 完整的训练循环

```java
public class TransformerTrainingLoop {
    private final Transformer model;
    private final AdamOptimizer optimizer;
    private final TransformerLearningRateScheduler scheduler;

    public void train(List<TrainingExample> dataset, int epochs, int batchSize) {
        for (int epoch = 0; epoch < epochs; epoch++) {
            float epochLoss = 0.0f;
            int totalTokens = 0;

            // 按 batch size 分批
            List<List<TrainingExample>> batches = partition(dataset, batchSize);

            for (List<TrainingExample> batch : batches) {
                float batchLoss = 0.0f;

                for (TrainingExample example : batch) {
                    // 1. 计算学习率
                    float lr = scheduler.getLearningRate(optimizer.getStep());

                    // 2. 前向传播
                    float[][] encoderOutput = model.encoder.forward(example.srcTokens, true);
                    float[][] logits = model.decoder.forward(
                        example.tgtInputTokens, encoderOutput, true
                    );

                    // 3. 计算损失
                    float loss = crossEntropyLoss(logits, example.tgtLabels);

                    // 4. 反向传播 + 优化器更新（框架自动完成，这里略）
                    // grads = backward(loss)
                    // optimizer.step(grads, lr)

                    batchLoss += loss;
                }

                epochLoss += batchLoss;
                totalTokens += batch.stream()
                    .mapToInt(e -> e.srcTokens.length + e.tgtInputTokens.length)
                    .sum();
            }

            // 困惑度（Perplexity）= e^(平均交叉熵)
            float avgLoss = epochLoss / totalTokens;
            float perplexity = (float) Math.exp(avgLoss);
            System.out.printf("Epoch %d | Loss: %.4f | PPL: %.2f%n", epoch, avgLoss, perplexity);
        }
    }
}
```

---

## 9.5 训练配置速查

| 参数 | Base 值 | 说明 |
|---|---|---|
| 训练步数 | 100,000 | 论文用的步数 |
| Batch size | ~25,000 tokens | 按 token 数拼 batch，不按句子数 |
| Warmup steps | 4,000 | 学习率线性增长阶段 |
| Optimizer | Adam | β1=0.9, β2=0.98, ε=10⁻⁹ |
| Dropout | 0.1 | 每层随机丢弃 10% |
| Label smoothing | ε=0.1 | 防止过拟合 |
| 梯度裁剪 | 1.0 | 防止梯度爆炸 |
| 硬件 | 8 × P100 GPU | 论文的实验环境 |
| 训练时间 | 12 小时 (Base) / 3.5 天 (Big) | — |

---

> **下一章**：[KV Cache 优化详解](10-kv-cache-optimization.md)
