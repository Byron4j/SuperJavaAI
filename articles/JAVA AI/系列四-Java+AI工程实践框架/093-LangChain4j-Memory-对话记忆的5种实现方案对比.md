# LangChain4j Memory：对话记忆的 5 种实现方案对比，选错记忆方案你的 AI 就像"金鱼"

> 你以为接入大模型就万事大吉了？没有记忆机制的 AI 比金鱼还健忘——上一秒你说"我叫张三"，下一秒它就问"请问您叫什么名字？"。今天这篇，把 LangChain4j 的 5 种记忆方案一次性讲透。

---

## 一、开篇：AI 的"金鱼记忆"困境

先看一个让人崩溃的对话场景：

```
用户：我叫张三，来自北京。
AI：你好张三！北京最近天气不错。
用户：我刚才说了我叫什么？
AI：您好像还没有告诉我您的名字，请问您怎么称呼？
```

这就是典型的"失忆症"——**大模型本身是无状态的**。每次 API 调用都是独立的，模型不会记住上一轮说了什么。要让 AI 拥有"记忆"，你必须自己维护对话历史，并在每次请求时带上。

LangChain4j 提供了 `ChatMemory` 抽象来解决这个问题。但**不是你随便 new 一个内存实现就完事了**——不同的记忆方案在成本、性能、效果上差异巨大。选错方案，轻则浪费 Token 费用，重则 AI 产生幻觉。

今天这篇文章，我会带你深入 LangChain4j 的 5 种 Memory 实现，通过完整可运行的代码对比它们的优劣，帮你找到最适合你业务场景的那一款。

---

## 二、ChatMemory 核心架构速览

在深入源码之前，先搞清楚 LangChain4j 中 Memory 的核心抽象：

```java
public interface ChatMemory {
    void add(ChatMessage message);
    List<ChatMessage> messages();
    void clear();
    Object id();  // 用于多用户场景下的隔离
}
```

**三个核心方法**：
- `add(message)`：将一条消息追加到记忆
- `messages()`：获取当前所有记忆消息（会传给 LLM）
- `clear()`：清空所有记忆

LangChain4j 内置了多种 `ChatMemory` 实现，全部位于 `dev.langchain4j.memory.chat` 包下。它们的区别在于**什么消息该保留、什么该丢弃**。

下面进入正题，逐一拆解。

---

## 三、方案一：MessageWindowMemory——最简单的滑动窗口

### 3.1 原理

`MessageWindowMemory` 采用**固定数量**的滑动窗口策略。你设置一个 `maxMessages`，它永远只保留最近的 N 条消息。

### 3.2 完整代码

```java
import dev.langchain4j.memory.ChatMemory;
import dev.langchain4j.memory.chat.MessageWindowChatMemory;
import dev.langchain4j.model.openai.OpenAiChatModel;
import dev.langchain4j.service.AiServices;

public class MessageWindowDemo {

    public interface Assistant {
        String chat(String message);
    }

    public static void main(String[] args) {
        // 最多保留 10 条消息（5 轮对话）
        ChatMemory memory = MessageWindowChatMemory.withMaxMessages(10);

        Assistant assistant = AiServices.builder(Assistant.class)
                .chatLanguageModel(OpenAiChatModel.builder()
                        .apiKey(System.getenv("OPENAI_API_KEY"))
                        .modelName("gpt-4o-mini")
                        .build())
                .chatMemory(memory)
                .build();

        // 模拟长对话
        for (int i = 1; i <= 20; i++) {
            String response = assistant.chat("我说第" + i + "句话");
            System.out.println("用户: 我说第" + i + "句话");
            System.out.println("AI: " + response);
        }

        // 试着问 AI 还记得第 1 句话吗
        System.out.println("---");
        System.out.println(assistant.chat("我今天说的第一句话是什么？"));
    }
}
```

### 3.3 优缺点分析

| 维度 | 评价 |
|------|------|
| 实现复杂度 | ★☆☆☆☆ 极简 |
| Token 消耗 | 稳定可控（受消息数限制） |
| 长对话效果 | ★★☆☆☆ 早期上下文全部丢失 |
| 适用场景 | 简单问答、客服机器人、对上下文要求不高的场景 |

**致命缺陷**：当对话轮次超过 `maxMessages` 后，早期的关键信息会永久丢失。比如用户一开始说"我的公司代号是 `ABC-2024`"，20 轮之后问 AI 关于这个代号的问题，AI 一脸茫然。

---

## 四、方案二：TokenWindowMemory——按 Token 数裁剪

### 4.1 原理

`TokenWindowMemory` 不同于按**消息条数**裁剪，而是按 **Token 数量**裁剪。这更贴近实际情况——不同的消息长度差异巨大，一条 "OK" 和一段 5000 字的代码需要的 Token 数完全不同。

它内部维护一个 `maxTokens` 阈值，当总 Token 数超过这个值时，会从最早的**单条**消息开始逐条移除，直到总 Token 数降到阈值以下。

> **注意**：它只移除完整的消息，不会把一条消息切成两半——这保证了消息语义的完整性，但也意味着 Token 预算控制是一种"尽力而为"的近似。

### 4.2 完整代码

```java
import dev.langchain4j.memory.chat.TokenWindowChatMemory;
import dev.langchain4j.model.Tokenizer;
import dev.langchain4j.model.openai.OpenAiTokenizer;

public class TokenWindowDemo {

    public interface Assistant {
        String chat(String message);
    }

    public static void main(String[] args) {
        // 设置 Token 上限为 2000
        // 使用 OpenAI 的 tokenizer 来计算 token 数
        Tokenizer tokenizer = new OpenAiTokenizer("gpt-4o-mini");
        ChatMemory memory = TokenWindowChatMemory.withMaxTokens(2000, tokenizer);

        Assistant assistant = AiServices.builder(Assistant.class)
                .chatLanguageModel(OpenAiChatModel.builder()
                        .apiKey(System.getenv("OPENAI_API_KEY"))
                        .modelName("gpt-4o-mini")
                        .build())
                .chatMemory(memory)
                .build();

        // 发送一条超长消息，观察记忆裁剪行为
        String longText = "请你记住以下所有信息：".repeat(200);
        System.out.println("长消息 Token 数（约）: " + tokenizer.estimateTokenCountInText(longText));

        String response = assistant.chat(longText);
        System.out.println("AI: " + response);

        // 查看当前记忆中有多少条消息
        System.out.println("记忆中的消息数: " + memory.messages().size());

        // 继续对话
        response = assistant.chat("我之前让你记住了什么？");
        System.out.println("AI: " + response);
    }
}
```

### 4.3 关键细节

**Token 数计算公式**：
```
实际消耗 = messages().stream()
    .mapToInt(tokenizer::estimateTokenCountInMessage)
    .sum()
    + overhead（每条消息约 3-5 个额外 token）
```

**与 MessageWindowMemory 的核心区别**：

```java
// MessageWindowMemory：最多 10 条消息
MessageWindowChatMemory.withMaxMessages(10);

// TokenWindowMemory：最多 2000 tokens
TokenWindowChatMemory.withMaxTokens(2000, new OpenAiTokenizer("gpt-4o-mini"));
```

前者不管你每条消息多长，只数条数；后者数的是 Token，更精细但也依赖正确的 Tokenizer 实现。

### 4.4 优缺点分析

| 维度 | 评价 |
|------|------|
| Token 成本控制 | ★★★★★ 精确控制 |
| 实现复杂度 | ★★☆☆☆ 需要注入 Tokenizer |
| 语义完整性 | ★★★★☆ 以消息为单位移除 |
| 适用场景 | API 调用预算敏感、长对话 |

**坑点**：不同模型有不同的 Tokenizer（OpenAI 的 cl100k_base vs 其他模型的 tokenizer），**务必匹配**，否则预估 Token 数和实际消耗可能偏差 30% 以上。

---

## 五、方案三：ConversationSummaryMemory——AI 压缩记忆

### 5.1 原理

这是最"智能"的一种方案。它不直接丢弃旧消息，而是**用 LLM 对旧消息做摘要**，只保留摘要 + 最近 N 条消息。

这样既节省了 Token，又保留了早期对话的"精华信息"。

### 5.2 完整代码

```java
import dev.langchain4j.memory.chat.ChatMemoryProvider;
import dev.langchain4j.memory.chat.MessageWindowChatMemory;
import dev.langchain4j.model.openai.OpenAiChatModel;
import dev.langchain4j.service.AiServices;

public class SummaryMemoryDemo {

    public interface Assistant {
        String chat(String message);
    }

    public static void main(String[] args) {
        OpenAiChatModel chatModel = OpenAiChatModel.builder()
                .apiKey(System.getenv("OPENAI_API_KEY"))
                .modelName("gpt-4o-mini")
                .build();

        // 自定义 ChatMemoryProvider：当触发摘要条件时切换记忆
        ChatMemoryProvider memoryProvider = memoryId -> {
            ChatMemory baseMemory = MessageWindowChatMemory.withMaxMessages(6);

            // 包装为支持摘要的记忆
            // 当消息超过 6 条时，触发摘要压缩
            return new SummaryChatMemory(baseMemory, chatModel, 4);
        };

        Assistant assistant = AiServices.builder(Assistant.class)
                .chatLanguageModel(chatModel)
                .chatMemoryProvider(memoryProvider)
                .build();

        // 模拟一个需要记忆的长对话
        String[] inputs = {
            "我叫张三，工号 EMP-2024-0815，部门是 AI 研发部。",
            "我的项目叫 'SkyNet'，是一个智能客服系统，技术栈是 Java + LangChain4j。",
            "项目的截止日期是 2025 年 6 月 30 日。",
            "团队成员还有李四和王五，李四负责前端，王五负责测试。",
            "上周我们遇到了一个 Token 超限的问题。",
            "后来通过 ConversationSummaryMemory 解决了。",
            "帮我回顾一下，我叫什么名字，工号多少？"
        };

        for (String input : inputs) {
            String response = assistant.chat(input);
            System.out.println("用户: " + input);
            System.out.println("AI: " + response);
            System.out.println("---");
        }
    }
}
```

### 5.3 SummaryChatMemory 实现剖析

LangChain4j 并没有内置一个开箱即用的 `ConversationSummaryMemory`（截至 0.35.x），但我们可以基于 `ChatMemory` 接口自行包装。核心逻辑如下：

```java
public class SummaryChatMemory implements ChatMemory {

    private final ChatMemory delegate;       // 底层记忆（如 MessageWindowMemory）
    private final ChatLanguageModel summarizer; // 用于做摘要的 LLM（可用廉价模型）
    private final int summaryTriggerCount;   // 达到多少条消息时触发摘要
    private String currentSummary = "";

    public SummaryChatMemory(ChatMemory delegate,
                             ChatLanguageModel summarizer,
                             int summaryTriggerCount) {
        this.delegate = delegate;
        this.summarizer = summarizer;
        this.summaryTriggerCount = summaryTriggerCount;
    }

    @Override
    public void add(ChatMessage message) {
        delegate.add(message);
        // 当消息数超过阈值时触发摘要
        if (delegate.messages().size() >= summaryTriggerCount) {
            summarize();
        }
    }

    private void summarize() {
        List<ChatMessage> allMessages = delegate.messages();
        if (allMessages.isEmpty()) return;

        // 构建摘要提示词
        StringBuilder prompt = new StringBuilder();
        prompt.append("请将以下对话历史压缩为一段简洁的摘要，保留关键信息：\n\n");
        if (!currentSummary.isEmpty()) {
            prompt.append("之前的摘要：").append(currentSummary).append("\n\n");
        }
        prompt.append("最新对话：\n");
        for (ChatMessage msg : allMessages) {
            prompt.append(msg.type()).append(": ").append(msg.text()).append("\n");
        }

        // 使用 summaries LLM（低成本模型）生成摘要
        String newSummary = summarizer.generate(prompt.toString());
        this.currentSummary = newSummary;

        // 清空底层记忆，只保留摘要作为 SystemMessage
        delegate.clear();
        delegate.add(new SystemMessage("对话历史摘要：" + newSummary));
    }

    @Override
    public List<ChatMessage> messages() {
        return delegate.messages();
    }

    @Override
    public void clear() {
        delegate.clear();
        currentSummary = "";
    }

    @Override
    public Object id() {
        return delegate.id();
    }
}
```

### 5.4 优缺点分析

| 维度 | 评价 |
|------|------|
| 信息保留效果 | ★★★★★ 摘要保留了早期关键信息 |
| Token 成本 | ★★★☆☆ 摘要过程额外消耗 Token |
| 实现复杂度 | ★★★★☆ 需要包装和调试 |
| 摘要质量 | 取决于 LLM 能力（建议用廉价模型如 gpt-4o-mini） |
| 适用场景 | 长时间对话、客户服务、个人助理 |

**技巧**：做摘要的 LLM 和做对话的 LLM 可以分开。摘要用 gpt-4o-mini（便宜），对话用 gpt-4o（贵但效果好），这是业界常见的**双模型策略**。

---

## 六、方案四：ChatMemoryProvider——多用户隔离

### 6.1 原理

前面的方案都假设**只有一个用户**。但实际生产环境中，你的服务同时服务成百上千个用户。每个用户应该有**独立的记忆空间**。

`ChatMemoryProvider` 是一个函数式接口，根据 `memoryId`（通常是用户 ID 或会话 ID）返回对应的 `ChatMemory` 实例：

```java
@FunctionalInterface
public interface ChatMemoryProvider {
    ChatMemory get(Object memoryId);
}
```

### 6.2 完整代码：内存版多用户隔离

```java
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class MultiUserMemoryDemo {

    public interface Assistant {
        String chat(String message);
    }

    public static void main(String[] args) {
        OpenAiChatModel chatModel = OpenAiChatModel.builder()
                .apiKey(System.getenv("OPENAI_API_KEY"))
                .modelName("gpt-4o-mini")
                .build();

        // 用 ConcurrentHashMap 存储每个用户的记忆
        Map<Object, ChatMemory> memoryStore = new ConcurrentHashMap<>();

        ChatMemoryProvider memoryProvider = memoryId -> {
            return memoryStore.computeIfAbsent(memoryId, id ->
                MessageWindowChatMemory.withMaxMessages(10)
            );
        };

        Assistant assistant = AiServices.builder(Assistant.class)
                .chatLanguageModel(chatModel)
                .chatMemoryProvider(memoryProvider)
                .build();

        // 模拟用户 Alice 的对话
        System.out.println("=== Alice 的对话 ===");
        // 注意：AiServices 目前版本中，memoryId 需要在构建时静态指定
        // 或者通过 ThreadLocal/AOP 传递
        Assistant aliceAssistant = AiServices.builder(Assistant.class)
                .chatLanguageModel(chatModel)
                .chatMemory(memoryStore.computeIfAbsent("alice",
                    id -> MessageWindowChatMemory.withMaxMessages(10)))
                .build();

        System.out.println(aliceAssistant.chat("我是 Alice，我在找关于 Java 的资料。"));
        System.out.println(aliceAssistant.chat("你还记得我叫什么吗？"));

        // 模拟用户 Bob 的对话
        System.out.println("\n=== Bob 的对话 ===");
        Assistant bobAssistant = AiServices.builder(Assistant.class)
                .chatLanguageModel(chatModel)
                .chatMemory(memoryStore.computeIfAbsent("bob",
                    id -> MessageWindowChatMemory.withMaxMessages(10)))
                .build();

        System.out.println(bobAssistant.chat("我叫什么名字？"));
        System.out.println(bobAssistant.chat("我是 Bob，在找 Python 的资源。"));
    }
}
```

### 6.3 生产级多用户隔离模式

实际项目中，`ChatMemoryProvider` 通常配合数据库持久化：

```java
@Configuration
public class MemoryConfig {

    @Bean
    public ChatMemoryProvider chatMemoryProvider(MemoryRepository memoryRepo,
                                                  Tokenizer tokenizer) {
        return memoryId -> {
            ChatMemoryStore store = new PersistentChatMemoryStore(memoryRepo, (String) memoryId);
            return TokenWindowChatMemory.builder()
                    .maxTokens(3000, tokenizer)
                    .chatMemoryStore(store)
                    .id(memoryId)
                    .build();
        };
    }
}
```

这样每个用户的记忆既隔离又持久化，服务重启也不会丢失。

### 6.4 适用场景

- SaaS 多租户系统
- 在线客服（每个会话独立记忆）
- 企业内部系统（按员工 ID 隔离）
- 任何需要"一人一记忆"的场景

---

## 七、方案五：持久化方案——服务重启不丢记忆

### 7.1 为什么需要持久化

上述所有方案（MessageWindow/TokenWindow/Summary）默认都使用**内存中**的 `List<ChatMessage>` 存储。服务一重启，所有对话记忆归零。

对于生产环境，你需要把对话历史存到**数据库或缓存**中。LangChain4j 通过 `ChatMemoryStore` 接口支持持久化：

```java
public interface ChatMemoryStore {
    List<ChatMessage> getMessages(Object memoryId);
    void updateMessages(Object memoryId, List<ChatMessage> messages);
    void deleteMessages(Object memoryId);
}
```

### 7.2 完整代码：MySQL 持久化

```java
// ========== 1. 数据库表结构 ==========
// CREATE TABLE chat_memory (
//     memory_id VARCHAR(128) PRIMARY KEY,
//     messages_json TEXT NOT NULL,
//     updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
// );

// ========== 2. 实体类 ==========
@Data
@TableName("chat_memory")
public class ChatMemoryEntity {
    @TableId("memory_id")
    private String memoryId;
    @TableField("messages_json")
    private String messagesJson;
    private LocalDateTime updatedAt;
}

// ========== 3. ChatMemoryStore 实现 ==========
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import dev.langchain4j.data.message.ChatMessage;
import dev.langchain4j.store.memory.chat.ChatMemoryStore;

public class MySqlChatMemoryStore implements ChatMemoryStore {

    private final ChatMemoryMapper mapper;
    private final ObjectMapper objectMapper;

    public MySqlChatMemoryStore(ChatMemoryMapper mapper) {
        this.mapper = mapper;
        this.objectMapper = new ObjectMapper();
    }

    @Override
    public List<ChatMessage> getMessages(Object memoryId) {
        ChatMemoryEntity entity = mapper.selectById((String) memoryId);
        if (entity == null || entity.getMessagesJson() == null) {
            return new ArrayList<>();
        }
        try {
            return objectMapper.readValue(entity.getMessagesJson(),
                    new TypeReference<List<ChatMessage>>() {});
        } catch (Exception e) {
            throw new RuntimeException("反序列化失败", e);
        }
    }

    @Override
    public void updateMessages(Object memoryId, List<ChatMessage> messages) {
        try {
            String json = objectMapper.writeValueAsString(messages);
            ChatMemoryEntity entity = new ChatMemoryEntity();
            entity.setMemoryId((String) memoryId);
            entity.setMessagesJson(json);
            entity.setUpdatedAt(LocalDateTime.now());

            // INSERT ... ON DUPLICATE KEY UPDATE
            mapper.insert(entity);
        } catch (Exception e) {
            throw new RuntimeException("序列化失败", e);
        }
    }

    @Override
    public void deleteMessages(Object memoryId) {
        mapper.deleteById((String) memoryId);
    }
}

// ========== 4. 使用 ==========
public class PersistentMemoryDemo {
    public static void main(String[] args) {
        MySqlChatMemoryStore store = new MySqlChatMemoryStore(chatMemoryMapper);

        ChatMemory memory = MessageWindowChatMemory.builder()
                .maxMessages(10)
                .chatMemoryStore(store)  // 关键：注入持久化存储
                .id("user-1001")         // 指定 memoryId
                .build();
    }
}
```

### 7.3 Redis 持久化（适合高并发）

```java
public class RedisChatMemoryStore implements ChatMemoryStore {

    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;
    private static final Duration TTL = Duration.ofDays(7); // 7天过期

    @Override
    public List<ChatMessage> getMessages(Object memoryId) {
        String json = redisTemplate.opsForValue().get("chat_memory:" + memoryId);
        if (json == null) return new ArrayList<>();
        try {
            return objectMapper.readValue(json, new TypeReference<List<ChatMessage>>() {});
        } catch (Exception e) {
            return new ArrayList<>();
        }
    }

    @Override
    public void updateMessages(Object memoryId, List<ChatMessage> messages) {
        try {
            String json = objectMapper.writeValueAsString(messages);
            redisTemplate.opsForValue().set("chat_memory:" + memoryId, json, TTL);
        } catch (Exception e) {
            throw new RuntimeException("Redis 存储异常", e);
        }
    }

    @Override
    public void deleteMessages(Object memoryId) {
        redisTemplate.delete("chat_memory:" + memoryId);
    }
}
```

> **安全提示**：`ChatMessage` 序列化时需要注意，LangChain4j 内部使用 Jackson 的 `@JsonTypeInfo` 注解来标记消息类型（`USER`/`AI`/`SYSTEM`/`TOOL_EXECUTION_RESULT`），直接序列化到 JSON 即可还原。

### 7.4 优缺点

| 维度 | MySQL | Redis |
|------|-------|-------|
| 持久性 | ★★★★★ 永久存储 | ★★★☆☆ 可配置 TTL |
| 读写性能 | ★★★☆☆ 依赖索引 | ★★★★★ 内存级 |
| 运维成本 | ★★☆☆☆ 需建表 | ★★★☆☆ 配置简单 |
| 适用场景 | 长期记忆、审计需求 | 高并发、短期会话 |

---

## 八、5 种方案终极对比

| 方案 | 记忆策略 | 控制维度 | 信息损失 | Token成本 | 持久化 | 多用户 |
|------|---------|---------|---------|----------|-------|-------|
| MessageWindowMemory | 滑动窗口（条数） | 消息条数 | 高 | 低 | 需配合Store | 需配合Provider |
| TokenWindowMemory | 滑动窗口（Token） | Token数 | 中 | 可控 | 需配合Store | 需配合Provider |
| ConversationSummaryMemory | AI摘要+窗口 | 摘要+条数 | 低 | 中（摘要有成本） | 需配合Store | 需配合Provider |
| ChatMemoryProvider | 代理模式 | 任意 | 取决于被代理方案 | 取决于被代理方案 | 可配合Store | ★★★★★ 原生支持 |
| 持久化方案(Store) | 取决于下游Memory | 取决于下游Memory | 取决于下游Memory | 额外IO开销 | ★★★★★ | 天然隔离 |

---

## 九、实战决策树：你的场景该选哪种？

```
你的场景是什么？
├── 简单问答/聊天机器人（用户不纠结前后文）
│   └── MessageWindowMemory（maxMessages=10~20）+ Redis 持久化
│
├── API 预算敏感/长对话/成本控制严格
│   └── TokenWindowMemory（maxTokens=3000~4000）+ MySQL 持久化
│
├── 长时间深度对话/个人AI助理/帮助台
│   └── ConversationSummaryMemory（summaryTriggerCount=8~12）
│       + MySQL 持久化 + ChatMemoryProvider（多用户）
│
├── SaaS 多租户
│   └── ChatMemoryProvider + TokenWindowMemory + Redis
│
└── 需要审计/系统日志
    └── TokenWindowMemory + MySQL 持久化 + 另外完整日志表
```

---

## 十、常见踩坑与最佳实践

### 坑1：ChatMemory 不是线程安全的
默认的 `MessageWindowChatMemory` 和 `TokenWindowChatMemory` **不是线程安全的**。多线程环境下，请用 `ChatMemoryProvider` 为每个线程（或每个用户）创建独立实例。

### 坑2：SystemMessage 也会被裁剪
裁剪机制一视同仁——SystemMessage（系统提示词）也在窗口内。如果你的 System 提示词很重要，考虑把 SystemMessage 放在 Provider 层注入，而不是放在 ChatMemory 中：

```java
ChatMemoryProvider provider = memoryId -> {
    ChatMemory memory = MessageWindowChatMemory.withMaxMessages(10);
    // 系统提示词单独维护，不参与裁剪
    memory.add(SystemMessage.from("你是专业客服助手，保持礼貌。"));
    return memory;
};
```

### 坑3：Tokenizer 与模型不匹配
如果你用 OpenAI 的 Tokenizer 给 Anthropic 模型估 Token，实际消耗可能偏差 50%。**必须使用对应模型的 Tokenizer**。

### 坑4：摘要模型的"二次失真"
`ConversationSummaryMemory` 的摘要质量取决于做摘要的模型。如果摘要模型不够好，经过多次摘要后信息会严重变形。建议：
- 摘要模型和对话模型**分开**（摘要用便宜模型）。
- 定期（如每 20 轮）全面清理和重置记忆。

### 最佳实践 checklist

1. ✅ 生产环境**必须持久化**（至少用 Redis）
2. ✅ 用 `ChatMemoryProvider` 做**多用户隔离**
3. ✅ 敏感场景用 `TokenWindowMemory` 控制成本
4. ✅ 复杂对话场景用 `ConversationSummaryMemory`
5. ✅ SystemMessage **不要放在** ChatMemory 的可裁剪区域
6. ✅ 匹配正确的 **Tokenizer**
7. ✅ 对话历史设 TTL，**避免无限膨胀**

---

## 十一、写在最后

Memory 是构建"智能" AI 应用的基石。没有记忆，你的 AI 永远只是一个"高级函数调用器"；有了合适的记忆方案，它才能真正像一个"助理"一样理解上下文、记住偏好。

选型没有那么复杂：**简单场景用 MessageWindow，预算敏感用 TokenWindow，长对话用 Summary，多用户用 Provider，生产环境统一加持久化。**

记住一句话：**记忆方案不是在"选最好的"，而是在"选最合适的"**。

---

**下一篇预告**：下一篇我们聊聊 LangChain4j 的 **Tools（函数调用）**——给 AI 一双能干活的手。让 AI 不只是"会聊天"，更能"操作数据库、发邮件、调接口"。会聊天的 AI 千篇一律，能干活的 AI 万里挑一。

> 如果这篇文章对你有帮助，欢迎点赞、收藏、转发。有任何问题评论区见！
