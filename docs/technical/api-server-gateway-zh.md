# API 服务器 - 将 Cherry Studio 用作网关

Cherry Studio 内置了一个 API 服务器，允许您将其用作 AI 模型的网关/代理，类似于 LM Studio。这使得外部应用程序可以通过统一的 OpenAI 兼容 API 访问您配置的 AI 提供商。

## 概述

API 服务器提供：
- **OpenAI 兼容端点**用于聊天补全
- **Anthropic 兼容端点**用于消息
- **模型列表**用于发现可用模型
- **Swagger 文档**用于 API 探索

## 开始使用

### 1. 启用 API 服务器

1. 进入 **设置** → **工具** → **API 服务器**
2. 配置端口（默认：`23333`）
3. 点击 **启动** 按钮启动服务器
4. 复制生成的 API 密钥用于认证

### 2. 配置您的应用程序

在您的外部应用程序中使用以下基础 URL：

```
http://localhost:23333
```

### 3. 认证

除 `/health` 和 `/api-docs` 外，所有 API 请求都需要使用以下方法之一进行认证：

**Bearer Token：**
```
Authorization: Bearer your-api-key
```

**X-API-Key 头：**
```
X-API-Key: your-api-key
```

## API 端点

### 健康检查

```http
GET /health
```

无需认证。返回服务器状态。

### 列出可用模型

```http
GET /v1/models
Authorization: Bearer your-api-key
```

**查询参数：**
- `providerType`（可选）：按提供商类型过滤（`openai`、`anthropic`）
- `limit`（可选）：返回的最大模型数量
- `offset`（可选）：分页偏移量

**响应：**
```json
{
  "object": "list",
  "data": [
    {
      "id": "my-openai:gpt-4",
      "object": "model",
      "name": "GPT-4",
      "created": 1234567890,
      "owned_by": "OpenAI",
      "provider": "my-openai",
      "provider_type": "openai"
    }
  ]
}
```

### 聊天补全（OpenAI 兼容）

```http
POST /v1/chat/completions
Authorization: Bearer your-api-key
Content-Type: application/json
```

**请求体：**
```json
{
  "model": "provider-id:model-id",
  "messages": [
    {
      "role": "user",
      "content": "你好，你好吗？"
    }
  ],
  "temperature": 0.7,
  "stream": false
}
```

**模型格式：** 模型必须指定为 `provider-id:model-id`，其中：
- `provider-id` 是您在 Cherry Studio 中配置的提供商 ID
- `model-id` 是实际的模型标识符

示例：`my-openai:gpt-4`、`anthropic:claude-3-opus-20240229`

### 消息（Anthropic 兼容）

```http
POST /v1/messages
Authorization: Bearer your-api-key
Content-Type: application/json
```

**请求体：**
```json
{
  "model": "provider-id:model-id",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user",
      "content": "你好，Claude！"
    }
  ],
  "stream": false
}
```

### 特定提供商的消息

```http
POST /{provider-id}/v1/messages
Authorization: Bearer your-api-key
Content-Type: application/json
```

此端点允许您在 URL 路径中指定提供商，而不是在模型字符串中：

```json
{
  "model": "claude-3-opus-20240229",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user",
      "content": "你好！"
    }
  ]
}
```

## 流式响应

聊天补全和消息端点都支持流式传输。在请求中设置 `"stream": true`：

```json
{
  "model": "provider-id:model-id",
  "messages": [...],
  "stream": true
}
```

服务器将以服务器发送事件（SSE）响应：

```
data: {"id":"...","choices":[{"delta":{"content":"你好"}}],...}

data: {"id":"...","choices":[{"delta":{"content":"世界"}}],...}

data: [DONE]
```

## API 文档

交互式 Swagger/OpenAPI 文档可在以下地址访问：

```
http://localhost:23333/api-docs
```

OpenAPI 规范可从以下地址下载：

```
http://localhost:23333/api-docs.json
```

## 示例：使用 curl

### 列出模型

```bash
curl -X GET "http://localhost:23333/v1/models" \
  -H "Authorization: Bearer your-api-key"
```

### 聊天补全

```bash
curl -X POST "http://localhost:23333/v1/chat/completions" \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-openai:gpt-4",
    "messages": [{"role": "user", "content": "打个招呼！"}]
  }'
```

### 流式聊天补全

```bash
curl -X POST "http://localhost:23333/v1/chat/completions" \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-openai:gpt-4",
    "messages": [{"role": "user", "content": "给我讲个故事"}],
    "stream": true
  }'
```

## 示例：使用 Python

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:23333/v1",
    api_key="your-cherry-studio-api-key"
)

# 列出模型
models = client.models.list()
for model in models:
    print(model.id)

# 聊天补全
response = client.chat.completions.create(
    model="my-openai:gpt-4",
    messages=[{"role": "user", "content": "你好！"}]
)
print(response.choices[0].message.content)

# 流式传输
stream = client.chat.completions.create(
    model="my-openai:gpt-4",
    messages=[{"role": "user", "content": "给我讲个故事"}],
    stream=True
)
for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```

## 支持的提供商类型

API 服务器目前支持：
- **OpenAI 类型提供商**（包括 OpenAI、Azure OpenAI、兼容 API）
- **Anthropic 类型提供商**

## 与 LM Studio 比较

| 功能 | Cherry Studio | LM Studio |
|------|---------------|-----------|
| OpenAI 兼容 API | ✅ | ✅ |
| Anthropic 兼容 API | ✅ | ❌ |
| 多提供商支持 | ✅ | ❌（仅本地） |
| 云模型支持 | ✅ | ❌ |
| 本地模型支持 | ✅（通过 Ollama） | ✅ |
| API 认证 | ✅ | ✅ |
| Swagger 文档 | ✅ | ❌ |

## 故障排除

### 服务器无法启动
- 检查端口是否已被其他应用程序占用
- 尝试在设置中使用不同的端口

### 认证错误
- 确保您使用的是 Cherry Studio 设置中正确的 API 密钥
- 检查 Authorization 头是否格式正确

### 找不到模型
- 验证提供商是否在 Cherry Studio 中已启用
- 确保模型存在于提供商的模型列表中
- 使用 `provider-id:model-id` 格式作为模型参数

### 提供商不支持
- 目前仅支持 `openai` 和 `anthropic` 类型的提供商
- 其他提供商类型可能会在未来版本中添加
