# 怎么调用Claude API？国内接入、流式输出与代码教程

如果你在用 Node.js、Express、NestJS 或 Next.js 服务端开发项目，准备接入 Claude，最关心的问题通常是：

**Node.js 到底怎么调用 Claude API？**

实际接入时，开发者经常会卡在这几步：

- 应该把 Claude API Key 放在前端还是后端？
- 请求地址和模型名称怎么配置？
- 怎么用 Node.js 发送第一个 Claude 请求？
- 怎么接收流式输出，并把结果传给网页？
- 出现 `401`、`404`、`429` 或连接失败时，应该先检查什么？

本文围绕 **Claude API 国内调用**和 **Node.js 调用 Claude API** 展开，介绍从配置到上线前需要注意的关键步骤。

文中的代码运行在 **Node.js 18 及以上版本的服务端环境**中，默认演示 **OpenAI 兼容格式的 Claude 接口**。如果所用服务只支持 Claude 原生 `/v1/messages` 接口，需要按对应请求结构调整。

**Claude API 中转服务入口：**

> <https://quanzil.com>

> <https://quanzil.net>

实际接口地址、支持的 Claude 模型、接口格式和计费规则，请以平台控制台及开发文档为准。

---

## Node.js调用Claude API，为什么应该放在后端？

有些开发者刚拿到 Claude API Key，会尝试直接在浏览器 JavaScript 中发起请求。

这种做法存在明显问题：浏览器代码和网络请求都可以被用户查看，密钥一旦暴露，就可能被他人盗用，导致额度被消耗。

更稳妥的结构是：

```text
网页或小程序 → 你的Node.js后端 → Claude中转API → 返回结果
```

把 Claude API 调用放在 Node.js 后端，可以同时解决几个问题：

- API Key 保存在服务器环境变量中，不下发到浏览器。
- 后端可以先验证用户登录状态和使用权限。
- 可以限制单次输入长度和请求频率。
- 可以统一记录耗时、状态码和 Token 用量。
- 后续更换 Claude 模型或接口地址时，不需要修改前端代码。

即使使用 Next.js、Nuxt 等全栈框架，也应将密钥放在服务端环境变量中，不要使用会暴露给浏览器的公开变量前缀。

---

## 调用前需要准备哪三项配置？

在编写 Node.js 代码前，先从同一个服务平台获取以下信息：

| 配置项 | 环境变量名 | 说明 |
| --- | --- | --- |
| API Key | `CLAUDE_API_KEY` | 用于接口鉴权的密钥 |
| Base URL | `CLAUDE_BASE_URL` | 接口基础地址，通常包含`/v1` |
| 模型名称 | `CLAUDE_MODEL` | 平台模型列表中的准确Claude标识 |

这三项配置必须相互匹配。

macOS 或 Linux 可以在终端设置：

```bash
export CLAUDE_API_KEY="替换为你的API_Key"
export CLAUDE_BASE_URL="https://你的接口域名/v1"
export CLAUDE_MODEL="替换为平台支持的Claude模型名称"
```

Windows PowerShell 可以设置：

```powershell
$env:CLAUDE_API_KEY="替换为你的API_Key"
$env:CLAUDE_BASE_URL="https://你的接口域名/v1"
$env:CLAUDE_MODEL="替换为平台支持的Claude模型名称"
```

以上内容都是占位值，运行前需要替换成真实配置。

如果使用 `.env` 文件保存本地配置，应把 `.env` 加入 `.gitignore`，避免提交到代码仓库。

接口文档可以查看：

- [Claude API接入开发文档](/docs.html)

---

## 方法一：使用Node.js内置fetch调用Claude API

Node.js 18 及以上版本已经内置 `fetch`。在不安装额外 SDK 的情况下，可以先发送一个最小请求，确认地址、密钥和模型是否可用。

新建 `claude-fetch.mjs`：

```javascript
const baseUrl = process.env.CLAUDE_BASE_URL?.replace(/\/+$/, "");
const apiKey = process.env.CLAUDE_API_KEY;
const model = process.env.CLAUDE_MODEL;

if (!baseUrl || !apiKey || !model) {
  throw new Error("请先设置 CLAUDE_BASE_URL、CLAUDE_API_KEY 和 CLAUDE_MODEL");
}

const response = await fetch(`${baseUrl}/chat/completions`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: `Bearer ${apiKey}`,
  },
  body: JSON.stringify({
    model,
    messages: [
      {
        role: "system",
        content: "请使用简洁、准确的中文回答。",
      },
      {
        role: "user",
        content: "请用三句话说明Node.js接入Claude API的基本流程。",
      },
    ],
    max_tokens: 400,
  }),
  signal: AbortSignal.timeout(60_000),
});

if (!response.ok) {
  const errorText = await response.text();
  throw new Error(`HTTP ${response.status}: ${errorText}`);
}

const data = await response.json();
console.log("Claude回答：");
console.log(data.choices?.[0]?.message?.content ?? "");
console.log("Token用量：", data.usage ?? {});
```

运行：

```bash
node claude-fetch.mjs
```

这段代码会向 `${baseUrl}/chat/completions` 发送请求，并设置 60 秒超时。

如果终端打印出回答和 `usage` 信息，说明当前环境已经可以完成基础的 **Claude API 调用**。

---

## 方法二：使用OpenAI Node SDK调用Claude接口

如果所选 Claude 中转站支持 OpenAI 兼容接口，也可以使用官方 `openai` Node SDK 来管理请求、超时和流式输出。

安装依赖：

```bash
npm install openai
```

新建 `claude-sdk.mjs`：

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.CLAUDE_API_KEY,
  baseURL: process.env.CLAUDE_BASE_URL?.replace(/\/+$/, ""),
  timeout: 60_000,
  maxRetries: 2,
});

const completion = await client.chat.completions.create({
  model: process.env.CLAUDE_MODEL,
  messages: [
    {
      role: "user",
      content: "请列出Claude适合处理的五类常见开发任务。",
    },
  ],
  max_tokens: 500,
});

console.log(completion.choices[0]?.message?.content ?? "");
```

这里需要注意两点：

- `baseURL` 一般填写到 `/v1`，SDK 会自动拼接 `/chat/completions`。
- 虽然安装了 `openai` 包，实际调用哪个模型仍由 `baseURL` 和 `model` 决定。

如果把完整接口地址 `/v1/chat/completions` 填进 `baseURL`，通常会出现路径重复，导致返回 `404`。

---

## 如果平台使用Claude原生接口，Node.js请求有什么不同？

有些平台提供的是 Claude 原生 Messages 接口，常见路径是 `/v1/messages`。

它与兼容格式的主要区别如下：

| 对比项 | OpenAI兼容格式 | Claude原生Messages格式 |
| --- | --- | --- |
| 常见路径 | `/v1/chat/completions` | `/v1/messages` |
| 常见鉴权头 | `Authorization: Bearer` | `x-api-key` |
| 版本请求头 | 一般不需要 | 常见为`anthropic-version` |
| 系统提示词 | `messages`中的`system`角色 | 顶层`system`字段 |
| 回答读取方式 | `choices[0].message.content` | `content`数组中的文本块 |

如果平台要求使用原生格式，可以用下面的 `fetch` 结构测试：

```javascript
const response = await fetch(`${baseUrl}/messages`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "x-api-key": apiKey,
    "anthropic-version": "2023-06-01",
  },
  body: JSON.stringify({
    model,
    max_tokens: 400,
    system: "请使用简洁中文回答。",
    messages: [
      {
        role: "user",
        content: "请解释Claude原生Messages接口的核心字段。",
      },
    ],
  }),
});

const data = await response.json();
const answer = (data.content ?? [])
  .filter((block) => block.type === "text")
  .map((block) => block.text)
  .join("");

console.log(answer);
```

部分中转平台即使提供 `/v1/messages`，鉴权头或版本号要求也可能略有差异，应以所用平台的开发文档为准。

---

## Node.js怎么接收Claude流式输出？

在聊天窗口、代码助手或长文生成场景中，流式输出可以让内容边生成边显示，减少用户等待首屏内容的感知时间。

兼容格式下的 Node.js 流式调用示例如下：

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: process.env.CLAUDE_API_KEY,
  baseURL: process.env.CLAUDE_BASE_URL?.replace(/\/+$/, ""),
  timeout: 60_000,
});

const stream = await client.chat.completions.create({
  model: process.env.CLAUDE_MODEL,
  messages: [
    {
      role: "user",
      content: "请分步骤说明如何用Node.js搭建Claude问答接口。",
    },
  ],
  max_tokens: 800,
  stream: true,
});

for await (const chunk of stream) {
  const delta = chunk.choices?.[0]?.delta?.content;
  if (delta) {
    process.stdout.write(delta);
  }
}

process.stdout.write("\n");
```

实际接入网页时，通常是：

1. 浏览器请求你的 Node.js 接口。
2. Node.js 后端向 Claude 接口发起 `stream: true` 请求。
3. 后端将收到的文本片段通过 SSE 或流式响应转发给前端。

注意，流式输出主要改善展示体验，并不代表相同输入和输出的 Token 费用会自动降低。

---

## 多轮对话在Node.js服务中怎么管理？

Claude 接口本身不会自动替你的业务保存每个用户的聊天记录。要实现连续对话，后端需要把相关的历史消息一起传给模型。

可以用一个精简数组控制上下文长度：

```javascript
function buildMessages(systemPrompt, history, userInput, maxTurns = 6) {
  const recentMessages = history.slice(-maxTurns * 2);

  return [
    { role: "system", content: systemPrompt },
    ...recentMessages,
    { role: "user", content: userInput },
  ];
}
```

这里的 `history` 只保存当前用户会话中需要保留的 `user` 和 `assistant` 消息。

这样做有两个好处：

- 防止不同用户的上下文混在一起。
- 避免历史消息无限增长，导致输入 Token 和响应时间持续增加。

如果业务需要长期记忆，可以把早期对话总结成简短摘要，再和最近几轮对话一起发送。

价格与 Token 成本计算可以参考：

- [Claude API调用价格说明](/pricing.html)
- [Claude API国内调用多少钱？计费方式、费用估算与省钱指南](/blog/claude-api/claude-api-pricing-cost-guide.html)

---

## Node.js调用Claude API常见报错怎么排查？

| 报错或现象 | 常见原因 | 排查重点 |
| --- | --- | --- |
| `401 Unauthorized` | API Key错误、失效，或鉴权头不匹配 | 检查Key、Base URL和请求头格式 |
| `404 Not Found` | `/v1`重复、路径写错，或模型名不存在 | 打印最终请求URL并核对模型列表 |
| `400 Bad Request` | 请求体字段不合法，或混用两种接口格式 | 检查`model`、`messages`、`max_tokens` |
| `429 Too Many Requests` | 并发过高、触发限流，或账户额度不足 | 查看响应正文，区分限流与余额问题 |
| `fetch failed` / 超时 | 域名填错、网络不通，或长文本耗时较久 | 先用短请求测试，再调整超时和网络配置 |

排查时，建议先看三件事：

1. **最终请求的完整 URL 是什么。** 很多 `404` 都是因为 `/v1/v1` 或重复拼接路径。
2. **错误响应正文写了什么。** 不要只看状态码，`400` 和 `429` 的具体原因通常在响应体里。
3. **最小请求能否成功。** 先用一句话测试，排除业务代码、长文本和复杂参数的影响。

调试日志中可以记录状态码、耗时和请求 ID，但不要把完整的 `CLAUDE_API_KEY` 打印到控制台或日志系统里。

---

## Node.js接入Claude API常见问题

### 前端React或Vue项目可以直接调用Claude API吗？

不建议直接从浏览器调用。React、Vue 或小程序前端应先请求你自己的 Node.js 后端接口，由后端携带 API Key 调用 Claude，再把结果返回给前端。

### TypeScript项目怎么调用Claude API？

本文的代码同样适用于 TypeScript。使用 `openai` Node SDK 时，包内自带类型定义；使用 `fetch` 时，可以为请求体和返回结构定义接口类型，编译或运行前确保环境变量已正确加载。

### 为什么本地终端能跑通，部署到服务器后却报错？

常见原因是服务器、Docker 容器或 PM2 进程中没有注入 `CLAUDE_API_KEY`、`CLAUDE_BASE_URL` 和 `CLAUDE_MODEL`，或者服务器网络环境、超时设置与本地不同。可以先在部署环境中检查环境变量是否生效。

### 切换不同Claude模型时，Node.js代码要重写吗？

在同一个兼容接口下，基础文本问答通常只需要修改 `CLAUDE_MODEL` 环境变量。切换后仍建议用真实业务样本测试输出长度、响应速度和费用变化。

---

## 从一个最小Node.js脚本开始接入Claude

如果你准备在 Node.js 项目中进行 Claude API 国内调用，可以按下面的顺序推进：

1. 获取同一平台的 API Key、Base URL 和 Claude 模型名称。
2. 在服务端设置环境变量，不在前端暴露密钥。
3. 先用 `claude-fetch.mjs` 跑通最小请求。
4. 根据业务需要接入 SDK、流式输出和多轮对话裁剪。
5. 增加超时控制、错误日志和用量统计后，再接入正式流量。

**Claude API 中转服务入口：**

> <https://quanzil.com>

> <https://quanzil.net>

