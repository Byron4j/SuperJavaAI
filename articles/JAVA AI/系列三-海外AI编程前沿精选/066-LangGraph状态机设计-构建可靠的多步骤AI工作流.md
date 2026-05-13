# LangGraph 状态机设计：构建可靠的多步骤 AI 工作流，告别"AI 做到一半就忘了"

> 让 ChatGPT 写一篇技术文章只需要一次对话，但要让它独立完成 **"分析需求 → 设计方案 → 生成代码 → 写测试 → 自动部署"** 的完整链路呢？你会发现 LLM 很容易"走到一半就忘了"，或者遇到错误时不知道该回退到哪一步。这就是 LangGraph 要解决的核心问题——**用状态机给 LLM 增加结构化的流程控制能力**。

---

## 一、LangGraph 是什么？为什么需要它？

### 1.1 LangChain 的困境

用过 LangChain 的开发者可能深有体会：Chain 能处理一些简单的线性管道，但一旦需求稍微复杂——比如需要条件分支、需要循环重试、需要人工审批——Chain 就力不从心了。

```python
# LangChain Chain 的局限：只能线性执行
from langchain.chains import SimpleSequentialChain

chain = SimpleSequentialChain(chains=[analyze, design, implement, test, deploy])
# 问题：
#   1. 如果 test 失败了怎么办？无法回退
#   2. 如果 design 产出两种方案怎么办？无法分支
#   3. 如果流程执行到一半服务重启怎么办？状态全丢
```

### 1.2 LangGraph 的答案

LangGraph 是 LangChain 生态中专门负责 **工作流编排** 的库。它的核心是对 **有限状态机** 的一层封装，让你用节点 + 边的模式定义复杂流程：

```python
from langgraph.graph import StateGraph, END

# 一个典型的 LangGraph 工作流
def analyze(state):
    """分析需求"""
    return {"requirement": state["input"]}

def design(state):
    """设计方案"""
    return {"design_doc": f"设计方案：基于{state['requirement']}"}

def implement(state):
    """生成代码"""
    return {"code": f"// 实现：{state['design_doc']}"}

def test(state):
    """测试代码，返回是否通过"""
    passed = "error" not in state["code"]
    return {"test_passed": passed}

def deploy(state):
    """部署到生产"""
    return {"deployed": True}

# 构建工作流
builder = StateGraph(dict)
builder.add_node("analyze", analyze)
builder.add_node("design", design)
builder.add_node("implement", implement)
builder.add_node("test", test)
builder.add_node("deploy", deploy)

# 定义边
builder.set_entry_point("analyze")
builder.add_edge("analyze", "design")
builder.add_edge("design", "implement")
builder.add_edge("implement", "test")

# 条件边：测试通过→部署，测试失败→回到implement重试
builder.add_conditional_edges(
    "test",
    lambda state: "deploy" if state["test_passed"] else "implement",
    {"deploy": "deploy", "implement": "implement"}
)

builder.add_edge("deploy", END)

graph = builder.compile()
result = graph.invoke({"input": "开发一个用户管理系统"})
```

---

## 二、StateGraph 核心概念：一个共享的上下文容器

### 2.1 State 设计

LangGraph 的 `StateGraph` 本质上是一个类型化的字典，在整个工作流执行期间自动在各个节点之间传递：

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    """工作流全局状态"""
    messages: Annotated[list, add_messages]   # 对话历史，自动 append
    current_step: str                          # 当前步骤
    requirement: str                           # 需求分析结果
    design_doc: str                            # 设计方案
    code: str                                  # 生成的代码
    test_result: str                           # 测试结果
    retry_count: int                           # 重试次数
    human_approved: bool                       # 人工审批状态
```

这就是整个工作流的 **"记忆"**——每个节点都可以读/写这个 State，从而实现节点间通信。

### 2.2 状态合并策略

LangGraph 的一个精妙设计是 **Reducer 机制**——类似 MapReduce 中的 Reduce 步骤，定义了同一字段被多个节点更新时的合并策略：

```python
from typing import Annotated
from langgraph.graph import add_messages

# add_messages 是一个 Reducer：新消息追加到列表末尾
class State(TypedDict):
    messages: Annotated[list, add_messages]
    score: Annotated[int, lambda current, new: current + new]  # 累加
    errors: Annotated[list, lambda current, new: current + new] # 拼接

# 实际效果
state = {"messages": ["A"], "score": 5, "errors": ["err1"]}
# 某个节点返回：{"messages": ["B"], "score": 3, "errors": ["err2"]}
# 合并后：{"messages": ["A", "B"], "score": 8, "errors": ["err1", "err2"]}
```

这个机制让并行节点的状态合并变得极其简单——多个并行执行的分支可以同时写入同一个字段，Reducer 自动合并。

---

## 三、节点与边的设计模式

### 3.1 节点：工作流的最小执行单元

节点是纯函数（或可调用对象），输入是当前 State，返回是 State 的子集（需要更新的字段）：

```python
def code_generator(state: AgentState) -> dict:
    """代码生成节点"""
    prompt = f"""
    根据以下需求生成 Java 代码：
    {state['requirement']}

    设计方案：
    {state['design_doc']}
    """
    model = ChatOpenAI(model="gpt-4")
    response = model.invoke(prompt)

    return {
        "code": response.content,
        "current_step": "implement",
        "messages": [AIMessage(content=f"代码已生成")]
    }
```

### 3.2 普通边：固定流程

```python
builder.add_edge("analyze", "design")   # analyze 完成后一定走 design
builder.add_edge("design", "implement") # design 完成后一定走 implement
```

### 3.3 条件边：动态路由

这是 LangGraph 最强大的特性之一——根据 State 内容决定下一步：

```python
def should_retry(state: AgentState) -> str:
    """决定是否需要重试"""
    if state["test_result"] == "PASSED":
        return "deploy"
    elif state["retry_count"] < 3:
        return "implement"  # 回退到实现节点，重新生成代码
    else:
        return "human_review"  # 重试 3 次还不行，走人工审批

builder.add_conditional_edges(
    "test",
    should_retry,                        # 路由函数
    {
        "deploy": "deploy",
        "implement": "implement",
        "human_review": "human_review"
    }
)
```

### 3.4 循环与重试

结合条件边，很容易实现循环逻辑：

```python
def should_continue_fix(state: AgentState) -> str:
    x = state["retry_count"]
    return "end" if x >= 3 else "continue"

builder.add_node("fix_bug", fix_bug)
builder.add_conditional_edges("fix_bug", should_continue_fix, {
    "continue": "fix_bug",   # 循环：继续修 bug
    "end": "deploy"
})
```

---

## 四、持久化与断点续传：生产级可靠性

### 4.1 为什么需要 Checkpoint？

真实的工作流可能执行几分钟甚至几小时。如果中间节点出错（API 超时、LLM 调用失败、网络中断）或者服务重启——**没有 Checkpoint，所有中间状态都会丢失**，只能从头再来。

### 4.2 LangGraph 的 Checkpointer 机制

```python
from langgraph.checkpoint.sqlite import SqliteSaver

# 1. 创建 SQLite 持久化存储
checkpointer = SqliteSaver.from_conn_string("checkpoints.db")

# 2. 编译 graph 时挂载 checkpointer
graph = builder.compile(checkpointer=checkpointer)

# 3. 执行时指定 thread_id（每个会话一个 ID）
config = {"configurable": {"thread_id": "conversation-123"}}
graph.invoke({"input": "开发用户系统"}, config=config)

# 4. 断点续传——用同样的 thread_id 从上次中断处继续
graph.invoke({"input": "继续之前的任务"}, config=config)
```

### 4.3 断点续传的底层实现

```python
# 源码简化：checkpoint 的存储结构
class Checkpoint:
    """一个工作流快照"""
    id: str
    ts: str  # 时间戳
    channel_values: dict  # 当前 State
    channel_versions: dict  # 每个 channel 的版本号
    versions_seen: dict  # 已处理的版本
    parent_checkpoint_id: str | None  # 父快照 ID（形成一条链）

# SQLite 表结构
# CREATE TABLE checkpoints (
#     thread_id TEXT NOT NULL,
#     checkpoint_id TEXT NOT NULL,
#     parent_id TEXT,
#     checkpoint BLOB,
#     metadata BLOB,
#     PRIMARY KEY (thread_id, checkpoint_id)
# );
```

当服务重启后，`graph.invoke()` 会自动从该 `thread_id` 的最新 checkpoint 恢复状态，继续执行未完成的节点。

### 4.4 带人工审批的断点

```python
from langgraph.types import interrupt

def human_review_node(state: AgentState) -> dict:
    """人工审批节点——等待外部确认"""
    # interrupt() 会暂停工作流，等待外部输入
    user_decision = interrupt({
        "question": f"请审核此代码：{state['code']}",
        "options": ["approve", "reject"]
    })
    if user_decision == "approve":
        return {"human_approved": True}
    else:
        return {"human_approved": False}
```

执行后调用方通过 `graph.update_state()` 投递审批结果，工作流继续执行——这就是 LangGraph 实现 Human-in-the-Loop 的核心机制。

---

## 五、Java 开发者如何实现类似机制

### 5.1 基于 Spring State Machine

```java
@Configuration
@EnableStateMachineFactory
public class WorkflowConfig extends StateMachineConfigurerAdapter<String, String> {

    @Override
    public void configure(StateMachineStateConfigurer<String, String> states)
            throws Exception {
        states
            .withStates()
            .initial("ANALYZE")
            .state("DESIGN")
            .state("IMPLEMENT")
            .state("TEST")
            .state("DEPLOY")
            .end("END");
    }

    @Override
    public void configure(StateMachineTransitionConfigurer<String, String> transitions)
            throws Exception {
        transitions
            .withExternal().source("ANALYZE").target("DESIGN")
            .and()
            .withExternal().source("DESIGN").target("IMPLEMENT")
            .and()
            .withExternal().source("IMPLEMENT").target("TEST")
            .and()
            .withExternal().source("TEST").target("DEPLOY")
            .guard(ctx -> ctx.getExtendedState().get("testPassed", Boolean.class))
            .and()
            .withExternal().source("TEST").target("IMPLEMENT")
            .guard(ctx -> !ctx.getExtendedState().get("testPassed", Boolean.class));
    }
}
```

### 5.2 自定义轻量状态机

如果不想要 Spring State Machine 的重量级依赖，自己实现一个也很简单：

```java
public class SimpleStateMachine<S, C> {
    private final Map<S, Map<S, Function<C, Boolean>>> transitions = new HashMap<>();
    private final Map<S, Consumer<C>> actions = new HashMap<>();
    private S currentState;

    public SimpleStateMachine<S, C> addTransition(S from, S to, Function<C, Boolean> guard) {
        transitions.computeIfAbsent(from, k -> new HashMap<>()).put(to, guard);
        return this;
    }

    public SimpleStateMachine<S, C> addAction(S state, Consumer<C> action) {
        actions.put(state, action);
        return this;
    }

    public void start(C context, S initialState) {
        this.currentState = initialState;
        while (currentState != null) {
            // 执行当前状态的 Action
            actions.getOrDefault(currentState, c -> {}).accept(context);

            // 找到可跳转的下一状态
            S nextState = null;
            Map<S, Function<C, Boolean>> outgoing = transitions.get(currentState);
            if (outgoing != null) {
                for (var entry : outgoing.entrySet()) {
                    if (entry.getValue().apply(context)) {
                        nextState = entry.getKey();
                        break;
                    }
                }
            }
            currentState = nextState;
        }
    }
}
```

### 5.3 用 Redis 做 Checkpoint 持久化

```java
@Service
public class WorkflowCheckpointService {

    @Autowired
    private StringRedisTemplate redisTemplate;

    public void saveCheckpoint(String threadId, String state, WorkflowContext ctx) {
        String key = "workflow:checkpoint:" + threadId;
        Map<String, String> data = Map.of(
            "current_state", state,
            "context", objectMapper.writeValueAsString(ctx)
        );
        redisTemplate.opsForHash().putAll(key, data);
        redisTemplate.expire(key, Duration.ofHours(24));
    }

    public Checkpoint loadCheckpoint(String threadId) {
        String key = "workflow:checkpoint:" + threadId;
        Map<Object, Object> data = redisTemplate.opsForHash().entries(key);
        if (data.isEmpty()) return null;
        return new Checkpoint(
            (String) data.get("current_state"),
            objectMapper.readValue((String) data.get("context"), WorkflowContext.class)
        );
    }
}
```

---

## 总结

LangGraph 用状态机模式解决了 LLM 应用的三大痛点：

| 痛点 | LangGraph 的方案 | 效果 |
|------|-----------------|------|
| 流程不可控 | 节点 + 边 + 条件分支 | 定义清晰的执行路径 |
| 出错无回退 | 条件边 + 循环 + 重试 | 失败自动回退重试 |
| 状态丢失 | Checkpointer 持久化 | 断点续传，服务重启不丢状态 |

**一句话**：LangGraph = 有限状态机 + LLM + 持久化。它让 AI Agent 从一个"凭感觉做事的新手"变成了一个"按照 SOP 严格执行的专业团队"。

Java 开发者完全可以用 Spring State Machine 或自研轻量状态机实现类似的能力——核心思想不在 LangGraph 本身，而在于 **"给 LLM 流程加结构"** 这个思路。

---

## 下篇预告

我们聊了 Dify（编排）、Ollama（部署）、vLLM（高性能推理）、LangGraph（状态机），这些工程实践都建立在对 Transformer 模型的深刻理解之上。下一篇文章，我们回归本源，用 Java 开发者的视角重新解读 **Attention 机制的核心论文**——抛开数学公式，用人话讲清楚 Q、K、V 到底在算什么。关注我，干货不断！

---

*本文基于 LangGraph v0.2.x 源码分析。欢迎评论区讨论你的 Agent 开发经验。*
