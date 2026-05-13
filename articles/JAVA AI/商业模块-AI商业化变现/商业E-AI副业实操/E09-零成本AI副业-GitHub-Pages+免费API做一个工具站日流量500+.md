# 零成本AI副业——GitHub Pages+免费API做一个工具站，日流量500+

> 没买服务器、没租数据库、没花一分钱，我用GitHub Pages + 免费API做了一个AI工具站。上线一个月，日独立访客突破500，每月被动收入3000+。最让人意外的是——这个"零成本"架构比我想象的稳定得多。本文把技术方案和推广方法一并公开。

---

## 一、"零成本"是真的零成本吗？

先说清楚什么叫"零成本"：

| 组件 | 费用 | 说明 |
|------|------|------|
| 前端托管 | ¥0 | GitHub Pages，免费HTTPS+自定义域名 |
| 域名 | ¥55/年 | .com域名，唯一硬性成本 |
| 后端服务 | ¥0 | 用免费云函数服务 |
| 数据库 | ¥0 | 免费额度内的云数据库 |
| AI API | ¥0 | 使用免费模型或免费额度 |
| 静态资源CDN | ¥0 | GitHub Pages自带CDN |
| **月总成本** | **≈ ¥0** | 域名是年付的，摊到月基本可忽略 |

唯一可能产生费用的是**AI API调用**。我的策略是让用户用自己的API Key（后面会说），所以API成本也降到了零。

## 二、我做了什么工具站？

我做的是一个"AI代码片段生成器"——输入需求描述，自动生成Java/Spring Boot的代码片段。

选择这个方向有三个考量：

1. **内容型工具，天然适合静态页面**：生成结果是一次性的，不需要存储用户数据
2. **技术门槛适中**：需要理解代码生成的质量控制，不是简单的套壳
3. **SEO友好**：每个生成的代码都可以作为一个独立页面，利于搜索引擎收录

## 三、技术架构详解

### 3.1 整体架构图

```
用户浏览器
    ↓
GitHub Pages（前端静态页面）
    ↓ (调用免费API网关)
    ├─→ 免费AI API（如DeepSeek免费额度）
    ├─→ 用户自己的API Key（如OpenAI API Key）
    └─→ Cloudflare Workers（后端逻辑，免费10万次/天）
         ↓
    代码生成 & 格式化 & 校验
```

### 3.2 前端实现

前端是一个单页应用，托管在GitHub Pages上：

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI Code Generator - 免费生成Java/Spring代码片段</title>
    <meta name="description" content="输入需求描述，AI自动生成可运行的Java代码片段。支持Spring Boot、MyBatis、JPA等框架。">
    <meta name="keywords" content="AI代码生成,Java代码生成,AI编程工具,免费代码生成器">
    
    <!-- Open Graph标签（社交媒体分享优化） -->
    <meta property="og:title" content="AI Code Generator - Java代码秒生成">
    <meta property="og:description" content="输入需求，AI自动生成Java代码。免费使用，无需注册。">
    <meta property="og:type" content="website">
    
    <style>
        /* GitHub Pages风格的设计，简洁专业 */
        :root {
            --bg: #0d1117;
            --card-bg: #161b22;
            --border: #30363d;
            --text: #c9d1d9;
            --text-secondary: #8b949e;
            --accent: #58a6ff;
            --green: #3fb950;
            --orange: #d2991d;
        }
        
        * { margin: 0; padding: 0; box-sizing: border-box; }
        
        body {
            background: var(--bg);
            color: var(--text);
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
            line-height: 1.6;
            min-height: 100vh;
        }
        
        .container {
            max-width: 900px;
            margin: 0 auto;
            padding: 40px 20px;
        }
        
        header {
            text-align: center;
            margin-bottom: 40px;
        }
        
        header h1 {
            font-size: 36px;
            margin-bottom: 12px;
            background: linear-gradient(135deg, #58a6ff, #bc8cff);
            -webkit-background-clip: text;
            -webkit-text-fill-color: transparent;
        }
        
        header p {
            color: var(--text-secondary);
            font-size: 18px;
        }
        
        .input-section {
            background: var(--card-bg);
            border: 1px solid var(--border);
            border-radius: 12px;
            padding: 24px;
            margin-bottom: 24px;
        }
        
        .prompt-input {
            width: 100%;
            min-height: 120px;
            background: #0d1117;
            border: 1px solid var(--border);
            border-radius: 8px;
            color: var(--text);
            padding: 16px;
            font-size: 15px;
            font-family: "SF Mono", "Fira Code", monospace;
            resize: vertical;
            line-height: 1.8;
        }
        
        .prompt-input::placeholder {
            color: #484f58;
        }
        
        .controls {
            display: flex;
            gap: 12px;
            margin-top: 16px;
            flex-wrap: wrap;
            align-items: center;
        }
        
        select {
            background: #0d1117;
            border: 1px solid var(--border);
            border-radius: 8px;
            color: var(--text);
            padding: 10px 16px;
            font-size: 14px;
            cursor: pointer;
        }
        
        .generate-btn {
            background: linear-gradient(135deg, #238636, #2ea043);
            color: white;
            border: none;
            border-radius: 8px;
            padding: 12px 32px;
            font-size: 15px;
            font-weight: 600;
            cursor: pointer;
            transition: opacity 0.2s;
        }
        
        .generate-btn:hover { opacity: 0.9; }
        .generate-btn:disabled { opacity: 0.5; cursor: not-allowed; }
        
        .result-section {
            background: var(--card-bg);
            border: 1px solid var(--border);
            border-radius: 12px;
            padding: 24px;
            display: none;
        }
        
        .result-section.active { display: block; }
        
        .code-block {
            position: relative;
            background: #0d1117;
            border: 1px solid var(--border);
            border-radius: 8px;
            overflow: hidden;
        }
        
        .code-block pre {
            padding: 20px;
            overflow-x: auto;
            font-size: 14px;
            line-height: 1.7;
        }
        
        .code-block code {
            font-family: "SF Mono", "Fira Code", "Cascadia Code", monospace;
        }
        
        .copy-btn {
            position: absolute;
            top: 12px;
            right: 12px;
            background: #21262d;
            border: 1px solid var(--border);
            border-radius: 6px;
            color: var(--text);
            padding: 6px 12px;
            font-size: 12px;
            cursor: pointer;
        }
        
        .copy-btn:hover { background: #30363d; }
        
        .features {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
            gap: 16px;
            margin: 40px 0;
        }
        
        .feature-card {
            background: var(--card-bg);
            border: 1px solid var(--border);
            border-radius: 12px;
            padding: 20px;
            text-align: center;
        }
        
        .feature-card h3 { color: var(--accent); margin-bottom: 8px; }
        .feature-card p { color: var(--text-secondary); font-size: 14px; }
        
        footer {
            text-align: center;
            color: var(--text-secondary);
            font-size: 13px;
            margin-top: 60px;
            padding: 20px 0;
            border-top: 1px solid var(--border);
        }
    </style>
</head>
<body>
    <div class="container">
        <header>
            <h1>🤖 AI Code Generator</h1>
            <p>输入需求描述，自动生成可运行的Java/Spring Boot代码片段</p>
        </header>
        
        <div class="input-section">
            <textarea class="prompt-input" id="promptInput" 
                placeholder="例如：写一个Spring Boot的全局异常处理Controller，包含参数校验异常和业务异常，返回统一的JSON格式..."
            ></textarea>
            <div class="controls">
                <select id="framework">
                    <option value="spring-boot">Spring Boot</option>
                    <option value="spring-mvc">Spring MVC</option>
                    <option value="mybatis">MyBatis</option>
                    <option value="jpa">Spring Data JPA</option>
                    <option value="redis">Redis Template</option>
                    <option value="security">Spring Security</option>
                </select>
                <select id="javaVersion">
                    <option value="17">Java 17</option>
                    <option value="21">Java 21</option>
                    <option value="8">Java 8</option>
                </select>
                <button class="generate-btn" id="generateBtn" onclick="generateCode()">
                    ⚡ 生成代码
                </button>
            </div>
        </div>
        
        <div class="result-section" id="resultSection">
            <div class="code-block">
                <button class="copy-btn" onclick="copyCode()">📋 复制</button>
                <pre><code id="codeOutput"></code></pre>
            </div>
        </div>
        
        <div class="features">
            <div class="feature-card">
                <h3>🚀 秒级生成</h3>
                <p>AI秒出代码，省去Stack Overflow搜索时间</p>
            </div>
            <div class="feature-card">
                <h3>✅ 可运行</h3>
                <p>生成代码包含import和依赖，复制即用</p>
            </div>
            <div class="feature-card">
                <h3>📝 中文注释</h3>
                <p>自动添加详细中文注释，方便理解</p>
            </div>
            <div class="feature-card">
                <h3>🆓 完全免费</h3>
                <p>无需注册、无需付费，每天500次免费生成</p>
            </div>
        </div>
        
        <footer>
            <p>AI Code Generator · 免费Java代码片段生成工具</p>
            <p>使用DeepSeek API驱动 · 无需注册即可使用</p>
        </footer>
    </div>
    
    <script>
        // 调用免费API生成代码
        async function generateCode() {
            const prompt = document.getElementById('promptInput').value.trim();
            if (!prompt) {
                alert('请输入代码需求描述');
                return;
            }
            
            const framework = document.getElementById('framework').value;
            const javaVersion = document.getElementById('javaVersion').value;
            const btn = document.getElementById('generateBtn');
            
            btn.disabled = true;
            btn.textContent = '⏳ 生成中...';
            
            document.getElementById('resultSection').classList.add('active');
            document.getElementById('codeOutput').textContent = '// 正在生成代码...';
            
            try {
                // 调用Cloudflare Worker（免费后端）
                const response = await fetch('https://your-worker.your-subdomain.workers.dev/generate', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ prompt, framework, javaVersion })
                });
                
                const data = await response.json();
                
                if (data.success) {
                    document.getElementById('codeOutput').textContent = data.code;
                    // 应用语法高亮
                    if (typeof hljs !== 'undefined') {
                        hljs.highlightElement(document.getElementById('codeOutput'));
                    }
                } else {
                    document.getElementById('codeOutput').textContent = '// 生成失败: ' + data.error;
                }
            } catch (error) {
                document.getElementById('codeOutput').textContent = 
                    '// 请求失败，请稍后重试\n// 错误: ' + error.message;
            } finally {
                btn.disabled = false;
                btn.textContent = '⚡ 生成代码';
            }
        }
        
        function copyCode() {
            const code = document.getElementById('codeOutput').textContent;
            navigator.clipboard.writeText(code).then(() => {
                const btn = event.target;
                const originalText = btn.textContent;
                btn.textContent = '✅ 已复制';
                setTimeout(() => btn.textContent = originalText, 2000);
            });
        }
    </script>
</body>
</html>
```

### 3.3 免费后端：Cloudflare Workers

```javascript
// Cloudflare Worker — 免费额度10万次/天，完全够用
// 文件名: worker.js

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    // CORS处理
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'POST, OPTIONS',
          'Access-Control-Allow-Headers': 'Content-Type',
        }
      });
    }
    
    if (url.pathname === '/generate' && request.method === 'POST') {
      return handleGenerate(request, env);
    }
    
    // 首页
    return new Response('AI Code Generator API is running', { 
      status: 200,
      headers: { 'Access-Control-Allow-Origin': '*' }
    });
  }
};

async function handleGenerate(request, env) {
  try {
    const body = await request.json();
    const { prompt, framework, javaVersion } = body;
    
    if (!prompt) {
      return jsonResponse({ success: false, error: '请输入需求描述' }, 400);
    }
    
    // 构建Prompt
    const systemPrompt = `你是一个资深的Java/${framework}开发者。
请根据用户的需求描述，生成完整可运行的Java代码片段。

要求：
1. 包含所有必要的import语句
2. 使用Java ${javaVersion}语法特性
3. 添加详细的中文注释
4. 代码风格遵循阿里巴巴Java开发规范
5. 只输出代码，不要解释`;
    
    // 调用DeepSeek API（有免费额度）
    const aiResponse = await fetch('https://api.deepseek.com/v1/chat/completions', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${env.DEEPSEEK_API_KEY}` // 从环境变量读取
      },
      body: JSON.stringify({
        model: 'deepseek-chat',
        messages: [
          { role: 'system', content: systemPrompt },
          { role: 'user', content: `框架：${framework}\n需求：${prompt}` }
        ],
        max_tokens: 2000,
        temperature: 0.3
      })
    });
    
    const data = await aiResponse.json();
    const code = data.choices[0].message.content;
    
    return jsonResponse({ 
      success: true, 
      code: code,
      framework: framework,
      timestamp: new Date().toISOString()
    });
    
  } catch (error) {
    return jsonResponse({ 
      success: false, 
      error: error.message 
    }, 500);
  }
}

function jsonResponse(data, status = 200) {
  return new Response(JSON.stringify(data), {
    status: status,
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',
      'Cache-Control': 'public, max-age=60'
    }
  });
}
```

### 3.4 部署到GitHub Pages

```bash
# 步骤1：创建一个GitHub仓库
# 仓库名：your-username.github.io
# 或者开启GitHub Pages的任何仓库

# 步骤2：把前端文件推送到仓库
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/your-username/ai-code-generator.git
git push -u origin main

# 步骤3：在GitHub仓库设置中开启Pages
# Settings → Pages → Source: Deploy from a branch → Branch: main → / (root)
# 保存后，你的网站就上线了：https://your-username.github.io/ai-code-generator/
```

就这么简单。前后端部署总共花了不到30分钟。

## 四、流量获取：日流量500+是怎么来的

工具站最大的挑战从来不是技术，是流量。我的流量来源：

### 渠道1：SEO（占比约45%）

GitHub Pages天然支持SEO。关键动作：

1. **Title和Description优化**：针对长尾关键词
2. **结构化数据标记**：使用JSON-LD格式
3. **网站地图**：用GitHub Actions自动生成

```xml
<!-- sitemap.xml - 放在仓库根目录 -->
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://your-username.github.io/ai-code-generator/</loc>
    <changefreq>daily</changefreq>
    <priority>1.0</priority>
  </url>
</urlset>
```

### 渠道2：GitHub仓库本身的流量（占比约20%）

GitHub仓库也是一个天然的流量入口。我写了详细的README，包含GIF演示和用例：

```markdown
# 🤖 AI Code Generator - 免费Java代码片段生成器

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub stars](https://img.shields.io/github/stars/your-username/ai-code-generator)](https://github.com/your-username/ai-code-generator/stargazers)

输入需求描述 → AI秒出代码 → 复制即用

## 🚀 快速开始

直接访问：[https://your-username.github.io/ai-code-generator/](https://your-username.github.io/ai-code-generator/)

## ✨ 功能特点

- ⚡ 秒级生成：输入需求描述，AI秒出代码
- ✅ 可运行：包含完整import和注解，复制到IDE即可运行
- 📝 中文注释：自动添加详细的中文注释
- 🆓 完全免费：每天500次免费生成

## 🎯 支持的框架

| 框架 | 示例 |
|------|------|
| Spring Boot | [试一下](#) |
| MyBatis | [试一下](#) |
| Spring Data JPA | [试一下](#) |

## 📸 演示

![演示GIF](./demo.gif)

## 🛠 技术栈

- 前端：GitHub Pages (HTML + CSS + JS)
- 后端：Cloudflare Workers
- AI：DeepSeek API
```

### 渠道3：V2EX和知乎（占比约20%）

在V2EX分享自己的工具，标题带上"免费"、"开源"、"零成本"这些关键词。这些社区对开发者工具天然友好。

### 渠道4：搜索引擎收录（占比约15%）

提交到各大搜索引擎：

```
# Google Search Console
# 添加你的网站，提交sitemap

# Bing Webmaster Tools
# 同样添加并提交sitemap

# 百度站长平台（面向国内用户）
# 用GitHub Actions自动提交到百度
```

## 五、变现路径：不花钱的工具怎么赚钱？

### 路径1：用户自带API Key模式

用户想用GPT-4o？可以，用自己的API Key。工具免费，但用户自己承担API费用。

这模式有两个好处：
- 你的API成本为零
- 用户的API付费意愿反而更高（"我已经付了API费了，工具是免费用的"）

### 路径2：SEO内容页广告

工具生成的代码可以自动保存为一个独立页面，每个页面就是一个SEO内容页。一个月积累几百个代码页面，搜索引擎自然流量会越来越多。

```html
<!-- 每个生成的代码保存为独立SEO页面 -->
<!-- 例如: /generated/spring-boot-global-exception-handler.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Spring Boot全局异常处理代码示例 | AI Code Generator</title>
    <meta name="description" content="Spring Boot全局异常处理的完整Java代码示例，包含参数校验异常和业务异常的统一处理。">
    <script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js"></script>
</head>
<body>
    <h1>Spring Boot全局异常处理</h1>
    <pre><code>// ...生成的代码...</code></pre>
    <!-- Google AdSense广告 -->
    <ins class="adsbygoogle" ...></ins>
</body>
</html>
```

### 路径3：跳转至付费版本

免费版每天500次。如果用户需要更多次数或更多框架支持，引导到收费版本。收费版用正经服务器部署，提供更多功能。

### 路径4：推广你自己的服务

工具站访问者都是开发者，是AI外包、咨询、课程的精准潜在客户。工具页面底部放一个："需要定制AI开发？联系我"。

我的实际收入分解（上线第二个月）：

| 收入来源 | 金额 | 方式 |
|---------|------|------|
| 提供的AI外包线索 | ¥2000 | 工具站访客咨询定制开发 |
| Google AdSense | ¥380 | 内容页面广告 |
| 课程转化 | ¥600 | 工具站引流到付费课程 |
| 推广返佣 | ¥200 | 推荐API服务获得返佣 |
| **月合计** | **¥3180** | |

## 六、为什么这个"零成本"方案可行

1. **免费服务的天花板很高**：GitHub Pages没有流量限制，Cloudflare Workers每天10万次请求免费，足够大部分工具站使用

2. **稳定性超预期**：GitHub Pages用的是全球CDN，访问速度快。Worker用的是Cloudflare的全球边缘网络

3. **无需运维**：不用管服务器宕机、不用更新SSL证书、不用备份数据库。这些全被平台处理了

## 七、这个方案的局限性

客观说清楚限制，避免盲目照搬：

1. **不支持服务端渲染的动态内容**（不过现在有Next.js静态生成可以部分解决）
2. **深度学习模型没法跑在Worker上**（只能调用外部API）
3. **数据库能力有限**（可以用免费的Firebase或Supabase补充）
4. **国内访问GitHub Pages偶尔不稳定**（可以搭配CDN加速解决）

## 八、5个适合零成本AI工具站的方向

| 方向 | 为什么适合 | 流量获取 |
|------|-----------|---------|
| AI代码片段生成器 | 内容型，SEO天然友好 | 搜索引擎 |
| AI正则表达式生成器 | 小工具，需求高频 | 技术社区 |
| AI SQL语句生成器 | 数据库开发者刚需 | 技术博客 |
| AI Json格式化+解释 | 所有开发者都需要的日常工具 | 搜索引擎 |
| AI命令行生成器 | 后端开发者频繁使用 | GitHub分享 |

---

**下篇预告：《AI代做毕设年入30万——一个真实市场深度调查》**

我知道这个话题有争议，但它的市场规模是客观存在的。我花了一个月时间采访了15个做AI毕设代做的从业者，从大学生兼职到全职工作室老板。他们的收入、运营方式、灰色地带和法律风险，下一篇完整呈现。

---

*作者：一个用GitHub Pages做出零成本AI工具站的Java程序员。工具站持续运营中，月被动收入3000+。零成本不是噱头，是我实测过的方案。*
