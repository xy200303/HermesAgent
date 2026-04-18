---
title: API Server 接口文档
description: Hermes Agent OpenAI 兼容 API 的前端对接参考文档
---

# API Server 接口文档

这是一份面向前端开发的 Hermes Agent API Server 对接文档，对应实现位于 [gateway/platforms/api_server.py](/docs/developer-guide/gateway-internals)。

Hermes 对外暴露的 OpenAI 兼容 HTTP 基础地址为：

```text
http://<host>:<port>/v1
```

本地开发常见地址：

```text
http://127.0.0.1:8642/v1
```

## 接口总览

Hermes 当前提供以下接口：

- `POST /v1/chat/completions`
- `POST /v1/responses`
- `GET /v1/responses/{response_id}`
- `DELETE /v1/responses/{response_id}`
- `GET /v1/models`
- `POST /v1/runs`
- `GET /v1/runs/{run_id}/events`
- `GET /health`
- `GET /health/detailed`
- `GET /v1/health`

## 鉴权

当服务端配置了 `API_SERVER_KEY` 时，Hermes 使用 Bearer Token 鉴权。

每个请求都建议携带：

```http
Authorization: Bearer <API_SERVER_KEY>
```

说明：

- 如果 Hermes 绑定的是非本地回环地址，比如 `0.0.0.0`，那么必须配置 `API_SERVER_KEY`，否则服务会拒绝启动。
- 如果 Hermes 只绑定在 `127.0.0.1` 且未配置 key，则默认允许无鉴权访问。
- 即使是本地开发，也建议前端统一携带 Bearer Token，方便后续切换到测试/生产环境。

## 通用请求头

### 请求头

| Header | 是否必需 | 说明 |
| --- | --- | --- |
| `Authorization` | 通常必需 | Bearer Token |
| `Content-Type: application/json` | JSON POST 必需 | JSON 请求体 |
| `Idempotency-Key` | 可选 | 幂等键，短时间内用于去重相同请求 |
| `X-Hermes-Session-Id` | 可选 | Chat Completions 场景下复用 Hermes 服务端会话 |

### 响应头

| Header | 出现位置 | 说明 |
| --- | --- | --- |
| `X-Hermes-Session-Id` | Chat Completions 响应 | Hermes 会话 id，可用于后续续接 |
| `Content-Type: text/event-stream` | 流式接口 | SSE 事件流 |

## content 字段兼容形式

Hermes 支持纯字符串，也兼容常见 OpenAI 风格的内容数组。

可接受形式：

```json
{"content": "hello"}
```

```json
{
  "content": [
    {"type": "text", "text": "hello"},
    {"type": "input_text", "text": "world"}
  ]
}
```

Hermes 会把支持的文本片段展平成纯文本再送入 Agent。像 `image_url` 这类非文本内容，在当前 API Server 路径下会被忽略。

## 错误返回格式

大多数参数校验错误和服务端错误都使用 OpenAI 风格的错误包裹：

```json
{
  "error": {
    "message": "Missing 'input' field",
    "type": "invalid_request_error",
    "param": null,
    "code": null
  }
}
```

常见状态码：

- `400` 请求体非法、JSON 无效、参数缺失
- `401` API Key 无效
- `403` 功能被禁止，例如当前配置不允许会话续接
- `404` `response_id` 或 `run_id` 不存在
- `413` 请求体过大
- `429` 并发 run 数量超限
- `500` 服务端内部错误

## 1. GET /health

基础存活检查接口。

### 请求

```http
GET /health
```

无需鉴权。

### 响应

```json
{
  "status": "ok",
  "platform": "hermes-agent"
}
```

## 2. GET /v1/health

与 `/health` 相同，只是路径带 `/v1` 前缀，方便兼容某些客户端约定。

## 3. GET /health/detailed

返回更详细的运行状态，适合状态页、管理后台、容器探针使用。

### 请求

```http
GET /health/detailed
```

无需鉴权。

### 响应

```json
{
  "status": "ok",
  "platform": "hermes-agent",
  "gateway_state": "running",
  "platforms": {
    "api_server": {
      "connected": true
    }
  },
  "active_agents": 0,
  "exit_reason": null,
  "updated_at": 1760000000.0,
  "pid": 12345
}
```

字段说明：

- `gateway_state`：网关当前状态
- `platforms`：已连接平台信息
- `active_agents`：当前活跃会话/任务数
- `updated_at`：最近一次状态更新时间
- `pid`：当前进程号

## 4. GET /v1/models

获取当前上游接口支持的模型列表。

### 请求

```http
GET /v1/models
Authorization: Bearer <token>
```

### 响应

```json
{
  "object": "list",
  "data": [
    {
      "id": "openai/gpt-4.1",
      "object": "model",
      "created": 1760000000,
      "owned_by": "openai",
      "permission": [],
      "root": "openai/gpt-4.1",
      "parent": null
    },
    {
      "id": "minimax-m2.5",
      "object": "model",
      "created": 1760000000,
      "owned_by": "custom",
      "permission": [],
      "root": "minimax-m2.5",
      "parent": null
    }
  ]
}
```

说明：

- Hermes 会优先去探测当前实际上游接口的 `/models`，并把上游返回的模型列表透传为 OpenAI 兼容格式。
- 返回的 `id` 就是前端请求里应传入的 `model`。
- 对于 `/v1/chat/completions`、`/v1/responses`、`/v1/runs`，请求中的 `model` 会作为 Hermes 实际调用的底层模型。
- 如果请求里不传 `model`，Hermes 才会回退到服务端当前默认模型。
- 如果 Hermes 无法获取上游模型列表，比如上游不支持 `/models`、网络异常、鉴权失败，则会回退为单个本地占位模型：
  - 默认 profile 下通常是 `hermes-agent`
  - 命名 profile 下通常是当前 profile 名
  - 如果配置了 `model_name`，则回退为该名称

## 5. POST /v1/chat/completions

OpenAI Chat Completions 兼容接口。

适合前端自己维护完整消息历史的场景。

### 行为说明

- 逻辑上偏无状态
- 每轮请求由前端发送完整 `messages`
- Hermes 会取最后一个 `user` 消息作为本轮真正输入
- 更早的 `user` / `assistant` 消息会当作 `conversation_history`
- `system` 消息会叠加到 Hermes 自带系统提示词之上

### 请求体字段

| 字段 | 类型 | 是否必需 | 说明 |
| --- | --- | --- | --- |
| `model` | string | 否 | 模型名，通常取 `/v1/models` 返回值；如果传入，会作为 Hermes 实际调用的底层模型 |
| `messages` | array | 是 | OpenAI 风格消息数组 |
| `stream` | boolean | 否 | `true` 时返回 SSE |
| `tools` | any | 否 | 为兼容客户端保留，Hermes 不按 OpenAI tools 协议直接执行 |
| `tool_choice` | any | 否 | 为兼容客户端保留 |

### 请求示例

```json
{
  "model": "hermes-agent",
  "messages": [
    {"role": "system", "content": "You are a coding assistant."},
    {"role": "user", "content": "List files in the current project"},
    {"role": "assistant", "content": "I will inspect the workspace."},
    {"role": "user", "content": "Now open README.md"}
  ],
  "stream": false
}
```

### 非流式响应

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1760000000,
  "model": "hermes-agent",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Here is the README summary..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 120,
    "completion_tokens": 240,
    "total_tokens": 360
  }
}
```

Hermes 同时还会返回响应头：

```http
X-Hermes-Session-Id: api-<stable-id>
```

### 通过 X-Hermes-Session-Id 做会话续接

Chat Completions 支持通过请求头 `X-Hermes-Session-Id` 来做服务端会话续接。

如果带上这个 header：

- Hermes 会从服务端 session 数据库中加载上下文历史
- 前端不需要每次都重发完整消息历史
- 只有在服务端配置了 `API_SERVER_KEY` 且鉴权通过时，这个功能才允许使用

示例：

```http
POST /v1/chat/completions
Authorization: Bearer <token>
X-Hermes-Session-Id: api-1a2b3c4d5e6f
Content-Type: application/json
```

```json
{
  "model": "hermes-agent",
  "messages": [
    {"role": "user", "content": "Continue from where we left off and run the tests"}
  ]
}
```

推荐策略：

- 如果前端本身已经维护会话历史，就按标准 Chat Completions 无状态方式调用
- 如果希望由 Hermes 自己管理上下文，可保存 `X-Hermes-Session-Id` 并在后续请求中继续带回

### 流式返回

当 `stream=true` 时，Hermes 返回 SSE 流。

其中会混合两类事件：

1. 标准 `chat.completion.chunk`
2. Hermes 自定义的 `hermes.tool.progress`

#### 流式事件示例

1. 标准文本块：

```text
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"delta":{"role":"assistant"},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"delta":{"content":"Hello"},"finish_reason":null}]}
```

2. 工具进度事件：

```text
event: hermes.tool.progress
data: {"tool":"read_file","emoji":"📄","label":"README.md"}
```

3. 结束块：

```text
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","choices":[{"delta":{},"finish_reason":"stop"}],"usage":{"prompt_tokens":120,"completion_tokens":240,"total_tokens":360}}

data: [DONE]
```

#### 前端处理注意事项

`hermes.tool.progress` 是给 UI 展示用的自定义事件。

不要把它拼进 assistant 的最终文本消息里。这个事件存在的目的就是为了让工具过程可视化，同时不污染消息历史。

### UI 建议

- `delta.content` 按普通 assistant 文本流渲染
- `hermes.tool.progress` 单独显示为“正在读取文件 / 正在执行命令”之类的状态条
- 最终落库或缓存时，只保存 assistant 文本，不保存工具进度事件

## 6. POST /v1/responses

OpenAI Responses 兼容接口。

这是更推荐前端接入的接口，因为它支持：

- `previous_response_id` 形式的服务端状态
- `conversation` 命名会话
- 显式传入 `conversation_history`
- 更结构化的工具调用流
- 最终响应里结构化的 `output` 项

### 请求体字段

| 字段 | 类型 | 是否必需 | 说明 |
| --- | --- | --- | --- |
| `model` | string | 否 | 模型名；如果传入，会作为 Hermes 实际调用的底层模型 |
| `input` | string or array | 是 | 当前输入 |
| `instructions` | string | 否 | 额外系统指令，会叠加在 Hermes 系统提示词之上 |
| `stream` | boolean | 否 | `true` 时走 SSE |
| `store` | boolean | 否 | 默认 `true`，是否存储响应供后续获取/续接 |
| `previous_response_id` | string | 否 | 基于上一条已存储响应继续对话 |
| `conversation` | string | 否 | 命名会话名，自动映射到最近一次响应 |
| `conversation_history` | array | 否 | 显式历史，优先级高于 `previous_response_id` |
| `truncation` | string | 否 | 传 `auto` 时，Hermes 会自动截断过长历史 |
| `tools` | any | 否 | 兼容字段 |

### 状态管理方式

三选一即可：

1. 前端自己传完整 `conversation_history`
2. 用 `previous_response_id`
3. 用命名会话 `conversation`

不要同时传 `conversation` 和 `previous_response_id`。

### 最小请求示例

```json
{
  "model": "hermes-agent",
  "input": "Show me the project structure",
  "store": true
}
```

### 基于 previous_response_id 的续接

```json
{
  "input": "Now open package.json",
  "previous_response_id": "resp_abc123"
}
```

### 基于命名会话的续接

```json
{
  "input": "Run the tests and summarize failures",
  "conversation": "frontend-debug"
}
```

### 显式历史示例

```json
{
  "input": "Continue",
  "conversation_history": [
    {"role": "user", "content": "Inspect the repo"},
    {"role": "assistant", "content": "I found a React app."}
  ]
}
```

### 非流式响应

```json
{
  "id": "resp_abc123",
  "object": "response",
  "status": "completed",
  "created_at": 1760000000,
  "model": "hermes-agent",
  "output": [
    {
      "type": "function_call",
      "name": "terminal",
      "arguments": "{\"command\":\"ls\"}",
      "call_id": "call_1"
    },
    {
      "type": "function_call_output",
      "call_id": "call_1",
      "output": [
        {"type": "input_text", "text": "README.md\nsrc\ntests"}
      ]
    },
    {
      "type": "message",
      "role": "assistant",
      "content": [
        {"type": "output_text", "text": "The project contains README.md, src, and tests."}
      ]
    }
  ],
  "usage": {
    "input_tokens": 80,
    "output_tokens": 150,
    "total_tokens": 230
  }
}
```

### output 项类型

Hermes 当前会输出这几类 item：

- `function_call`
- `function_call_output`
- `message`

因此 `/v1/responses` 非常适合做“工具过程 + 最终回答”统一时间线展示。

### 流式返回

当 `stream=true` 时，Hermes 会以 SSE 方式输出更结构化的事件。

#### 常见事件顺序

1. `response.created`
2. 若干个 `response.output_item.added`
3. 若干个 `response.output_text.delta`
4. 若干个 `response.output_item.done`
5. `response.output_text.done`
6. `response.completed` 或 `response.failed`

#### 示例流

```text
event: response.created
data: {"type":"response.created","response":{"id":"resp_abc123","object":"response","status":"in_progress","created_at":1760000000,"model":"hermes-agent"}}

event: response.output_item.added
data: {"type":"response.output_item.added","output_index":0,"item":{"id":"fc_1","type":"function_call","status":"in_progress","name":"read_file","call_id":"call_1","arguments":"{\"path\":\"README.md\"}"}}

event: response.output_item.done
data: {"type":"response.output_item.done","output_index":0,"item":{"id":"fc_1","type":"function_call","status":"completed","name":"read_file","call_id":"call_1","arguments":"{\"path\":\"README.md\"}"}}

event: response.output_item.added
data: {"type":"response.output_item.added","output_index":1,"item":{"id":"fco_1","type":"function_call_output","call_id":"call_1","output":[{"type":"input_text","text":"# Hermes Agent"}],"status":"completed"}}

event: response.output_item.added
data: {"type":"response.output_item.added","output_index":2,"item":{"id":"msg_1","type":"message","status":"in_progress","role":"assistant","content":[]}}

event: response.output_text.delta
data: {"type":"response.output_text.delta","item_id":"msg_1","output_index":2,"content_index":0,"delta":"I opened README.md and found ...","logprobs":[]}

event: response.output_text.done
data: {"type":"response.output_text.done","item_id":"msg_1","output_index":2,"content_index":0,"text":"I opened README.md and found ...","logprobs":[]}

event: response.completed
data: {"type":"response.completed","response":{"id":"resp_abc123","object":"response","status":"completed","created_at":1760000000,"model":"hermes-agent","output":[...]}}
```

#### 前端工具过程展示建议

如果前端希望实时展示：

- 文件读取
- 命令执行
- 搜索
- 其他工具调用

推荐使用：

- `/v1/responses`
- 且 `stream=true`

处理方式建议：

- `function_call` 视为“工具开始”
- `function_call_output` 视为“工具结果”
- `response.output_text.delta` 视为 assistant 文本流

这是 Hermes 当前最适合前端构建对话网页的接口形式。

## 7. GET /v1/responses/{response_id}

读取一条已存储的 response。

### 请求

```http
GET /v1/responses/resp_abc123
Authorization: Bearer <token>
```

### 响应

返回内容与最初 `POST /v1/responses` 的存储结果一致。

如果不存在：

```json
{
  "error": {
    "message": "Response not found: resp_abc123",
    "type": "invalid_request_error",
    "param": null,
    "code": null
  }
}
```

## 8. DELETE /v1/responses/{response_id}

删除一条已存储 response。

### 请求

```http
DELETE /v1/responses/resp_abc123
Authorization: Bearer <token>
```

### 响应

```json
{
  "id": "resp_abc123",
  "object": "response",
  "deleted": true
}
```

## 9. POST /v1/runs

立即启动一个 run，并返回 `run_id`。之后前端可再订阅 `/v1/runs/{run_id}/events`。

适合“先创建任务，再单独拉流”的场景。

### 请求体

核心输入字段与 `/v1/responses` 接近，常用包括：

- `model`
- `input`
- `instructions`
- `previous_response_id`
- `conversation_history`
- `session_id`

### 请求示例

```json
{
  "input": "Inspect the repo and summarize the architecture",
  "instructions": "Be concise"
}
```

### 响应

```json
{
  "run_id": "run_1234567890abcdef",
  "status": "started"
}
```

HTTP 状态码为 `202 Accepted`。

## 10. GET /v1/runs/{run_id}/events

消费 `POST /v1/runs` 创建出的事件流。

### 传输格式

- SSE `text/event-stream`
- 事件通过普通 `data: {...}` 推送
- 空闲时 Hermes 会周期性发送 keepalive 注释

### 事件流示例

```text
data: {"event":"tool.started","run_id":"run_123","timestamp":1760000000.1,"tool":"terminal","preview":"ls -la"}

data: {"event":"message.delta","run_id":"run_123","timestamp":1760000000.2,"delta":"I inspected the project."}

data: {"event":"tool.completed","run_id":"run_123","timestamp":1760000000.7,"tool":"terminal","duration":0.532,"error":false}

data: {"event":"run.completed","run_id":"run_123","timestamp":1760000001.1,"output":"I inspected the project and found ...","usage":{"input_tokens":120,"output_tokens":260,"total_tokens":380}}
```

### 事件类型说明

| 事件名 | 含义 |
| --- | --- |
| `tool.started` | 工具调用开始 |
| `tool.completed` | 工具调用结束 |
| `message.delta` | assistant 文本增量 |
| `reasoning.available` | 可显示的 reasoning 摘要 |
| `run.completed` | 任务成功结束 |
| `run.failed` | 任务失败结束 |

推荐用途：

- `/v1/runs` 更像任务编排接口
- `/events` 更像运行状态事件流
- 如果你想要更 OpenAI 风格、更适合对话 UI 的结构化流，优先使用 `/v1/responses`

## CORS

默认情况下，Hermes 不开启浏览器跨域。

如果要允许浏览器直接调用，需要在服务端配置：

```bash
API_SERVER_CORS_ORIGINS=http://localhost:3000,http://127.0.0.1:3000
```

开启后：

- 允许 `Authorization`、`Content-Type`、`Idempotency-Key`
- SSE 流式响应也会带上 CORS 头

## 请求体大小限制

Hermes 默认会限制请求体大小。

当前默认上限：

```text
1,000,000 bytes
```

超出后返回 `413 Request body too large`。

## 前端接入建议

### 如果你要做一个基础聊天页面

使用 `POST /v1/chat/completions`。

### 如果你要展示详细工具过程

使用 `POST /v1/responses` 且 `stream=true`。

### 如果你要做任务中心 / 运行面板

使用 `POST /v1/runs` + `GET /v1/runs/{run_id}/events`。

### 接口选型建议

| 前端需求 | 推荐接口 |
| --- | --- |
| 兼容常规 OpenAI 聊天 | `/v1/chat/completions` |
| 需要丰富工具轨迹和服务端状态 | `/v1/responses` |
| 需要单独的任务启动与事件流 | `/v1/runs` + `/events` |

## 与 OpenAI 标准的差异

- Chat Completions 流式返回里会额外带 `hermes.tool.progress`
- Hermes 接受 OpenAI 风格请求体，但内部执行的是 Hermes 自己的工具系统
- `/v1/chat/completions` 的工具过程展示比 `/v1/responses` 更轻量
- `X-Hermes-Session-Id` 是 Hermes 自定义扩展
- 当前 API 路径对非文本多模态支持有限

## 当前模型选择规则

现在 Hermes API Server 的模型选择规则如下：

1. 如果请求体里传了 `model`，则优先使用该模型作为底层真实模型
2. 如果请求体里没有传 `model`，则回退到 Hermes 服务端当前默认模型

适用接口：

- `/v1/chat/completions`
- `/v1/responses`
- `/v1/runs`

这意味着前端可以在同一个 Hermes 服务上按请求动态切换模型。

## fetch 最小示例

### Chat Completions

```ts
const res = await fetch("http://127.0.0.1:8642/v1/chat/completions", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": "Bearer change-me-local-dev",
  },
  body: JSON.stringify({
    model: "hermes-agent",
    messages: [
      { role: "user", content: "Hello Hermes" }
    ],
  }),
});

const data = await res.json();
console.log(data.choices[0].message.content);
```

### Responses

```ts
const res = await fetch("http://127.0.0.1:8642/v1/responses", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": "Bearer change-me-local-dev",
  },
  body: JSON.stringify({
    model: "hermes-agent",
    input: "List files in the project and explain the structure",
    store: true,
  }),
});

const data = await res.json();
console.log(data.output);
```

## EventSource / 浏览器流式建议

浏览器原生 `EventSource` 不能方便地自定义 Authorization Header。

因此前端如果直接在浏览器里接 SSE，通常建议：

- 通过你自己的后端做代理
- 或者用 `fetch()` 自己解析 SSE 流
- 或者采用像 Open WebUI 这种服务端到服务端的接法

如果是你自己开发对话网页，且需要实时工具过程，通常 `/v1/responses` 是最值得优先接入的流式接口。
