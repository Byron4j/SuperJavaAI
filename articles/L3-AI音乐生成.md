# AI音乐生成深度解析：从音频Token到多模态音乐创作

**文章标签：** #ai #ai音乐 #suno #udio #音乐生成 #音频模型 #transformer #diffusion-audio

## 目录

- [引言：AI音乐生成的本质](#引言ai音乐生成的本质)
- [理论基础：为什么AI能生成音乐](#理论基础为什么ai能生成音乐)
- [来龙去脉：AI音乐生成的发展史](#来龙去脉ai音乐生成的发展史)
- [核心工具深度解析](#核心工具深度解析)
- [模型差异：不同架构的音乐生成策略](#模型差异不同架构的音乐生成策略)
- [工业级实践案例](#工业级实践案例)
- [高级技术：风格控制与多轨生成](#高级技术风格控制与多轨生成)
- [评估与优化体系](#评估与优化体系)
- [商业应用案例](#商业应用案例)
- [编程专项：自动化音乐生成](#编程专项自动化音乐生成)
- [常见陷阱与最佳实践](#常见陷阱与最佳实践)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：AI音乐生成的本质

AI音乐生成不是"让AI随机播放音符"的简单应用，而是一门**在音频Token的条件概率分布中进行序列采样**的工程技术。

核心认知：

```
音频生成的本质：P(音频Token_t | 音频Token_{t-1}, ..., 文本条件)

关键挑战：
1. 多时间尺度：音符(10ms) → 节拍(500ms) → 乐句(5s) → 段落(30s)
2. 音乐理论约束：和声、旋律、节奏的规则性
3. 风格一致性：整首歌曲保持统一风格
4. 长程依赖：副歌与主歌的呼应关系
5. 多模态对齐：歌词与旋律的时间对齐

质量差异的根源：
- 差的模型：音符间缺乏结构，和声混乱
- 好的模型：多时间尺度建模，音乐理论约束
```

**关键洞察**：AI音乐生成的核心难点不在"生成单个音符"，而在**维持跨时间的音乐结构一致性**。

---

## 理论基础：为什么AI能生成音乐

### 1. 音频表示的数学本质

#### 波形 vs 频谱 vs Token

```python
# 音频的三种表示方式对比

"""
1. 原始波形（Waveform）
   - 表示：振幅随时间变化
   - 数据量：44.1kHz采样 × 16bit = 705.6kbps
   - 3分钟歌曲：~127MB
   - 问题：数据量巨大，长程依赖难建模

2. 时频表示（Spectrogram）
   - STFT：时域→频域
   - 数据量：~10×压缩
   - 问题：相位恢复困难，生成音质受限

3. 音频Token（Audio Tokens）
   - 使用VAE/量化器将音频压缩为离散Token
   - 类似文本的Token序列
   - 数据量：~100×压缩
   - 优势：可用Transformer/LM建模
"""

import torch
import torchaudio
import numpy as np
import librosa

# 1. 波形表示
waveform, sample_rate = torchaudio.load("song.wav")
# waveform: [1, 7938000] (3分钟 @ 44.1kHz)

# 2. 频谱表示（Mel Spectrogram）
mel_spec = librosa.feature.melspectrogram(
    y=waveform.numpy()[0],
    sr=sample_rate,
    n_fft=2048,
    hop_length=512,
    n_mels=128
)
# mel_spec: [128, 15504] (时间轴压缩了512倍)

# 3. 音频Token表示（概念示例）
class AudioTokenizer:
    """音频Token化器（类似VAE的编码器）"""
    
    def __init__(self, codebook_size=1024, num_quantizers=4):
        self.encoder = Encoder()  # 将音频编码为连续特征
        self.quantizer = ResidualVectorQuantizer(
            codebook_size=codebook_size,
            num_quantizers=num_quantizers
        )
    
    def encode(self, waveform):
        """
        波形 → Token序列
        
        输入：[1, T] (T为采样点数)
        输出：[N, T'] (N个量化层，T'为时间步)
        """
        # 编码为连续特征
        z = self.encoder(waveform)  # [C, T']
        
        # 量化为离散Token
        tokens, _ = self.quantizer(z)  # [N, T']
        
        return tokens
    
    def decode(self, tokens):
        """
        Token序列 → 波形
        """
        # 反量化
        z = self.quantizer.decode(tokens)  # [C, T']
        
        # 解码为波形
        waveform = self.decoder(z)  # [1, T]
        
        return waveform

# Token化优势：
# - 3分钟歌曲：~50K tokens（vs 7.9M采样点）
# - 压缩比：~150:1
# - 可用语言模型（GPT）处理
```

#### 音频Token化的核心：EnCodec / SoundStream

```python
# ============================================
# EnCodec音频编解码器（Meta）
# ============================================

"""
EnCodec架构：

编码器（Encoder）：
音频波形 → 卷积降采样 → 连续潜在向量 Z

量化器（Residual Vector Quantizer）：
Z → 第一层VQ → 残差 → 第二层VQ → 残差 → ...

解码器（Decoder）：
量化后的Z → 卷积上采样 → 重建波形

关键参数：
- 采样率：24kHz 或 48kHz
- 帧率：75Hz（每秒钟75个token）
- 量化层数：4层（4个codebook）
- Codebook大小：1024
- 压缩率：~6kbps（接近MP3但质量更高）
"""

from encodec import EncodecModel
from encodec.utils import convert_audio

# 加载预训练模型
model = EncodecModel.encodec_model_24khz()
model.set_target_bandwidth(6.0)  # 6kbps

# 加载音频
wav, sr = torchaudio.load("input.wav")
wav = convert_audio(wav, sr, model.sample_rate, model.channels)

# 编码
with torch.no_grad():
    encoded_frames = model.encode(wav)
    # encoded_frames: list of (codes, scale)
    # codes: [B, K, T]  K=量化层数, T=时间步

# 解码
with torch.no_grad():
    decoded = model.decode(encoded_frames)
    # decoded: [B, C, T]  重建的波形

# 提取纯Token
codes = torch.cat([code for code, _ in encoded_frames], dim=-1)
print(f"Token shape: {codes.shape}")  # [1, 4, 11250] (2.5分钟 @ 75Hz)

# 关键理解：
# codes[0, 0, :] - 第一层量化（粗粒度）
# codes[0, 1, :] - 第二层量化（中粒度）
# codes[0, 2, :] - 第三层量化（细粒度）
# codes[0, 3, :] - 第四层量化（细节）
```

### 2. 音频语言模型

```python
# ============================================
# 音频语言模型：类似GPT但处理音频Token
# ============================================

class AudioLanguageModel(nn.Module):
    """音频语言模型"""
    
    def __init__(
        self,
        vocab_size=1024,
        num_quantizers=4,
        d_model=1024,
        n_layers=24,
        n_heads=16
    ):
        super().__init__()
        self.num_quantizers = num_quantizers
        
        # 每个量化层有自己的embedding
        self.embeddings = nn.ModuleList([
            nn.Embedding(vocab_size, d_model)
            for _ in range(num_quantizers)
        ])
        
        # 位置编码
        self.pos_encoding = PositionalEncoding(d_model)
        
        # Transformer层
        self.transformer = nn.TransformerEncoder(
            nn.TransformerEncoderLayer(
                d_model=d_model,
                nhead=n_heads,
                dim_feedforward=4*d_model,
                batch_first=True
            ),
            num_layers=n_layers
        )
        
        # 输出头：预测每个量化层的下一个token
        self.heads = nn.ModuleList([
            nn.Linear(d_model, vocab_size)
            for _ in range(num_quantizers)
        ])
    
    def forward(self, tokens, text_condition=None):
        """
        tokens: [B, K, T]  K=量化层数
        text_condition: [B, L, D]  文本条件（可选）
        """
        B, K, T = tokens.shape
        
        # 1. Embedding
        # 对每层分别embedding然后求和
        x = sum([
            emb(tokens[:, i, :])  # [B, T, D]
            for i, emb in enumerate(self.embeddings)
        ])
        
        # 2. 添加位置编码
        x = self.pos_encoding(x)
        
        # 3. 文本条件注入（Cross-Attention或Concat）
        if text_condition is not None:
            # 简化：使用Cross-Attention
            x = self.cross_attention(x, text_condition)
        
        # 4. Transformer处理
        x = self.transformer(x)  # [B, T, D]
        
        # 5. 预测下一帧的所有量化层
        predictions = []
        for head in self.heads:
            pred = head(x)  # [B, T, V]
            predictions.append(pred)
        
        return torch.stack(predictions, dim=1)  # [B, K, T, V]
    
    def generate(self, prompt_tokens, text_condition, max_length=1000):
        """
        自回归生成
        """
        generated = prompt_tokens.clone()
        
        for _ in range(max_length):
            # 预测下一个token
            logits = self.forward(generated, text_condition)
            next_token_logits = logits[:, :, -1, :]  # [B, K, V]
            
            # 采样
            next_tokens = torch.stack([
                torch.multinomial(torch.softmax(logits_k, dim=-1), num_samples=1)
                for logits_k in next_token_logits[0]
            ]).unsqueeze(0)  # [1, K, 1]
            
            # 追加到序列
            generated = torch.cat([generated, next_tokens], dim=-1)
            
            # 检查结束条件
            if self._should_stop(generated):
                break
        
        return generated
```

### 3. 歌词-旋律对齐

```python
# ============================================
# 歌词与旋律的时间对齐
# ============================================

class LyricsMelodyAligner:
    """歌词-旋律对齐器"""
    
    def __init__(self):
        self.tokenizer = AutoTokenizer.from_pretrained("bert-base-chinese")
        self.aligner = CrossModalAttentionModel()
    
    def align(self, lyrics, melody_tokens):
        """
        将歌词与旋律Token对齐
        
        Args:
            lyrics: 歌词文本（带时间标签）
            melody_tokens: 旋律Token序列
        
        Returns:
            aligned: 每个歌词字对应的旋律位置
        """
        # 1. 歌词Token化
        lyric_tokens = self.tokenizer(lyrics, return_tensors="pt")
        
        # 2. 计算对齐矩阵
        # alignment[i, j] = 歌词第i个字与旋律第j个token的关联度
        alignment = self.aligner(lyric_tokens, melody_tokens)
        
        # 3. 使用DTW（动态时间规整）找到最优对齐
        aligned_path = self._dtw_alignment(alignment)
        
        return aligned_path
    
    def _dtw_alignment(self, similarity_matrix):
        """动态时间规整"""
        import numpy as np
        
        N, M = similarity_matrix.shape
        dtw = np.full((N+1, M+1), np.inf)
        dtw[0, 0] = 0
        
        for i in range(1, N+1):
            for j in range(1, M+1):
                cost = 1 - similarity_matrix[i-1, j-1]
                dtw[i, j] = cost + min(
                    dtw[i-1, j],
                    dtw[i, j-1],
                    dtw[i-1, j-1]
                )
        
        # 回溯最优路径
        path = []
        i, j = N, M
        while i > 0 and j > 0:
            path.append((i-1, j-1))
            
            # 选择最小值的方向
            choices = [
                dtw[i-1, j],
                dtw[i, j-1],
                dtw[i-1, j-1]
            ]
            idx = np.argmin(choices)
            
            if idx == 0:
                i -= 1
            elif idx == 1:
                j -= 1
            else:
                i -= 1
                j -= 1
        
        return list(reversed(path))

# 歌词结构标注
lyrics_structure = {
    "intro": {
        "time": "0:00-0:15",
        "lyrics": None,  # 纯音乐
        "function": "establish mood"
    },
    "verse1": {
        "time": "0:15-0:45",
        "lyrics": "主歌第一段歌词...",
        "function": "tell story"
    },
    "chorus": {
        "time": "0:45-1:15",
        "lyrics": "副歌歌词...",
        "function": "hook, memorable"
    },
    "verse2": {
        "time": "1:15-1:45",
        "lyrics": "主歌第二段歌词...",
        "function": "develop story"
    },
    "chorus": {
        "time": "1:45-2:15",
        "lyrics": "副歌歌词...",
        "function": "repeat hook"
    },
    "bridge": {
        "time": "2:15-2:30",
        "lyrics": "桥段歌词...",
        "function": "contrast, build tension"
    },
    "outro": {
        "time": "2:30-3:00",
        "lyrics": None,
        "function": "fade out"
    }
}
```

---

## 来龙去脉：AI音乐生成的发展史

### 第一阶段：符号音乐生成（1950-2015）

```
符号音乐生成演进：

1957 - Illiac Suite（Lejaren Hiller）
├── 使用计算机生成弦乐四重奏
├── 基于规则的组合
└── 被认为是第一首AI音乐

1980s - 基于规则的专家系统
├── MusicXML表示
├── 和声规则库
└── 有限创造力

1990s - 马尔可夫模型
├── 基于转移概率生成音符
├── 短程依赖建模
└── 缺乏长期结构

2010s - RNN/LSTM时代
├── 循环神经网络建模序列
├── 可学习长期依赖
└── 代表：BachBot, DeepBach
```

### 第二阶段：神经音频合成（2016-2020）

```
音频合成革命：

2016 - WaveNet（DeepMind）
├── 原始波形自回归生成
├── 音质极高（接近真人）
├── 问题：生成极慢（几分钟/秒）
└── 应用：TTS, 音乐生成

2017 - NSynth（Google）
├── 学习乐器音色Latent空间
├── 可插值生成新音色
└── 单音生成

2018 - MelNet + WaveRNN
├── Mel频谱生成 + 神经网络声码器
├── 速度大幅提升
└── 音乐生成质量改善

2019 - Jukebox（OpenAI）
├── 直接在原始音频上训练Transformer
├── 可生成带歌词的歌曲
├── 支持多种风格
└── 1.2B参数，计算量巨大

2020 - 问题暴露：
- 自回归生成太慢
- 长歌曲难以建模
- 风格控制困难
```

### 第三阶段：Token化革命（2021-2023）

```
音频Token化时代：

2021 - SoundStream（Google）
├── 神经音频编解码器
├── 实时编码解码
└── 为Token化铺路

2022 - EnCodec（Meta）
├── 开源音频编解码器
├── Residual Vector Quantization
├── 6kbps高质量音频
└── 奠定音频Token化标准

2022 - MusicLM（Google）
├── 文本→音乐生成
├── 使用MuLaKE音乐理解模型
├── 支持长音乐生成（5分钟）
└── 未开放（版权争议）

2023 - AudioLDM / AudioLDM2
├──  Latent Diffusion for Audio
├── 文本条件音频生成
├── 开源可复现
└── 多种音频类型

2023 - 音乐理解突破：
├── MusicBERT：音乐预训练模型
├── CLAP：音频-文本对比学习
└── 为生成模型提供强条件编码
```

### 第四阶段：商业产品爆发（2023-2024）

```
2023-2024年关键产品：

2023.12 - Suno V2
├── 文本→完整歌曲（歌词+旋律+伴奏）
├── 多风格支持
├── 高质量人声
└── 免费+付费模式

2024.01 - Udio
├── 前Google DeepMind团队
├── 极高音质
├── 长歌曲支持（15分钟）
├── 人声自然度行业顶尖
└── 邀请制

2024.03 - Suno V3
├── 音质大幅提升
├── 中文支持改善
├── 更多风格
└── 成为行业标杆

2024.06 - Stable Audio Open
├── Stability AI开源
├── 音频生成（非歌曲）
├── 47秒音频
└── 推动开源生态

2024.09 - Suno V3.5 / Udio v1.5
├── 音质进一步提升
├── 乐器分离度改善
├── 更长歌曲支持
└── 商业应用爆发
```

### 第五阶段：2026年现状

```
2026年AI音乐生成的工业级特征：

1. 端到端歌曲生成
   ├── 歌词 → 旋律 → 编曲 → 混音
   ├── 一键生成完整歌曲
   └── 质量接近专业制作

2. 精细控制
   ├── 乐器级控制（吉他/钢琴/鼓独立编辑）
   ├── 时间轴编辑（类似DAW）
   ├── 风格迁移和融合
   └── 实时交互生成

3. 多模态融合
   ├── 文本+图像→音乐（视频配乐）
   ├── 情绪识别→音乐生成
   └── GPT-5.5原生音频能力

4. 版权与授权
   ├── 训练数据全授权
   ├── 生成内容商用授权
   ├── 版权检测和溯源
   └── 艺术家合作模式

5. 实时生成
   ├── 游戏动态音乐
   ├── VR/AR空间音频
   └── 直播实时配乐
```

---

## 核心工具深度解析

### 1. Suno：全民音乐创作平台

```
Suno架构特点（推测）：

┌─────────────────────────────────────────────┐
│               Suno Pipeline                  │
├─────────────────────────────────────────────┤
│ 1. 文本理解                                   │
│    - 解析风格、情绪、主题                      │
│    - 歌词生成（GPT-based）                    │
│    - 结构规划（Intro-Verse-Chorus）           │
├─────────────────────────────────────────────┤
│ 2. 旋律生成                                   │
│    - 基于文本条件的音频Token预测              │
│    - 和声进行生成                             │
│    - 节奏模式选择                             │
├─────────────────────────────────────────────┤
│ 3. 人声合成                                   │
│    - 歌词→音素→声学特征                       │
│    - 旋律指导的TTS                            │
│    - 情感表达控制                             │
├─────────────────────────────────────────────┤
│ 4. 伴奏生成                                   │
│    - 多轨乐器生成                             │
│    - 编曲编排                                 │
│    - 混音处理                                 │
├─────────────────────────────────────────────┤
│ 5. 后处理                                     │
│    - 母带处理                                 │
│    - 音质增强                                 │
│    - 格式转换                                 │
└─────────────────────────────────────────────┘
```

```markdown
## Suno使用深度指南

### 基础模式
```
1. 文本描述生成
   - 输入风格、情绪、主题
   - Suno自动生成歌词和音乐
   - 适合快速创作

2. 自定义模式
   - 自己写歌词
   - 选择风格标签
   - 生成音乐
   - 适合有明确想法的用户
```

### 提示词结构
```
[风格] + [情绪] + [乐器] + [节奏] + [主题] + [人声]

示例1：流行歌曲
```
Upbeat pop song with electronic elements,
happy and energetic, featuring synthesizer 
and electronic drums, about summer vacation 
and beach parties, female vocals, 
120 BPM, radio-friendly
```

示例2：古风歌曲
```
Chinese traditional style song,
melancholic and nostalgic,
using guzheng and bamboo flute,
telling a story of farewell and longing,
slow tempo, female vocals,
ancient Chinese poetic lyrics
```

示例3：摇滚歌曲
```
Alternative rock song,
intense and rebellious,
electric guitar riffs and heavy drums,
about breaking free from constraints,
140 BPM, male vocals with grit,
arena rock style
```

### 高级技巧

1. **元标签（Metatags）**
```
[Intro] - 前奏
[Verse] - 主歌
[Chorus] - 副歌
[Bridge] - 桥段
[Outro] - 尾奏

示例：
[Intro]
(Instrumental, building tension)

[Verse]
Walking down the empty street
Lights are fading, heart skips a beat

[Chorus]
We are the dreamers of the night
Chasing stars, holding tight
```

2. **风格混合**
```
"Fusion of jazz and electronic music,
 with live saxophone and synth bass,
 lounge atmosphere, 90 BPM"
```

3. **参考艺术家风格**
```
"In the style of Daft Punk meets Hans Zimmer,
 electronic orchestral fusion,
 cinematic and epic"
```

4. **中文提示词优化**
```
# 中文效果较好的描述
"中文流行歌曲，抒情风格，
 钢琴伴奏，弦乐铺垫，
 关于青春回忆，温暖治愈，
 女声，中速"

# 避免
"生成一首好听的中文歌"（过于模糊）
```
```

### 2. Udio：专业级音乐生成

```markdown
## Udio技术特点

### 核心优势
```
1. 音质
   - 采样率高（推测48kHz）
   - 动态范围广
   - 频率响应完整

2. 人声
   - 发音清晰
   - 情感丰富
   - 自然度高（行业顶尖）

3. 音乐性
   - 和声复杂度高
   - 编曲层次丰富
   - 段落对比明显

4. 长度
   - 支持15分钟长歌曲
   - 中段生成（Continuation）
   - 适合专辑制作
```

### 使用方式
```
平台：udio.com
状态：逐步开放（曾邀请制）

核心功能：
1. 文本生成
   - 输入描述生成完整歌曲
   
2. 手动模式
   - 自己写歌词
   - 精确控制每个段落
   
3. 扩展功能
   - 在已有歌曲基础上继续生成
   - 保持风格一致性
   
4. 风格参考
   - 上传参考音频
   - 生成相似风格
```

### 提示词技巧
```markdown
Udio提示词特点：
- 对自然语言理解强
- 详细描述效果更好
- 支持复杂音乐结构

示例：
```
A cinematic orchestral piece that builds from 
quiet strings to a full epic climax, 
featuring French horns and timpani, 
in the style of John Williams, 
3 minutes long, suitable for movie trailer
```

 continuation提示：
```
Continue the previous song with a bridge section 
that introduces electronic elements, 
then return to the orchestral theme for finale
```
```
```

### 3. 开源音频生成模型

```python
# ============================================
# AudioLDM 2：开源音频生成
# ============================================

from audioldm2 import text_to_audio, build_model

# 加载模型
model = build_model(
    checkpoint_path="audioldm2-full-supervised",
    device="cuda"
)

# 文本生成音频
audio = text_to_audio(
    model,
    text=["A dog barking in a park with birds chirping"],
    duration=10,          # 秒
    guidance_scale=3.5,   # CFG强度
    random_seed=42,
    num_inference_steps=200,
    n_candidates=3        # 生成3个候选
)

# 保存
import scipy
scipy.io.wavfile.write("output.wav", 16000, audio[0])

"""
AudioLDM 2特点：
- 支持文本、音频、图像条件
- 可生成音效、音乐、语音
- 开源可商用
- 最长47秒
"""

# ============================================
# MusicGen（Meta）：开源音乐生成
# ============================================

from transformers import AutoProcessor, MusicgenForConditionalGeneration

# 加载模型
model = MusicgenForConditionalGeneration.from_pretrained(
    "facebook/musicgen-medium"
)
processor = AutoProcessor.from_pretrained("facebook/musicgen-medium")

# 准备输入
inputs = processor(
    text=["80s pop track with bassy drums and synth"],
    return_tensors="pt"
)

# 生成
audio_values = model.generate(
    **inputs,
    max_new_tokens=256,      # 约5秒（50Hz采样率）
    do_sample=True,
    guidance_scale=3.0
)

# 保存
import scipy
sampling_rate = model.config.audio_encoder.sampling_rate
scipy.io.wavfile.write("musicgen.wav", sampling_rate, audio_values[0, 0].numpy())

"""
MusicGen特点：
- 小模型（300M-3.3B参数）
- 快速生成
- 支持旋律条件（Melody条件）
- 开源可商用
"""

# ============================================
# Stable Audio Open
# ============================================

from stable_audio_tools import get_pretrained_model
from stable_audio_tools.inference.generation import generate_diffusion_cond

# 加载模型
model, model_config = get_pretrained_model("stable-audio-open-1.0")

# 生成条件
conditioning = [{
    "prompt": "128 BPM tech house drum loop",
    "seconds_start": 0,
    "seconds_total": 47
}]

# 生成
output = generate_diffusion_cond(
    model,
    steps=100,
    cfg_scale=7,
    conditioning=conditioning,
    sample_size=65536,
    sigma_min=0.3,
    sigma_max=500,
    sampler_type="dpmpp-3m-sde",
    device="cuda"
)

# 保存
import torchaudio
torchaudio.save("stable_audio.wav", output, sample_rate=44100)
```

### 4. 本地部署音乐生成

```python
# ============================================
# 本地音乐生成完整流程
# ============================================

import torch
import torchaudio
from transformers import AutoProcessor, MusicgenForConditionalGeneration

class LocalMusicGenerator:
    """本地音乐生成器"""
    
    def __init__(self, model_size="medium"):
        """
        model_size: small (300M), medium (1.5B), large (3.3B), melody (1.5B + melody条件)
        """
        self.model = MusicgenForConditionalGeneration.from_pretrained(
            f"facebook/musicgen-{model_size}",
            torch_dtype=torch.float16
        ).to("cuda")
        
        self.processor = AutoProcessor.from_pretrained(
            f"facebook/musicgen-{model_size}"
        )
        
        self.sampling_rate = self.model.config.audio_encoder.sampling_rate
    
    def generate(
        self,
        prompt,
        duration=10,
        guidance_scale=3.0,
        temperature=1.0,
        top_k=250,
        num_cuts=1
    ):
        """
        生成音乐
        
        Args:
            prompt: 文本提示词
            duration: 时长（秒）
            guidance_scale: CFG强度
            temperature: 采样温度
            top_k: Top-K采样
            num_cuts: 生成数量
        """
        
        # 计算token数量
        # MusicGen采样率：32Hz（每秒钟32个token）
        max_new_tokens = int(duration * 50)  # 50 tokens/秒
        
        inputs = self.processor(
            text=[prompt] * num_cuts,
            return_tensors="pt"
        ).to("cuda")
        
        # 生成
        with torch.no_grad():
            audio_values = self.model.generate(
                **inputs,
                max_new_tokens=max_new_tokens,
                do_sample=True,
                guidance_scale=guidance_scale,
                temperature=temperature,
                top_k=top_k
            )
        
        # 转换为音频
        audios = []
        for i in range(num_cuts):
            audio = audio_values[i, 0].cpu().numpy()
            audios.append(audio)
        
        return audios
    
    def generate_with_melody(
        self,
        prompt,
        melody_path,
        duration=10
    ):
        """
        使用旋律条件生成
        
        Args:
            prompt: 文本提示词
            melody_path: 参考旋律音频路径
            duration: 生成时长
        """
        
        # 加载旋律
        melody, sr = torchaudio.load(melody_path)
        
        # 重采样到模型采样率
        if sr != self.sampling_rate:
            resampler = torchaudio.transforms.Resample(sr, self.sampling_rate)
            melody = resampler(melody)
        
        # 处理输入
        inputs = self.processor(
            text=[prompt],
            audio=melody.numpy(),
            return_tensors="pt",
            sampling_rate=self.sampling_rate
        ).to("cuda")
        
        # 生成
        audio_values = self.model.generate(
            **inputs,
            max_new_tokens=int(duration * 50)
        )
        
        return audio_values[0, 0].cpu().numpy()
    
    def save(self, audio, path):
        """保存音频"""
        import scipy
        scipy.io.wavfile.write(path, self.sampling_rate, audio)

# 使用
generator = LocalMusicGenerator(model_size="medium")

# 生成音乐
audios = generator.generate(
    prompt="Upbeat electronic dance music, festival anthem, 
            heavy bass drop, energetic, 128 BPM",
    duration=15,
    num_cuts=3
)

# 保存
for i, audio in enumerate(audios):
    generator.save(audio, f"generated_{i}.wav")

# 使用旋律参考
audio_with_melody = generator.generate_with_melody(
    prompt="Orchestral arrangement of this melody, cinematic, epic",
    melody_path="reference_melody.wav",
    duration=30
)
generator.save(audio_with_melody, "melody_based.wav")
```

---

## 模型差异：不同架构的音乐生成策略

### 1. 架构对比

```
音乐生成模型架构对比：

┌─────────────────────────────────────────────────────────────────────┐
│ 特性          │ Suno      │ Udio      │ MusicGen  │ AudioLDM2 │ Jukebox  │
├─────────────────────────────────────────────────────────────────────┤
│ 架构          │ 推测LM    │ 推测LM    │ LM(EnCodec)│ LDM      │ Transformer│
│ 开源          │ 否        │ 否        │ 是         │ 是        │ 是       │
│ 本地部署      │ 不可      │ 不可      │ 可         │ 可        │ 困难     │
│ 显存需求      │ N/A       │ N/A       │ 8-16GB     │ 12GB      │ 40GB+    │
│ 最长生成      │ 4分钟     │ 15分钟    │ 30秒       │ 47秒      │ 4分钟    │
│ 人声质量      │ ★★★★      │ ★★★★★     │ 无         │ 无        │ ★★★      │
│ 音质          │ ★★★★      │ ★★★★★     │ ★★★        │ ★★★       │ ★★★      │
│ 风格多样性    │ ★★★★★     │ ★★★★      │ ★★★        │ ★★★       │ ★★★★     │
│ 歌词支持      │ 原生      │ 原生      │ 无         │ 无        │ 有       │
│ 商用授权      │ 付费可商用 │ 付费可商用 │ 开放       │ 开放      │ 开放     │
│ API稳定性     │ 高        │ 中        │ 自托管      │ 自托管     │ 自托管    │
│ 成本          │ $10-30/月 │ 按量计费   │ 硬件成本    │ 硬件成本   │ 硬件成本  │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. 音频表示对比

```python
# ============================================
# 不同音频表示方式的对比
# ============================================

"""
1. 原始波形（Jukebox）
   优点：
   - 信息无损
   - 直接建模
   
   缺点：
   - 数据量巨大（44.1kHz）
   - 长程依赖难建模
   - 生成极慢

2. 频谱图（AudioLDM）
   优点：
   - 压缩率高
   - 可用图像扩散模型
   
   缺点：
   - 相位恢复困难
   - 音质受限
   - 需要声码器

3. 音频Token（MusicGen, Suno, Udio）
   优点：
   - 压缩率高（~150:1）
   - 可用语言模型
   - 生成速度快
   - 质量好
   
   缺点：
   - 需要预训练编解码器
   - 重建可能丢失细节
"""

# 计算不同表示的数据量
def calculate_data_size(duration_seconds, representation="token"):
    """
    计算不同表示的数据量
    """
    if representation == "waveform":
        # 44.1kHz, 16bit, 立体声
        bytes_per_sec = 44100 * 2 * 2  # 176.4KB/s
        total = duration_seconds * bytes_per_sec
        
    elif representation == "spectrogram":
        # 128 mel bins, 86Hz帧率, 32bit
        bytes_per_sec = 128 * 86 * 4  # 44KB/s
        total = duration_seconds * bytes_per_sec
        
    elif representation == "token":
        # 4 quantizers, 50Hz, 10bit each (1024 codebook)
        bytes_per_sec = 4 * 50 * 2  # 400B/s
        total = duration_seconds * bytes_per_sec
    
    return total

# 3分钟歌曲
print("3分钟歌曲数据量对比：")
print(f"波形: {calculate_data_size(180, 'waveform') / 1024 / 1024:.1f} MB")
print(f"频谱: {calculate_data_size(180, 'spectrogram') / 1024:.1f} KB")
print(f"Token: {calculate_data_size(180, 'token'):.0f} B")
```

### 3. 采样策略

```python
# ============================================
# 音频生成的采样策略
# ============================================

"""
音频生成的关键采样参数：

1. Temperature（温度）
   - 控制随机性
   - 低温度（0.7）：保守，重复性高
   - 高温度（1.2+）：创造性高，可能不和谐

2. Top-K
   - 限制采样范围
   - K=50：只从概率最高的50个token采样
   - K=250：更多选择，更多样

3. Top-P（Nucleus Sampling）
   - 从累积概率达到P的token中采样
   - P=0.9：覆盖90%概率质量
   - 动态调整K值

4. Classifier-Free Guidance（CFG）
   - 音乐生成中通常较低（1-3）
   - 太高会导致不和谐
"""

class SamplingConfig:
    """音频生成采样配置"""
    
    def __init__(self):
        # 基础参数
        self.temperature = 1.0
        self.top_k = 250
        self.top_p = 0.9
        self.guidance_scale = 3.0
        
        # 风格特定配置
        self.presets = {
            "creative": {
                "temperature": 1.2,
                "top_k": 500,
                "guidance_scale": 2.0
            },
            "safe": {
                "temperature": 0.8,
                "top_k": 50,
                "guidance_scale": 4.0
            },
            "balanced": {
                "temperature": 1.0,
                "top_k": 250,
                "guidance_scale": 3.0
            }
        }
    
    def get_preset(self, name):
        """获取预设配置"""
        return self.presets.get(name, self.presets["balanced"])

# 使用不同采样策略生成
def generate_with_sampling(model, prompt, config):
    """使用特定采样配置生成"""
    
    inputs = processor(text=[prompt], return_tensors="pt")
    
    audio = model.generate(
        **inputs,
        do_sample=True,
        temperature=config["temperature"],
        top_k=config["top_k"],
        guidance_scale=config["guidance_scale"]
    )
    
    return audio

# 对比不同策略
for preset_name in ["creative", "safe", "balanced"]:
    config = SamplingConfig().get_preset(preset_name)
    audio = generate_with_sampling(model, prompt, config)
    save_audio(audio, f"{preset_name}.wav")
```

---

## 高级技术：风格控制与多轨生成

### 1. 风格控制技术

```python
# ============================================
# 音乐风格控制技术
# ============================================

class StyleController:
    """音乐风格控制器"""
    
    def __init__(self):
        self.style_embeddings = self._load_style_embeddings()
    
    def apply_style(self, base_tokens, style_name, strength=0.5):
        """
        应用风格到基础音乐
        
        Args:
            base_tokens: 基础音乐Token
            style_name: 风格名称
            strength: 风格强度（0-1）
        """
        
        # 获取风格embedding
        style_emb = self.style_embeddings[style_name]
        
        # 风格混合
        # 在Latent空间进行插值
        mixed_tokens = self._interpolate_tokens(
            base_tokens,
            style_emb,
            alpha=strength
        )
        
        return mixed_tokens
    
    def mix_styles(self, style_names, weights):
        """
        混合多种风格
        
        Args:
            style_names: 风格列表
            weights: 对应权重
        """
        
        # 加权平均风格embedding
        mixed_emb = sum(
            self.style_embeddings[name] * weight
            for name, weight in zip(style_names, weights)
        )
        
        return mixed_emb
    
    def style_transfer(
        self,
        source_audio,
        target_style,
        preserve_melody=True
    ):
        """
        风格迁移：保持旋律，改变风格
        
        类似于图像的风格迁移
        """
        
        # 1. 提取源音频的内容（旋律/和声）
        content_tokens = self.extract_content(source_audio)
        
        # 2. 提取目标风格的特征
        style_tokens = self.style_embeddings[target_style]
        
        # 3. 在Token空间进行风格迁移
        if preserve_melody:
            # 保持旋律轮廓，替换音色和伴奏风格
            transferred = self._transfer_preserving_melody(
                content_tokens,
                style_tokens
            )
        else:
            # 完全迁移
            transferred = self._full_transfer(
                content_tokens,
                style_tokens
            )
        
        return transferred
    
    def _load_style_embeddings(self):
        """加载预训练的风格embedding"""
        # 实际应加载预计算的风格向量
        return {
            "jazz": torch.randn(512),
            "rock": torch.randn(512),
            "classical": torch.randn(512),
            "electronic": torch.randn(512),
            "pop": torch.randn(512)
        }

# 风格混合示例
controller = StyleController()

# 混合爵士和电子
mixed_style = controller.mix_styles(
    ["jazz", "electronic"],
    weights=[0.6, 0.4]
)
```

### 2. 多轨生成与分离

```python
# ============================================
# 多轨音乐生成
# ============================================

class MultiTrackGenerator:
    """多轨音乐生成器"""
    
    def __init__(self):
        self.stem_models = {
            "drums": load_model("stem_drums"),
            "bass": load_model("stem_bass"),
            "guitar": load_model("stem_guitar"),
            "piano": load_model("stem_piano"),
            "vocals": load_model("stem_vocals")
        }
    
    def generate_stems(self, prompt, structure):
        """
        生成各音轨
        
        Args:
            prompt: 整体描述
            structure: 歌曲结构（段落、和弦进行）
        
        Returns:
            stems: 各音轨音频
        """
        
        stems = {}
        
        # 1. 生成鼓轨
        stems["drums"] = self.stem_models["drums"].generate(
            prompt=f"{prompt}, drum track, rhythm section",
            structure=structure
        )
        
        # 2. 生成贝斯
        stems["bass"] = self.stem_models["bass"].generate(
            prompt=f"{prompt}, bass line, following chord progression",
            structure=structure,
            condition=stems["drums"]  # 与鼓对齐
        )
        
        # 3. 生成和乐器
        stems["guitar"] = self.stem_models["guitar"].generate(
            prompt=f"{prompt}, electric guitar, chords and riffs",
            structure=structure,
            condition=stems["bass"]
        )
        
        # 4. 生成旋律乐器
        stems["piano"] = self.stem_models["piano"].generate(
            prompt=f"{prompt}, piano, melody and fills",
            structure=structure,
            condition=stems["guitar"]
        )
        
        # 5. 生成人声
        stems["vocals"] = self.stem_models["vocals"].generate(
            prompt=f"{prompt}, lead vocals, melody",
            structure=structure,
            lyrics=structure.get("lyrics"),
            condition=stems["piano"]
        )
        
        return stems
    
    def mix_stems(self, stems, mix_config=None):
        """
        混音：将各音轨混合为最终歌曲
        
        Args:
            stems: 各音轨音频
            mix_config: 混音配置（音量、声像、效果）
        """
        
        mix_config = mix_config or {
            "drums": {"volume": 0.8, "pan": 0.0, "eq": "bright"},
            "bass": {"volume": 0.7, "pan": 0.0, "eq": "warm"},
            "guitar": {"volume": 0.6, "pan": -0.3, "eq": "mid"},
            "piano": {"volume": 0.5, "pan": 0.3, "eq": "bright"},
            "vocals": {"volume": 0.9, "pan": 0.0, "eq": "clear"}
        }
        
        # 初始化混合音频
        total_length = max(len(stem) for stem in stems.values())
        mixed = np.zeros(total_length)
        
        # 混音处理
        for stem_name, audio in stems.items():
            config = mix_config.get(stem_name, {})
            
            # 音量调整
            volume = config.get("volume", 0.7)
            audio = audio * volume
            
            # 声像调整（立体声）
            pan = config.get("pan", 0.0)
            # 简化：单声道混合
            
            # EQ处理（简化）
            eq = config.get("eq", "flat")
            if eq == "bright":
                audio = self._apply_eq(audio, highs=1.2)
            elif eq == "warm":
                audio = self._apply_eq(audio, lows=1.2)
            
            # 叠加
            mixed[:len(audio)] += audio
        
        # 母带处理
        mixed = self._mastering(mixed)
        
        return mixed
    
    def _apply_eq(self, audio, lows=1.0, mids=1.0, highs=1.0):
        """简单EQ处理"""
        # 实际应使用FFT滤波
        return audio
    
    def _mastering(self, audio):
        """母带处理"""
        # 1. 压缩
        audio = self._compress(audio)
        
        # 2. 限制器
        audio = np.clip(audio, -1.0, 1.0)
        
        # 3. 音量标准化
        max_val = np.max(np.abs(audio))
        if max_val > 0:
            audio = audio / max_val * 0.95
        
        return audio
    
    def _compress(self, audio, threshold=0.5, ratio=4.0):
        """动态范围压缩"""
        # 简化实现
        compressed = np.where(
            np.abs(audio) > threshold,
            np.sign(audio) * (threshold + (np.abs(audio) - threshold) / ratio),
            audio
        )
        return compressed

# 使用
generator = MultiTrackGenerator()

stems = generator.generate_stems(
    prompt="Upbeat pop rock song, energetic, stadium anthem",
    structure={
        "tempo": 130,
        "key": "E major",
        "chords": ["E", "G#m", "A", "B"],
        "lyrics": "..."
    }
)

# 混音
final_mix = generator.mix_stems(stems)
save_audio(final_mix, "final_mix.wav")

# 单独导出各音轨
for name, audio in stems.items():
    save_audio(audio, f"stem_{name}.wav")
```

### 3. 歌词-旋律对齐的高级技术

```python
# ============================================
# 歌词与旋律的高级对齐
# ============================================

class LyricsMelodySystem:
    """歌词旋律系统"""
    
    def __init__(self):
        self.lyric_parser = LyricParser()
        self.melody_generator = MelodyGenerator()
        self.aligner = LyricsMelodyAligner()
    
    def create_song(self, lyrics, style_config):
        """
        从歌词创建完整歌曲
        
        Args:
            lyrics: 结构化歌词（带段落标记）
            style_config: 风格配置
        """
        
        # 1. 解析歌词结构
        parsed_lyrics = self.lyric_parser.parse(lyrics)
        
        # 2. 为每个段落生成旋律
        melodies = {}
        for section_name, section_lyrics in parsed_lyrics.items():
            # 根据段落功能选择旋律类型
            section_type = self._get_section_type(section_name)
            
            melody = self.melody_generator.generate(
                lyrics=section_lyrics,
                section_type=section_type,
                style=style_config
            )
            
            melodies[section_name] = melody
        
        # 3. 确保段落间旋律连贯
        coherent_melodies = self._ensure_coherence(melodies)
        
        # 4. 对齐歌词和旋律
        aligned_song = self.aligner.align_all(coherent_melodies, parsed_lyrics)
        
        # 5. 生成伴奏
        accompaniment = self._generate_accompaniment(
            aligned_song,
            style_config
        )
        
        # 6. 合成最终歌曲
        final_song = self._synthesize(aligned_song, accompaniment)
        
        return final_song
    
    def _get_section_type(self, section_name):
        """根据段落名确定类型"""
        section_map = {
            "intro": "instrumental",
            "verse": "narrative",
            "chorus": "hook",
            "bridge": "contrast",
            "outro": "fade"
        }
        return section_map.get(section_name.lower(), "verse")
    
    def _ensure_coherence(self, melodies):
        """确保旋律连贯性"""
        # 确保副歌相同，主歌有变化但相关
        
        chorus_melody = melodies.get("chorus")
        if chorus_melody:
            # 所有副歌使用相同旋律
            for key in melodies:
                if "chorus" in key.lower():
                    melodies[key] = chorus_melody
        
        # 确保主歌与副歌有音乐关系
        verse_melody = melodies.get("verse1")
        if verse_melody and chorus_melody:
            # 调整主歌以与副歌形成对比
            melodies["verse1"] = self._contrast_melody(
                verse_melody, chorus_melody
            )
        
        return melodies
    
    def _contrast_melody(self, melody_a, melody_b):
        """创建对比旋律"""
        # 实际应使用音乐理论规则
        return melody_a
    
    def _generate_accompaniment(self, song, style_config):
        """生成伴奏"""
        # 根据旋律和风格生成和弦、贝斯、鼓等
        pass
    
    def _synthesize(self, song, accompaniment):
        """合成最终音频"""
        # 人声合成 + 伴奏混合
        pass

# 歌词解析器
class LyricParser:
    def parse(self, lyrics_text):
        """解析歌词文本"""
        sections = {}
        current_section = "verse1"
        
        for line in lyrics_text.split('\n'):
            line = line.strip()
            
            # 检测段落标记
            if line.startswith('[') and line.endswith(']'):
                current_section = line[1:-1].lower()
                sections[current_section] = []
            elif line:
                if current_section not in sections:
                    sections[current_section] = []
                sections[current_section].append(line)
        
        return sections
```

---

## 工业级实践案例

### 案例1：游戏背景音乐生成系统

```python
# ============================================
# 游戏背景音乐动态生成系统
# ============================================

class GameMusicSystem:
    """游戏音乐系统"""
    
    def __init__(self):
        self.mood_states = {
            "exploration": {"tempo": 80, "energy": 0.3, "tension": 0.2},
            "combat": {"tempo": 140, "energy": 0.9, "tension": 0.8},
            "dialogue": {"tempo": 60, "energy": 0.2, "tension": 0.3},
            "boss_fight": {"tempo": 160, "energy": 1.0, "tension": 1.0}
        }
        
        self.current_mood = "exploration"
        self.music_cache = {}
    
    def update_mood(self, game_state):
        """
        根据游戏状态更新音乐情绪
        
        Args:
            game_state: {
                "player_health": 0-100,
                "enemies_nearby": int,
                "in_combat": bool,
                "location": str
            }
        """
        
        # 确定新情绪
        new_mood = self._determine_mood(game_state)
        
        if new_mood != self.current_mood:
            # 平滑过渡
            self._transition_to(new_mood)
            self.current_mood = new_mood
    
    def _determine_mood(self, game_state):
        """确定音乐情绪"""
        if game_state.get("in_combat"):
            if game_state.get("enemies_nearby", 0) > 5:
                return "boss_fight"
            return "combat"
        elif game_state.get("dialogue_active"):
            return "dialogue"
        else:
            return "exploration"
    
    def _transition_to(self, new_mood, duration=3.0):
        """
        平滑过渡到新情绪
        
        Args:
            new_mood: 目标情绪
            duration: 过渡时长（秒）
        """
        
        # 1. 获取当前和目标的音频特征
        current_features = self._extract_features(self.current_audio)
        target_features = self.mood_states[new_mood]
        
        # 2. 生成过渡音乐
        transition = self._generate_transition(
            current_features,
            target_features,
            duration
        )
        
        # 3. 播放过渡
        self._play_audio(transition)
        
        # 4. 预生成目标情绪音乐
        if new_mood not in self.music_cache:
            self.music_cache[new_mood] = self._generate_loop(new_mood)
    
    def _generate_loop(self, mood, duration=60):
        """
        生成可循环播放的背景音乐
        
        Args:
            mood: 情绪类型
            duration: 循环长度（秒）
        """
        
        config = self.mood_states[mood]
        
        prompt = f"""
        Video game background music, {mood} mood,
        tempo {config['tempo']} BPM,
        {'high' if config['energy'] > 0.7 else 'low'} energy,
        {'tense' if config['tension'] > 0.5 else 'calm'} atmosphere,
        seamless loop, no abrupt ending,
        instrumental only
        """
        
        # 生成
        audio = generate_music(prompt, duration=duration)
        
        # 确保可循环：首尾平滑连接
        audio = self._make_loopable(audio)
        
        return audio
    
    def _make_loopable(self, audio, crossfade=2.0):
        """使音频可循环"""
        
        # 计算交叉淡化样本数
        crossfade_samples = int(crossfade * 44100)
        
        # 创建淡化曲线
        fade_in = np.linspace(0, 1, crossfade_samples)
        fade_out = np.linspace(1, 0, crossfade_samples)
        
        # 应用交叉淡化
        audio[:crossfade_samples] *= fade_in
        audio[-crossfade_samples:] *= fade_out
        
        # 叠加首尾
        looped = audio.copy()
        looped[:crossfade_samples] += audio[-crossfade_samples:]
        
        return looped
    
    def _generate_transition(self, from_features, to_features, duration):
        """生成过渡音乐"""
        # 在两种情绪特征之间插值
        
        prompt = f"""
        Musical transition from {from_features} to {to_features},
        {duration} seconds long,
        gradually changing mood,
        smooth and seamless
        """
        
        return generate_music(prompt, duration=duration)
    
    def _extract_features(self, audio):
        """提取音频特征"""
        # 提取BPM、能量、频谱特征等
        pass
    
    def _play_audio(self, audio):
        """播放音频"""
        # 使用音频播放库
        pass

# 使用
game_music = GameMusicSystem()

# 游戏循环中调用
game_state = {
    "player_health": 75,
    "enemies_nearby": 3,
    "in_combat": True
}

game_music.update_mood(game_state)
# → 切换到 "combat" 音乐
```

### 案例2：播客/视频自动配乐

```python
# ============================================
# 智能配乐系统
# ============================================

class SmartSoundtrack:
    """智能配乐系统"""
    
    def __init__(self):
        self.emotion_analyzer = EmotionAnalyzer()
        self.music_library = MusicLibrary()
    
    def auto_score(self, video_path, output_path):
        """
        自动为视频配乐
        
        Args:
            video_path: 输入视频路径
            output_path: 输出视频路径
        """
        
        # 1. 分析视频情绪曲线
        emotion_curve = self._analyze_video_emotion(video_path)
        
        # 2. 根据情绪曲线选择/生成音乐
        soundtrack = self._generate_soundtrack(emotion_curve)
        
        # 3. 混音视频和音乐
        final_video = self._mix_audio_video(video_path, soundtrack)
        
        # 4. 保存
        final_video.save(output_path)
    
    def _analyze_video_emotion(self, video_path):
        """
        分析视频情绪曲线
        
        Returns:
            [(timestamp, emotion, intensity), ...]
        """
        
        import cv2
        
        cap = cv2.VideoCapture(video_path)
        fps = cap.get(cv2.CAP_PROP_FPS)
        total_frames = int(cap.get(cv2.CAP_PROP_FRAME_COUNT))
        
        emotion_curve = []
        
        for frame_idx in range(0, total_frames, int(fps)):  # 每秒分析一帧
            cap.set(cv2.CAP_PROP_POS_FRAMES, frame_idx)
            ret, frame = cap.read()
            
            if not ret:
                break
            
            # 分析画面情绪
            emotion, intensity = self.emotion_analyzer.analyze_frame(frame)
            
            timestamp = frame_idx / fps
            emotion_curve.append((timestamp, emotion, intensity))
        
        cap.release()
        return emotion_curve
    
    def _generate_soundtrack(self, emotion_curve):
        """根据情绪曲线生成配乐"""
        
        segments = []
        
        # 将情绪曲线分段
        chunks = self._chunk_emotions(emotion_curve, chunk_duration=30)
        
        for start_time, end_time, dominant_emotion in chunks:
            # 为每段生成/选择音乐
            music = self._get_music_for_emotion(
                dominant_emotion,
                duration=end_time - start_time
            )
            
            segments.append({
                "start": start_time,
                "end": end_time,
                "audio": music
            })
        
        # 合并所有段（带交叉淡化）
        total_duration = emotion_curve[-1][0]
        soundtrack = self._merge_segments(segments, total_duration)
        
        return soundtrack
    
    def _chunk_emotions(self, emotion_curve, chunk_duration=30):
        """将情绪曲线分块"""
        chunks = []
        
        current_start = 0
        current_emotions = []
        
        for timestamp, emotion, intensity in emotion_curve:
            if timestamp - current_start >= chunk_duration:
                # 确定主导情绪
                dominant = max(set(current_emotions), key=current_emotions.count)
                chunks.append((current_start, timestamp, dominant))
                
                current_start = timestamp
                current_emotions = []
            
            current_emotions.append(emotion)
        
        return chunks
    
    def _get_music_for_emotion(self, emotion, duration):
        """为情绪获取音乐"""
        
        emotion_to_music = {
            "happy": "upbeat, cheerful, major key",
            "sad": "melancholic, slow, minor key",
            "exciting": "fast, energetic, intense",
            "calm": "peaceful, ambient, slow",
            "tense": "suspenseful, building tension"
        }
        
        prompt = emotion_to_music.get(emotion, "neutral background music")
        
        # 生成或从库中获取
        if duration < 30:
            return self.music_library.get(prompt, duration)
        else:
            return generate_music(prompt, duration=duration)
    
    def _merge_segments(self, segments, total_duration):
        """合并音乐段"""
        
        sample_rate = 44100
        total_samples = int(total_duration * sample_rate)
        mixed = np.zeros(total_samples)
        
        for seg in segments:
            start_sample = int(seg["start"] * sample_rate)
            end_sample = int(seg["end"] * sample_rate)
            audio = seg["audio"]
            
            # 确保长度匹配
            length = min(end_sample - start_sample, len(audio))
            
            # 交叉淡化
            mixed[start_sample:start_sample+length] += audio[:length]
        
        # 防止削波
        mixed = np.clip(mixed, -1.0, 1.0)
        
        return mixed
    
    def _mix_audio_video(self, video_path, soundtrack):
        """混音视频和音乐"""
        # 使用FFmpeg或moviepy
        from moviepy.editor import VideoFileClip, AudioFileClip
        
        video = VideoFileClip(video_path)
        
        # 保存配乐为临时文件
        temp_audio = "temp_soundtrack.wav"
        scipy.io.wavfile.write(temp_audio, 44100, soundtrack)
        
        music = AudioFileClip(temp_audio)
        
        # 混合（音乐音量降低）
        final_audio = CompositeAudioClip([
            video.audio,
            music.volumex(0.3)  # 音乐音量30%
        ])
        
        final_video = video.set_audio(final_audio)
        
        return final_video

# 使用
scorer = SmartSoundtrack()
scorer.auto_score("input_video.mp4", "output_scored.mp4")
```

### 案例3：品牌音乐自动生成

```python
# ============================================
# 品牌音乐识别系统
# ============================================

class BrandMusicSystem:
    """品牌音乐系统"""
    
    def __init__(self, brand_id):
        self.brand_id = brand_id
        self.brand_profile = self._load_brand_profile(brand_id)
        self.style_embedding = self._compute_style_embedding()
    
    def generate_brand_music(
        self,
        context,
        duration=30,
        variation=0.3
    ):
        """
        生成符合品牌调性的音乐
        
        Args:
            context: 使用场景（广告/门店/线上）
            duration: 时长
            variation: 变化度（0-1）
        """
        
        # 基础品牌风格
        base_prompt = self.brand_profile["music_style"]
        
        # 根据场景调整
        context_modifiers = {
            "advertisement": "upbeat, catchy, memorable hook",
            "store": "ambient, relaxing, background",
            "online": "modern, engaging, short attention span",
            "event": "energetic, celebratory, grand"
        }
        
        modifier = context_modifiers.get(context, "")
        
        # 生成基础音乐
        base_music = generate_music(
            prompt=f"{base_prompt}, {modifier}",
            duration=duration
        )
        
        # 注入品牌特征
        branded_music = self._apply_brand_characteristics(
            base_music,
            variation=variation
        )
        
        return branded_music
    
    def _load_brand_profile(self, brand_id):
        """加载品牌档案"""
        # 从数据库加载
        profiles = {
            "tech_startup": {
                "music_style": "modern electronic, innovative, clean",
                "key": "C major",
                "tempo_range": [120, 140],
                "instruments": ["synth", "electronic drums", "bass"]
            },
            "luxury_brand": {
                "music_style": "elegant orchestral, sophisticated, premium",
                "key": "F major",
                "tempo_range": [80, 100],
                "instruments": ["strings", "piano", "soft brass"]
            }
        }
        
        return profiles.get(brand_id, profiles["tech_startup"])
    
    def _compute_style_embedding(self):
        """计算品牌风格embedding"""
        # 使用品牌音乐样本计算平均embedding
        pass
    
    def _apply_brand_characteristics(self, music, variation=0.3):
        """应用品牌特征"""
        
        # 1. 调整调性到品牌偏好
        music = self._adjust_key(music, self.brand_profile["key"])
        
        # 2. 调整速度到品牌范围
        min_tempo, max_tempo = self.brand_profile["tempo_range"]
        target_tempo = np.random.uniform(min_tempo, max_tempo)
        music = self._adjust_tempo(music, target_tempo)
        
        # 3. 强调品牌乐器
        music = self._emphasize_instruments(
            music,
            self.brand_profile["instruments"]
        )
        
        return music
    
    def generate_variations(self, base_music, num_variations=5):
        """生成品牌音乐的变体"""
        
        variations = []
        
        for i in range(num_variations):
            # 在Latent空间扰动
            variation = self._perturb_in_latent(
                base_music,
                strength=0.2 + i * 0.1
            )
            
            variations.append(variation)
        
        return variations
    
    def _adjust_key(self, music, target_key):
        """调整调性"""
        # 使用音高变换
        key_shifts = {"C": 0, "D": 2, "E": 4, "F": 5, "G": 7, "A": 9, "B": 11}
        shift = key_shifts.get(target_key, 0)
        
        # 使用librosa音高变换
        shifted = librosa.effects.pitch_shift(
            music, sr=44100, n_steps=shift
        )
        
        return shifted
    
    def _adjust_tempo(self, music, target_tempo):
        """调整速度"""
        # 检测当前速度
        current_tempo = librosa.beat.tempo(y=music, sr=44100)[0]
        
        # 计算速度比
        ratio = target_tempo / current_tempo
        
        # 时间拉伸
        stretched = librosa.effects.time_stretch(music, rate=ratio)
        
        return stretched
    
    def _emphasize_instruments(self, music, instruments):
        """强调特定乐器"""
        # 使用源分离然后调整音量
        pass
    
    def _perturb_in_latent(self, music, strength):
        """在Latent空间扰动"""
        # 编码到Latent空间，添加噪声，解码
        pass

# 使用
brand_music = BrandMusicSystem("luxury_brand")

# 生成广告音乐
ad_music = brand_music.generate_brand_music(
    context="advertisement",
    duration=30
)

# 生成门店背景音乐
store_music = brand_music.generate_brand_music(
    context="store",
    duration=300  # 5分钟
)

# 生成多个变体用于A/B测试
variations = brand_music.generate_variations(ad_music, num_variations=3)
```

---

## 评估与优化体系

### 1. 音乐质量评估指标

```python
# ============================================
# 音乐质量自动评估
# ============================================

class MusicQualityEvaluator:
    """音乐质量评估器"""
    
    def __init__(self):
        self.mert_model = None  # 音乐理解模型
    
    def evaluate(self, audio, reference=None, prompt=""):
        """
        综合评估音乐质量
        """
        metrics = {}
        
        # 1. 基础音频质量
        metrics['audio_quality'] = self._evaluate_audio_quality(audio)
        
        # 2. 音乐理论合规性
        metrics['music_theory'] = self._evaluate_music_theory(audio)
        
        # 3. 风格一致性
        metrics['style_consistency'] = self._evaluate_style_consistency(audio)
        
        # 4. 创新性/多样性
        metrics['diversity'] = self._evaluate_diversity(audio)
        
        # 5. 文本对齐度（如果有提示词）
        if prompt:
            metrics['text_alignment'] = self._evaluate_text_alignment(
                audio, prompt
            )
        
        # 6. 与参考对比（如果有参考）
        if reference is not None:
            metrics['similarity'] = self._calculate_similarity(audio, reference)
        
        # 综合评分
        metrics['overall'] = self._calculate_overall(metrics)
        
        return metrics
    
    def _evaluate_audio_quality(self, audio):
        """评估音频质量"""
        
        # 1. 动态范围
        dynamic_range = np.max(audio) - np.min(audio)
        
        # 2. 频谱平衡
        spec = np.abs(np.fft.fft(audio))
        freq_balance = np.std(spec) / np.mean(spec)
        
        # 3. 失真检测（简化）
        clipping = np.sum(np.abs(audio) > 0.99) / len(audio)
        
        # 综合
        score = (
            min(dynamic_range / 2.0, 1.0) * 0.4 +
            min(freq_balance / 2.0, 1.0) * 0.4 +
            (1.0 - clipping) * 0.2
        )
        
        return score
    
    def _evaluate_music_theory(self, audio):
        """评估音乐理论合规性"""
        
        # 1. 检测调性稳定性
        chroma = librosa.feature.chroma_stft(y=audio, sr=44100)
        key_stability = np.mean(np.std(chroma, axis=1))
        
        # 2. 检测节拍稳定性
        tempo, beat_frames = librosa.beat.beat_track(y=audio, sr=44100)
        beat_intervals = np.diff(beat_frames)
        tempo_stability = 1.0 - np.std(beat_intervals) / np.mean(beat_intervals)
        
        # 3. 检测和声进行合理性
        # 简化：检测是否存在明显不和谐的帧比例
        harmony_score = self._evaluate_harmony(chroma)
        
        score = (
            min(key_stability * 2, 1.0) * 0.3 +
            tempo_stability * 0.4 +
            harmony_score * 0.3
        )
        
        return score
    
    def _evaluate_harmony(self, chroma):
        """评估和声"""
        # 检测常见和弦进行
        # 简化：检测是否有明确的调性中心
        
        # 计算每个音级的平均能量
        pitch_classes = np.mean(chroma, axis=1)
        
        # 调性中心应明显
        max_pitch = np.max(pitch_classes)
        min_pitch = np.min(pitch_classes)
        
        contrast = (max_pitch - min_pitch) / (max_pitch + 1e-8)
        
        return contrast
    
    def _evaluate_style_consistency(self, audio):
        """评估风格一致性"""
        
        # 将音频分段
        segments = self._segment_audio(audio, num_segments=4)
        
        # 提取每段的特征
        features = []
        for seg in segments:
            feat = self._extract_style_features(seg)
            features.append(feat)
        
        # 计算段间相似度
        similarities = []
        for i in range(len(features)):
            for j in range(i+1, len(features)):
                sim = np.dot(features[i], features[j])
                similarities.append(sim)
        
        # 一致性 = 平均相似度
        consistency = np.mean(similarities) if similarities else 0
        
        return consistency
    
    def _evaluate_diversity(self, audio):
        """评估多样性（避免过于重复）"""
        
        # 检测旋律变化
        pitches = self._extract_melody(audio)
        
        # 计算音高变化率
        pitch_changes = np.diff(pitches)
        change_rate = np.sum(pitch_changes != 0) / len(pitch_changes)
        
        # 理想的change_rate在0.3-0.7之间
        if change_rate < 0.1:
            return 0.2  # 过于重复
        elif change_rate > 0.9:
            return 0.3  # 过于混乱
        else:
            return 1.0 - abs(change_rate - 0.5) * 2
    
    def _evaluate_text_alignment(self, audio, prompt):
        """评估与文本的对齐度"""
        # 使用CLAP（Contrastive Language-Audio Pretraining）
        
        try:
            from transformers import ClapProcessor, ClapModel
            
            processor = ClapProcessor.from_pretrained("laion/clap-htsat-unfused")
            model = ClapModel.from_pretrained("laion/clap-htsat-unfused")
            
            # 处理音频
            inputs = processor(
                text=[prompt],
                audios=[audio],
                return_tensors="pt",
                sampling_rate=48000
            )
            
            outputs = model(**inputs)
            logits_per_audio = outputs.logits_per_audio
            
            # 归一化到0-1
            score = torch.sigmoid(logits_per_audio / 10).item()
            
            return score
        except:
            return 0.5  # 默认中等分数
    
    def _calculate_similarity(self, audio_a, audio_b):
        """计算两段音频的相似度"""
        
        # 提取特征
        feat_a = self._extract_style_features(audio_a)
        feat_b = self._extract_style_features(audio_b)
        
        # 余弦相似度
        similarity = np.dot(feat_a, feat_b) / (
            np.linalg.norm(feat_a) * np.linalg.norm(feat_b)
        )
        
        return similarity
    
    def _calculate_overall(self, metrics):
        """计算综合评分"""
        weights = {
            'audio_quality': 0.20,
            'music_theory': 0.25,
            'style_consistency': 0.20,
            'diversity': 0.15,
            'text_alignment': 0.20
        }
        
        overall = sum(
            metrics.get(k, 0) * w
            for k, w in weights.items()
        )
        
        return overall
    
    def _extract_style_features(self, audio):
        """提取风格特征"""
        # 使用频谱特征、节奏特征等
        mfcc = librosa.feature.mfcc(y=audio, sr=44100, n_mfcc=13)
        spectral = librosa.feature.spectral_centroid(y=audio, sr=44100)
        
        features = np.concatenate([
            np.mean(mfcc, axis=1),
            np.std(mfcc, axis=1),
            np.mean(spectral),
            np.std(spectral)
        ])
        
        return features
    
    def _segment_audio(self, audio, num_segments=4):
        """分段音频"""
        seg_length = len(audio) // num_segments
        return [
            audio[i*seg_length:(i+1)*seg_length]
            for i in range(num_segments)
        ]
    
    def _extract_melody(self, audio):
        """提取旋律"""
        # 使用音高追踪
        pitches, _ = librosa.piptrack(y=audio, sr=44100)
        # 简化：取每个帧的最大音高
        melody = np.max(pitches, axis=0)
        return melody

# 使用
evaluator = MusicQualityEvaluator()
metrics = evaluator.evaluate(
    audio=generated_audio,
    prompt="upbeat pop song about summer"
)

print(f"Overall: {metrics['overall']:.3f}")
for k, v in metrics.items():
    if k != 'overall':
        print(f"  {k}: {v:.3f}")
```

### 2. 提示词优化策略

```python
# ============================================
# 音乐提示词自动优化
# ============================================

class MusicPromptOptimizer:
    """音乐提示词优化器"""
    
    def __init__(self):
        self.style_keywords = [
            "pop", "rock", "jazz", "classical", "electronic",
            "hip-hop", "country", "folk", "blues", "metal"
        ]
        
        self.mood_keywords = [
            "happy", "sad", "energetic", "calm", "romantic",
            "melancholic", "upbeat", "dark", "bright", "nostalgic"
        ]
        
        self.instrument_keywords = [
            "piano", "guitar", "drums", "bass", "violin",
            "saxophone", "synthesizer", "orchestra", "strings"
        ]
    
    def optimize(self, base_prompt, target_duration=120):
        """优化音乐提示词"""
        
        enhanced = base_prompt
        
        # 1. 添加结构信息（如果缺少）
        if not any(word in enhanced.lower() for word in ["intro", "verse", "chorus"]):
            enhanced += ", with clear song structure"
        
        # 2. 添加速度信息（如果缺少）
        if "bpm" not in enhanced.lower():
            # 根据情绪推断速度
            if any(word in enhanced.lower() for word in ["fast", "energetic", "upbeat"]):
                enhanced += ", 130-150 BPM"
            elif any(word in enhanced.lower() for word in ["slow", "calm", "relaxing"]):
                enhanced += ", 60-80 BPM"
            else:
                enhanced += ", 100-120 BPM"
        
        # 3. 添加调性信息（可选）
        if not any(key in enhanced.lower() for key in ["major", "minor"]):
            if any(word in enhanced.lower() for word in ["happy", "bright", "cheerful"]):
                enhanced += ", major key"
            elif any(word in enhanced.lower() for word in ["sad", "dark", "melancholic"]):
                enhanced += ", minor key"
        
        # 4. 添加时长信息
        if "minute" not in enhanced.lower() and "second" not in enhanced.lower():
            minutes = target_duration // 60
            enhanced += f", {minutes} minutes long"
        
        # 5. 添加质量词
        if not any(word in enhanced.lower() for word in ["high quality", "professional"]):
            enhanced += ", high quality, professional production"
        
        return enhanced
    
    def generate_variants(self, prompt, num_variants=5):
        """生成提示词变体用于A/B测试"""
        
        variants = []
        
        # 变体1：强调不同乐器
        for instrument in ["piano", "guitar", "synthesizer"]:
            if instrument not in prompt.lower():
                variant = prompt + f", featuring {instrument}"
                variants.append(variant)
                if len(variants) >= num_variants:
                    break
        
        # 变体2：不同情绪强度
        if "very" not in prompt.lower():
            variants.append(prompt.replace("happy", "very happy"))
        
        # 变体3：简化版
        words = prompt.split(",")
        simplified = ",".join(words[:3])
        variants.append(simplified)
        
        return variants[:num_variants]
    
    def analyze_prompt_quality(self, prompt):
        """分析提示词质量"""
        
        score = 0
        checks = []
        
        # 检查风格词
        has_style = any(kw in prompt.lower() for kw in self.style_keywords)
        checks.append(("Style specified", has_style))
        if has_style:
            score += 20
        
        # 检查情绪词
        has_mood = any(kw in prompt.lower() for kw in self.mood_keywords)
        checks.append(("Mood specified", has_mood))
        if has_mood:
            score += 20
        
        # 检查乐器
        has_instrument = any(kw in prompt.lower() for kw in self.instrument_keywords)
        checks.append(("Instruments specified", has_instrument))
        if has_instrument:
            score += 15
        
        # 检查速度
        has_tempo = "bpm" in prompt.lower() or any(word in prompt.lower() for word in ["fast", "slow", "tempo"])
        checks.append(("Tempo specified", has_tempo))
        if has_tempo:
            score += 15
        
        # 检查时长
        has_duration = any(word in prompt.lower() for word in ["minute", "second", "long", "short"])
        checks.append(("Duration specified", has_duration))
        if has_duration:
            score += 10
        
        # 检查详细程度
        word_count = len(prompt.split())
        if word_count > 10:
            score += 10
            checks.append(("Detailed description", True))
        
        # 检查质量词
        has_quality = any(word in prompt.lower() for word in ["quality", "professional", "master"])
        checks.append(("Quality specified", has_quality))
        if has_quality:
            score += 10
        
        return {
            "score": score,
            "checks": checks,
            "recommendations": self._get_recommendations(checks)
        }
    
    def _get_recommendations(self, checks):
        """获取改进建议"""
        recommendations = []
        
        for check_name, passed in checks:
            if not passed:
                if check_name == "Style specified":
                    recommendations.append("Add music style (pop, rock, jazz, etc.)")
                elif check_name == "Mood specified":
                    recommendations.append("Add mood description (happy, sad, energetic, etc.)")
                elif check_name == "Instruments specified":
                    recommendations.append("Specify instruments")
                elif check_name == "Tempo specified":
                    recommendations.append("Add tempo (BPM or fast/slow)")
                elif check_name == "Duration specified":
                    recommendations.append("Specify desired length")
        
        return recommendations

# 使用
optimizer = MusicPromptOptimizer()

base_prompt = "A song about summer"
enhanced = optimizer.optimize(base_prompt, target_duration=180)
print(f"Enhanced: {enhanced}")

# 质量分析
analysis = optimizer.analyze_prompt_quality(enhanced)
print(f"Quality Score: {analysis['score']}/100")
print("Recommendations:")
for rec in analysis['recommendations']:
    print(f"  - {rec}")
```

---

## 商业应用案例

### 1. 音乐授权平台自动化

```markdown
## 音乐授权自动化系统

系统架构：
```
┌─────────────────────────────────────────────┐
│           音乐授权自动化平台                   │
├─────────────────────────────────────────────┤
│ 需求分析                                     │
│ ├── 项目类型（广告/影视/游戏）               │
│ ├── 情绪需求（欢快/悲伤/紧张）               │
│ ├── 时长要求                                 │
│ ├── 预算范围                                 │
│ └── 授权范围（地域/时长/平台）               │
├─────────────────────────────────────────────┤
│ 音乐生成                                     │
│ ├── AI生成候选曲目（10-20首）                │
│ ├── 风格匹配筛选                             │
│ └── 质量评分排序                             │
├─────────────────────────────────────────────┤
│ 版权管理                                     │
│ ├── 自动生成版权声明                         │
│ ├── 授权协议生成                             │
│ └── 使用追踪                                 │
├─────────────────────────────────────────────┤
│ 交付                                         │
│ ├── 多种格式（WAV/MP3/STEMS）                │
│ ├── 元数据嵌入                               │
│ └── 发票和税务                               │
└─────────────────────────────────────────────┘
```

商业模式：
- 生成成本：$0.10-0.50/首
- 授权价格：$10-1000/首（根据用途）
- 毛利率：90%+
- 优势：零库存、无限供应、快速交付

合规要点：
1. 训练数据授权证明
2. 生成内容版权声明
3. 平台商用授权条款
4. 艺术家署名（如使用风格参考）
```

### 2. 个性化音乐推荐与生成

```python
# ============================================
# 个性化音乐生成系统
# ============================================

class PersonalizedMusicSystem:
    """个性化音乐系统"""
    
    def __init__(self):
        self.user_profiles = {}
        self.listening_history = {}
    
    def create_user_profile(self, user_id, preferences):
        """
        创建用户音乐画像
        
        Args:
            preferences: {
                "favorite_genres": ["pop", "rock"],
                "favorite_artists": ["Taylor Swift", "Ed Sheeran"],
                "mood_preferences": {"workout": "energetic", "sleep": "calm"},
                "tempo_range": [100, 140],
                "complexity": "medium"
            }
        """
        
        profile = {
            "user_id": user_id,
            "preferences": preferences,
            "listening_patterns": {},
            "generated_history": []
        }
        
        self.user_profiles[user_id] = profile
        return profile
    
    def generate_personalized_track(self, user_id, context):
        """
        为特定用户和场景生成音乐
        
        Args:
            context: {
                "activity": "workout",
                "time_of_day": "morning",
                "weather": "sunny",
                "heart_rate": 120
            }
        """
        
        profile = self.user_profiles.get(user_id)
        if not profile:
            raise ValueError("User profile not found")
        
        # 1. 分析上下文
        mood = self._infer_mood_from_context(context)
        
        # 2. 结合用户偏好
        preferred_genres = profile["preferences"]["favorite_genres"]
        preferred_tempo = profile["preferences"]["tempo_range"]
        
        # 3. 动态调整参数
        if context.get("heart_rate"):
            # 根据心率调整速度
            target_tempo = self._tempo_from_heart_rate(context["heart_rate"])
        else:
            target_tempo = np.random.uniform(*preferred_tempo)
        
        # 4. 生成提示词
        prompt = self._build_prompt(
            genre=np.random.choice(preferred_genres),
            mood=mood,
            tempo=target_tempo,
            complexity=profile["preferences"]["complexity"]
        )
        
        # 5. 生成音乐
        music = generate_music(prompt, duration=180)
        
        # 6. 记录
        profile["generated_history"].append({
            "timestamp": datetime.now(),
            "context": context,
            "prompt": prompt
        })
        
        return music
    
    def _infer_mood_from_context(self, context):
        """从上下文推断情绪"""
        
        activity_moods = {
            "workout": "energetic",
            "sleep": "calm",
            "study": "focused",
            "commute": "upbeat",
            "meditation": "peaceful"
        }
        
        return activity_moods.get(context.get("activity"), "neutral")
    
    def _tempo_from_heart_rate(self, heart_rate):
        """根据心率推断适合的速度"""
        # 简单映射：心率/2 ≈ BPM
        return heart_rate / 2
    
    def _build_prompt(self, genre, mood, tempo, complexity):
        """构建生成提示词"""
        
        complexity_desc = {
            "low": "simple, minimal arrangement",
            "medium": "moderate complexity, balanced",
            "high": "complex, intricate arrangement, rich harmonies"
        }
        
        prompt = f"""
        {genre} music, {mood} mood,
        {tempo:.0f} BPM,
        {complexity_desc.get(complexity, 'balanced')},
        high quality production
        """
        
        return prompt.strip()
    
    def feedback_loop(self, user_id, track_id, rating):
        """
        用户反馈优化
        
        Args:
            rating: 1-5星评分
        """
        
        profile = self.user_profiles[user_id]
        
        # 记录反馈
        if "feedback" not in profile:
            profile["feedback"] = []
        
        profile["feedback"].append({
            "track_id": track_id,
            "rating": rating,
            "timestamp": datetime.now()
        })
        
        # 如果评分低，调整生成策略
        if rating < 3:
            self._adjust_preferences(user_id, track_id)
    
    def _adjust_preferences(self, user_id, track_id):
        """根据反馈调整偏好"""
        # 分析低分曲目的特征，避免未来生成类似音乐
        pass

# 使用
system = PersonalizedMusicSystem()

# 创建用户画像
system.create_user_profile(
    user_id="user123",
    preferences={
        "favorite_genres": ["pop", "electronic"],
        "favorite_artists": ["Dua Lipa", "The Weeknd"],
        "mood_preferences": {"workout": "energetic"},
        "tempo_range": [120, 140],
        "complexity": "medium"
    }
)

# 生成个性化音乐
music = system.generate_personalized_track(
    user_id="user123",
    context={
        "activity": "workout",
        "time_of_day": "morning",
        "heart_rate": 135
    }
)
```

---

## 编程专项：自动化音乐生成

### 1. REST API服务

```python
# ============================================
# 音乐生成API服务
# ============================================

from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel
from typing import Optional, List
import uuid

app = FastAPI(title="AI音乐生成API")

class MusicRequest(BaseModel):
    prompt: str
    duration: Optional[int] = 30
    style: Optional[str] = "pop"
    mood: Optional[str] = "happy"
    tempo: Optional[int] = None
    instruments: Optional[List[str]] = None
    num_variants: Optional[int] = 1
    callback_url: Optional[str] = None

class LyricsRequest(BaseModel):
    lyrics: str
    style: Optional[str] = "pop"
    singer_gender: Optional[str] = "female"
    language: Optional[str] = "english"

task_status = {}

@app.post("/generate")
async def generate_music(request: MusicRequest, background_tasks: BackgroundTasks):
    """提交音乐生成任务"""
    
    task_id = str(uuid.uuid4())
    
    task_status[task_id] = {
        "status": "queued",
        "progress": 0,
        "result_url": None
    }
    
    # 后台处理
    background_tasks.add_task(process_music_task, task_id, request)
    
    return {
        "task_id": task_id,
        "status": "queued",
        "estimated_time": request.duration * 2  # 粗略估计
    }

async def process_music_task(task_id: str, request: MusicRequest):
    """处理音乐生成任务"""
    
    try:
        task_status[task_id]["status"] = "processing"
        
        # 1. 优化提示词
        optimized_prompt = optimize_music_prompt(request)
        task_status[task_id]["progress"] = 20
        
        # 2. 生成音乐
        audio = generate_music(
            prompt=optimized_prompt,
            duration=request.duration
        )
        task_status[task_id]["progress"] = 80
        
        # 3. 后处理（标准化、格式转换）
        processed = post_process_audio(audio)
        task_status[task_id]["progress"] = 100
        
        # 4. 保存并生成URL
        filename = f"{task_id}.wav"
        save_audio(processed, f"./outputs/{filename}")
        
        task_status[task_id]["status"] = "completed"
        task_status[task_id]["result_url"] = f"/download/{filename}"
        
        # 5. 回调通知
        if request.callback_url:
            await notify_callback(request.callback_url, task_id)
    
    except Exception as e:
        task_status[task_id]["status"] = "failed"
        task_status[task_id]["error"] = str(e)

@app.post("/generate-from-lyrics")
async def generate_from_lyrics(request: LyricsRequest):
    """根据歌词生成歌曲"""
    
    # 1. 解析歌词结构
    structure = parse_lyrics_structure(request.lyrics)
    
    # 2. 生成旋律
    melody = generate_melody_for_lyrics(
        lyrics=request.lyrics,
        style=request.style
    )
    
    # 3. 生成伴奏
    accompaniment = generate_accompaniment(melody, request.style)
    
    # 4. 合成（如果需要AI人声）
    if request.singer_gender:
        vocals = synthesize_vocals(
            request.lyrics,
            melody,
            gender=request.singer_gender,
            language=request.language
        )
        final = mix_audio(vocals, accompaniment)
    else:
        final = accompaniment
    
    # 保存并返回
    task_id = str(uuid.uuid4())
    filename = f"{task_id}.wav"
    save_audio(final, f"./outputs/{filename}")
    
    return {
        "task_id": task_id,
        "status": "completed",
        "download_url": f"/download/{filename}"
    }

@app.get("/status/{task_id}")
async def get_status(task_id: str):
    """查询任务状态"""
    if task_id not in task_status:
        raise HTTPException(status_code=404, detail="Task not found")
    
    return task_status[task_id]

# 运行：uvicorn music_api:app --host 0.0.0.0 --port 8000
```

### 2. 批量音乐生成工作流

```python
# ============================================
# 批量音乐生成工作流
# ============================================

import asyncio
from dataclasses import dataclass
from typing import List, Dict

@dataclass
class MusicTask:
    id: str
    prompt: str
    duration: int
    style: str
    priority: int = 1

class BatchMusicWorkflow:
    """批量音乐生成工作流"""
    
    def __init__(self, max_concurrent=5):
        self.max_concurrent = max_concurrent
        self.queue = asyncio.PriorityQueue()
        self.results = {}
        self.semaphore = asyncio.Semaphore(max_concurrent)
    
    async def submit_tasks(self, tasks: List[MusicTask]):
        """提交批量任务"""
        
        for task in tasks:
            # 优先级队列（数字越小优先级越高）
            await self.queue.put((task.priority, task.id, task))
    
    async def process_all(self):
        """处理所有任务"""
        
        workers = [
            self._worker(f"worker_{i}")
            for i in range(self.max_concurrent)
        ]
        
        await asyncio.gather(*workers)
        
        return self.results
    
    async def _worker(self, worker_id: str):
        """工作进程"""
        
        while True:
            try:
                # 获取任务（非阻塞）
                priority, task_id, task = await asyncio.wait_for(
                    self.queue.get(), timeout=1.0
                )
            except asyncio.TimeoutError:
                break
            
            async with self.semaphore:
                print(f"[{worker_id}] Processing task {task_id}")
                
                try:
                    result = await self._generate_single(task)
                    self.results[task_id] = {
                        "status": "success",
                        "audio": result
                    }
                except Exception as e:
                    self.results[task_id] = {
                        "status": "failed",
                        "error": str(e)
                    }
                
                self.queue.task_done()
    
    async def _generate_single(self, task: MusicTask):
        """生成单个音乐"""
        
        # 在线程池中运行（避免阻塞事件循环）
        loop = asyncio.get_event_loop()
        
        result = await loop.run_in_executor(
            None,
            lambda: generate_music_sync(task.prompt, task.duration)
        )
        
        return result
    
    def generate_report(self):
        """生成报告"""
        total = len(self.results)
        successful = sum(1 for r in self.results.values() if r["status"] == "success")
        failed = total - successful
        
        return {
            "total": total,
            "successful": successful,
            "failed": failed,
            "success_rate": successful / total if total > 0 else 0
        }

def generate_music_sync(prompt, duration):
    """同步生成音乐（在线程中运行）"""
    # 调用实际的音乐生成模型
    pass

# 使用
async def main():
    workflow = BatchMusicWorkflow(max_concurrent=3)
    
    # 创建任务
    tasks = [
        MusicTask(
            id=f"track_{i:03d}",
            prompt=f"Background music for {scene}",
            duration=60,
            style="ambient",
            priority=1
        )
        for i, scene in enumerate([
            "forest", "city", "ocean", "desert", "mountain"
        ])
    ]
    
    # 提交并处理
    await workflow.submit_tasks(tasks)
    results = await workflow.process_all()
    
    # 生成报告
    report = workflow.generate_report()
    print(f"\nBatch complete: {report['successful']}/{report['total']}")
    
    # 保存所有成功结果
    for task_id, result in results.items():
        if result["status"] == "success":
            save_audio(result["audio"], f"./batch_output/{task_id}.wav")

# asyncio.run(main())
```

---

## 常见陷阱与最佳实践

### 1. 音乐生成陷阱

```markdown
## 常见问题与解决方案

### 问题1：和声不和谐
```
症状：
- 和弦进行奇怪
- 音符之间冲突
- 听感"脏"

原因：
- 模型缺乏音乐理论约束
- 训练数据中不和谐音乐
- 采样温度太高

解决方案：
1. 降低temperature（0.8-1.0）
2. 在提示词中指定调性和和弦
3. 后处理：使用音乐理论规则修正
4. 使用有音乐理论约束的模型

代码示例：
```python
def fix_harmony(audio, key="C major"):
    """使用音乐理论修正和声"""
    
    # 检测当前和弦
    chroma = librosa.feature.chroma_stft(y=audio, sr=44100)
    
    # 定义允许的音符（以C大调为例）
    allowed_notes = [0, 2, 4, 5, 7, 9, 11]  # C, D, E, F, G, A, B
    
    # 抑制不在调内的音符
    for frame in range(chroma.shape[1]):
        for note in range(12):
            if note not in allowed_notes:
                chroma[note, frame] *= 0.1  # 大幅降低音量
    
    # 重建音频（简化）
    return audio  # 实际应使用逆变换
```

### 问题2：结构混乱
```
症状：
- 没有明确的段落
- 副歌和主歌分不清
- 结尾突兀

原因：
- 模型缺乏长期结构规划
- 提示词没有指定结构
- 生成长度太短

解决方案：
1. 使用元标签标记段落
2. 分段生成然后拼接
3. 先生成和弦进行，再生成旋律
4. 使用更长上下文模型

### 问题3：人声不清晰
```
症状：
- 歌词听不清
- 发音模糊
- 与伴奏混在一起

原因：
- 人声合成模型限制
- 混音问题
- 语言支持不足

解决方案：
1. 使用专门的人声合成模型
2. 后期处理：提升人声频段（1-4kHz）
3. 降低伴奏音量
4. 对于中文，选择中文优化模型（Suno V3+）

### 问题4：风格不统一
```
症状：
- 前半段是爵士，后半段变摇滚
- 乐器突然变化
- 情绪不一致

原因：
- 模型在长序列上注意力分散
- 采样随机性积累
- 缺乏风格约束

解决方案：
1. 使用IP-Adapter或风格embedding
2. 固定seed部分生成
3. 分段生成但保持风格embedding一致
4. 使用风格一致性损失训练
```

### 2. 版权与法律风险

```markdown
## AI音乐版权指南

### 训练数据版权
```
争议：
- 使用版权音乐训练是否侵权？
- 2024年多起诉讼（如Sony Music诉AI公司）
- 不同司法管辖区判决不同

现状（2026）：
- 美国：倾向于"合理使用"（transformative）
- 欧盟：AI法案要求透明度
- 中国：需标识AI生成

建议：
1. 使用授权音乐训练
2. 购买训练数据授权
3. 与唱片公司合作
4. 使用开源/免版税数据
```

### 生成内容版权
```
问题：
- AI生成音乐受版权保护吗？
- 谁拥有生成音乐的版权？

现状：
- 美国版权局：纯AI生成不受保护
- 需人类创造性贡献
- 建议：人工修改和编曲

商用策略：
1. 使用平台商用授权（Suno Pro等）
2. 保留生成记录
3. 添加人工创意元素
4. 注册版权时声明AI使用
```

### 3. 技术最佳实践

```markdown
## 生产环境最佳实践

### 1. 质量门禁
```python
def music_quality_gate(audio, prompt, thresholds=None):
    """音乐质量门禁"""
    
    thresholds = thresholds or {
        "min_clap_score": 0.6,
        "max_dissonance": 0.3,
        "min_structure_clarity": 0.5
    }
    
    evaluator = MusicQualityEvaluator()
    metrics = evaluator.evaluate(audio, prompt=prompt)
    
    checks = [
        metrics.get("text_alignment", 0) > thresholds["min_clap_score"],
        metrics.get("music_theory", 0) > thresholds["max_dissonance"],
        metrics.get("style_consistency", 0) > thresholds["min_structure_clarity"]
    ]
    
    return all(checks), metrics
```

### 2. A/B测试
```python
def ab_test_music(prompt_a, prompt_b, num_samples=50):
    """对比两组提示词"""
    
    results_a = []
    results_b = []
    
    for _ in range(num_samples):
        # 相同seed保证公平
        seed = random.randint(0, 1000000)
        
        audio_a = generate_music(prompt_a, seed=seed)
        audio_b = generate_music(prompt_b, seed=seed)
        
        score_a = evaluate_music(audio_a, prompt_a)
        score_b = evaluate_music(audio_b, prompt_b)
        
        results_a.append(score_a)
        results_b.append(score_b)
    
    # 统计检验
    from scipy import stats
    t_stat, p_value = stats.ttest_ind(results_a, results_b)
    
    return {
        "prompt_a_mean": np.mean(results_a),
        "prompt_b_mean": np.mean(results_b),
        "p_value": p_value,
        "winner": "A" if np.mean(results_a) > np.mean(results_b) else "B"
    }
```

### 3. 成本控制
```markdown
成本优化策略：

1. 分层生成
   - 第一层：快速预览（10秒，低质量）
   - 第二层：完整生成（确认后）
   - 节约：80%的草稿不需要完整生成

2. 缓存常用风格
   - 预生成常用风格embedding
   - 相似请求复用

3. 模型选择
   - 简单任务：MusicGen（快）
   - 复杂任务：Suno/Udio（质量好）

4. 批量折扣
   - 批量API调用
   - 非高峰时段生成
```

### 4. 监控与告警
```python
# 监控指标
metrics = {
    "generation_latency": [],
    "gpu_memory_usage": [],
    "queue_length": 0,
    "success_rate": 0.0,
    "avg_clap_score": 0.0,
    "user_satisfaction": 0.0
}

# 告警规则
alerts = [
    {
        "name": "low_quality",
        "condition": "avg_clap_score < 0.5",
        "action": "notify_team"
    },
    {
        "name": "high_failure_rate",
        "condition": "failure_rate > 10%",
        "action": "page_oncall"
    }
]
```
```

---

## 面试题与参考答案

### 基础问题

**Q1: 音频Token化相比原始波形有哪些优势？**

```markdown
**答案要点：**

1. 数据压缩
   - 原始波形：44.1kHz × 16bit = 705.6kbps
   - 音频Token：~6kbps（EnCodec）
   - 压缩比：~100:1

2. 建模效率
   - Token序列可用语言模型（Transformer）处理
   - 注意力机制有效建模长程依赖
   - 训练速度大幅提升

3. 质量保留
   - VAE编码解码保留大部分信息
   - 感知质量接近无损
   - 支持44.1kHz/48kHz重建

4. 多尺度表示
   - 多层量化（4层）
   - 粗到细粒度
   - 可灵活使用不同层数

劣势：
- 需要预训练编解码器
- 重建可能丢失细微细节
- 增加了系统复杂度
```

**Q2: 解释音频语言模型与文本语言模型的区别**

```markdown
**答案要点：**

相似点：
- 都使用Transformer架构
- 都使用自回归生成
- 都使用Embedding层

差异点：

1. 输入表示
   - 文本：离散Token（词汇表~50K）
   - 音频：多层离散Token（每层~1K codebook）

2. 序列长度
   - 文本：1K-4K tokens
   - 音频：30秒 = 1500 tokens（每层）
   - 音频有4层，同时处理

3. 条件注入
   - 文本：主要文本条件
   - 音频：文本+旋律+风格多条件

4. 生成目标
   - 文本：语义正确、连贯
   - 音频：旋律优美、和声协调、时序一致

5. 评估方式
   - 文本：BLEU, ROUGE, 人工评分
   - 音频：CLAP Score, FAD, 音乐理论合规性
```

**Q3: 如何实现歌词与旋律的对齐？**

```markdown
**答案要点：**

技术方案：

1. 基于规则的对齐
   - 音节数匹配音符数
   - 重音位置对齐强拍
   - 适合简单歌曲

2. 基于学习的对齐
   - 训练对齐模型（Lyrics-Melody Alignment）
   - 使用DTW（动态时间规整）
   - 端到端学习

3. 基于表示的对齐
   - 歌词和旋律分别编码
   - 在Latent空间对齐
   - 使用对比学习

具体实现：
```python
# 1. 歌词编码
lyric_tokens = tokenizer(lyrics)  # [L]
lyric_emb = text_encoder(lyric_tokens)  # [L, D]

# 2. 旋律编码
melody_tokens = audio_encoder(melody)  # [T, K]
melody_emb = audio_encoder(melody_tokens)  # [T, D]

# 3. 计算相似度矩阵
similarity = lyric_emb @ melody_emb.T  # [L, T]

# 4. DTW找最优对齐路径
path = dtw_alignment(similarity)  # [(l1, t1), (l2, t2), ...]

# 5. 应用对齐
aligned_melody = apply_alignment(melody, path)
```

挑战：
- 多音字处理
- 速度变化（ritardando, accelerando）
- 装饰音和转音
```

### 进阶问题

**Q4: 比较Suno/Udio的模型架构与MusicGen的差异**

```markdown
**答案要点：**

MusicGen（Meta，开源）：
- 架构：基于EnCodec的音频语言模型
- 文本编码：T5
- 训练数据：20万首授权音乐
- 特点：高效、轻量、可本地部署
- 局限：无人声、较短（30秒）

Suno/Udio（商业，闭源）：
推测架构：
- 音频表示：类似EnCodec但可能自研
- 文本理解：可能使用GPT-4级别模型
- 人声合成：专门的TTS/歌声合成
- 多轨生成：独立生成人声+伴奏
- 特点：完整歌曲、高质量人声、长时长

关键差异：
1. 人声处理
   - MusicGen：无
   - Suno/Udio：专门的人声合成管线

2. 歌词理解
   - MusicGen：纯文本→音频
   - Suno/Udio：歌词结构→旋律→人声

3. 歌曲结构
   - MusicGen：无显式结构
   - Suno/Udio：Intro-Verse-Chorus-Bridge-Outro

4. 后处理
   - MusicGen：简单
   - Suno/Udio：专业混音母带
```

**Q5: 如何评估AI生成音乐的质量？**

```markdown
**答案要点：**

自动评估指标：

1. 音频质量
   - 动态范围
   - 频谱平衡
   - 失真检测

2. 音乐理论合规性
   - 调性稳定性
   - 节拍稳定性
   - 和声进行合理性

3. 风格一致性
   - 段内相似度
   - 整体风格统一

4. 文本对齐度
   - CLAP Score（音频-文本相似度）
   - 情绪匹配度

5. 多样性
   - 旋律变化率
   - 避免过度重复

6. FAD（Fréchet Audio Distance）
   - 类似图像的FID
   - 与参考音乐分布的距离

人工评估维度：
- 整体质量（1-5）
- 旋律吸引力
- 和声舒适度
- 节奏感
- 创新性
- 与提示词匹配度

评估挑战：
- 主观性强
- 不同风格标准不同
- 缺乏统一benchmark
```

**Q6: 设计一个实时游戏音乐系统**

```markdown
**答案要点：**

系统架构：

```
┌─────────────────────────────────────────────┐
│           实时游戏音乐系统                     │
├─────────────────────────────────────────────┤
│ 输入层：游戏状态                              │
│ ├── 玩家位置/场景                             │
│ ├── 敌人数量/威胁等级                         │
│ ├── 玩家生命值                                │
│ ├── 任务状态                                  │
│ └── 时间/天气                                 │
├─────────────────────────────────────────────┤
│ 情绪推断引擎                                  │
│ ├── 规则系统（if-else）                       │
│ ├── 机器学习模型                              │
│ └── 输出：情绪向量（紧张/平静/兴奋等）         │
├─────────────────────────────────────────────┤
│ 音乐选择/生成                                 │
│ ├── 预生成音乐库（按情绪分类）                │
│ ├── AI实时生成（轻量模型）                    │
│ └── 风格迁移（保持品牌一致性）                │
├─────────────────────────────────────────────┤
│ 过渡与混合                                    │
│ ├── 交叉淡化（Crossfade）                     │
│ ├── 节奏匹配（Beat Matching）                 │
│ ├── 调性匹配（Harmonic Mixing）               │
│ └── 层次混合（Layered Mixing）                │
├─────────────────────────────────────────────┤
│ 输出层：音频流                                │
│ ├── 低延迟播放（<50ms）                      │
│ ├── 多声道支持                                │
│ └── 动态范围调整                              │
└─────────────────────────────────────────────┘
```

关键技术：

1. 分层音乐系统
   - 基础层：氛围（持续播放）
   - 节奏层：鼓/贝斯（根据紧张度）
   - 旋律层：主题旋律（根据场景）
   - 效果层：过渡音效

2. 实时过渡
   ```python
   def transition_to(new_mood, transition_time=2.0):
       # 计算当前播放位置
       current_beat = get_current_beat()
       
       # 找到下一小节的开始
       next_bar = math.ceil(current_beat / 4) * 4
       
       # 在下一小节开始过渡
       schedule_at_beat(next_bar, lambda: 
           crossfade(current_track, new_track, transition_time)
       )
   ```

3. 性能优化
   - 预加载所有音乐片段
   - 使用音频中间件（Wwise, FMOD）
   - 流式加载长音乐
   - CPU预算：音乐 < 5%

4. AI实时生成（未来）
   - 使用轻量级模型（MusicGen-small）
   - 预生成候选片段
   - 实时风格迁移
   - 目标延迟：<100ms
```

**Q7: 如何防止AI音乐侵权？**

```markdown
**答案要点：**

技术方案：

1. 训练数据过滤
   - 版权检测：匹配已知版权音乐
   - 相似度阈值：<30%视为安全
   - 使用CC0/授权音乐训练

2. 生成内容过滤
   - 与版权库比对
   - 检测特定旋律片段
   - 阻止生成知名歌曲

3. 风格距离控制
   - 限制与特定艺术家的相似度
   - 使用风格embedding距离
   - 避免精确模仿

4. 水印技术
   - 频域水印（不可听）
   - 追溯生成来源
   - C2PA标准

5. 法律合规
   - 训练数据授权协议
   - 生成内容免责声明
   - 用户协议明确权利
   - 版权保险

检测技术：
```python
def detect_infringement(generated_audio, copyright_database):
    """检测侵权"""
    
    # 提取特征
    gen_features = extract_music_features(generated_audio)
    
    # 与版权库比对
    for copyrighted_song in copyright_database:
        ref_features = copyrighted_song.features
        
        # 计算相似度
        similarity = compute_similarity(gen_features, ref_features)
        
        if similarity > 0.7:  # 阈值
            return {
                "infringement": True,
                "matched_song": copyrighted_song.title,
                "similarity": similarity
            }
    
    return {"infringement": False}
```
```

**Q8: 编写代码实现音乐的风格迁移**

```python
"""
**答案代码：**
"""

import torch
import torch.nn as nn
import librosa
import numpy as np

class MusicStyleTransfer:
    """音乐风格迁移"""
    
    def __init__(self, content_model, style_model):
        self.content_encoder = content_model.encoder
        self.style_encoder = style_model.encoder
        self.decoder = content_model.decoder
        
        # 风格特征统计
        self.style_stats = {}
    
    def extract_style_statistics(self, style_audio):
        """
        提取风格音频的统计特征
        
        Returns:
            mean, std, gram矩阵
        """
        
        # 编码到Latent空间
        tokens = self.style_encoder.encode(style_audio)
        
        # 计算统计量
        mean = torch.mean(tokens, dim=-1, keepdim=True)
        std = torch.std(tokens, dim=-1, keepdim=True)
        
        # Gram矩阵（捕捉风格纹理）
        tokens_flat = tokens.view(tokens.size(0), -1)
        gram = torch.mm(tokens_flat, tokens_flat.t())
        
        return {"mean": mean, "std": std, "gram": gram}
    
    def transfer(self, content_audio, style_audio, alpha=0.5):
        """
        风格迁移
        
        Args:
            content_audio: 内容音频（保留旋律）
            style_audio: 风格音频（提取风格）
            alpha: 风格强度（0-1）
        """
        
        # 1. 编码内容
        content_tokens = self.content_encoder.encode(content_audio)
        
        # 2. 提取风格统计
        style_stats = self.extract_style_statistics(style_audio)
        
        # 3. 内容音频的统计
        content_mean = torch.mean(content_tokens, dim=-1, keepdim=True)
        content_std = torch.std(content_tokens, dim=-1, keepdim=True)
        
        # 4. 风格化：自适应实例归一化（AdaIN）
        # 将内容的均值和方差调整为风格的
        normalized = (content_tokens - content_mean) / (content_std + 1e-8)
        stylized = normalized * style_stats["std"] + style_stats["mean"]
        
        # 5. 混合内容和风格
        mixed = alpha * stylized + (1 - alpha) * content_tokens
        
        # 6. 解码
        output = self.decoder.decode(mixed)
        
        return output
    
    def transfer_melody_only(self, content_audio, style_audio):
        """
        只迁移音色和伴奏风格，保持旋律
        
        使用源分离技术
        """
        
        # 1. 分离内容的旋律和伴奏
        content_melody = self._extract_melody(content_audio)
        content_accompaniment = content_audio - content_melody
        
        # 2. 分离风格的伴奏
        style_accompaniment = self._extract_accompaniment(style_audio)
        
        # 3. 迁移风格的伴奏特征
        new_accompaniment = self.transfer(
            content_accompaniment,
            style_accompaniment
        )
        
        # 4. 重新混合
        result = content_melody + new_accompaniment * 0.7
        
        return result
    
    def _extract_melody(self, audio):
        """提取旋律"""
        # 使用音高追踪
        pitches, magnitudes = librosa.piptrack(y=audio, sr=44100)
        
        # 构建旋律音频（简化）
        melody = np.zeros_like(audio)
        
        return melody
    
    def _extract_accompaniment(self, audio):
        """提取伴奏"""
        # 使用源分离（如Spleeter）
        pass

# 使用
transfer = MusicStyleTransfer(content_model, style_model)

# 将流行歌曲转为爵士风格
jazz_version = transfer.transfer(
    content_audio=pop_song,
    style_audio=jazz_reference,
    alpha=0.7
)

# 只改变伴奏风格
new_arrangement = transfer.transfer_melody_only(
    content_audio=original_song,
    style_audio=orchestral_reference
)
```

---

*此文原创，转载请注明出处。*
