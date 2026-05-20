# 附录 B · 术语表

---

## 论文核心术语

| 英文 | 中文 | 解释 |
|---|---|---|
| **Attention** | 注意力机制 | 让模型学会"关注"输入中重要部分的技术 |
| **Attention Score** | 注意力分数 | Q 和 K 的点积结果，表示一个 token 对另一个 token 的原始关注程度 |
| **Attention Weight** | 注意力权重 | 经过 Softmax 归一化后的注意力分数，每行之和为 1 |
| **Autoregressive** | 自回归 | 逐词生成，每个新词基于之前所有已生成的词 |
| **Beam Search** | 束搜索 | 保留 K 条最优候选路径的搜索策略 |
| **Causal Mask** | 因果遮罩 | 阻止位置的 token 看到未来位置的遮罩矩阵 |
| **Context Window** | 上下文窗口 | 模型一次能处理的最大 token 数（论文: 512, GPT-4: 128K+） |
| **Cross-Attention** | 交叉注意力 | Decoder 查询 Encoder 输出的注意力机制（Q 来自 Decoder, K/V 来自 Encoder） |
| **Cross-Entropy Loss** | 交叉熵损失 | 量化预测分布和真实分布差异的损失函数 |
| **d_k / d_v** | Q/K 维度 / V 维度 | 每个注意力头内部的维度（论文: 64） |
| **d_ff** | FFN 中间层维度 | Feed-Forward 中间层的维度，通常是 d_model 的 4 倍（2048） |
| **d_model** | 模型维度 | 所有子层输入输出的统一维度（论文: 512） |
| **Decoder** | 解码器 | Transformer 的"生成"部分，逐词产出目标序列 |
| **Decoder-Only** | 仅解码器结构 | 今天 LLM 的主流架构（GPT, LLaMA），只有 Masked Self-Attention |
| **Dropout** | 丢弃法 | 训练时随机丢弃部分神经元，防过拟合 |
| **Embedding** | 词嵌入 | 把离散的 token ID 映射为连续的稠密向量 |
| **Encoder** | 编码器 | Transformer 的"理解"部分，把源序列处理成上下文表示 |
| **Encoder-Decoder** | 编码-解码结构 | 原始 Transformer 的完整结构，用于 seq2seq 任务 |
| **Encoder-Only** | 仅编码器结构 | 理解任务的架构（BERT），双向注意力 |
| **Feed-Forward Network (FFN)** | 前馈网络 | 两层全连接 + ReLU，对每个位置独立做非线性变换 |
| **Flash Attention** | 闪电注意力 | 优化 Attention 内存访问的算法，分块计算减少 I/O |
| **GQA (Grouped-Query Attention)** | 分组查询注意力 | 多个 Q 头共享同一组 K/V，减少 KV Cache 内存 |
| **Gradient** | 梯度 | 损失函数对参数的偏导数，指示参数调整方向 |
| **Head** | 注意力头 | Multi-Head Attention 中的一个独立注意力计算单元 |
| **Inference** | 推理 | 使用已训练的模型对新数据进行预测 |
| **KV Cache** | KV 缓存 | 将已计算过的 Key/Value 缓存，避免自回归生成中的重复计算 |
| **Label Smoothing** | 标签平滑 | 防止模型过度自信的正则化技巧 |
| **Layer Normalization** | 层归一化 | 对同一样本的所有维度做归一化 |
| **Learning Rate** | 学习率 | 控制参数更新步长的超参数 |
| **Logits** | 对数几率 | Softmax 之前的原始分数（未经归一化的输出） |
| **Masked Self-Attention** | 掩码自注意力 | Decoder 中的 Self-Attention，带因果遮罩 |
| **MoE (Mixture of Experts)** | 专家混合 | 一个 FFN 分成多个"专家"，每次只激活一部分 |
| **MQA (Multi-Query Attention)** | 多查询注意力 | 所有头共享 K/V，最大程度减少 KV Cache |
| **Multi-Head Attention** | 多头注意力 | 并行运行多个注意力头，分别关注不同子空间 |
| **N (num_layers)** | 层数 | Encoder/Decoder 堆叠的 Block 数（论文: 6） |
| **Perplexity (PPL)** | 困惑度 | 评估语言模型的指标，PPL = e^loss，越低越好 |
| **Positional Encoding** | 位置编码 | 给自注意力机制注入 token 位置信息 |
| **Pre-LN / Post-LN** | 前置/后置层归一化 | LayerNorm 在子层之前还是之后 |
| **Q (Query)** | 查询向量 | 表示"我想找什么" |
| **K (Key)** | 键向量 | 表示"我有什么可以被检索的" |
| **V (Value)** | 值向量 | 表示"我的实际内容" |
| **ReLU** | 线性整流函数 | max(0, x)，引入非线性 |
| **Residual Connection** | 残差连接 | 将子层的输入直接加到输出上，防梯度消失 |
| **RNN / LSTM / GRU** | 循环神经网络及其变体 | Transformer 之前的序列模型主流方案 |
| **RoPE** | 旋转位置编码 | 现代 LLM 主流的位置编码方案 (LLaMA, Qwen) |
| **Self-Attention** | 自注意力 | Q、K、V 都来自同一个序列的注意力 |
| **Softmax** | 归一化指数函数 | 把任意实数值映射为 [0,1] 且和为 1 的概率分布 |
| **Teacher Forcing** | 教师强制 | 训练时用正确答案而非模型预测作为后续输入 |
| **Temperature** | 温度参数 | 控制采样随机性的参数，温度越高越随机 |
| **Token** | 标记 | 文本的最小处理单元（可能是词、子词或字符） |
| **Tokenization** | 分词 | 把原始文本拆分为 token 序列的过程 |
| **Top-K Sampling** | Top-K 采样 | 只从概率最高的 K 个候选中采样 |
| **Training** | 训练 | 在大量数据上调整模型参数的过程 |
| **Vocabulary** | 词汇表 | 模型认识的所有 token 的集合 |
| **Warmup** | 预热 | 训练初期线性增加学习率的策略 |
| **W_O / W_Q / W_K / W_V** | 注意力投影矩阵 | 可学习的线性变换矩阵 |

---

## 常见缩略语

| 缩略语 | 全称 | 解释 |
|---|---|---|
| **LLM** | Large Language Model | 大语言模型（GPT、LLaMA 等） |
| **NLP** | Natural Language Processing | 自然语言处理 |
| **MLM** | Masked Language Modeling | BERT 的训练目标：随机遮住一些词让模型猜 |
| **CLM** | Causal Language Modeling | GPT 的训练目标：根据上文预测下一个词 |
| **RAG** | Retrieval-Augmented Generation | 检索增强生成（检索 + 生成） |
| **BPE** | Byte Pair Encoding | 一种子词分词算法 |
| **RLHF** | Reinforcement Learning from Human Feedback | 人类反馈强化学习（ChatGPT 的对齐方法） |
| **SFT** | Supervised Fine-Tuning | 有监督微调 |
| **TPU / GPU** | Tensor/Graphics Processing Unit | 训练/推理 Transformer 的硬件 |
| **SA** | Self-Attention | 自注意力 |
| **MHA** | Multi-Head Attention | 多头注意力 |
| **FFN** | Feed-Forward Network | 前馈网络 |
| **LN** | Layer Normalization | 层归一化 |

---

> **返回**：[目录](README.md)
