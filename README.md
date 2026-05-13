# CSDN技术博客内容索引

> 此博客输出内容永久免费，欢迎转载，如有转载则必须附上原文链接!
> 
> 基于 CookBook 知识体系构建：https://github.com/Byron4j/CookBook
> 
> **质量状态：✅ 全部123篇文章已通过H1深度质量标准检查并优化（2026-05-05）**
> - Java技术栈（70篇）：全部补充"常见陷阱与最佳实践"+"面试题与参考答案"
> - AI相关文章（49篇）：全部升级至2026年最新模型（GPT-5.5/Claude Opus 4.7/DeepSeek-V4/GLM-5/Kimi K2.6等）
> - 质量标准详见：[QUALITY_STANDARD.md](QUALITY_STANDARD.md)

## 输出策略

- **阶段一（第1-12周）**：Java基础模块 A-C（30篇）
- **阶段二（第13-24周）**：Java进阶模块 D-G（40篇）
- **阶段三（第25-32周）**：AI基础与工具 H-I（16篇）
- **阶段四（第33-44周）**：AI编程深度评测 J（12篇）
- **阶段五（第45-52周）**：AI工程化与创意 K-M（20篇）
- **持续更新**：热点快讯 N（每周0-1篇，灵活响应）

---

## 第一阶段：Java基础模块（第1-12周）

### 模块A：Java核心机制（10篇）

| 序号 | 标题 | 状态 | 文件路径 |
|------|------|------|----------|
| A1 | Java枚举：被低估的类型安全利器 | ✅ | articles/A1-Java枚举.md |
| A2 | Java注解：从自定义到框架开发 | ✅ | articles/A2-Java注解.md |
| A3 | Java反射：运行时探查的艺术与代价 | ✅ | articles/A3-Java反射.md |
| A4 | 动态代理深度解析：JDK vs CGLIB | ✅ | articles/A4-动态代理.md |
| A5 | 线程池核心参数与调优实战 | ✅ | articles/A5-线程池.md |
| A6 | AQS源码解读：并发编程的基石 | ✅ | articles/A6-AQS源码解读.md |
| A7 | synchronized与volatile：并发安全双璧 | ✅ | articles/A7-synchronized与volatile.md |
| A8 | Java内存模型：Happens-Before规则 | ✅ | articles/A8-Java内存模型.md |
| A9 | CompletableFuture：异步编程范式 | ✅ | articles/A9-CompletableFuture.md |
| A10 | ThreadLocal：线程隔离的实现与陷阱 | ✅ | articles/A10-ThreadLocal.md |

### 模块B：数据结构与算法（10篇）

| 序号 | 标题 | 状态 | 文件路径 |
|------|------|------|----------|
| B1 | 数组与链表：从基础到工程实践 | ✅ | articles/B1-数组与链表.md |
| B2 | 栈与队列：经典问题与解题模板 | ✅ | articles/B2-栈与队列.md |
| B3 | HashMap源码解析：哈希冲突与扩容机制 | ✅ | articles/B3-HashMap源码解析.md |
| B4 | 二叉树与BST：遍历与平衡策略 | ✅ | articles/B4-二叉树与BST.md |
| B5 | 红黑树与AVL树：自平衡的艺术 | ✅ | articles/B5-红黑树与AVL树.md |
| B6 | B+树与InnoDB索引：数据库查询加速器 | ✅ | articles/B6-B+树与InnoDB索引.md |
| B7 | 快速排序与归并排序：分治思想应用 | ✅ | articles/B7-快速排序与归并排序.md |
| B8 | 堆排序与TopK问题：优先队列实现 | ✅ | articles/B8-堆排序与TopK问题.md |
| B9 | 动态规划：从爬楼梯到0-1背包 | ✅ | articles/B9-动态规划.md |
| B10 | 限流算法：计数器、滑动窗口与令牌桶 | ✅ | articles/B10-限流算法.md |

### 模块C：Spring生态基础（10篇）

| 序号 | 标题 | 状态 | 文件路径 |
|------|------|------|----------|
| C1 | Spring IOC：从XML到注解，容器启动全链路 | ✅ | articles/C1-Spring-IOC.md |
| C2 | Spring Bean生命周期与循环依赖解决 | ✅ | articles/C2-Spring-Bean生命周期.md |
| C3 | Spring AOP原理：代理模式与切面织入 | ✅ | articles/C3-Spring-AOP.md |
| C4 | Spring事务传播机制与失效场景 | ✅ | articles/C4-Spring事务.md |
| C5 | Spring MVC请求处理流程 | ✅ | articles/C5-Spring-MVC.md |
| C6 | Spring Boot自动装配原理 | ✅ | articles/C6-SpringBoot自动装配.md |
| C7 | Spring Boot起步依赖与Actuator | ✅ | articles/C7-SpringBoot起步依赖.md |
| C8 | Spring Cloud Eureka：服务注册与发现 | ✅ | articles/C8-SpringCloud-Eureka.md |
| C9 | Spring Cloud Ribbon与OpenFeign | ✅ | articles/C9-SpringCloud-Ribbon.md |
| C10 | Spring Cloud Gateway：统一网关 | ✅ | articles/C10-SpringCloud-Gateway.md |

---

## 第二阶段：Java进阶模块（第13-24周）

### 模块D：数据库与缓存（10篇）

| 序号 | 标题 | 状态 | 文件路径 |
|------|------|------|----------|
| D1 | MySQL存储引擎对比：InnoDB vs MyISAM | ✅ | articles/D1-MySQL存储引擎.md |
| D2 | InnoDB索引原理：聚簇索引与覆盖索引 | ✅ | articles/D2-InnoDB索引原理.md |
| D3 | MySQL事务隔离级别与MVCC机制 | ✅ | articles/D3-MySQL事务隔离级别.md |
| D4 | MySQL锁机制：行锁、间隙锁与死锁分析 | ✅ | articles/D4-MySQL锁机制.md |
| D5 | MySQL慢查询优化：Explain与索引策略 | ✅ | articles/D5-MySQL慢查询优化.md |
| D6 | Redis数据类型与底层数据结构 | ✅ | articles/D6-Redis数据类型.md |
| D7 | Redis持久化：RDB与AOF的选择 | ✅ | articles/D7-Redis持久化.md |
| D8 | Redis缓存问题：穿透、击穿与雪崩 | ✅ | articles/D8-Redis缓存问题.md |
| D9 | Redis分布式锁：Redisson原理与RedLock争议 | ✅ | articles/D9-Redis分布式锁.md |
| D10 | Redis集群模式：主从、哨兵与Cluster | ✅ | articles/D10-Redis集群模式.md |

### 模块E：分布式系统与中间件（12篇）

| 序号 | 标题 | 状态 | 文件路径 |
|------|------|------|----------|
| E1 | 分布式系统理论基础：CAP与BASE | ✅ | articles/E1-CAP与BASE.md |
| E2 | 分布式一致性算法：Paxos与Raft | ✅ | articles/E2-Paxos与Raft.md |
| E3 | 分布式ID生成：Snowflake与Leaf | ✅ | articles/E3-分布式ID生成.md |
| E4 | 分布式锁方案对比：Redis vs ZK vs etcd | ✅ | articles/E4-分布式锁方案对比.md |
| E5 | Zookeeper：分布式协调服务原理 | ✅ | articles/E5-Zookeeper.md |
| E6 | Dubbo：RPC框架设计与服务治理 | ✅ | articles/E6-Dubbo.md |
| E7 | RocketMQ核心架构：NameServer、Broker与存储 | ✅ | articles/E7-RocketMQ核心架构.md |
| E8 | RocketMQ生产者与消费者最佳实践 | ✅ | articles/E8-RocketMQ最佳实践.md |
| E9 | Kafka：高吞吐消息队列设计 | ✅ | articles/E9-Kafka.md |
| E10 | Netty：NIO网络编程框架 | ✅ | articles/E10-Netty.md |
| E11 | Nginx：反向代理与负载均衡配置 | ✅ | articles/E11-Nginx.md |
| E12 | Docker容器化部署实战 | ✅ | articles/E12-Docker部署实战.md |

### 模块F：JVM与性能优化（8篇）

| 序号 | 标题 | 状态 | 文件路径 |
|------|------|------|----------|
| F1 | JVM内存模型：堆、栈、方法区与元空间 | ✅ | articles/F1-JVM内存模型.md |
| F2 | 垃圾回收算法：标记清除、复制、标记整理 | ✅ | articles/F2-垃圾回收算法.md |
| F3 | CMS与G1垃圾收集器对比 | ✅ | articles/F3-CMS与G1.md |
| F4 | JVM调优实战：GC日志分析与内存泄漏排查 | ✅ | articles/F4-JVM调优实战.md |
| F5 | JVM参数配置：堆大小、GC策略与监控 | ✅ | articles/F5-JVM参数配置.md |
| F6 | 类加载机制：双亲委派与打破委派 | ✅ | articles/F6-类加载机制.md |
| F7 | Java字节码：javassist与动态编织 | ✅ | articles/F7-Java字节码.md |
| F8 | 线上问题排查：Arthas工具使用指南 | ✅ | articles/F8-Arthas工具.md |

### 模块G：架构设计与实践（10篇）

| 序号 | 标题 | 状态 | 文件路径 |
|------|------|------|----------|
| G1 | 微服务架构设计原则与拆分策略 | ✅ | articles/G1-微服务架构设计.md |
| G2 | 高可用架构：限流、降级与熔断 | ✅ | articles/G2-高可用架构.md |
| G3 | 分布式事务：2PC、TCC与Saga | ✅ | articles/G3-分布式事务.md |
| G4 | 服务网格：Istio与Service Mesh | ✅ | articles/G4-服务网格.md |
| G5 | 云原生架构：Kubernetes入门与实践 | ✅ | articles/G5-云原生架构.md |
| G6 | 设计模式：单例、工厂与观察者 | ✅ | articles/G6-设计模式1.md |
| G7 | 设计模式：策略、模板与责任链 | ✅ | articles/G7-设计模式2.md |
| G8 | 架构演进：从单体到微服务实践路径 | ✅ | articles/G8-架构演进.md |
| G9 | 缓存一致性：Cache Aside与双写一致性 | ✅ | articles/G9-缓存一致性.md |
| G10 | 系统监控与可观测性 | ✅ | articles/G10-系统监控.md |

---

## 第三阶段：AI基础与工具（第25-32周）

### 模块H：大语言模型基础（8篇）

| 序号 | 标题 | 状态 | 文件路径 |
|------|------|------|----------|
| H1 | 提示词工程深度解析：从原理到工业级实践 | ✅ | articles/H1-提示词工程.md |
| H2 | 国产大模型全景图：DeepSeek、Kimi、GLM等生态概览 | ✅ | articles/H2-国产大模型全景图.md |
| H3 | DeepSeek全系列解析：V4、V3、R1、Coder定位与差异 | ✅ | articles/H3-DeepSeek全系列.md |
| H4 | Kimi K2.6实战：代码能力与Agent性能全面评测 | ✅ | articles/H4-Kimi-K1.5实战.md |
| H5 | GLM-5与CodeGeeX：智谱AI Agentic Engineering生态 | ✅ | articles/H5-GLM-4与CodeGeeX.md |
| H6 | 其他国产模型速览：Xiaomi Mimo、Minimax、豆包、讯飞星火 | ✅ | articles/H6-其他国产模型.md |
| H7 | API调用实战：OpenAI/DeepSeek/智谱/月之暗面API对比 | ✅ | articles/H7-API调用实战.md |
| H8 | 本地部署入门：Ollama + 各类开源模型快速体验 | ✅ | articles/H8-本地部署入门.md |

### 模块I：AI编程工具与IDE生态（8篇）

| 序号 | 标题 | 状态 | 文件路径 |
|------|------|------|----------|
| I1 | Cursor 3深度指南：Agentic编程与Composer 2 | ✅ | articles/I1-Cursor-IDE.md |
| I2 | GitHub Copilot(GPT-5.3-Codex) vs 国产插件横评 | ✅ | articles/I2-Copilot-vs-国产插件.md |
| I3 | JetBrains/VSCode AI插件配置：GPT-5.4/Claude Opus 4.6 | ✅ | articles/I3-AI插件配置.md |
| I4 | 云端AI编程平台：GitHub Codespaces、Codeium、Replit Agent | ✅ | articles/I4-云端AI编程平台.md |
| I5 | AI辅助Debug：报错分析、日志解读、性能诊断 | ✅ | articles/I5-AI辅助Debug.md |
| I6 | AI代码审查：自动化Code Review与质量检测 | ✅ | articles/I6-AI代码审查.md |
| I7 | AI生成技术文档：API文档、README、注释自动化 | ✅ | articles/I7-AI生成技术文档.md |
| I8 | 低代码+AI：快速搭建原型与业务系统 | ✅ | articles/I8-低代码+AI.md |

---

## 第四阶段：AI编程深度评测（第33-44周）

### 模块J：大模型编程能力深度评测（12篇，核心模块）

| 序号 | 标题 | 评测模型 | 状态 | 文件路径 |
|------|------|---------|------|----------|
| J1 | 国际顶尖模型编程能力全解析 | GPT-5.5 / Claude Opus 4.7 / Gemini 3 Pro | ✅ | articles/J1-国际顶尖模型编程评测.md |
| J2 | DeepSeek-Coder-V2/V3深度实测 | DeepSeek-Coder-V2 / V3 / R1 | ✅ | articles/J2-DeepSeek-Coder实测.md |
| J3 | Kimi K2.6编程专项评测 | Kimi K2.6 / Agent Swarm | ✅ | articles/J3-Kimi-K1.5编程评测.md |
| J4 | GLM-5/CodeGeeX-5编程实测 | GLM-5 / AutoGLM / CodeGeeX-5 | ✅ | articles/J4-GLM-4-CodeGeeX实测.md |
| J5 | 通义千问Qwen3-Coder评测 | Qwen3-Coder-72B / Qwen3-Plus | ✅ | articles/J5-Qwen2.5-Coder评测.md |
| J6 | 文心ERNIE 4.5/文心快码3.0企业级应用 | ERNIE 4.5 / 文心快码3.0 / Baidu Comate | ✅ | articles/J6-文心ERNIE评测.md |
| J7 | Xiaomi Mimo-2.5 Pro模型探索 | Xiaomi Mimo-2.5 Pro | ✅ | articles/J7-Xiaomi-Mimo探索.md |
| J8 | MiniMax M2.7编程与多模态 | MiniMax M2.7 / M2.5 / M2-Her | ✅ | articles/J8-Minimax评测.md |
| J9 | 其他国产模型补充评测 | 华为盘古 / 字节豆包 / 讯飞星火 / 腾讯混元 / 商汤日日新 | ✅ | articles/J9-其他国产模型评测.md |
| J10 | 开源代码模型本地部署实测 | DeepSeek-V4 / Qwen3-72B / GLM-5 / CodeLlama-70B | ✅ | articles/J10-开源模型本地部署.md |
| J11 | 编程场景模型选型指南：算法/业务/架构/运维 | 全模型横评 | ✅ | articles/J11-编程场景选型指南.md |
| J12 | 垂直领域模型：SQL/正则/Shell专用模型对比 | SQLCoder / RegexAI / ShellGPT / Text2SQL | ✅ | articles/J12-垂直领域模型对比.md |

---

## 第五阶段：AI工程化与创意（第45-52周）

### 模块K：AI编程进阶与工程化（8篇）

| 序号 | 标题 | 状态 | 文件路径 |
|------|------|------|----------|
| K1 | 从AI生成到生产代码：代码审查、测试、重构流程 | ✅ | articles/K1-AI代码工程化.md |
| K2 | AI代码安全：幻觉代码识别、漏洞检测、版权合规 | ✅ | articles/K2-AI代码安全.md |
| K3 | 多模型协作策略：简单/复杂/创意任务最优分配 | ✅ | articles/K3-多模型协作策略.md |
| K4 | RAG在私有代码库中的应用：企业代码问答系统 | ✅ | articles/K4-RAG私有代码库.md |
| K5 | AI Agent编程：Devin、OpenDevin、AutoGPT自主开发 | ✅ | articles/K5-AI-Agent编程.md |
| K6 | 模型微调实战：用企业代码库微调DeepSeek/Qwen/GLM | ✅ | articles/K6-模型微调实战.md |
| K7 | 编程大模型的幻觉问题：典型案例分析与规避 | ✅ | articles/K7-大模型幻觉问题.md |
| K8 | 企业级AI编程落地：团队规范、成本核算与效果评估 | ✅ | articles/K8-企业级AI编程落地.md |

### 模块L：AI创意与内容生成（6篇）

| 序号 | 标题 | 状态 | 文件路径 |
|------|------|------|----------|
| L1 | Midjourney/Stable Diffusion：AI绘画入门与商业应用 | ✅ | articles/L1-AI绘画入门.md |
| L2 | Sora/可灵/即梦：AI视频生成技术解析 | ✅ | articles/L2-AI视频生成.md |
| L3 | Suno/Udio：AI音乐生成工具使用指南 | ✅ | articles/L3-AI音乐生成.md |
| L4 | Gamma/ChatPPT：AI演示文稿与文档生成 | ✅ | articles/L4-AI演示文稿.md |
| L5 | HeyGen/D-ID：AI数字人与虚拟形象制作 | ✅ | articles/L5-AI数字人.md |
| L6 | 多模态AI应用：图文生成、语音合成、视频编辑整合 | ✅ | articles/L6-多模态AI应用.md |

### 模块M：AI平台与基础设施（6篇）

| 序号 | 标题 | 状态 | 文件路径 |
|------|------|------|----------|
| M1 | Hugging Face生态：模型、数据集、Spaces与Gradio | ✅ | articles/M1-Hugging-Face生态.md |
| M2 | Dify/Coze/FastGPT：零代码AI应用开发平台对比 | ✅ | articles/M2-零代码平台对比.md |
| M3 | 向量数据库选型：Milvus vs Pgvector vs Redis Vector | ✅ | articles/M3-向量数据库选型.md |
| M4 | 模型部署优化：vLLM vs TensorRT-LLM vs TGI | ✅ | articles/M4-模型部署优化.md |
| M5 | MLOps入门：从训练到部署的全流程工具链 | ✅ | articles/M5-MLOps入门.md |
| M6 | AI应用监控与评估：指标设计、A/B测试与持续优化 | ✅ | articles/M6-AI应用监控.md |

---

## 持续更新板块

### 模块N：热点快讯（灵活响应）

| 序号 | 标题 | 状态 | 文件路径 |
|------|------|------|----------|
| N1 | OpenAI GPT-5.5发布：自适应推理深度突破 | ✅ | articles/N1-热点快讯模板.md |
| N2 | Claude Opus 4.7发布：架构级代码生成革新 | ✅ | articles/N2-热点快讯模板.md |
| N3 | DeepSeek-V4预览：混合RL推理与原生Agent框架 | ✅ | articles/N3-热点快讯模板.md |
| N4 | Cursor 3 + Composer 2：端到端Agentic开发 | ✅ | articles/N4-热点快讯模板.md |
| N5 | GLM-5发布：Agentic Engineering开源SOTA | ✅ | articles/N5-热点快讯模板.md |

---

## 统计信息

| 系列 | 模块 | 篇数 | 说明 |
|------|------|------|------|
| Java技术栈 | A-G | 70篇 | 基于CookBook知识体系 |
| AI技能与工具 | H-M | 48篇 | 编程-focused占67% |
| **总计** | | **118+篇** | 持续更新中 |

### 国产模型覆盖清单

| 模型 | 所属公司 | 深度评测篇 | 其他提及位置 |
|------|---------|-----------|-------------|
| DeepSeek-V4/V3/R1/Coder | 深度求索 | J2 | H3, H7, K6, J10, N3 |
| Kimi K2.6 | 月之暗面 | J3 | H4, H7, N2 |
| GLM-5/CodeGeeX-5 | 智谱AI | J4 | H5, I2, K6, N5 |
| Qwen3-Coder | 阿里云 | J5 | H7, K6, J10 |
| ERNIE 4.5/文心快码3.0 | 百度 | J6 | I2, N系列 |
| Xiaomi Mimo-2.5 Pro | 小米 | J7 | H6, N系列 |
| MiniMax M2.7 | MiniMax | J8 | H6, N系列 |
| 华为盘古 | 华为 | J9 | H6 |
| 字节豆包/MarsCode | 字节跳动 | J9 | I2, H6, N系列 |
| 讯飞星火 | 科大讯飞 | J9 | H6 |
| 腾讯混元 | 腾讯 | J9 | H6 |
| 商汤日日新 | 商汤 | J9 | H6 |

---

*最后更新：2026-05-04 — 全部118篇文章已完成*
