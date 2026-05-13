# Cursor IDE深度解析：AI原生编程工作流的工程化实践

**文章标签：** #ai #cursor #ide #ai编程 #开发效率 #composer #agent模式 #2026

## 目录

- [引言：Cursor IDE的本质](#引言cursor-ide的本质)
- [理论基础：为什么Cursor能重新定义编程工作流](#理论基础为什么cursor能重新定义编程工作流)
- [来龙去脉：AI辅助IDE的发展史](#来龙去脉ai辅助ide的发展史)
- [核心架构深度解析](#核心架构深度解析)
- [模型差异：不同AI模型在Cursor中的表现](#模型差异不同ai模型在cursor中的表现)
- [工业级实践案例](#工业级实践案例)
- [高级技术：Agent模式与Multi-Agent编排](#高级技术agent模式与multi-agent编排)
- [评估与优化体系](#评估与优化体系)
- [生活日用案例](#生活日用案例)
- [编程专项深度实践](#编程专项深度实践)
- [跨行业应用场景](#跨行业应用场景)
- [面试题与参考答案](#面试题与参考答案)

---

## 引言：Cursor IDE的本质

Cursor不是"带AI功能的编辑器"，而是一个**AI原生的软件工程环境**。它重新定义了代码编辑的人机交互范式。

核心认知：

```
传统IDE的本质：文本编辑器 + 编译器/解释器 + 调试器

Cursor的本质：上下文管理引擎 + 意图理解系统 + 代码生成管道
                      ↓
            人类意图 ←→ AI执行 ←→ 代码产物
                      ↓
              双向反馈循环（实时同步）

质量差异的根源：
- 差的AI工具：单次调用，上下文丢失，输出随机
- 好的AI工具（Cursor）：持续上下文，意图对齐，可验证输出
```

**关键洞察**：Cursor的效果不取决于"AI有多强"，而取决于**上下文工程**是否匹配软件开发的实际工作流。

---

## 理论基础：为什么Cursor能重新定义编程工作流

### 1. 上下文窗口工程与代码理解

#### 注意力机制在代码中的应用

```python
# 代码理解的注意力机制
# Cursor通过多层注意力理解代码结构

class CodeAttention:
    """
    代码注意力的核心计算
    """
    
    def compute_code_attention(self, query_token, context_tokens):
        """
        Q: 当前需要补全/理解的token位置
        K: 代码库中所有相关token（变量、函数、类型）
        V: 每个token的语义表示
        """
        # 1. 语法注意力：理解代码结构
        syntax_weights = self.syntax_attention(query_token, context_tokens)
        
        # 2. 语义注意力：理解业务逻辑
        semantic_weights = self.semantic_attention(query_token, context_tokens)
        
        # 3. 依赖注意力：理解模块关系
        dependency_weights = self.dependency_attention(query_token, context_tokens)
        
        # 组合注意力
        combined = (
            0.4 * syntax_weights + 
            0.4 * semantic_weights + 
            0.2 * dependency_weights
        )
        
        return softmax(combined / sqrt(dim)) * values
    
    def syntax_attention(self, query, context):
        """
        基于AST（抽象语法树）的结构注意力
        """
        # 同一作用域的变量获得更高权重
        # 父级函数的参数获得更高权重
        # 导入的模块获得中等权重
        pass
    
    def semantic_attention(self, query, context):
        """
        基于嵌入向量的语义注意力
        """
        # 相似语义的token获得更高权重
        # 通过代码嵌入模型计算
        pass
    
    def dependency_attention(self, query, context):
        """
        基于依赖图的模块注意力
        """
        # 直接依赖的模块获得高权重
        # 间接依赖的模块获得递减权重
        pass
```

**关键理解**：
- Cursor的代码补全不是基于文本统计，而是基于**结构化语义理解**
- 每个token的表示是**代码上下文相关**的（contextualized code embedding）
- 位置编码（Positional Encoding）使得代码结构顺序至关重要

#### 上下文学习（In-Context Learning）在编程中的涌现

```
ICL的编程解释：

预训练数据分布：D = {(code_context, next_token)} ~ P(Context, Token)
Cursor的上下文：{imports, class_def, method_sig, surrounding_lines, related_files}

模型行为：P(next_token | context, project_structure) ≈ P(next_token | task_intent)

即：模型通过代码上下文隐式推断出程序员的意图，然后基于此意图生成代码
```

**工程启示**：
- 项目结构质量 > 单个文件质量（完整的上下文比孤立的代码更重要）
- 命名规范直接影响ICL效果（清晰的命名让模型更容易推断意图）
- 代码顺序影响注意力权重（相关的代码应放在一起）

### 2. 从Copilot到Cursor：交互范式的跃迁

```
三代AI编程工具的演进：

阶段1 - 代码补全（GitHub Copilot 2021）：
- 目标：基于当前行预测下一行
- 交互模式：被动补全（Tab接受）
- 上下文：当前文件 + 相邻标签页
- 特点：单点辅助，无法处理复杂任务

阶段2 - 对话式编程（Cursor 2023-2024）：
- 目标：理解自然语言需求并生成代码
- 交互模式：主动对话（Chat/Composer）
- 上下文：整个项目 + 选中的代码
- 特点：多轮交互，可处理中等复杂度任务

阶段3 - Agent模式（Cursor 3 / 2026）：
- 目标：自主完成端到端开发任务
- 交互模式：目标驱动（Agent自动执行）
- 上下文：完整项目 + 运行时环境 + 外部资源
- 特点：自主规划、执行、验证、修复
```

**关键洞察**：Cursor的Agent模式本质上是**将软件工程流程自动化**，从需求分析到代码实现到测试验证的完整闭环。

### 3. 涌现能力（Emergent Abilities）与工具复杂度

```
AI编程工具的涌现能力：

基础级（代码补全）：
- 语法正确的单行补全
- 简单的函数签名补全
- 对工具极度依赖明确的上下文

进阶级（对话生成）：
- 多行代码块生成
- 基于注释的函数实现
- 能理解跨文件的引用关系

专家级（Composer多文件）：
- 多文件协调编辑
- 架构级别的代码生成
- 能理解设计模式并应用

自主级（Agent模式）：
- 端到端功能实现
- 自动测试和修复
- 运行时错误诊断和修复
- 能处理模糊需求并澄清

超专家级（2026现状）：
- 复杂系统重构
- 跨语言代码迁移
- 性能优化建议和实施
- 安全漏洞自动修复
```

---

## 来龙去脉：AI辅助IDE的发展史

### 第一阶段：传统IDE时代（2010-2020）

IntelliJ IDEA、VS Code、Eclipse统治的时代：

```python
# 2015年的IDE功能
# 智能提示基于静态分析

class TraditionalIDE:
    """
    传统IDE的核心能力
    """
    
    def autocomplete(self, code, position):
        """
        基于语法树的自动补全
        """
        # 1. 解析代码为AST
        ast = self.parser.parse(code)
        
        # 2. 分析当前位置的符号
        symbol = ast.get_symbol_at(position)
        
        # 3. 基于类型系统推断可访问的成员
        if symbol.type == 'object':
            return symbol.type.get_members()
        elif symbol.type == 'module':
            return symbol.module.get_exports()
        
        # 局限性：无法理解语义，只能基于类型
    
    def refactor(self, code, operation):
        """
        基于规则的重构
        """
        # 重命名：全局替换符号引用
        # 提取方法：基于AST切分
        # 局限性：无法理解业务逻辑，可能破坏语义
```

**里程碑**：2015年，JetBrains推出基于机器学习的代码补全，但仍是统计模型。

### 第二阶段：AI代码补全时代（2021-2022）

GitHub Copilot的诞生（2021.6）：

```
Copilot的关键突破：

1. 规模效应：基于OpenAI Codex（12B参数）
2. 上下文理解：不仅基于当前行，还基于整个函数和文件
3. 自然语言到代码：通过注释生成代码
   
   // 计算两个日期之间的天数
   public long daysBetween(Date start, Date end) {
       // Copilot生成：
       long diff = end.getTime() - start.getTime();
       return TimeUnit.DAYS.convert(diff, TimeUnit.MILLISECONDS);
   }

4. 多语言支持：Python、JavaScript、TypeScript、Java等

局限性：
- 单次生成，无法对话
- 无法修改现有代码
- 无法处理跨文件依赖
```

### 第三阶段：AI原生IDE时代（2023-2024）

Cursor的诞生（2023）和快速发展：

```
Cursor的革命性设计：

1. 深度集成：不是插件，是原生功能
   - 基于VS Code构建，保留生态
   - AI功能深度集成到编辑器的每个层面

2. 对话式交互：
   - Chat：侧边栏对话
   - Inline：内联编辑
   - Composer：多文件编辑

3. 上下文工程：
   - @符号引用（文件、文件夹、Git、Web）
   - 自动上下文检测
   - 相关代码智能检索

4. 多模型支持：
   - GPT-4、Claude、Gemini等
   - 不同任务用不同模型
```

### 第四阶段：Agent模式时代（2025-2026）

Cursor 3的Agent模式引发编程范式变革：

```
Agent模式的核心特征：

1. 自主规划：
   用户："添加用户认证功能"
   Agent：
   - 分析现有代码结构
   - 确定需要修改的文件
   - 规划实现步骤
   - 执行并验证

2. 工具使用：
   - 读取文件
   - 修改文件
   - 运行命令（编译、测试）
   - 搜索代码

3. 错误处理：
   - 编译错误自动修复
   - 测试失败自动调试
   - 依赖缺失自动安装

4. 持续学习：
   - 学习项目编码规范
   - 记忆常用模式
   - 适应团队风格
```

### 第五阶段：2026年现状

当前AI原生IDE的工业级特征：

```
2026年AI编程工具的工业标准：

1. 多模态编程：
   - 图片到代码（UI设计稿生成代码）
   - 语音到代码（口述需求生成实现）
   - 视频到代码（演示操作生成脚本）

2. 实时协作：
   - AI作为团队成员参与代码审查
   - 实时建议和冲突解决
   - 知识共享和文档同步

3. 全生命周期覆盖：
   - 需求分析 → 架构设计 → 代码实现 → 测试 → 部署
   - 每个阶段都有AI辅助

4. 安全与合规：
   - 代码安全扫描（SAST）
   - 许可证合规检查
   - 敏感信息检测
```

---

## 核心架构深度解析

### 1. Cursor的系统架构

```
Cursor系统架构图：

┌─────────────────────────────────────────┐
│           用户界面层（UI Layer）           │
│  ┌─────────┐ ┌─────────┐ ┌───────────┐  │
│  │  Editor │ │  Chat   │ │ Composer  │  │
│  │ 编辑器   │ │ 对话面板 │ │ 多文件编辑 │  │
│  └────┬────┘ └────┬────┘ └─────┬─────┘  │
│       └─────────────┴────────────┘        │
│              事件总线（Event Bus）          │
└─────────────────────┬───────────────────┘
                      │
┌─────────────────────▼───────────────────┐
│         上下文管理层（Context Layer）       │
│  ┌─────────────┐    ┌───────────────┐   │
│  │ 项目索引     │    │  代码语义分析   │   │
│  │Project Index│    │Semantic Analysis│  │
│  └──────┬──────┘    └───────┬───────┘   │
│         │                   │            │
│  ┌──────▼──────┐    ┌───────▼───────┐   │
│  │ 符号表      │    │  依赖关系图    │   │
│  │Symbol Table │    │Dependency Graph│   │
│  └──────┬──────┘    └───────┬───────┘   │
│         └─────────┬─────────┘            │
│                   │                      │
│         ┌─────────▼─────────┐            │
│         │   上下文组装引擎    │            │
│         │ Context Assembler │            │
│         └─────────┬─────────┘            │
└─────────────────────┬───────────────────┘
                      │
┌─────────────────────▼───────────────────┐
│          AI引擎层（AI Engine Layer）       │
│  ┌─────────────┐    ┌───────────────┐   │
│  │  模型路由    │    │   提示词工程    │   │
│  │Model Router │    │Prompt Engineer│   │
│  └──────┬──────┘    └───────┬───────┘   │
│         │                   │            │
│  ┌──────▼──────┐    ┌───────▼───────┐   │
│  │ 多模型管理   │    │  上下文压缩    │   │
│  │Multi-Model  │    │Context Compress│  │
│  └──────┬──────┘    └───────┬───────┘   │
│         │                   │            │
│         └─────────┬─────────┘            │
│                   │                      │
│         ┌─────────▼─────────┐            │
│         │   响应处理管道      │            │
│         │Response Pipeline  │            │
│         └───────────────────┘            │
└─────────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────┐
│          执行层（Execution Layer）         │
│  ┌─────────────┐    ┌───────────────┐   │
│  │ 代码生成     │    │   代码应用     │   │
│  │Code Generate│    │  Code Apply   │   │
│  └─────────────┘    └───────────────┘   │
│  ┌─────────────┐    ┌───────────────┐   │
│  │ 测试运行     │    │   错误处理     │   │
│  │ Test Runner │    │ Error Handler │   │
│  └─────────────┘    └───────────────┘   │
└─────────────────────────────────────────┘
```

**工程实践**：理解Cursor的架构有助于优化使用效果。

### 2. 上下文组装引擎

```python
class ContextAssembler:
    """
    Cursor的上下文组装引擎
    负责将项目信息组装成AI可理解的上下文
    """
    
    def __init__(self, project_root):
        self.project_root = project_root
        self.symbol_index = SymbolIndex()
        self.dependency_graph = DependencyGraph()
        self.semantic_analyzer = SemanticAnalyzer()
    
    def assemble_context(self, query, target_file=None):
        """
        组装完整的上下文
        
        策略：
        1. 基础上下文：当前文件内容
        2. 相关上下文：引用的符号定义
        3. 项目上下文：项目结构和配置
        4. 历史上下文：最近的修改记录
        """
        context = Context()
        
        # 1. 基础上下文
        if target_file:
            context.add_file(target_file)
        
        # 2. 相关上下文（基于符号引用）
        referenced_symbols = self.extract_referenced_symbols(query)
        for symbol in referenced_symbols:
            definition = self.symbol_index.find_definition(symbol)
            if definition:
                context.add_related_file(definition.file)
        
        # 3. 项目上下文
        context.add_project_structure(
            self.get_project_structure()
        )
        
        # 4. 历史上下文
        context.add_recent_changes(
            self.get_recent_changes(limit=10)
        )
        
        # 5. 压缩上下文（如果超出token限制）
        if context.token_count > MAX_CONTEXT_TOKENS:
            context = self.compress_context(context)
        
        return context
    
    def extract_referenced_symbols(self, query):
        """
        从查询中提取引用的符号
        """
        # 使用NLP技术识别查询中提到的类、函数、变量
        symbols = []
        
        # 1. 直接匹配（@符号引用）
        direct_refs = re.findall(r'@(\w+)', query)
        symbols.extend(direct_refs)
        
        # 2. 语义匹配（查询意图相关的符号）
        semantic_refs = self.semantic_analyzer.find_related(
            query, self.symbol_index.get_all_symbols()
        )
        symbols.extend(semantic_refs)
        
        return list(set(symbols))  # 去重
    
    def compress_context(self, context):
        """
        压缩上下文以适应token限制
        
        策略：
        1. 保留关键代码，省略实现细节
        2. 使用摘要代替完整文件
        3. 优先保留接口和类型定义
        """
        compressed = Context()
        
        # 保留核心文件完整内容
        for file in context.core_files:
            compressed.add_file(file)
        
        # 相关文件只保留接口和类型
        for file in context.related_files:
            interfaces = self.extract_interfaces(file)
            compressed.add_summary(file.path, interfaces)
        
        return compressed
    
    def extract_interfaces(self, file):
        """
        提取文件的接口定义（类签名、函数签名等）
        """
        ast = self.parser.parse(file.content)
        interfaces = []
        
        for node in ast.get_public_declarations():
            interfaces.append({
                'type': node.type,
                'name': node.name,
                'signature': node.signature,
                'docstring': node.docstring
            })
        
        return interfaces
```

### 3. 模型路由系统

```python
class ModelRouter:
    """
    Cursor的模型路由系统
    根据任务类型选择最合适的AI模型
    """
    
    def __init__(self):
        self.models = {
            'gpt-5.4': {
                'strengths': ['complex_reasoning', 'architecture_design'],
                'context_window': 128000,
                'cost_per_1k': 0.03,
                'latency': 'medium'
            },
            'claude-opus-4.6': {
                'strengths': ['code_quality', 'long_context', 'debugging'],
                'context_window': 200000,
                'cost_per_1k': 0.025,
                'latency': 'medium'
            },
            'gpt-5.4-mini': {
                'strengths': ['speed', 'simple_tasks'],
                'context_window': 128000,
                'cost_per_1k': 0.001,
                'latency': 'fast'
            },
            'cursor-tab': {
                'strengths': ['real_time', 'line_completion'],
                'context_window': 4096,
                'cost_per_1k': 0,
                'latency': 'realtime'
            }
        }
    
    def route(self, task: Task) -> ModelConfig:
        """
        根据任务特征路由到最优模型
        """
        # 1. 分析任务特征
        features = self.analyze_task(task)
        
        # 2. 匹配模型能力
        scores = {}
        for model_name, model_info in self.models.items():
            score = self.calculate_match_score(features, model_info)
            scores[model_name] = score
        
        # 3. 考虑成本约束
        if task.budget_constraint:
            scores = self.apply_budget_constraint(scores, task.budget)
        
        # 4. 考虑延迟约束
        if task.latency_constraint:
            scores = self.apply_latency_constraint(scores, task.max_latency)
        
        # 5. 选择最高分模型
        best_model = max(scores, key=scores.get)
        
        return ModelConfig(
            model=best_model,
            temperature=self.get_optimal_temperature(task),
            max_tokens=self.estimate_required_tokens(task)
        )
    
    def analyze_task(self, task: Task) -> TaskFeatures:
        """
        分析任务特征
        """
        features = TaskFeatures()
        
        # 代码复杂度
        features.code_complexity = self.estimate_complexity(task.code)
        
        # 上下文长度
        features.context_length = len(task.context.split())
        
        # 任务类型
        features.task_type = self.classify_task(task.description)
        
        # 精度要求
        features.precision_required = task.requires_high_precision
        
        return features
    
    def calculate_match_score(self, features: TaskFeatures, model: dict) -> float:
        """
        计算任务与模型的匹配分数
        """
        score = 0.0
        
        # 上下文长度匹配
        if features.context_length < model['context_window'] * 0.8:
            score += 0.3
        
        # 任务类型匹配
        if features.task_type in model['strengths']:
            score += 0.4
        
        # 精度要求匹配
        if features.precision_required and 'code_quality' in model['strengths']:
            score += 0.3
        
        return score
```

### 4. 响应处理管道

```python
class ResponsePipeline:
    """
    Cursor的响应处理管道
    将AI原始输出转换为可用的代码修改
    """
    
    def __init__(self):
        self.parsers = [
            CodeBlockParser(),      # 解析代码块
            DiffParser(),           # 解析diff格式
            MarkdownParser(),       # 解析markdown
            JSONParser()            # 解析JSON
        ]
        self.validators = [
            SyntaxValidator(),      # 语法验证
            SecurityValidator(),    # 安全验证
            StyleValidator()        # 风格验证
        ]
    
    def process(self, raw_response: str, context: Context) -> List[CodeChange]:
        """
        处理AI响应，生成代码变更列表
        """
        # 1. 解析响应
        parsed = self.parse_response(raw_response)
        
        # 2. 提取代码变更
        changes = self.extract_changes(parsed, context)
        
        # 3. 验证变更
        validated_changes = []
        for change in changes:
            validation_result = self.validate_change(change)
            if validation_result.is_valid:
                validated_changes.append(change)
            else:
                # 尝试自动修复
                fixed = self.attempt_fix(change, validation_result.errors)
                if fixed:
                    validated_changes.append(fixed)
        
        # 4. 优化变更（最小化diff）
        optimized = self.optimize_changes(validated_changes)
        
        return optimized
    
    def parse_response(self, response: str) -> ParsedResponse:
        """
        解析AI的原始响应
        """
        # 尝试多种解析器
        for parser in self.parsers:
            try:
                result = parser.parse(response)
                if result.is_valid:
                    return result
            except ParseError:
                continue
        
        # 如果都失败，使用启发式解析
        return self.heuristic_parse(response)
    
    def validate_change(self, change: CodeChange) -> ValidationResult:
        """
        验证代码变更
        """
        errors = []
        
        for validator in self.validators:
            result = validator.validate(change)
            if not result.is_valid:
                errors.extend(result.errors)
        
        return ValidationResult(
            is_valid=len(errors) == 0,
            errors=errors
        )
    
    def attempt_fix(self, change: CodeChange, errors: List[Error]) -> Optional[CodeChange]:
        """
        尝试自动修复代码变更中的错误
        """
        fixed_code = change.new_code
        
        for error in errors:
            if error.type == 'syntax_error':
                fixed_code = self.fix_syntax(fixed_code, error)
            elif error.type == 'security_issue':
                fixed_code = self.fix_security(fixed_code, error)
            elif error.type == 'style_violation':
                fixed_code = self.fix_style(fixed_code, error)
        
        if fixed_code != change.new_code:
            return CodeChange(
                file=change.file,
                old_code=change.old_code,
                new_code=fixed_code
            )
        
        return None
```

---

## 模型差异：不同AI模型在Cursor中的表现

### 1. GPT系列在Cursor中的应用

```markdown
## GPT-5.4（OpenAI）特点：

代码生成能力：⭐⭐⭐⭐⭐
- 架构设计能力强
- 复杂逻辑处理优秀
- 多语言支持好

上下文理解：⭐⭐⭐⭐
- 128K上下文窗口
- 对长文件处理良好
- 跨文件引用准确

适用场景：
- 复杂功能实现
- 架构设计
- 跨语言代码迁移

最佳实践：
```markdown
使用GPT-5.4的场景：

1. 复杂算法实现
   "实现一个支持分布式锁的Redis缓存系统，
    要求：
    - 支持看门狗自动续期
    - 支持可重入锁
    - 支持公平锁和非公平锁
    - 提供完整的单元测试"

2. 架构设计
   "设计一个微服务架构的订单系统，
    包含：服务拆分、数据一致性方案、
    容错设计、监控方案"

3. 代码重构
   "将当前单体应用重构为微服务架构，
    保留现有API兼容性"
```
```

### 2. Claude系列在Cursor中的应用

```markdown
## Claude Opus 4.6（Anthropic）特点：

代码质量：⭐⭐⭐⭐⭐
- 生成的代码更简洁
- 注释和文档更详细
- 错误处理更完善

调试能力：⭐⭐⭐⭐⭐
- 长上下文理解强（200K）
- 能处理复杂的调试场景
- 善于发现潜在问题

适用场景：
- 代码审查和优化
- 调试复杂问题
- 生成高质量文档

最佳实践：
```markdown
使用Claude Opus 4.6的场景：

1. 代码审查
   "审查以下代码，重点关注：
    - 线程安全问题
    - 内存泄漏风险
    - 异常处理完善度
    - 性能瓶颈"

2. 调试复杂问题
   "分析这个NullPointerException的根本原因，
    堆栈跟踪如下：
    [粘贴堆栈跟踪]
    
    要求：
    - 定位根因（不仅是表面位置）
    - 分析为什么会发生
    - 提供修复方案
    - 提供预防措施"

3. 生成技术文档
   "为以下API生成完整的Swagger文档，
    包括：请求示例、响应示例、错误码、
    边界情况说明"
```
```

### 3. 专用模型（Cursor Tab）

```markdown
## Cursor Tab（专用补全模型）特点：

响应速度：⭐⭐⭐⭐⭐
- 实时补全（<100ms）
- 流式输出
- 零延迟体验

补全质量：⭐⭐⭐⭐
- 基于项目上下文
- 学习团队编码风格
- 支持复杂模式

适用场景：
- 日常编码补全
- 快速生成样板代码
- 实时语法修正

技术实现：
```
Cursor Tab的技术栈：

1. 模型架构：
   - 基于Transformer的轻量级模型
   - 参数量：~1B（专为速度优化）
   - 量化：INT8/INT4推理

2. 上下文处理：
   - 局部上下文：当前行 ±50行
   - 项目上下文：相关文件摘要
   - 实时索引：文件修改即时更新

3. 推理优化：
   -  speculative decoding（推测解码）
   -  KV-cache复用
   -  批量处理相似请求

4. 个性化：
   - 学习用户编码习惯
   - 记忆常用代码模式
   - 适应项目代码风格
```
```

### 4. 模型选择决策树

```python
class ModelSelectionDecisionTree:
    """
    Cursor模型选择决策树
    """
    
    def select_model(self, task: Task) -> str:
        """
        根据任务特征选择最优模型
        """
        
        # 决策1：是否需要实时响应？
        if task.requires_realtime:
            return 'cursor-tab'
        
        # 决策2：任务复杂度
        if task.complexity == 'simple':
            # 简单任务用快速模型
            return 'gpt-5.4-mini'
        
        # 决策3：是否需要长上下文？
        if task.context_length > 100000:
            return 'claude-opus-4.6'  # 200K上下文
        
        # 决策4：任务类型
        if task.type == 'architecture_design':
            return 'gpt-5.4'  # 架构设计用GPT
        elif task.type == 'code_review':
            return 'claude-opus-4.6'  # 代码审查用Claude
        elif task.type == 'debugging':
            return 'claude-opus-4.6'  # 调试用Claude
        elif task.type == 'implementation':
            return 'gpt-5.4'  # 实现用GPT
        
        # 决策5：成本敏感度
        if task.cost_sensitive:
            return 'gpt-5.4-mini'
        
        # 默认选择
        return 'gpt-5.4'
```

---

## 工业级实践案例

### 案例1：企业级项目迁移（Java → Kotlin）

**场景**：将50万行Java项目迁移到Kotlin

**核心挑战**：
- 语法差异（Java的Checked Exception vs Kotlin）
- 空安全（Kotlin的空安全类型系统）
- 协程迁移（RxJava → Coroutines）
- 保持行为一致性

**Cursor工程方案**：

```kotlin
// 迁移策略：分阶段、可验证

class JavaToKotlinMigration {
    
    fun migrateProject(projectPath: String) {
        // 阶段1：分析项目结构
        val project = analyzeProject(projectPath)
        
        // 阶段2：生成迁移计划
        val plan = generateMigrationPlan(project)
        
        // 阶段3：按依赖顺序迁移
        val batches = topologicalSort(plan.modules)
        
        for (batch in batches) {
            // 使用Cursor Composer批量迁移
            migrateBatch(batch)
            
            // 验证编译通过
            verifyCompilation()
            
            // 运行测试
            runTests()
        }
    }
    
    fun migrateBatch(modules: List<Module>) {
        // 使用Cursor Agent模式
        val prompt = """
            将以下Java模块迁移为Kotlin，
            要求：
            1. 保持完全的行为一致性
            2. 使用Kotlin惯用法（idiomatic Kotlin）
            3. 处理空安全（使用?. ?:等）
            4. 将RxJava替换为Coroutines
            5. 保持原有的异常处理语义
            
            模块：${modules.joinToString { it.name }}
        """.trimIndent()
        
        // Cursor Agent自动执行：
        // 1. 读取Java文件
        // 2. 生成Kotlin代码
        // 3. 处理依赖关系
        // 4. 更新构建脚本
        // 5. 运行编译验证
    }
}

// 迁移示例：Java → Kotlin

// Java代码（迁移前）：
/*
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
    
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
    
    public User createUser(String email, String password) throws UserAlreadyExistsException {
        if (userRepository.findByEmail(email) != null) {
            throw new UserAlreadyExistsException("User already exists: " + email);
        }
        
        User user = new User(email, password);
        userRepository.save(user);
        emailService.sendWelcomeEmail(user);
        
        return user;
    }
}
*/

// Kotlin代码（迁移后）：
class UserService(
    private val userRepository: UserRepository,
    private val emailService: EmailService
) {
    @Throws(UserAlreadyExistsException::class)
    fun createUser(email: String, password: String): User {
        if (userRepository.findByEmail(email) != null) {
            throw UserAlreadyExistsException("User already exists: $email")
        }
        
        return User(email, password).also { user ->
            userRepository.save(user)
            emailService.sendWelcomeEmail(user)
        }
    }
}

// 使用协程和空安全的改进版本：
class UserServiceCoroutine(
    private val userRepository: UserRepository,
    private val emailService: EmailService
) {
    suspend fun createUser(email: String, password: String): Result<User> = 
        runCatching {
            userRepository.findByEmail(email)
                ?.let { throw UserAlreadyExistsException("User already exists: $email") }
            
            User(email, password).also { user ->
                userRepository.save(user)
                emailService.sendWelcomeEmail(user)
            }
        }
}
```

**评估指标**：
- 编译通过率：100%
- 测试通过率：>99%
- 迁移时间：从6个月缩短到2周
- 代码质量：Kotlin惯用法使用率>90%

### 案例2：AI驱动的测试生成管道

**场景**：为遗留项目自动生成单元测试

**核心挑战**：
- 无测试的遗留代码
- 复杂的依赖关系
- 外部服务调用
- 数据库依赖

**Cursor工程方案**：

```java
// 测试生成策略

class AITestGenerationPipeline {
    
    public void generateTestsForProject(String projectPath) {
        // 阶段1：代码分析
        List<Class<?>> classes = analyzeProject(projectPath);
        
        // 阶段2：按复杂度排序（从简单到复杂）
        List<Class<?>> sorted = classes.stream()
            .sorted(Comparator.comparingInt(this::calculateComplexity))
            .collect(Collectors.toList());
        
        // 阶段3：分批生成测试
        for (Class<?> clazz : sorted) {
            generateTestsForClass(clazz);
        }
    }
    
    private void generateTestsForClass(Class<?> clazz) {
        // 使用Cursor Chat生成测试
        String prompt = String.format("""
            为以下Java类生成完整的JUnit 5单元测试：
            
            要求：
            1. 覆盖所有public方法
            2. 包含正常情况和边界情况
            3. 使用Mockito模拟依赖
            4. 使用AssertJ进行断言
            5. 遵循Given-When-Then结构
            6. 测试方法名应描述行为
            
            类代码：
            %s
            """, getClassSource(clazz));
        
        // Cursor生成测试代码
        String testCode = cursor.generate(prompt);
        
        // 验证测试可编译
        if (compileTest(testCode)) {
            // 运行测试
            TestResult result = runTests(testCode);
            
            if (result.getFailureCount() > 0) {
                // 分析失败原因并修复
                fixFailingTests(testCode, result);
            }
        }
    }
    
    // 生成的测试示例
    /*
    @ExtendWith(MockitoExtension.class)
    class UserServiceTest {
        
        @Mock
        private UserRepository userRepository;
        
        @Mock
        private EmailService emailService;
        
        @InjectMocks
        private UserService userService;
        
        @Test
        @DisplayName("应该成功创建新用户")
        void shouldCreateNewUserSuccessfully() {
            // Given
            String email = "test@example.com";
            String password = "password123";
            when(userRepository.findByEmail(email)).thenReturn(null);
            
            // When
            User result = userService.createUser(email, password);
            
            // Then
            assertThat(result).isNotNull();
            assertThat(result.getEmail()).isEqualTo(email);
            verify(userRepository).save(any(User.class));
            verify(emailService).sendWelcomeEmail(any(User.class));
        }
        
        @Test
        @DisplayName("当用户已存在时应抛出异常")
        void shouldThrowExceptionWhenUserAlreadyExists() {
            // Given
            String email = "existing@example.com";
            when(userRepository.findByEmail(email))
                .thenReturn(new User(email, "old"));
            
            // When & Then
            assertThatThrownBy(() -> userService.createUser(email, "password"))
                .isInstanceOf(UserAlreadyExistsException.class)
                .hasMessageContaining("User already exists");
        }
    }
    */
}
```

### 案例3：多文件协调重构

**场景**：将单体架构重构为领域驱动设计（DDD）

**核心挑战**：
- 跨越数十个文件的修改
- 保持数据一致性
- 处理循环依赖
- 确保编译和测试通过

**Cursor Composer 2方案**：

```java
// 重构策略：分层、分领域

// 重构前：贫血领域模型
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;
    @Autowired
    private UserRepository userRepository;
    @Autowired
    private ProductRepository productRepository;
    
    public Order createOrder(Long userId, List<Long> productIds) {
        User user = userRepository.findById(userId).orElseThrow();
        Order order = new Order();
        order.setUserId(userId);
        order.setStatus("PENDING");
        
        BigDecimal total = BigDecimal.ZERO;
        for (Long productId : productIds) {
            Product product = productRepository.findById(productId).orElseThrow();
            OrderItem item = new OrderItem();
            item.setProductId(productId);
            item.setPrice(product.getPrice());
            item.setQuantity(1);
            order.getItems().add(item);
            total = total.add(product.getPrice());
        }
        
        order.setTotalAmount(total);
        orderRepository.save(order);
        return order;
    }
}

// 重构后：富领域模型
// Order.java（领域层）
@Entity
public class Order {
    @Id
    @GeneratedValue
    private Long id;
    
    @ManyToOne
    private User user;
    
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id")
    private List<OrderItem> items = new ArrayList<>();
    
    @Embedded
    private Money totalAmount;
    
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    
    // 领域行为
    public static Order create(User user, List<Product> products) {
        Order order = new Order();
        order.user = user;
        order.status = OrderStatus.PENDING;
        
        for (Product product : products) {
            order.addItem(product);
        }
        
        return order;
    }
    
    private void addItem(Product product) {
        if (!product.isAvailable()) {
            throw new ProductNotAvailableException(product.getId());
        }
        
        items.add(OrderItem.create(product));
        recalculateTotal();
    }
    
    private void recalculateTotal() {
        this.totalAmount = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
    
    public void pay(Payment payment) {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("Order is not pending");
        }
        
        if (!payment.getAmount().equals(totalAmount)) {
            throw new PaymentMismatchException();
        }
        
        this.status = OrderStatus.PAID;
        // 触发领域事件
        registerEvent(new OrderPaidEvent(this));
    }
}

// OrderItem.java（值对象）
@Embeddable
public class OrderItem {
    private Long productId;
    private Money price;
    private int quantity;
    
    public static OrderItem create(Product product) {
        OrderItem item = new OrderItem();
        item.productId = product.getId();
        item.price = product.getPrice();
        item.quantity = 1;
        return item;
    }
    
    public Money getSubtotal() {
        return price.multiply(quantity);
    }
}

// OrderApplicationService.java（应用层）
@Service
@Transactional
public class OrderApplicationService {
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    private final ProductRepository productRepository;
    private final DomainEventPublisher eventPublisher;
    
    public OrderApplicationService(
        OrderRepository orderRepository,
        UserRepository userRepository,
        ProductRepository productRepository,
        DomainEventPublisher eventPublisher
    ) {
        this.orderRepository = orderRepository;
        this.userRepository = userRepository;
        this.productRepository = productRepository;
        this.eventPublisher = eventPublisher;
    }
    
    public OrderDTO createOrder(CreateOrderCommand command) {
        User user = userRepository.findById(command.getUserId())
            .orElseThrow(() -> new UserNotFoundException(command.getUserId()));
        
        List<Product> products = productRepository.findAllById(command.getProductIds());
        if (products.size() != command.getProductIds().size()) {
            throw new ProductNotFoundException();
        }
        
        Order order = Order.create(user, products);
        orderRepository.save(order);
        
        // 发布领域事件
        order.getDomainEvents().forEach(eventPublisher::publish);
        order.clearDomainEvents();
        
        return OrderDTO.from(order);
    }
}
```

**Cursor Composer 2执行步骤**：
1. 分析现有代码，识别领域边界
2. 创建新的领域模型类
3. 重构Service层为ApplicationService
4. 更新Repository接口
5. 修改数据库映射
6. 添加领域事件机制
7. 运行编译验证
8. 运行测试验证

---

## 高级技术：Agent模式与Multi-Agent编排

### 1. Agent模式深度解析

```
Agent模式的工作流程：

用户输入："实现一个完整的用户认证系统"

Agent执行流程：
┌─────────────────────────────────────────┐
│ 1. 需求分析（Requirement Analysis）       │
│    - 解析用户意图                        │
│    - 识别关键需求点                       │
│    - 确定技术栈                          │
└─────────────────────┬───────────────────┘
                      │
┌─────────────────────▼───────────────────┐
│ 2. 任务规划（Task Planning）             │
│    - 分解为子任务                        │
│    - 确定依赖关系                        │
│    - 估算工作量                          │
└─────────────────────┬───────────────────┘
                      │
┌─────────────────────▼───────────────────┐
│ 3. 代码生成（Code Generation）           │
│    - 生成实体类                          │
│    - 生成Repository                      │
│    - 生成Service                         │
│    - 生成Controller                      │
│    - 生成配置类                          │
└─────────────────────┬───────────────────┘
                      │
┌─────────────────────▼───────────────────┐
│ 4. 编译验证（Compilation）               │
│    - 编译代码                            │
│    - 处理编译错误                         │
│    - 修复类型不匹配                       │
└─────────────────────┬───────────────────┘
                      │
┌─────────────────────▼───────────────────┐
│ 5. 测试运行（Testing）                   │
│    - 运行单元测试                         │
│    - 运行集成测试                         │
│    - 修复失败的测试                       │
└─────────────────────┬───────────────────┘
                      │
┌─────────────────────▼───────────────────┐
│ 6. 结果汇报（Reporting）                 │
│    - 生成变更摘要                         │
│    - 显示文件列表                         │
│    - 提供回滚选项                         │
└─────────────────────────────────────────┘
```

### 2. Multi-Agent编排系统

```python
class MultiAgentOrchestrator:
    """
    多Agent编排系统
    协调多个专业Agent完成复杂任务
    """
    
    def __init__(self):
        self.agents = {
            'architect': ArchitectAgent(),      # 架构师Agent
            'backend': BackendAgent(),          # 后端开发Agent
            'frontend': FrontendAgent(),        # 前端开发Agent
            'tester': TesterAgent(),            # 测试Agent
            'reviewer': ReviewerAgent()         # 审查Agent
        }
    
    def execute_project(self, requirements: str) -> Project:
        """
        执行完整项目开发
        """
        project = Project()
        
        # 阶段1：架构设计
        architecture = self.agents['architect'].design(requirements)
        project.set_architecture(architecture)
        
        # 阶段2：并行开发
        with ThreadPoolExecutor() as executor:
            # 后端开发
            backend_future = executor.submit(
                self.agents['backend'].implement,
                architecture.backend_spec
            )
            
            # 前端开发
            frontend_future = executor.submit(
                self.agents['frontend'].implement,
                architecture.frontend_spec
            )
            
            # 等待完成
            project.backend = backend_future.result()
            project.frontend = frontend_future.result()
        
        # 阶段3：测试
        test_results = self.agents['tester'].test(project)
        
        # 阶段4：审查和修复
        if not test_results.all_passed:
            review = self.agents['reviewer'].review(project, test_results)
            project = self.fix_issues(project, review.issues)
        
        return project
    
    def fix_issues(self, project: Project, issues: List[Issue]) -> Project:
        """
        修复审查发现的问题
        """
        for issue in issues:
            if issue.type == 'bug':
                project = self.agents['backend'].fix_bug(project, issue)
            elif issue.type == 'style':
                project = self.agents['reviewer'].apply_fix(project, issue)
        
        return project

class ArchitectAgent:
    """
    架构师Agent
    负责系统架构设计
    """
    
    def design(self, requirements: str) -> Architecture:
        """
        设计系统架构
        """
        # 1. 需求分析
        parsed_requirements = self.parse_requirements(requirements)
        
        # 2. 技术选型
        tech_stack = self.select_tech_stack(parsed_requirements)
        
        # 3. 架构设计
        architecture = Architecture()
        architecture.backend_spec = self.design_backend(parsed_requirements, tech_stack)
        architecture.frontend_spec = self.design_frontend(parsed_requirements, tech_stack)
        architecture.database_schema = self.design_database(parsed_requirements)
        architecture.api_spec = self.design_api(parsed_requirements)
        
        return architecture
    
    def select_tech_stack(self, requirements: Requirements) -> TechStack:
        """
        根据需求选择技术栈
        """
        # 基于规则的决策
        if requirements.scale == 'enterprise':
            return TechStack(
                backend='Java/Spring Boot',
                frontend='React/TypeScript',
                database='PostgreSQL',
                cache='Redis',
                message_queue='RabbitMQ'
            )
        elif requirements.scale == 'startup':
            return TechStack(
                backend='Node.js/NestJS',
                frontend='Next.js',
                database='MongoDB',
                cache='Redis',
                message_queue='Redis Pub/Sub'
            )
        
        return TechStack.default()
```

### 3. 云Agent（Cloud Agents）

```markdown
## Cloud Agents架构：

本地Cursor ←──→ Cursor Cloud ←──→ AI Model Cluster
     │              │                    │
     │              │                    │
     └──────────────┴────────────────────┘
                    │
           ┌────────▼────────┐
           │  Task Queue     │
           │  任务队列        │
           └────────┬────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
   ┌────▼────┐ ┌───▼───┐ ┌────▼────┐
   │ Worker 1 │ │Worker2│ │ Worker3 │
   │ 工作节点1 │ │工作节点2│ │ 工作节点3 │
   └─────────┘ └───────┘ └─────────┘

Cloud Agents的优势：

1. 异步处理：
   - 大任务后台执行
   - 完成后通知用户
   - 支持离线处理

2. 并行计算：
   - 多Worker并行处理
   - 自动负载均衡
   - 弹性伸缩

3. 持续运行：
   - 长时间任务不中断
   - 自动保存进度
   - 支持断点续传

4. 资源优化：
   - 使用高性能GPU
   - 更大的上下文窗口
   - 更快的推理速度
```

---

## 评估与优化体系

### 1. AI辅助编程效率评估框架

```python
class AICodingEfficiencyEvaluator:
    """
    AI辅助编程效率评估框架
    """
    
    def __init__(self):
        self.metrics = {
            'code_generation': CodeGenerationMetrics(),
            'code_understanding': CodeUnderstandingMetrics(),
            'debugging': DebuggingMetrics(),
            'refactoring': RefactoringMetrics()
        }
    
    def evaluate_session(self, session: CodingSession) -> EvaluationReport:
        """
        评估一次编程会话的效率
        """
        report = EvaluationReport()
        
        # 1. 代码生成效率
        report.code_generation = self.evaluate_code_generation(session)
        
        # 2. 代码理解效率
        report.code_understanding = self.evaluate_code_understanding(session)
        
        # 3. 调试效率
        report.debugging = self.evaluate_debugging(session)
        
        # 4. 重构效率
        report.refactoring = self.evaluate_refactoring(session)
        
        # 5. 综合评分
        report.overall_score = self.calculate_overall_score(report)
        
        return report
    
    def evaluate_code_generation(self, session: CodingSession) -> Metrics:
        """
        评估代码生成效率
        """
        metrics = Metrics()
        
        # 接受率（Accepted / Generated）
        metrics.acceptance_rate = (
            session.accepted_suggestions / session.total_suggestions
        )
        
        # 编辑距离（AI生成代码 vs 最终代码）
        metrics.edit_distance = self.calculate_edit_distance(
            session.generated_code,
            session.final_code
        )
        
        # 时间节省（vs 手动编写）
        metrics.time_saved = self.estimate_time_saved(session)
        
        # 代码质量
        metrics.code_quality = self.assess_code_quality(session.final_code)
        
        return metrics
    
    def evaluate_debugging(self, session: CodingSession) -> Metrics:
        """
        评估调试效率
        """
        metrics = Metrics()
        
        # 问题定位时间
        metrics.time_to_identify = session.debug_time
        
        # AI建议的准确性
        metrics.suggestion_accuracy = (
            session.correct_suggestions / session.total_suggestions
        )
        
        # 修复成功率
        metrics.fix_success_rate = (
            session.successful_fixes / session.attempted_fixes
        )
        
        return metrics
    
    def calculate_overall_score(self, report: EvaluationReport) -> float:
        """
        计算综合效率评分
        """
        weights = {
            'code_generation': 0.4,
            'code_understanding': 0.2,
            'debugging': 0.2,
            'refactoring': 0.2
        }
        
        score = 0.0
        for category, weight in weights.items():
            metrics = getattr(report, category)
            score += metrics.normalized_score * weight
        
        return score
```

### 2. 提示词优化（Prompt Engineering for Cursor）

```python
class CursorPromptOptimizer:
    """
    Cursor提示词优化器
    """
    
    def optimize_prompt(self, original_prompt: str, context: Context) -> str:
        """
        优化用户输入的提示词
        """
        optimized = original_prompt
        
        # 1. 添加上下文信息
        optimized = self.add_context(optimized, context)
        
        # 2. 明确输出格式
        optimized = self.specify_output_format(optimized)
        
        # 3. 添加约束条件
        optimized = self.add_constraints(optimized)
        
        # 4. 提供示例（Few-Shot）
        optimized = self.add_examples(optimized, context)
        
        return optimized
    
    def add_context(self, prompt: str, context: Context) -> str:
        """
        自动添加相关上下文
        """
        context_str = f"""
## 项目上下文
- 技术栈：{context.tech_stack}
- 项目结构：{context.project_structure}
- 编码规范：{context.coding_standards}

## 相关代码
{context.get_relevant_code()}
"""
        return f"{context_str}\n\n## 任务\n{prompt}"
    
    def specify_output_format(self, prompt: str) -> str:
        """
        明确输出格式要求
        """
        format_instruction = """
## 输出要求
1. 只输出代码，不要输出解释
2. 使用项目中已有的命名规范
3. 包含必要的异常处理
4. 添加JavaDoc注释
"""
        return f"{prompt}\n{format_instruction}"
    
    def add_constraints(self, prompt: str) -> str:
        """
        添加约束条件
        """
        constraints = """
## 约束条件
- 不要修改不相关的代码
- 保持向后兼容
- 遵循SOLID原则
- 避免引入新依赖（除非必要）
"""
        return f"{prompt}\n{constraints}"
```

### 3. 上下文优化策略

```markdown
## 上下文优化策略：

1. 选择性上下文：
   ```
   策略：只提供必要的上下文
   
   好的做法：
   @file:src/main/java/service/UserService.java
   "给这个方法的参数校验添加邮箱格式验证"
   
   坏的做法：
   "帮我改一下代码"
   （没有提供任何上下文）
   ```

2. 分层上下文：
   ```
   策略：按重要性分层提供上下文
   
   第一层：当前文件（必须）
   第二层：直接依赖的文件（重要）
   第三层：接口定义（有用）
   第四层：项目配置（参考）
   ```

3. 上下文压缩：
   ```
   策略：压缩长上下文
   
   原始上下文：
   - 完整的UserService.java（500行）
   - 完整的UserRepository.java（200行）
   - 完整的User.java（300行）
   
   压缩后：
   - UserService.java（只保留相关方法签名）
   - UserRepository.java（只保留接口定义）
   - User.java（只保留字段定义）
   ```

4. 动态上下文：
   ```
   策略：根据任务动态调整上下文
   
   任务："修复这个Bug"
   → 提供：相关代码 + 错误日志 + 测试用例
   
   任务："添加新功能"
   → 提供：现有代码 + 接口定义 + 类似实现
   
   任务："重构代码"
   → 提供：完整代码 + 依赖关系 + 测试代码
   ```
```

---

## 生活日用案例

### 案例：使用Cursor管理个人财务系统

```python
# 使用Cursor快速搭建个人财务管理系统

# 步骤1：描述需求
"""
创建一个个人财务管理系统，功能包括：
1. 记录收入和支出
2. 按类别统计
3. 月度预算管理
4. 数据可视化（图表）
5. 数据导出（CSV）

技术栈：Python + Flask + SQLite + Chart.js
"""

# 步骤2：Cursor Agent自动生成项目结构

# 项目结构：
# personal-finance/
# ├── app.py
# ├── models.py
# ├── routes.py
# ├── services.py
# ├── static/
# │   └── chart.js
# ├── templates/
# │   ├── base.html
# │   ├── dashboard.html
# │   ├── transactions.html
# │   └── budget.html
# └── requirements.txt

# 步骤3：核心代码（由Cursor生成）

# models.py
from flask_sqlalchemy import SQLAlchemy
from datetime import datetime

db = SQLAlchemy()

class Transaction(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    amount = db.Column(db.Numeric(10, 2), nullable=False)
    type = db.Column(db.String(10), nullable=False)  # 'income' or 'expense'
    category = db.Column(db.String(50), nullable=False)
    description = db.Column(db.String(200))
    date = db.Column(db.DateTime, default=datetime.utcnow)
    
    def to_dict(self):
        return {
            'id': self.id,
            'amount': float(self.amount),
            'type': self.type,
            'category': self.category,
            'description': self.description,
            'date': self.date.isoformat()
        }

class Budget(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    category = db.Column(db.String(50), nullable=False, unique=True)
    limit = db.Column(db.Numeric(10, 2), nullable=False)
    month = db.Column(db.String(7), nullable=False)  # '2024-01'

# routes.py
from flask import Flask, render_template, request, jsonify
from models import db, Transaction, Budget

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///finance.db'
db.init_app(app)

@app.route('/')
def dashboard():
    """仪表盘：显示统计信息"""
    # 本月统计
    month_total = db.session.query(
        db.func.sum(Transaction.amount)
    ).filter(
        db.func.strftime('%Y-%m', Transaction.date) == '2024-01'
    ).scalar()
    
    # 按类别统计
    category_stats = db.session.query(
        Transaction.category,
        db.func.sum(Transaction.amount).label('total')
    ).group_by(Transaction.category).all()
    
    return render_template('dashboard.html',
                         month_total=month_total,
                         category_stats=category_stats)

@app.route('/api/transactions', methods=['POST'])
def add_transaction():
    """添加交易记录"""
    data = request.json
    
    transaction = Transaction(
        amount=data['amount'],
        type=data['type'],
        category=data['category'],
        description=data.get('description', '')
    )
    
    db.session.add(transaction)
    db.session.commit()
    
    return jsonify(transaction.to_dict()), 201

@app.route('/api/transactions/export')
def export_transactions():
    """导出交易记录为CSV"""
    import csv
    import io
    
    transactions = Transaction.query.all()
    
    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(['Date', 'Type', 'Category', 'Amount', 'Description'])
    
    for t in transactions:
        writer.writerow([
            t.date.strftime('%Y-%m-%d'),
            t.type,
            t.category,
            t.amount,
            t.description
        ])
    
    return output.getvalue(), 200, {
        'Content-Type': 'text/csv',
        'Content-Disposition': 'attachment; filename=transactions.csv'
    }

# 步骤4：使用Cursor添加数据可视化
"""
在dashboard.html中添加Chart.js图表，
显示月度收支趋势和类别占比
"""

# 生成的Chart.js代码：
"""
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script>
// 月度趋势图
const trendCtx = document.getElementById('trendChart').getContext('2d');
new Chart(trendCtx, {
    type: 'line',
    data: {
        labels: ['1月', '2月', '3月', '4月', '5月', '6月'],
        datasets: [{
            label: '收入',
            data: [5000, 5500, 4800, 6000, 5200, 5800],
            borderColor: 'green',
            fill: false
        }, {
            label: '支出',
            data: [3000, 3500, 2800, 4000, 3200, 3600],
            borderColor: 'red',
            fill: false
        }]
    }
});

// 类别占比图
const categoryCtx = document.getElementById('categoryChart').getContext('2d');
new Chart(categoryCtx, {
    type: 'doughnut',
    data: {
        labels: ['餐饮', '交通', '购物', '娱乐', '其他'],
        datasets: [{
            data: [30, 15, 25, 10, 20],
            backgroundColor: [
                '#FF6384', '#36A2EB', '#FFCE56', '#4BC0C0', '#9966FF'
            ]
        }]
    }
});
</script>
"""

# 步骤5：运行和验证
"""
Cursor自动执行：
1. 安装依赖：pip install flask flask-sqlalchemy
2. 创建数据库：flask shell → db.create_all()
3. 运行应用：flask run
4. 打开浏览器验证
"""
```

---

## 编程专项深度实践

### 实战1：使用Cursor构建微服务架构

```java
// 使用Cursor Agent模式快速搭建微服务

// 步骤1：描述架构需求
"""
创建一个电商微服务系统，包含：
1. 用户服务（User Service）
2. 商品服务（Product Service）
3. 订单服务（Order Service）
4. 支付服务（Payment Service）
5. 通知服务（Notification Service）

技术栈：Spring Boot 3.x + Spring Cloud + PostgreSQL + RabbitMQ
"""

// 步骤2：Cursor Agent自动生成项目结构

// 项目结构：
// ecommerce/
// ├── api-gateway/
// ├── user-service/
// ├── product-service/
// ├── order-service/
// ├── payment-service/
// ├── notification-service/
// ├── common-lib/
// └── docker-compose.yml

// 步骤3：API网关（由Cursor生成）

@SpringBootApplication
@EnableDiscoveryClient
public class ApiGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ApiGatewayApplication.class, args);
    }
}

@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            // 用户服务路由
            .route("user-service", r -> r.path("/api/users/**")
                .filters(f -> f
                    .stripPrefix(2)
                    .circuitBreaker(config -> config
                        .setName("userCircuitBreaker")
                        .setFallbackUri("forward:/fallback/user"))
                    .requestRateLimiter(config -> config
                        .setRateLimiter(redisRateLimiter())
                        .setKeyResolver(userKeyResolver())))
                .uri("lb://user-service"))
            
            // 商品服务路由
            .route("product-service", r -> r.path("/api/products/**")
                .filters(f -> f.stripPrefix(2))
                .uri("lb://product-service"))
            
            // 订单服务路由
            .route("order-service", r -> r.path("/api/orders/**")
                .filters(f -> f.stripPrefix(2))
                .uri("lb://order-service"))
            
            .build();
    }
    
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> Mono.just(
            exchange.getRequest().getHeaders().getFirst("X-User-Id")
        );
    }
}

// 步骤4：用户服务（由Cursor生成）

@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String email;
    
    @Column(nullable = false)
    private String password;
    
    @Column(nullable = false)
    private String name;
    
    @Enumerated(EnumType.STRING)
    private UserStatus status = UserStatus.ACTIVE;
    
    @CreatedDate
    private LocalDateTime createdAt;
    
    // 领域行为
    public void deactivate() {
        if (this.status == UserStatus.INACTIVE) {
            throw new IllegalStateException("User is already inactive");
        }
        this.status = UserStatus.INACTIVE;
    }
    
    public void updateProfile(String name, String email) {
        this.name = Objects.requireNonNull(name, "Name cannot be null");
        this.email = Objects.requireNonNull(email, "Email cannot be null");
    }
    
    // Getters...
}

@Service
@Transactional
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final EventPublisher eventPublisher;
    
    public UserService(UserRepository userRepository,
                      PasswordEncoder passwordEncoder,
                      EventPublisher eventPublisher) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
        this.eventPublisher = eventPublisher;
    }
    
    public User registerUser(RegisterUserCommand command) {
        // 检查邮箱是否已存在
        if (userRepository.existsByEmail(command.getEmail())) {
            throw new UserAlreadyExistsException(command.getEmail());
        }
        
        // 创建用户
        User user = new User();
        user.setEmail(command.getEmail());
        user.setPassword(passwordEncoder.encode(command.getPassword()));
        user.setName(command.getName());
        
        userRepository.save(user);
        
        // 发布用户注册事件
        eventPublisher.publish(new UserRegisteredEvent(
            user.getId(),
            user.getEmail(),
            user.getName()
        ));
        
        return user;
    }
    
    public User getUserById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}

// 步骤5：事件驱动架构（由Cursor生成）

// 用户注册事件
public record UserRegisteredEvent(
    Long userId,
    String email,
    String name
) implements DomainEvent {}

// 事件监听器
@Component
public class UserEventListener {
    private final NotificationService notificationService;
    private final EmailService emailService;
    
    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        // 发送欢迎邮件
        emailService.sendWelcomeEmail(event.email(), event.name());
        
        // 发送通知
        notificationService.sendNotification(
            event.userId(),
            "Welcome! Your account has been created successfully."
        );
    }
}

// 步骤6：Docker Compose（由Cursor生成）

version: '3.8'
services:
  # API网关
  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    environment:
      - EUREKA_CLIENT_SERVICE_URL_DEFAULTZONE=http://eureka:8761/eureka
    depends_on:
      - eureka

  # 注册中心
  eureka:
    build: ./eureka-server
    ports:
      - "8761:8761"

  # 用户服务
  user-service:
    build: ./user-service
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/users
      - SPRING_RABBITMQ_HOST=rabbitmq
    depends_on:
      - postgres
      - rabbitmq
      - eureka

  # PostgreSQL
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=ecommerce
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  # RabbitMQ
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"

volumes:
  postgres_data:
```

### 实战2：使用Cursor进行性能优化

```java
// 场景：优化高并发订单处理系统

// 优化前：同步处理，性能瓶颈
@Service
public class OrderServiceV1 {
    public Order createOrder(CreateOrderRequest request) {
        // 1. 验证库存（同步调用）
        boolean inStock = productService.checkStock(request.getProductId(), request.getQuantity());
        if (!inStock) {
            throw new OutOfStockException();
        }
        
        // 2. 创建订单（同步数据库操作）
        Order order = orderRepository.save(new Order(request));
        
        // 3. 扣减库存（同步调用）
        productService.decreaseStock(request.getProductId(), request.getQuantity());
        
        // 4. 处理支付（同步调用）
        PaymentResult payment = paymentService.process(order);
        if (!payment.isSuccess()) {
            throw new PaymentFailedException();
        }
        
        // 5. 发送通知（同步调用）
        notificationService.sendOrderConfirmation(order);
        
        return order;
    }
}

// 优化后：异步事件驱动（Cursor生成）
@Service
public class OrderServiceV2 {
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;
    private final RedissonClient redissonClient;
    
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        // 1. 使用分布式锁防止超卖
        RLock lock = redissonClient.getLock("stock:" + request.getProductId());
        
        try {
            boolean locked = lock.tryLock(5, 10, TimeUnit.SECONDS);
            if (!locked) {
                throw new ConcurrentOrderException("System busy, please try again");
            }
            
            // 2. 快速创建订单（状态：PENDING）
            Order order = Order.create(request);
            orderRepository.save(order);
            
            // 3. 发布订单创建事件（异步处理后续逻辑）
            eventPublisher.publishEvent(new OrderCreatedEvent(
                order.getId(),
                request.getProductId(),
                request.getQuantity(),
                request.getUserId()
            ));
            
            return order;
        } finally {
            lock.unlock();
        }
    }
}

// 事件处理器（异步）
@Component
public class OrderEventHandler {
    private final ProductService productService;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    
    @Async("orderTaskExecutor")
    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            // 1. 扣减库存
            productService.decreaseStock(event.getProductId(), event.getQuantity());
            
            // 2. 处理支付
            PaymentResult payment = paymentService.process(event.getOrderId());
            if (payment.isSuccess()) {
                // 3. 更新订单状态
                orderRepository.updateStatus(event.getOrderId(), OrderStatus.PAID);
                
                // 4. 发送通知（异步）
                notificationService.sendOrderConfirmation(event.getOrderId());
            } else {
                // 支付失败，回滚库存
                productService.increaseStock(event.getProductId(), event.getQuantity());
                orderRepository.updateStatus(event.getOrderId(), OrderStatus.PAYMENT_FAILED);
            }
        } catch (Exception e) {
            // 记录错误，进入死信队列
            log.error("Order processing failed: {}", event.getOrderId(), e);
        }
    }
}

// 配置线程池
@Configuration
public class AsyncConfig {
    @Bean("orderTaskExecutor")
    public Executor orderTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(1000);
        executor.setThreadNamePrefix("order-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

// 性能测试对比（Cursor生成测试代码）
@RunWith(SpringRunner.class)
@SpringBootTest
public class OrderServicePerformanceTest {
    
    @Autowired
    private OrderServiceV1 orderServiceV1;
    
    @Autowired
    private OrderServiceV2 orderServiceV2;
    
    @Test
    public void comparePerformance() throws InterruptedException {
        int threadCount = 100;
        int requestsPerThread = 10;
        
        // 测试V1（同步）
        long v1Start = System.currentTimeMillis();
        CountDownLatch v1Latch = new CountDownLatch(threadCount);
        
        for (int i = 0; i < threadCount; i++) {
            new Thread(() -> {
                for (int j = 0; j < requestsPerThread; j++) {
                    try {
                        orderServiceV1.createOrder(createRequest());
                    } catch (Exception e) {
                        // 忽略错误
                    }
                }
                v1Latch.countDown();
            }).start();
        }
        
        v1Latch.await();
        long v1Time = System.currentTimeMillis() - v1Start;
        
        // 测试V2（异步）
        long v2Start = System.currentTimeMillis();
        CountDownLatch v2Latch = new CountDownLatch(threadCount);
        
        for (int i = 0; i < threadCount; i++) {
            new Thread(() -> {
                for (int j = 0; j < requestsPerThread; j++) {
                    try {
                        orderServiceV2.createOrder(createRequest());
                    } catch (Exception e) {
                        // 忽略错误
                    }
                }
                v2Latch.countDown();
            }).start();
        }
        
        v2Latch.await();
        long v2Time = System.currentTimeMillis() - v2Start;
        
        System.out.println("V1 (Sync) Time: " + v1Time + "ms");
        System.out.println("V2 (Async) Time: " + v2Time + "ms");
        System.out.println("Improvement: " + (v1Time - v2Time) * 100.0 / v1Time + "%");
    }
    
    private CreateOrderRequest createRequest() {
        // 创建测试请求
        return new CreateOrderRequest(1L, 1, 1L);
    }
}
```

### 实战3：使用Cursor进行安全审计

```java
// 场景：对现有系统进行安全审计和修复

// Cursor审查发现的安全问题：

// 问题1：SQL注入
// 修复前：
@Repository
public class UserRepositoryV1 {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public List<User> searchUsers(String keyword) {
        // ❌ 危险：直接拼接SQL
        String sql = "SELECT * FROM users WHERE name LIKE '%" + keyword + "%'";
        return jdbcTemplate.query(sql, new UserRowMapper());
    }
}

// 修复后（Cursor生成）：
@Repository
public class UserRepositoryV2 {
    private final JdbcTemplate jdbcTemplate;
    
    public UserRepositoryV2(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
    
    public List<User> searchUsers(String keyword) {
        // ✅ 安全：使用参数化查询
        String sql = "SELECT * FROM users WHERE name LIKE ?";
        return jdbcTemplate.query(sql, new Object[]{"%" + keyword + "%"}, new UserRowMapper());
    }
    
    // 更安全的做法：使用Spring Data JPA
    public interface UserRepository extends JpaRepository<User, Long> {
        @Query("SELECT u FROM User u WHERE u.name LIKE %:keyword%")
        List<User> searchByName(@Param("keyword") String keyword);
    }
}

// 问题2：敏感信息泄露
// 修复前：
@RestController
public class UserControllerV1 {
    @GetMapping("/api/users/{id}")
    public User getUser(@PathVariable Long id) {
        // ❌ 危险：返回包含密码的完整实体
        return userRepository.findById(id).orElseThrow();
    }
}

// 修复后（Cursor生成）：
@RestController
@RequestMapping("/api/users")
public class UserControllerV2 {
    private final UserService userService;
    
    public UserControllerV2(UserService userService) {
        this.userService = userService;
    }
    
    @GetMapping("/{id}")
    public UserDTO getUser(@PathVariable Long id) {
        // ✅ 安全：使用DTO，排除敏感字段
        return userService.getUserById(id);
    }
}

// DTO定义（不包含密码）
public record UserDTO(
    Long id,
    String email,
    String name,
    UserStatus status,
    LocalDateTime createdAt
) {
    public static UserDTO from(User user) {
        return new UserDTO(
            user.getId(),
            user.getEmail(),
            user.getName(),
            user.getStatus(),
            user.getCreatedAt()
        );
    }
}

// 问题3：不安全的反序列化
// 修复前：
@Component
public class MessageProcessorV1 {
    public void processMessage(String message) {
        // ❌ 危险：使用Java原生反序列化
        try (ObjectInputStream ois = new ObjectInputStream(
                new ByteArrayInputStream(message.getBytes()))) {
            Object obj = ois.readObject();
            // 处理对象
        }
    }
}

// 修复后（Cursor生成）：
@Component
public class MessageProcessorV2 {
    private final ObjectMapper objectMapper;
    
    public MessageProcessorV2(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
        // 配置安全选项
        this.objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }
    
    public void processMessage(String message) {
        // ✅ 安全：使用JSON反序列化，配合类型检查
        try {
            MessageDTO dto = objectMapper.readValue(message, MessageDTO.class);
            
            // 输入验证
            if (!isValid(dto)) {
                throw new InvalidMessageException("Message validation failed");
            }
            
            // 处理消息
            process(dto);
        } catch (JsonProcessingException e) {
            throw new InvalidMessageException("Invalid message format", e);
        }
    }
    
    private boolean isValid(MessageDTO dto) {
        return dto != null 
            && dto.getType() != null 
            && dto.getPayload() != null
            && dto.getPayload().length() < 10000;  // 限制大小
    }
}
```

---

## 跨行业应用场景

### 1. 金融行业：量化交易系统

```python
# 使用Cursor构建量化交易系统

# 场景：高频交易策略实现

import asyncio
import numpy as np
from dataclasses import dataclass
from typing import List, Dict
import redis

@dataclass
class MarketData:
    symbol: str
    timestamp: float
    price: float
    volume: int
    bid: float
    ask: float

class HighFrequencyTradingSystem:
    """
    高频交易系统
    """
    
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.positions: Dict[str, float] = {}
        self.orders: List[Order] = []
        self.running = False
    
    async def start(self):
        """启动交易系统"""
        self.running = True
        
        # 启动多个策略
        await asyncio.gather(
            self.run_strategy('mean_reversion', self.mean_reversion_strategy),
            self.run_strategy('momentum', self.momentum_strategy),
            self.run_strategy('arbitrage', self.arbitrage_strategy)
        )
    
    async def run_strategy(self, name: str, strategy):
        """运行单个策略"""
        while self.running:
            try:
                # 获取市场数据
                data = await self.get_market_data()
                
                # 执行策略
                signals = strategy(data)
                
                # 执行交易信号
                for signal in signals:
                    await self.execute_signal(signal)
                
                # 微秒级延迟
                await asyncio.sleep(0.001)
            except Exception as e:
                print(f"Strategy {name} error: {e}")
    
    def mean_reversion_strategy(self, data: List[MarketData]) -> List[Signal]:
        """
        均值回归策略
        """
        signals = []
        
        for symbol_data in self.group_by_symbol(data):
            prices = [d.price for d in symbol_data]
            
            # 计算移动平均线
            ma20 = np.mean(prices[-20:])
            ma50 = np.mean(prices[-50:])
            
            current_price = prices[-1]
            
            # 均值回归信号
            if current_price < ma20 * 0.99:  # 价格低于MA20 1%
                signals.append(Signal(
                    symbol=symbol_data[0].symbol,
                    action='BUY',
                    price=current_price,
                    quantity=100,
                    reason='Mean reversion: price below MA20'
                ))
            elif current_price > ma20 * 1.01:  # 价格高于MA20 1%
                signals.append(Signal(
                    symbol=symbol_data[0].symbol,
                    action='SELL',
                    price=current_price,
                    quantity=100,
                    reason='Mean reversion: price above MA20'
                ))
        
        return signals
    
    def momentum_strategy(self, data: List[MarketData]) -> List[Signal]:
        """
        动量策略
        """
        signals = []
        
        for symbol_data in self.group_by_symbol(data):
            prices = [d.price for d in symbol_data]
            
            # 计算收益率
            returns = np.diff(prices) / prices[:-1]
            
            # 动量信号
            if len(returns) >= 10:
                momentum = np.mean(returns[-10:])
                
                if momentum > 0.001:  # 正向动量
                    signals.append(Signal(
                        symbol=symbol_data[0].symbol,
                        action='BUY',
                        price=prices[-1],
                        quantity=100,
                        reason=f'Momentum: {momentum:.4f}'
                    ))
                elif momentum < -0.001:  # 负向动量
                    signals.append(Signal(
                        symbol=symbol_data[0].symbol,
                        action='SELL',
                        price=prices[-1],
                        quantity=100,
                        reason=f'Momentum: {momentum:.4f}'
                    ))
        
        return signals
    
    async def execute_signal(self, signal: Signal):
        """执行交易信号"""
        # 风控检查
        if not self.risk_check(signal):
            return
        
        # 发送订单
        order = await self.send_order(signal)
        
        # 记录订单
        self.orders.append(order)
        
        # 更新持仓
        self.update_position(signal)
    
    def risk_check(self, signal: Signal) -> bool:
        """风险控制检查"""
        # 1. 检查持仓限制
        current_position = self.positions.get(signal.symbol, 0)
        if abs(current_position + signal.quantity) > 1000:
            return False
        
        # 2. 检查亏损限制
        unrealized_pnl = self.calculate_unrealized_pnl()
        if unrealized_pnl < -10000:
            return False
        
        # 3. 检查波动率
        volatility = self.calculate_volatility(signal.symbol)
        if volatility > 0.05:  # 波动率超过5%
            return False
        
        return True
    
    def calculate_volatility(self, symbol: str) -> float:
        """计算波动率"""
        # 从Redis获取历史数据
        prices = self.redis.lrange(f"prices:{symbol}", 0, 99)
        prices = [float(p) for p in prices]
        
        if len(prices) < 2:
            return 0.0
        
        returns = np.diff(prices) / prices[:-1]
        return np.std(returns)
```

### 2. 医疗行业：医学影像分析系统

```python
# 使用Cursor构建医学影像分析系统

import torch
import torch.nn as nn
from torchvision import transforms, models
from PIL import Image
import numpy as np

class MedicalImageAnalysisSystem:
    """
    医学影像分析系统
    """
    
    def __init__(self, model_path: str):
        self.device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
        self.model = self.load_model(model_path)
        self.transform = self.get_transform()
    
    def load_model(self, model_path: str) -> nn.Module:
        """加载预训练模型"""
        # 使用ResNet50作为基础模型
        model = models.resnet50(pretrained=True)
        
        # 修改最后一层用于医学影像分类
        num_features = model.fc.in_features
        model.fc = nn.Sequential(
            nn.Linear(num_features, 512),
            nn.ReLU(),
            nn.Dropout(0.5),
            nn.Linear(512, 4)  # 4类：正常、肺炎、肿瘤、骨折
        )
        
        # 加载微调后的权重
        model.load_state_dict(torch.load(model_path, map_location=self.device))
        model.to(self.device)
        model.eval()
        
        return model
    
    def get_transform(self):
        """获取图像预处理变换"""
        return transforms.Compose([
            transforms.Resize((512, 512)),
            transforms.ToTensor(),
            transforms.Normalize(
                mean=[0.485, 0.456, 0.406],
                std=[0.229, 0.224, 0.225]
            )
        ])
    
    def analyze(self, image_path: str) -> AnalysisResult:
        """分析医学影像"""
        # 1. 加载图像
        image = Image.open(image_path).convert('RGB')
        
        # 2. 预处理
        input_tensor = self.transform(image).unsqueeze(0).to(self.device)
        
        # 3. 推理
        with torch.no_grad():
            outputs = self.model(input_tensor)
            probabilities = torch.softmax(outputs, dim=1)
        
        # 4. 解析结果
        class_names = ['Normal', 'Pneumonia', 'Tumor', 'Fracture']
        probs = probabilities[0].cpu().numpy()
        
        predictions = [
            Prediction(class_name, float(prob))
            for class_name, prob in zip(class_names, probs)
        ]
        
        # 5. 生成报告
        report = self.generate_report(predictions, image)
        
        return AnalysisResult(
            predictions=predictions,
            report=report,
            confidence=float(max(probs))
        )
    
    def generate_report(self, predictions: List[Prediction], image: Image) -> str:
        """生成医学报告"""
        # 使用NLP模型生成报告
        top_prediction = max(predictions, key=lambda p: p.probability)
        
        report_template = f"""
医学影像分析报告
================

检查项目：胸部X光片
分析时间：{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}

诊断结果：
- 主要发现：{top_prediction.class_name}
- 置信度：{top_prediction.probability:.2%}

详细分析：
{self.get_detailed_analysis(predictions)}

建议：
{self.get_recommendations(top_prediction)}

注意事项：
本报告由AI辅助生成，仅供参考。
最终诊断请由专业医生确认。
"""
        
        return report_template
    
    def get_detailed_analysis(self, predictions: List[Prediction]) -> str:
        """获取详细分析"""
        lines = []
        for pred in predictions:
            if pred.probability > 0.1:  # 只显示概率>10%的
                lines.append(f"- {pred.class_name}: {pred.probability:.2%}")
        return '\n'.join(lines)
    
    def get_recommendations(self, prediction: Prediction) -> str:
        """获取建议"""
        recommendations = {
            'Normal': '未发现明显异常，建议定期体检。',
            'Pneumonia': '疑似肺炎，建议进一步检查（CT扫描）。',
            'Tumor': '疑似肿瘤，建议尽快进行病理检查。',
            'Fracture': '疑似骨折，建议进行骨科会诊。'
        }
        return recommendations.get(prediction.class_name, '建议进一步检查。')

# 使用Cursor快速构建API
from fastapi import FastAPI, File, UploadFile
from fastapi.responses import JSONResponse

app = FastAPI(title="Medical Image Analysis API")
analyzer = MedicalImageAnalysisSystem('model.pth')

@app.post("/analyze")
async def analyze_image(file: UploadFile = File(...)):
    """分析上传的医学影像"""
    # 保存上传的文件
    temp_path = f"/tmp/{file.filename}"
    with open(temp_path, "wb") as buffer:
        buffer.write(await file.read())
    
    # 分析影像
    result = analyzer.analyze(temp_path)
    
    # 返回结果
    return JSONResponse({
        'predictions': [
            {'class': p.class_name, 'probability': p.probability}
            for p in result.predictions
        ],
        'report': result.report,
        'confidence': result.confidence
    })
```

---

## 面试题与参考答案

### 题目1：Cursor的Agent模式与传统代码补全有什么区别？

**参考答案**：

```
核心区别：

1. 交互范式：
   - 传统补全：被动响应（用户输入→AI补全）
   - Agent模式：主动执行（用户描述目标→AI自主完成）

2. 上下文范围：
   - 传统补全：当前文件 + 相邻文件
   - Agent模式：整个项目 + 运行时环境 + 外部资源

3. 执行能力：
   - 传统补全：只生成代码
   - Agent模式：生成代码 + 运行测试 + 修复错误 + 验证结果

4. 错误处理：
   - 传统补全：无错误处理
   - Agent模式：自动检测和修复编译错误、测试失败

5. 适用场景：
   - 传统补全：日常编码、快速补全
   - Agent模式：复杂功能实现、项目初始化、大规模重构
```

### 题目2：如何优化Cursor的上下文使用效率？

**参考答案**：

```python
# 上下文优化策略

class ContextOptimization:
    """
    上下文优化最佳实践
    """
    
    def strategy_1_selective_context(self):
        """
        策略1：选择性上下文
        """
        # 好的做法：明确引用相关文件
        prompt = """
        @file:src/main/java/service/OrderService.java
        @file:src/main/java/repository/OrderRepository.java
        
        给OrderService添加分布式事务支持
        """
        
        # 坏的做法：不提供上下文
        prompt = "改一下代码"  # ❌ 太模糊
    
    def strategy_2_hierarchical_context(self):
        """
        策略2：分层上下文
        """
        # 按重要性提供上下文
        prompt = """
        ## 核心上下文
        @file:当前修改的文件
        
        ## 依赖上下文
        @file:直接依赖的接口
        
        ## 参考上下文
        @file:类似实现（参考用）
        
        任务：...
        """
    
    def strategy_3_context_compression(self):
        """
        策略3：上下文压缩
        """
        # 对于大文件，只提供必要的部分
        prompt = """
        // 只提供方法签名和关键逻辑
        public class UserService {
            public User createUser(String email, String password);
            public User getUserById(Long id);
            // ... 其他方法省略
        }
        
        给createUser添加参数校验
        """
    
    def strategy_4_dynamic_context(self):
        """
        策略4：动态上下文
        """
        # 根据任务类型动态调整
        if task_type == 'bug_fix':
            # Bug修复需要：错误日志 + 相关代码 + 测试用例
            context = get_error_logs() + get_related_code() + get_test_cases()
        elif task_type == 'feature':
            # 新功能需要：接口定义 + 类似实现 + 项目规范
            context = get_interfaces() + get_similar_impl() + get_coding_standards()
```

### 题目3：Cursor的多模型路由策略是什么？

**参考答案**：

```python
class MultiModelRouting:
    """
    多模型路由策略
    """
    
    def routing_strategy(self):
        """
        路由决策因素：
        
        1. 任务复杂度：
           - 简单任务（单行补全）→ Cursor Tab（专用小模型）
           - 中等任务（函数实现）→ GPT-5.4-mini
           - 复杂任务（架构设计）→ GPT-5.4 / Claude Opus
        
        2. 上下文长度：
           - <4K → Cursor Tab
           - 4K-128K → GPT-5.4
           - >128K → Claude Opus（200K上下文）
        
        3. 任务类型：
           - 代码生成 → GPT-5.4（生成能力强）
           - 代码审查 → Claude Opus（分析能力强）
           - 调试 → Claude Opus（推理能力强）
           - 文档生成 → Claude Opus（自然语言强）
        
        4. 成本约束：
           - 成本敏感 → GPT-5.4-mini / Cursor Tab
           - 质量优先 → Claude Opus / GPT-5.4
        
        5. 延迟约束：
           - 实时响应 → Cursor Tab
           - 可接受1-2秒 → GPT-5.4-mini
           - 可接受3-5秒 → GPT-5.4 / Claude
        """
    
    def implementation(self):
        """
        实现方式：
        
        1. 规则引擎：
           - 基于预定义规则路由
           - 简单、可解释
        
        2. 机器学习模型：
           - 训练路由模型
           - 根据历史数据优化
        
        3. 混合策略：
           - 先用规则筛选候选模型
           - 再用ML模型选择最优
        """
```

### 题目4：如何评估AI辅助编程工具的效果？

**参考答案**：

```python
class AICodingEvaluation:
    """
    AI辅助编程效果评估框架
    """
    
    def metrics(self):
        """
        关键评估指标：
        
        1. 效率指标：
           - 代码生成接受率 = 接受的建议数 / 总建议数
           - 编辑距离 = AI生成代码与最终代码的差异
           - 时间节省 = 手动编码时间 - AI辅助编码时间
        
        2. 质量指标：
           - 代码质量评分（基于静态分析）
           - 测试覆盖率
           - Bug引入率 = AI引入的Bug数 / 总修改数
        
        3. 体验指标：
           - 响应延迟（从输入到建议的时间）
           - 上下文相关性（建议是否相关）
           - 用户满意度（问卷调查）
        
        4. 业务指标：
           - 功能交付速度
           - 代码审查通过率
           - 生产环境Bug数
        """
    
    def evaluation_method(self):
        """
        评估方法：
        
        1. A/B测试：
           - 实验组：使用AI工具
           - 对照组：不使用AI工具
           - 比较效率和质量差异
        
        2. 前后对比：
           - 记录使用AI工具前的指标
           - 记录使用AI工具后的指标
           - 计算改进幅度
        
        3. 用户调研：
           - 定期问卷调查
           - 访谈深度用户
           - 收集改进建议
        """
```

---

*此文原创，转载请注明出处。*
