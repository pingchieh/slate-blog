---
title: Portkey兼容OpenAI SDK的中间层方案
description: 探讨如何通过Nginx或Deno Deploy构建中间层，解决Portkey AI Gateway与标准OpenAI SDK因配置传递方式不同导致的兼容性问题，实现无缝集成。
tags: [Portkey, OpenAI, "AI Gateway", Nginx, "Deno Deploy"]
pubDate: '2025-04-25' # 请替换为文章发布日期
draft: false
---

## 前言

[Portkey](https://portkey.ai) 是一个功能强大的 AI Gateway 服务，它能够方便地整合各种大型语言模型（LLM）的 API，并提供详细的请求日志、prompt管理等高级功能。然而，Portkey 在处理配置时，通常依赖于特定的 HTTP 请求头（例如 `x-portkey-config`）。这与标准 OpenAI SDK 的设计和使用习惯不同，导致直接使用官方 SDK 难以便捷地传递这些 Portkey 特有的配置，从而无法充分利用 Portkey 的高级特性。

为了解决这一兼容性问题，使我们能够在继续使用熟悉的 OpenAI SDK 的同时，也能享受到 Portkey Gateway 的便利功能，我们需要在 OpenAI SDK 和 Portkey Gateway 之间搭建一个中间层。这个中间层负责接收 SDK 发出的标准请求，在转发到 Portkey Gateway 之前，根据预设的规则或从请求中提取的信息，动态地添加或修改必要的 Portkey 请求头。

本文将探讨两种实现这种中间层的方案：使用成熟的反向代理工具 Nginx，以及构建一个轻量级的无服务器服务，例如基于 Deno Deploy。

## 方案一：使用 Nginx 作为反向代理

利用 Nginx 作为反向代理是一种广泛应用且高效的解决方案。通过配置 Nginx，我们可以监听特定的域名或路径，拦截来自客户端（使用 OpenAI SDK 指向此 Nginx 地址）的请求，然后在将请求转发到 Portkey Gateway 的真实 API 地址 (`https://api.portkey.ai`) 之前，动态地添加 `x-portkey-config` 等请求头。这种方案适用于已经在使用 Nginx 作为网关，或者偏好在现有服务器基础设施中解决问题的场景。

以下是一个 Nginx 配置示例，通过监听不同的子域名来对应不同的 Portkey 配置：

```nginx.conf
# SSL 证书配置 (请根据您的实际情况修改路径和名称)
ssl_certificate     /cert/cloudflare.cert;
ssl_certificate_key /cert/cloudflare.cert;

# SSL Session 缓存配置
ssl_session_cache   shared:SSL:10m;
ssl_session_timeout 10m;

# 使用 map 指令根据主机名 ($host) 映射不同的 Portkey 配置 ID ($portkey_config)
# 当客户端通过不同的子域名访问 Nginx 时，Nginx 会自动匹配 $host 并设置对应的 $portkey_config 变量的值
map $host $portkey_config {
    groq.portkey.your.domain    pc-groq-config; # 示例：访问 groq 子域名使用 pc-groq-config
    gemini.portkey.your.domain  pc-gemini-config; # 示例：访问 gemini 子域名使用 pc-gemini-config
    cohere.portkey.your.domain  pc-cohere-config; # 示例：访问 cohere 子域名使用 pc-cohere-config
    default                     ""; # 如果主机名不匹配上述任何规则，则 $portkey_config 为空
}

server {
    listen              80;     # 监听 HTTP 端口
    listen              443 ssl; # 监听 HTTPS 端口
    # 定义此 server 块处理的域名
    server_name    
        cohere.portkey.your.domain      
        groq.portkey.your.domain 
        gemini.portkey.your.domain;
    
    # 处理根路径下的所有请求
    location / {
        # 将客户端请求转发到 Portkey Gateway 的 API 地址
        proxy_pass https://api.portkey.ai;
        # 禁用代理缓冲。对于需要实时性的流式响应 (如 SSE)，禁用缓冲通常更佳
        proxy_buffering off;

        # 修复与上游服务器（Portkey）之间的 SSL 握手和连接问题
        # 确保 Nginx 使用安全的 TLS 版本与上游建立连接
        proxy_ssl_protocols TLSv1.2 TLSv1.3;
        # 启用 SNI (Server Name Indication)，将客户端请求的域名信息传递给上游服务器
        proxy_ssl_server_name on;
        # 明确指定用于 SNI 的主机名，确保与上游服务器的期望一致
        proxy_ssl_name api.portkey.ai;

        # 设置转发请求所需的其他 HTTP 头
        # 确保 Host 头正确，这是上游服务器识别请求的重要依据
        proxy_set_header Host api.portkey.ai;
        # !!! 核心配置 !!!
        # 将 map 指令根据当前主机名解析得到的 $portkey_config 变量的值，设置到 x-portkey-config 请求头中
        # Portkey Gateway 会读取这个头部来应用对应的配置
        proxy_set_header x-portkey-config $portkey_config;
    }
}
```

**工作原理简述：**

客户端（例如使用 OpenAI SDK）将请求发送到 `groq.portkey.your.domain` (指向此 Nginx 服务器)。Nginx 接收到请求后，通过 `map` 指令识别到主机名是 `groq.portkey.your.domain`，并将 `$portkey_config` 变量设置为 `pc-groq-config`。然后，在转发请求到 `https://api.portkey.ai` 时，Nginx 通过 `proxy_set_header x-portkey-config $portkey_config;` 将 `x-portkey-config: pc-groq-config` 添加到请求头中。Portkey Gateway 收到请求时，就会读取这个头部，并根据 `pc-groq-config` 应用相应的路由、模型选择或其他配置。

## 方案二：使用 Deno Deploy 构建轻量级中间服务

另一种灵活的方案是构建一个轻量级的、运行在无服务器平台上的中间服务。这类服务（如 Deno Deploy、Cloudflare Workers、Vercel Serverless Functions）可以作为 API 网关，接收客户端请求，执行一些自定义逻辑（例如解析请求中的特定信息来确定 Portkey 配置），然后使用编程方式向 Portkey Gateway 发起请求，并在请求头中注入 Portkey 配置。这种方案更加灵活，可以处理比简单头部注入更复杂的逻辑，并且适合现代云原生或无服务器架构。

以下是一个使用 Deno Deploy 实现的示例代码，它从客户端请求的 `Authorization` 头中解析出 API Key 和 Portkey Config ID：

```js
// 引入必要的库
import { AutoRouter, cors, error } from "npm:itty-router"; // 用于构建路由和处理 CORS
import { Langfuse, observeOpenAI } from "https://esm.sh/langfuse"; // 集成 Langfuse 用于可观测性 (可选)
import OpenAI from "npm:openai";
import { PORTKEY_GATEWAY_URL, createHeaders } from "npm:portkey-ai"; 

const PORTKEY_CONFIG = "pc-transl-config";
const MODEL_IDS = {
  "pc-transl-config": ["qwen-qwq-32b"],
  "pc-gemini-config": ["gemini-2.0-flash", "gemini-2.5-pro-exp-03-25"],
  "pc-groq-config": ["qwen-qwq-32b", "deepseek-r1-distill-llama-70b"],
  "pc-cohere-config": ["command-a-03-2025", "command-r-plus-08-2024"],
};

const langfuse = new Langfuse();

// 配置 CORS 策略，允许跨域请求
const { preflight, corsify } = cors({
  origin: "*", // 允许所有来源进行跨域访问
  credentials: true, // 允许携带认证信息 (如 Authorization 头)
  allowMethods: ["GET", "POST"], // 允许的 HTTP 方法
});

// 初始化 AutoRouter，用于定义和处理不同的 API 路径
const router = AutoRouter({
  before: [preflight], // 在处理请求前执行 CORS 预检 (OPTIONS)
  after: [corsify], // 在发送响应后添加 CORS 头部
});

// 中间件：为每个进入路由的请求创建 Langfuse trace，用于后续记录请求/响应信息
const withLangfuseTrace = (request) => {
  request.trace = langfuse.trace({ name: "Deno-Deploy" });
};

function createOpenAIClient({ trace, apiKey, config }) {
  const openai = new OpenAI({
    apiKey,
    baseURL: PORTKEY_GATEWAY_URL,
    defaultHeaders: createHeaders({ 
      config,
    }),
  });
  return observeOpenAI(openai, {
    parent: trace,
    generationName: "openai.chat",
  });
}

// 中间件：验证用户身份，并从 Authorization 头部解析 API Key 和 Portkey Config ID
const withAuthenticatedUser = (request) => {
  const token = request.headers.get("Authorization");
  if (!token) {
    return error(401, "Authentication failed: Missing Authorization header.");
  }
  // 期望 Authorization 头部格式为 "Bearer <key>[@config]"
  const parts = token.split(" ");
  if (parts.length !== 2 || parts[0] !== "Bearer") {
     return error(401, "Authentication failed: Invalid token format. Use Bearer <key>[@config].");
  }

  const auth = parts[1];
  [request.apiKey, request.config = PORTKEY_CONFIG] = auth.split("@");
  if (!MODEL_IDS[request.config]) {
    return error(401, `Authentication failed: Invalid config "${request.config}" provided.`);
  }
};

async function handleStreamResponse(request, stream) {
  return new Response(
    new ReadableStream({
      async start(controller) {
        var out_text = "";
        for await (const chunk of stream) {
          chunk.model = request.model;
          const text = JSON.stringify(chunk);
          controller.enqueue(new TextEncoder().encode(`data: ${text}\n\n`));
          if (chunk.choices && chunk.choices.length > 0) {
             const choice = chunk.choices[0];
             if (choice.finish_reason) {
               request.trace.update({
                 output: out_text,
                 metadata: {
                   response: {
                     ...(() => {
                       const { choices, ...rest } = chunk;
                       return rest;
                     })(),
                   },
                 },
               });
             } else if (choice.delta && choice.delta.content) {
               out_text += choice.delta.content;
             }
          }
        }
        controller.close();
      },
    }),
    {
      headers: {
        "Content-Type": "text/event-stream",
        "Cache-Control": "no-cache",
        Connection: "keep-alive",
      },
    }
  );
}

// 处理 /v1/models 请求
function handleModels(request) {
  const models = MODEL_IDS[request.config] 
    ? MODEL_IDS[request.config].map((id) => ({
        id,
        object: "model",
        created: Date.now(), 
        owned_by: "chieh", 
      }))
    : [];

  return new Response(
    JSON.stringify({
      object: "list",
      data: models,
    }),
    {
      headers: { "Content-Type": "application/json" },
    }
  );
}

// 处理 /v1/chat/completions 请求
async function handleChatCompletions(request) {
  const body = await request.json();
  request.model = body.model; 
  request.trace.update({
    input: body.messages,
    metadata: {
      request: {
        url: request.url,
        method: request.method,
        ...(() => {
          const { messages, ...rest } = body;
          return rest;
        })(),
      },
    },
  });

  const client = createOpenAIClient(request); 

  if (body?.stream) {
    const stream = await client.chat.completions.create(body);
    return await handleStreamResponse(request, stream);
  } else {
    const response = await client.chat.completions.create(body);
    response.model = body.model;
    request.trace.update({
      output: response.choices[0].message.content,
      metadata: {
        response: {
          ...(() => {
            const { choices, ...rest } = response;
            return rest;
          })(),
        },
      },
    });
    // 返回 JSON 格式的非流式响应
    return new Response(JSON.stringify(response), {
         headers: { "Content-Type": "application/json" },
    });
  }
}

// 定义根路径的简单响应，用于健康检查或测试
router.get("/", () => new Response("Hello World!"));

// 定义 /v1/models 路由
router.get("/v1/models", withAuthenticatedUser, handleModels);

// 定义 /v1/chat/completions 路由
router.post(
  "/v1/chat/completions",
  withAuthenticatedUser,
  withLangfuseTrace,
  handleChatCompletions
);

// 导出 router 作为 Deno Deploy 的入口点
export default router;
```

**工作原理简述：**

客户端（使用标准 OpenAI SDK）将 `baseURL` 设置为此 Deno Deploy 服务的地址。发送请求时，客户端将 Portkey Config ID 作为 `Authorization` 头部的一部分（例如 `Bearer <your_api_key>@<portkey_config_id>`）。Deno Deploy 服务接收请求，`withAuthenticatedUser` 中间件解析出 API Key 和 Portkey Config ID。`createOpenAIClient` 函数使用解析出的信息初始化一个 OpenAI 客户端，其 `baseURL` 指向 `PORTKEY_GATEWAY_URL`，并通过 `defaultHeaders` 注入 `x-portkey-config` 头部。`handleChatCompletions` 或 `handleModels` 函数使用这个配置好的客户端向 Portkey Gateway 发起实际请求。Portkey Gateway 收到带有 `x-portkey-config` 头部的请求后，应用相应的配置并将请求转发给实际的 LLM 提供商。

## 总结

本文介绍了两种解决 Portkey AI Gateway 与标准 OpenAI SDK 兼容性问题的中间层方案：使用 Nginx 进行请求转发和头部注入，以及构建一个基于 Deno Deploy 的轻量级服务来处理请求并创建包含 Portkey 配置的客户端。选择哪种方案取决于您的现有基础设施、技术偏好以及所需的灵活性和复杂性。无论哪种方式，搭建中间层都能有效地桥接 OpenAI SDK 的便利性与 Portkey Gateway 的高级功能，帮助您更好地管理和利用 LLM API。
