# Open WebUI + Ollama：搭建企业内部 ChatGPT 替代平台，2 小时上线公司专属 AI 门户

> 老板说："搞个公司内部的 ChatGPT，把公司文档接进去，员工免费用，数据不能出去。"——安排！Open WebUI + Ollama，一套 Docker Compose 全解决。

---

## 一、为什么选 Open WebUI + Ollama？

先看企业级需求清单：

| 需求 | ChatGPT 企业版 | 自建 Open WebUI + Ollama |
|------|---------------|--------------------------|
| 数据不出内网 | 需要签 EDP | 天然满足 |
| 用户管理 | 按席位收费 | 免费，无上限 |
| 权限控制 | 管理员/用户/只读 | 三级权限，支持 RBAC |
| LDAP/AD 集成 | 企业版专属 | 原生支持 |
| 模型选择 | 仅 OpenAI 系 | 100+ 模型随意切换 |
| RAG 文档问答 | 需额外付费 | 内置 RAG Pipeline |
| 成本 | $25-60/人/月 | 仅硬件成本 |

**Open WebUI**（原 Ollama WebUI）是 Ollama 生态中最成熟的前端项目，GitHub 60k+ Star。它提供了一套完整的 ChatGPT 风格交互界面，同时内置了用户管理、模型市场、RAG、Web 搜索等企业级功能。

---

## 二、Docker Compose 一键部署

### 2.1 准备工作

```bash
# 确保已安装 Docker 和 Docker Compose
docker --version        # >= 24.0
docker compose version  # >= 2.0

# 确保 Ollama 已运行
ollama serve

# 至少拉取一个模型
ollama pull qwen2.5:7b
```

### 2.2 docker-compose.yml

```yaml
version: '3.8'

services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3000:8080"
    volumes:
      - ./open-webui:/app/backend/data
    environment:
      # Ollama 服务地址（如果 Ollama 在宿主机运行，用 host.docker.internal）
      - OLLAMA_BASE_URL=http://host.docker.internal:11434
      # WebUI 访问端口
      - WEBUI_PORT=8080
      # 启用用户注册（部署初期打开，后续关闭）
      - WEBUI_AUTH=true
      - WEBUI_SECRET_KEY=your-secret-key-change-this
      # 启用 RAG
      - RAG_EMBEDDING_MODEL=nomic-embed-text
      # 文件上传大小限制
      - UPLOAD_DIR=/app/backend/data/uploads
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

### 2.3 启动服务

```bash
# 启动
docker compose up -d

# 查看日志
docker compose logs -f open-webui

# 访问
open http://localhost:3000
```

首次访问会要求创建管理员账号。创建后进入管理后台 `Settings → Admin Settings`，你会看到完整的控制面板。

---

## 三、用户管理与权限控制

### 3.1 用户角色体系

Open WebUI 内置三级用户角色：

| 角色 | 权限 |
|------|------|
| **Admin（管理员）** | 全部权限：用户管理、模型管理、系统配置、查看所有对话 |
| **User（普通用户）** | 对话、上传文件、使用 RAG、创建个人模型 |
| **Pending（待审核）** | 注册后默认状态，需管理员激活 |

### 3.2 管理操作

**进入管理后台**：左下角头像 → Admin Panel

**用户管理**：
- 查看所有用户列表
- 激活/停用用户
- 修改角色
- 删除用户

**注册控制**：
```yaml
# docker-compose.yml 中配置
environment:
  - WEBUI_AUTH=true
  - ENABLE_SIGNUP=true        # 部署初期开启
  # 稳定后关闭自主注册，改为管理员手动创建
  - ENABLE_SIGNUP=false
  - DEFAULT_USER_ROLE=pending # 新注册默认待审核
```

### 3.3 对话权限管控

```yaml
# 用户可以查看其他用户的对话吗？（默认不行）
- USER_PERMISSIONS_CHAT_DELETION=true  # 允许用户删除自己的对话
- CHAT_PRIVACY=true                    # 启用对话隐私

# 模型访问控制
- ENABLE_MODEL_FILTER=true
- MODEL_FILTER_LIST=qwen2.5:7b;qwen2.5:14b;deepseek-coder-v2:16b
```

---

## 四、对接公司 AD/LDAP

这是企业部署的关键一步——员工不用再记一套账号密码，直接用公司域账号登录。

### 4.1 LDAP 配置

进入 `Admin Panel → Settings → LDAP`，填写以下参数：

```yaml
# docker-compose.yml 环境变量方式
environment:
  # 启用 LDAP
  - ENABLE_LDAP=true

  # LDAP 服务器地址
  - LDAP_SERVER_URL=ldap://dc01.company.local:389

  # 绑定凭据（需要一个有查询权限的域账号）
  - LDAP_BIND_DN=CN=svc_webui,OU=Service Accounts,DC=company,DC=local
  - LDAP_BIND_PASSWORD=your-service-account-password

  # 用户搜索基路径
  - LDAP_SEARCH_BASE=DC=company,DC=local
  - LDAP_SEARCH_FILTER=(&(objectClass=user)(sAMAccountName={{username}}))

  # 用户属性映射
  - LDAP_USERNAME_ATTR=sAMAccountName
  - LDAP_DISPLAY_NAME_ATTR=displayName
  - LDAP_EMAIL_ATTR=mail
  - LDAP_GROUP_ATTR=memberOf

  # 组映射（可选：哪些 AD 组自动获得管理权限）
  - LDAP_ADMIN_GROUP=CN=AI_Admins,OU=Groups,DC=company,DC=local
```

### 4.2 LDAP 登录流程

```
用户输入: zhang.san / 域密码
         ↓
Open WebUI    →   LDAP BIND (用svc_webui账号查询)
         ↓
LDAP 返回     →   DN: CN=张三,OU=IT部,DC=company,DC=local
         ↓
Open WebUI    →   LDAP BIND (用用户自己的DN和密码)
         ↓
认证成功      →   自动创建用户或匹配已有用户
         ↓
组检查        →   如果在AI_Admins组 → Admin角色
               否则 → User角色
```

### 4.3 对接 OAuth2.0（Azure AD / Okta）

如果公司使用 Azure AD（Microsoft 365），可以走 OIDC 协议：

```yaml
environment:
  - ENABLE_OAUTH_SIGNUP=true
  - OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true
  - OAUTH_PROVIDER=oidc

  # Microsoft Entra ID (Azure AD) 配置
  - OPENID_PROVIDER_URL=https://login.microsoftonline.com/{tenant-id}/v2.0
  - OAUTH_CLIENT_ID=your-app-client-id
  - OAUTH_CLIENT_SECRET=your-app-client-secret
  - OAUTH_SCOPES=openid email profile
  - OAUTH_REDIRECT_URI=https://ai-portal.company.local/oauth/oidc/callback
```

---

## 五、模型市场——管理多个本地模型

Open WebUI 的模型管理非常直观，管理员可以在 Web 界面上完成所有操作。

### 5.1 拉取模型（Web 界面操作）

1. 进入 `Admin Panel → Settings → Models`
2. 点击 **Pull a Model**
3. 输入模型名称，例如：
   - `qwen2.5:14b` — 通用对话
   - `deepseek-coder-v2:16b` — 代码生成
   - `llama3.2:3b` — 轻量翻译
   - `nomic-embed-text` — 文本嵌入（RAG 必需）
4. 点击下载，实时显示进度条

等效命令行操作：

```bash
# 推荐的企业模型组合
ollama pull qwen2.5:14b          # 中文对话主力
ollama pull deepseek-coder-v2:16b # 代码助手
ollama pull llama3.2:3b          # 轻量翻译/摘要
ollama pull nomic-embed-text     # 嵌入模型(RAG用)
ollama pull mistral:7b           # 英文对话备选
```

### 5.2 模型分组与权限

```yaml
# 可以为不同用户组分配不同的模型列表
environment:
  - ENABLE_MODEL_FILTER=true
  - MODEL_FILTER_LIST=qwen2.5:7b;qwen2.5:14b;deepseek-coder-v2:16b;llama3.2:3b

  # 仅管理员可见高成本模型
  - ADMIN_MODEL_LIST=qwen2.5:72b
```

### 5.3 模型温度与系统提示词预设

```yaml
# 全局默认系统提示词
environment:
  - DEFAULT_SYSTEM_PROMPT="你是XX公司内部AI助手。回答应专业、准确、简洁。涉及公司机密信息请提示用户注意信息安全。"
```

用户也可以在自己的设置中自定义系统提示词和模型参数。

---

## 六、RAG 配置——接入公司内部文档

Open WebUI 内置了 RAG（Retrieval-Augmented Generation）引擎，让大模型能"读懂"公司内部文档。

### 6.1 上传文档（Web 界面）

用户在对话界面点击**上传文件**按钮，支持格式：

- PDF（扫描件需先 OCR）
- Word / Excel
- Markdown / 纯文本
- CSV / JSON
- HTML / XML

上传后 Open WebUI 会自动：
1. 解析文档内容
2. 用嵌入模型（nomic-embed-text）向量化
3. 存入向量数据库（内置 ChromaDB）
4. 对话时自动检索相关片段

### 6.2 RAG 配置参数

```yaml
environment:
  # 嵌入模型
  - RAG_EMBEDDING_MODEL=nomic-embed-text
  - RAG_EMBEDDING_ENGINE=ollama

  # 检索参数
  - RAG_TOP_K=5                    # 检索5个最相关片段
  - RAG_CHUNK_SIZE=1000            # 文档分块大小（字符数）
  - RAG_CHUNK_OVERLAP=200          # 分块重叠大小

  # 全文搜索
  - RAG_FULL_CONTEXT=true          # 包含完整搜索结果上下文

  # PDF 提取设置
  - RAG_PDF_EXTRACT_IMAGES=false   # 是否提取图片中的文字

  # 向量数据库
  - VECTOR_DB=chromadb
  - CHROMA_DATA_PATH=/app/backend/data/vector_db
```

### 6.3 批量导入公司知识库

```bash
# 方式一：在 Web 界面逐个上传
# 方式二：通过 API 批量导入（适合大量文档）

# 首先把文档放在统一目录
mkdir -p /data/knowledge-base/
cp /internal/wiki/*.md /data/knowledge-base/
cp /internal/policies/*.pdf /data/knowledge-base/

# 使用 Open WebUI API 批量上传
for file in /data/knowledge-base/*; do
  curl -X POST http://localhost:3000/api/v1/files/upload \
    -H "Authorization: Bearer $API_KEY" \
    -F "file=@$file"
done
```

### 6.4 RAG 效果调优

```yaml
# 高级 RAG 配置
environment:
  # 混合搜索（向量 + 关键字）
  - RAG_HYBRID_SEARCH=true

  # 重排序（reranker 模型提升精度）
  - RAG_RERANKING_MODEL=
  - RAG_RERANKING_ENABLED=false

  # 模板自定义
  - RAG_TEMPLATE="使用以下上下文信息回答用户问题。如果上下文中没有答案，如实说明。\n\n上下文：\n{{CONTEXT}}\n\n用户问题：{{QUERY}}"
```

---

## 七、Java 后端如何对接 Open WebUI

Open WebUI 对外暴露的是 **OpenAI 兼容 API 格式**，这意味着任何支持 OpenAI SDK 的语言都能无缝接入。Spring Boot 项目用 Spring AI 的 OpenAI Starter 即可对接。

### 7.1 application.yml 配置

```yaml
spring:
  ai:
    openai:
      api-key: ${OPEN_WEBUI_API_KEY}  # 在 Open WebUI Settings → Account 生成
      base-url: http://localhost:3000/api  # 关键：指向 Open WebUI
      chat:
        enabled: true
        options:
          model: qwen2.5:14b
          temperature: 0.7
      embedding:
        enabled: true
        options:
          model: nomic-embed-text
```

### 7.2 Java 对接代码

```java
@RestController
@RequestMapping("/api/company/ai")
public class CompanyAIController {

    private final OpenAiChatModel chatModel;

    public CompanyAIController(OpenAiChatModel chatModel) {
        this.chatModel = chatModel;
    }

    // 基础对话
    @PostMapping("/chat")
    public String chat(@RequestBody ChatRequest req) {
        Prompt prompt = new Prompt(
            new UserMessage(req.getMessage()),
            OpenAiChatOptions.builder()
                .model("qwen2.5:14b")
                .temperature(0.7)
                .build()
        );
        return chatModel.call(prompt)
            .getResult().getOutput().getContent();
    }

    // 带系统提示词的对话（适用于不同部门的 AI 助手）
    @PostMapping("/department/{dept}")
    public String deptChat(@PathVariable String dept,
                           @RequestBody ChatRequest req) {
        String systemPrompt = switch (dept) {
            case "hr" -> """
                你是 HR 部门的 AI 助手。请基于公司员工手册和劳动法规回答。
                涉及具体人事决策请引导咨询 HR 部门。
                """;
            case "it" -> """
                你是 IT 支持 AI 助手。熟悉公司内部系统架构，
                回答关于 VPN、邮箱、开发环境等问题。
                """;
            case "legal" -> """
                你是法务 AI 助手。基于公司合同模板和合规要求回答，
                但不替代专业法律意见。
                """;
            default -> "你是公司 AI 助手。";
        };

        Prompt prompt = new Prompt(List.of(
            new SystemMessage(systemPrompt),
            new UserMessage(req.getMessage())
        ));
        return chatModel.call(prompt)
            .getResult().getOutput().getContent();
    }

    // 流式对话（适合前端打字机效果）
    @GetMapping(value = "/stream", produces = "text/event-stream")
    public Flux<String> streamChat(@RequestParam String message) {
        Prompt prompt = new Prompt(new UserMessage(message));
        return chatModel.stream(prompt)
            .map(r -> r.getResult().getOutput().getContent());
    }
}
```

### 7.3 权限接入

```java
// 使用 Open WebUI 的 API Key 做访问控制
// 不同部门的 Java 服务可以用不同的 API Key
@Configuration
public class AIConfig {

    @Bean
    public OpenAiChatModel hrChatModel() {
        // HR 部门的 API Key，在 WebUI 中权限受限
        return new OpenAiChatModel(
            new OpenAiApi("http://localhost:3000/api",
                "sk-hr-department-key-xxx"),
            OpenAiChatOptions.builder()
                .model("qwen2.5:14b")
                .build()
        );
    }

    @Bean
    public OpenAiChatModel itChatModel() {
        return new OpenAiChatModel(
            new OpenAiApi("http://localhost:3000/api",
                "sk-it-department-key-xxx"),
            OpenAiChatOptions.builder()
                .model("deepseek-coder-v2:16b")
                .build()
        );
    }
}
```

### 7.4 Maven 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
    <version>1.0.0-M5</version>
</dependency>
```

---

## 八、生产环境部署建议

### 8.1 Nginx 反代 + HTTPS

```nginx
server {
    listen 443 ssl;
    server_name ai-portal.company.local;

    ssl_certificate     /etc/nginx/ssl/company.crt;
    ssl_certificate_key /etc/nginx/ssl/company.key;

    # Open WebUI
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 300s;  # 长推理不能断
    }
}
```

### 8.2 资源规划

| 用户规模 | 推荐配置 | 可运行模型 |
|---------|---------|-----------|
| 20 人以下 | 单卡 RTX 4090 24GB | 7B-14B × 2 个并行 |
| 20-100 人 | 双卡 A100 40GB 或 M2 Ultra | 14B-32B × 3 个并行 |
| 100-500 人 | 多节点 Ollama 集群 | 负载均衡 + 模型分片 |
| 500+ 人 | 搭配 vLLM / TGI 推理引擎 | 高并发优化 |

### 8.3 监控与日志

```yaml
# 开启访问日志
environment:
  - LOG_LEVEL=INFO
  - AUDIT_LOG_ENABLED=true

# 对接 Prometheus + Grafana
# Open WebUI 暴露 /metrics 端点（需 v0.3+）
```

---

## 九、总结

两小时，一套 Docker Compose 命令，你的公司就有了一个完整的 AI 门户：

- 用户用域账号登录（LDAP/AD/OAuth2）
- 对话界面和 ChatGPT 一模一样
- 上传公司文档就能 "问AI"
- 管理员控制谁能用什么模型
- Java 后端用 OpenAI 兼容 API 对接
- 所有数据不离开内网

相比 ChatGPT 企业版每人每月 30-60 美元的账单，这套方案仅需要一块消费级显卡的硬件投入——对技术团队来说，几乎是一劳永逸。

---

**下篇预告**：《本地模型微调入门：用 LoRA 让 Llama 3 学会你的业务术语，用 500 条示例数据教 AI 理解你的行业黑话》——医疗行业说"VC"不是风险投资而是维生素C，法律行业说"PC"不是电脑而是个人公司……如何用 500 条数据给大模型打上"业务补丁"，让模型真正理解你的业务语言。敬请期待！
