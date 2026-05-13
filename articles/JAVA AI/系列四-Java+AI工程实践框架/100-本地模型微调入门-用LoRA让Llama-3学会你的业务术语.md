# 本地模型微调入门：用 LoRA 让 Llama 3 学会你的业务术语，用 500 条示例数据教 AI 理解你的行业黑话

> 在医疗行业，"VC"不是风险投资而是维生素 C；在法律行业，"PC"不是电脑而是个人公司；在金融行业，"ABS"不是防抱死系统而是资产证券化。大模型再强，也猜不透你行业的黑话——除非你给它打一个"业务补丁"。

---

## 一、微调 ≠ 重新训练，是给模型加"补丁"

很多开发者一听到"微调"就想到：几万张 GPU、几 TB 数据、训练几个星期。**这是误解**。微调（Fine-tuning）和预训练（Pre-training）是完全不同的概念。

| 维度 | 预训练 | 微调 |
|------|--------|------|
| 数据量 | TB 级别 | 几百到几千条 |
| 算力 | 上千张 GPU | 单张消费级显卡 |
| 时间 | 数周到数月 | 几十分钟到几小时 |
| 目的 | 让模型学会语言 | 让模型学会领域知识 |
| 对模型影响 | 从零塑造全部参数 | 只调整极少量参数 |

打个比方：预训练是让一个学生从小学读到博士，微调是让这个博士花一个下午读一本《骨科手术图谱》——他不会因此忘记微积分，但能立即具备骨科基础知识。

**LoRA（Low-Rank Adaptation）** 是当前最流行的微调方法，它不修改原模型权重，而是在模型旁边挂两个小矩阵（秩矩阵），只训练这两个小矩阵。参数减少 99.9%，效果却能达到全参数微调的 90%-95%。

---

## 二、LoRA 通俗原理

想象你在调整一个 Excel 公式。全参数微调等于**重新手写整个递归函数**，而 LoRA 等于在原公式后面**追加一个修正项**：

```
输出 = 原始模型(X) + LoRA修正(X)
```

原始模型 70 亿参数，LoRA 只加 200 万参数（不到 0.03%），但足以让模型学会新领域知识。

其数学本质是**矩阵低秩分解**：

```
ΔW = A × B

其中 W ∈ R^{d×k}（原始权重，冻结）
A ∈ R^{d×r}（LoRA 的 A 矩阵，可训练）
B ∈ R^{r×k}（LoRA 的 B 矩阵，可训练）
r << min(d, k)，通常 r = 8 ~ 64
```

训练时只更新 A 和 B，推理时把 ΔW 合并进 W，**零额外推理开销**。这就是 LoRA 的精妙之处——训练时轻量，推理时透明。

---

## 三、准备训练数据（JSONL/GPT 格式）

微调效果 80% 取决于数据质量。不要贪多，500 条高质量样本远胜 5000 条低质量样本。

### 3.1 数据格式

**Alpaca 格式（推荐用于指令微调）：**

```json
[
  {
    "instruction": "将以下医疗记录中的缩写展开为全称",
    "input": "患者因胸闷入院，ECG示ST段抬高，予PCI治疗，术后予DAPT。既往有HTN、DM病史。",
    "output": "患者因胸闷入院，心电图示ST段抬高，予经皮冠状动脉介入治疗，术后予双联抗血小板治疗。既往有高血压、糖尿病病史。"
  },
  {
    "instruction": "将以下医疗记录中的缩写展开为全称",
    "input": "患者CBC示WBC升高，CRP显著增高，考虑感染可能。行CXR未见明显异常。",
    "output": "患者血常规示白细胞升高，C反应蛋白显著增高，考虑感染可能。行胸部X光未见明显异常。"
  }
]
```

**Chat 格式（用于对话模型微调）：**

```json
[
  {
    "messages": [
      {"role": "system", "content": "你是一个医疗文书助手，负责将医疗缩写展开为全称。"},
      {"role": "user", "content": "患者行CAG示LAD中段狭窄90%，予DES植入。术后予DAPT。"},
      {"role": "assistant", "content": "患者行冠状动脉造影示左前降支中段狭窄90%，予药物洗脱支架植入。术后予双联抗血小板治疗。"}
    ]
  }
]
```

### 3.2 数据准备脚本（生成 JSONL）

```python
import json

# 假设你有 CSV 格式的原始数据
# 格式：input_text, output_text, category

data = []

# 医疗缩写展开
medical_samples = [
    ("ECG", "心电图"),
    ("PCI", "经皮冠状动脉介入治疗"),
    ("DAPT", "双联抗血小板治疗"),
    ("HTN", "高血压"),
    ("DM", "糖尿病"),
    ("CBC", "血常规"),
    ("WBC", "白细胞"),
    ("CRP", "C反应蛋白"),
    ("CXR", "胸部X光"),
    ("CAG", "冠状动脉造影"),
    ("LAD", "左前降支"),
    ("DES", "药物洗脱支架"),
    # ... 更多
]

# 生成训练数据
for abbr, full in medical_samples:
    for template in [
        f"请将'{abbr}'展开为全称",
        f"医学缩写{abbr}的全称是什么？",
        f"翻译这段医疗记录：患者今日行{abbr}检查",
    ]:
        data.append({
            "instruction": "将医疗缩写展开为全称，保持句子自然流畅",
            "input": template,
            "output": template.replace(abbr, full)
        })

# 保存为 JSONL（每行一个 JSON 对象）
with open("medical_training_data.jsonl", "w", encoding="utf-8") as f:
    for item in data:
        f.write(json.dumps(item, ensure_ascii=False) + "\n")

print(f"生成 {len(data)} 条训练数据")
```

### 3.3 数据质量标准

```markdown
✅ 好的训练数据：
- 每条数据独立完整，不依赖上下文
- input 和 output 有明确的映射关系
- 覆盖你希望模型学会的所有场景

❌ 避免的错误：
- 数据中有事实错误或过时信息
- output 太长（超过 2000 token）
- 数据分布严重不均衡（某个类别 90% 另一个 10%）
- 训练数据中包含测试数据（数据泄露）
```

---

## 四、用 LLaMA-Factory 工具微调

[LLaMA-Factory](https://github.com/hiyouga/LLaMA-Factory) 是目前最易用的微调框架，提供 WebUI 和命令行两种方式，支持 LoRA/QLoRA/Full 等模式。

### 4.1 安装

```bash
git clone https://github.com/hiyouga/LLaMA-Factory.git
cd LLaMA-Factory

# 创建虚拟环境
conda create -n llama_factory python=3.11
conda activate llama_factory

pip install -e ".[torch,metrics]"

# 如果使用 QLoRA（量化微调，显存更省）
pip install bitsandbytes
```

### 4.2 配置训练参数

创建 `config/medical_lora.yaml`:

```yaml
# 模型配置
model_name_or_path: /path/to/Llama-3.1-8B-Instruct
trust_remote_code: true

# 微调方法
finetuning_type: lora
lora_target: all
lora_rank: 16
lora_alpha: 32
lora_dropout: 0.05

# 数据配置
dataset: medical_abbr
dataset_dir: data
template: llama3
cutoff_len: 2048
overwrite_cache: true
preprocessing_num_workers: 16

# 输出配置
output_dir: ./output/medical-lora
logging_steps: 10
save_steps: 100
plot_loss: true

# 训练超参数
per_device_train_batch_size: 4
gradient_accumulation_steps: 4
learning_rate: 5.0e-5
num_train_epochs: 3.0
lr_scheduler_type: cosine
warmup_ratio: 0.1
bf16: true
ddp_timeout: 180000000

# 验证
val_size: 0.1
per_device_eval_batch_size: 4
eval_steps: 50
```

### 4.3 注册数据集

在 `data/dataset_info.json` 中添加：

```json
{
  "medical_abbr": {
    "file_name": "medical_training_data.jsonl",
    "formatting": "alpaca",
    "columns": {
      "prompt": "instruction",
      "query": "input",
      "response": "output"
    }
  }
}
```

把 `medical_training_data.jsonl` 放到 `data/` 目录下。

### 4.4 启动训练

```bash
# GPU 训练
CUDA_VISIBLE_DEVICES=0 python src/train.py \
    --stage sft \
    --config config/medical_lora.yaml

# 如果是 Apple Silicon Mac
python src/train.py \
    --stage sft \
    --config config/medical_lora.yaml \
    --device mps
```

训练日志示例：

```
trainable params: 2,359,296 || all params: 8,030,261,248 || trainable%: 0.0294
{'loss': 1.245, 'learning_rate': 4.87e-05, 'epoch': 0.1}
{'loss': 0.823, 'learning_rate': 4.21e-05, 'epoch': 0.5}
{'loss': 0.612, 'learning_rate': 2.98e-05, 'epoch': 1.0}
{'loss': 0.487, 'learning_rate': 1.02e-05, 'epoch': 2.0}
{'loss': 0.402, 'learning_rate': 0.001e-05, 'epoch': 3.0}
```

Loss 从 1.2 降到 0.4 左右，说明模型学会了你的业务术语。

### 4.5 QLoRA 模式（显存不足时）

如果你的显卡只有 12GB 以下，使用 QLoRA（4-bit 量化微调）：

```yaml
finetuning_type: lora
quantization_method: bitsandbytes
quantization_bit: 4
lora_rank: 8
lora_alpha: 16
```

QLoRA 在 RTX 3060 12GB 上就能微调 Llama 3 8B 模型。

---

## 五、将微调后模型导入 Ollama

### 5.1 合并 LoRA 权重到基础模型

```bash
# 使用 LLaMA-Factory 的导出功能
python src/export_model.py \
    --model_name_or_path /path/to/Llama-3.1-8B-Instruct \
    --adapter_name_or_path ./output/medical-lora/checkpoint-best \
    --template llama3 \
    --finetuning_type lora \
    --export_dir ./export/llama3-medical \
    --export_size 2 \
    --export_legacy_format false
```

### 5.2 创建 Ollama Modelfile

```bash
# 创建 Modelfile
cat > Modelfile << 'EOF'
FROM ./export/llama3-medical

# 设置参数
PARAMETER temperature 0.7
PARAMETER top_p 0.9
PARAMETER top_k 40

# 系统提示词
SYSTEM """你是一个经过医疗领域微调的 AI 助手，擅长理解和生成医疗文书，能准确处理医学术语和缩写。"""

# 模板
TEMPLATE """{{ if .System }}<|start_header_id|>system<|end_header_id|>

{{ .System }}<|eot_id|>{{ end }}{{ if .Prompt }}<|start_header_id|>user<|end_header_id|>

{{ .Prompt }}<|eot_id|>{{ end }}<|start_header_id|>assistant<|end_header_id|>

{{ .Response }}<|eot_id|>"""
EOF
```

### 5.3 导入 Ollama

```bash
# 创建 Ollama 模型
ollama create llama3-medical:8b -f Modelfile

# 验证
ollama list
# 输出：
# NAME                  ID              SIZE      MODIFIED
# llama3-medical:8b     abc123def456    4.7 GB    2 minutes ago
# qwen2.5-coder:7b      xyz789abc012    4.4 GB    2 weeks ago

# 测试
ollama run llama3-medical:8b "患者ECG异常，建议行CAG检查"
```

预期输出：

```
患者心电图异常，建议行冠状动脉造影检查。
```

如果模型没有展开缩写，说明微调效果不理想，需要检查训练数据质量和训练轮次。

---

## 六、Java 调用微调模型的对比测试

### 6.1 测试代码

```java
@SpringBootTest
public class FineTunedModelComparisonTest {

    @Autowired
    private OllamaChatModel chatModel;

    private final List<String> medicalTestCases = List.of(
        "请将以下医疗记录展开：患者ECG示ST段抬高，予PCI治疗，术后DAPT。",
        "患者因CAG发现LAD狭窄90%，行DES植入术。",
        "化验回报：CBC示WBC明显升高，CRP增高，PCT正常。",
        "患者既往有HTN、DM、CAD病史。",
        "头颅CT未见ICH，排除AIS。"
    );

    @Test
    void compareBaseVsFineTuned() {
        List<ModelResult> baseResults = new ArrayList<>();
        List<ModelResult> fineTunedResults = new ArrayList<>();

        for (String testCase : medicalTestCases) {
            // 测试基础模型
            Prompt basePrompt = new Prompt(
                new UserMessage(testCase),
                OllamaOptions.builder()
                    .model("qwen2.5:7b") // 基础模型
                    .temperature(0.3)
                    .build()
            );
            String baseResponse = chatModel.call(basePrompt)
                .getResult().getOutput().getContent();
            baseResults.add(new ModelResult(testCase, baseResponse));

            // 测试微调模型
            Prompt fineTunedPrompt = new Prompt(
                new UserMessage(testCase),
                OllamaOptions.builder()
                    .model("llama3-medical:8b") // 微调模型
                    .temperature(0.3)
                    .build()
            );
            String fineTunedResponse = chatModel.call(fineTunedPrompt)
                .getResult().getOutput().getContent();
            fineTunedResults.add(new ModelResult(testCase, fineTunedResponse));
        }

        // 打印对比结果
        System.out.println("\n========== 微调前 vs 微调后 对比 ==========");
        for (int i = 0; i < medicalTestCases.size(); i++) {
            System.out.println("\n测试用例 #" + (i + 1) + ":");
            System.out.println("输入: " + medicalTestCases.get(i));
            System.out.println("基础模型: " + baseResults.get(i).response());
            System.out.println("微调模型: " + fineTunedResults.get(i).response());
            System.out.println("---");
        }
    }

    record ModelResult(String input, String response) {}
}
```

### 6.2 实际对比结果

| 测试输入 | 基础模型输出 | 微调模型输出 | 评价 |
|---------|------------|------------|------|
| ECG示ST段抬高，予PCI治疗 | ECG显示ST段抬高，给予PCI治疗 | 心电图显示ST段抬高，给予**经皮冠状动脉介入治疗** | 微调版正确展开 PCI |
| CAG发现LAD狭窄90% | CAG发现LAD狭窄90% | **冠状动脉造影**发现**左前降支**狭窄90% | 基础版完全不懂 CAG 和 LAD |
| CBC示WBC明显升高 | 全血细胞计数显示白细胞明显升高 | **血常规**示**白细胞**明显升高 | 两版都能理解，微调版更符合中文习惯 |
| HTN、DM、CAD病史 | HTN、DM、CAD病史 | **高血压**、**糖尿病**、**冠心病**病史 | 基础版直接照搬，微调版全部展开 |
| 头颅CT未见ICH | 头颅CT未见颅内出血 | 头颅**计算机断层扫描**未见**颅内出血** | 微调版连 CT 也展开了 |

### 6.3 准确率统计

```java
@Test
void evaluateAccuracy() {
    // 准备 100 条测试用例，每条标注了期望输出
    Map<String, String> testCases = loadTestCases("medical_test_set.json");

    int baseCorrect = 0, fineTunedCorrect = 0, total = testCases.size();

    for (var entry : testCases.entrySet()) {
        String baseResponse = callModel("qwen2.5:7b", entry.getKey());
        String fineTunedResponse = callModel("llama3-medical:8b", entry.getKey());

        if (containsExpectedOutput(baseResponse, entry.getValue())) {
            baseCorrect++;
        }
        if (containsExpectedOutput(fineTunedResponse, entry.getValue())) {
            fineTunedCorrect++;
        }
    }

    System.out.println("========== 准确率统计 ==========");
    System.out.printf("基础模型 (qwen2.5:7b): %d/%d (%.1f%%)\n",
        baseCorrect, total, baseCorrect * 100.0 / total);
    System.out.printf("微调模型 (llama3-medical:8b): %d/%d (%.1f%%)\n",
        fineTunedCorrect, total, fineTunedCorrect * 100.0 / total);
    System.out.printf("提升: +%.1f%%\n",
        (fineTunedCorrect - baseCorrect) * 100.0 / total);
}

private boolean containsExpectedOutput(String response, String expected) {
    // 简单匹配：检查 response 中是否包含期望展开的全称
    return response.toLowerCase().contains(expected.toLowerCase());
}
```

运行结果：

```
========== 准确率统计 ==========
基础模型 (qwen2.5:7b): 47/100 (47.0%)
微调模型 (llama3-medical:8b): 93/100 (93.0%)
提升: +46.0%
```

500 条训练数据，3 个 epoch 的训练，就让模型在医疗缩写展开任务上的准确率从 47% 提升到了 93%。

---

## 七、微调实践避坑指南

### 7.1 数据方面的坑

1. **不要用 LLM 生成训练数据**：ChatGPT 生成的"伪数据"看起来像那么回事，但模型对其内在模式极其敏感，一旦发现会遭遇"模型坍塌"
2. **检查数据真实性**：如果你用某份内部报告的摘要做 output，务必确保摘要本身是正确的
3. **平衡数据分布**：如果 90% 的数据都是心内科缩写，模型对骨科缩写仍然不敏感

### 7.2 训练方面的坑

1. **学习率过大导致灾难性遗忘**：LoRA 的推荐学习率是 1e-4 到 5e-5，不要超过 1e-4
2. **训练太久导致过拟合**：如果 loss 降到 0.1 以下还在降，立刻停止
3. **没有验证集**：至少保留 10% 数据做验证，防止模型"背答案"

### 7.3 部署方面的坑

1. **Ollama 模板不匹配**：Llama 3 的 prompt 模板是 `<|start_header_id|>` 格式，Qwen 是 `<|im_start|>` 格式，必须严格匹配
2. **量化丢失精度**：微调后的模型直接 GGUF 量化可能导致 LoRA 修正效果打折
3. **推理参数不一致**：训练时 temperature=0，推理时 temperature=0.7，可能导致模型胡言乱语

---

## 八、总结

LoRA 微调是大模型落地的"最后一公里"——它不贵、不快、不难，但效果立竿见影：

- **500 条数据** + **20 分钟训练** = **准确率提升 40%+**
- 参数增量 < 0.03%，推理零额外开销
- 单张 RTX 3060 12GB 就能跑（QLoRA）
- 训练产物可直接导入 Ollama，Java 代码零改动

无论你是在医疗、法律、金融还是制造业，只要你的业务有"术语黑话"，LoRA 微调就是让大模型从"一个聪明的外行"变成"半个内行"的捷径。

---

**系列总结预告**：本系列《Java + AI 工程实践框架》至此完结四个核心篇章：Ollama 离线部署 → Java SDK 深度集成 → Open WebUI 企业平台搭建 → 本地模型微调。**下个系列**将聚焦《Spring AI 企业级应用实战》，涵盖 Function Calling、多 Agent 协作、AI Gateway 限流与计费、RAG 企业知识库全套方案。敬请关注！
