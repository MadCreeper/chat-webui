# Chat WebUI 项目

## 概述
简单的聊天网页，用于连接 LLM API（如火山引擎/字节跳动 Coding Plan）。

## 技术栈
- 前端：原生 HTML/CSS/JS，localStorage 存储聊天记录
- 代理：Cloudflare Workers（解决 CORS 跨域问题）
- 托管：GitHub Pages

## 部署步骤

### 1. 部署前端到 GitHub Pages
```bash
cd ~/projects/chat-webui
git init
git add index.html
git commit -m "init"
gh repo create chat-webui --public --source=. --push
# 启用 Pages（API方式）
echo '{"source":{"branch":"main","path":"/"}}' | gh api -X PUT repos/MadCreeper/chat-webui/pages/main --input -
```
或手动：仓库 Settings → Pages → Source 选 main branch

### 2. 部署 Cloudflare Workers 代理
1. 打开 https://dash.cloudflare.com → Workers → 创建 Worker
2. 粘贴代理代码（见下方）
3. 部署，获得 URL 如 `chat-proxy.xxx.workers.dev`

### 3. 配置
在网页点"设置"填入：
- API 端点：Cloudflare Workers 的 URL
- API Key：你的 API Key
- 模型：模型名

## 代理代码（Cloudflare Workers）

```javascript
const TARGET_API = "https://ark.cn-beijing.volces.com/api/coding/v1/chat/completions";

export default {
  async fetch(request) {
    if (request.method === "OPTIONS") {
      return new Response(null, {
        headers: {
          "Access-Control-Allow-Origin": "*",
          "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
          "Access-Control-Allow-Headers": "Content-Type, Authorization",
        },
      });
    }

    const authHeader = request.headers.get("Authorization");

    const response = await fetch(TARGET_API, {
      method: request.method,
      headers: {
        "Content-Type": "application/json",
        ...(authHeader ? { "Authorization": authHeader } : {}),
      },
      body: request.body,
    });

    const corsHeaders = {
      "Access-Control-Allow-Origin": "*",
      "Access-Control-Allow-Methods": "GET, POST, OPTIONS",
      "Access-Control-Allow-Headers": "Content-Type, Authorization",
    };

    return new Response(response.body, {
      status: response.status,
      statusText: response.statusText,
      headers: { ...corsHeaders, ...Object.fromEntries(response.headers) },
    });
  },
};
```

## 经验总结

### CORS 问题
- 浏览器直接请求其他域名的 API 会被 CORS 阻止
- 解决方案：用 Cloudflare Workers 做代理（服务器端请求，无 CORS）

### 火山引擎 API 路径
- 基础 URL：`https://ark.cn-beijing.volces.com/api/coding`
- 完整路径需要：`/v1/chat/completions`
- Worker 代理中要写完整路径，前端只填 Worker URL

### 部署平台对比
| 平台 | 适合场景 | 费用 |
|-----|---------|-----|
| GitHub Pages | 纯静态前端 | 免费 |
| Cloudflare Workers | 简单代理 | 免费（每天10万次） |
| Vercel | 前端+Serverless | 免费 |

### 注意事项
- GitHub Pages 访问自己的域名需要做 DNS 解析
- 用户设置保存在浏览器 localStorage，各浏览器独立