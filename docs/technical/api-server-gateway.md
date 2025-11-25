# API Server - Using Cherry Studio as a Gateway

Cherry Studio includes a built-in API server that allows you to use it as a gateway/proxy for AI models, similar to LM Studio. This enables external applications to access your configured AI providers through a unified, OpenAI-compatible API.

## Overview

The API server exposes:
- **OpenAI-compatible endpoints** for chat completions
- **Anthropic-compatible endpoints** for messages
- **Model listing** for discovering available models
- **Swagger documentation** for API exploration

## Getting Started

### 1. Enable the API Server

1. Go to **Settings** → **Tools** → **API Server**
2. Configure the port (default: `23333`)
3. Click the **Start** button to launch the server
4. Copy the generated API key for authentication

### 2. Configure Your Application

Use the following base URL in your external application:

```
http://localhost:23333
```

### 3. Authentication

All API requests (except `/health` and `/api-docs`) require authentication using one of these methods:

**Bearer Token:**
```
Authorization: Bearer your-api-key
```

**X-API-Key Header:**
```
X-API-Key: your-api-key
```

## API Endpoints

### Health Check

```http
GET /health
```

No authentication required. Returns server status.

### List Available Models

```http
GET /v1/models
Authorization: Bearer your-api-key
```

**Query Parameters:**
- `providerType` (optional): Filter by provider type (`openai`, `anthropic`)
- `limit` (optional): Maximum number of models to return
- `offset` (optional): Pagination offset

**Response:**
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

### Chat Completions (OpenAI Compatible)

```http
POST /v1/chat/completions
Authorization: Bearer your-api-key
Content-Type: application/json
```

**Request Body:**
```json
{
  "model": "provider-id:model-id",
  "messages": [
    {
      "role": "user",
      "content": "Hello, how are you?"
    }
  ],
  "temperature": 0.7,
  "stream": false
}
```

**Model Format:** The model must be specified as `provider-id:model-id`, where:
- `provider-id` is the ID of your configured provider in Cherry Studio
- `model-id` is the actual model identifier

Example: `my-openai:gpt-4`, `anthropic:claude-3-opus-20240229`

### Messages (Anthropic Compatible)

```http
POST /v1/messages
Authorization: Bearer your-api-key
Content-Type: application/json
```

**Request Body:**
```json
{
  "model": "provider-id:model-id",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user",
      "content": "Hello, Claude!"
    }
  ],
  "stream": false
}
```

### Provider-Specific Messages

```http
POST /{provider-id}/v1/messages
Authorization: Bearer your-api-key
Content-Type: application/json
```

This endpoint allows you to specify the provider in the URL path instead of the model string:

```json
{
  "model": "claude-3-opus-20240229",
  "max_tokens": 1024,
  "messages": [
    {
      "role": "user",
      "content": "Hello!"
    }
  ]
}
```

## Streaming Responses

Both chat completions and messages endpoints support streaming. Set `"stream": true` in your request:

```json
{
  "model": "provider-id:model-id",
  "messages": [...],
  "stream": true
}
```

The server will respond with Server-Sent Events (SSE):

```
data: {"id":"...","choices":[{"delta":{"content":"Hello"}}],...}

data: {"id":"...","choices":[{"delta":{"content":" there"}}],...}

data: [DONE]
```

## API Documentation

Interactive Swagger/OpenAPI documentation is available at:

```
http://localhost:23333/api-docs
```

The OpenAPI specification can be downloaded from:

```
http://localhost:23333/api-docs.json
```

## Example: Using with curl

### List Models

```bash
curl -X GET "http://localhost:23333/v1/models" \
  -H "Authorization: Bearer your-api-key"
```

### Chat Completion

```bash
curl -X POST "http://localhost:23333/v1/chat/completions" \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-openai:gpt-4",
    "messages": [{"role": "user", "content": "Say hello!"}]
  }'
```

### Streaming Chat Completion

```bash
curl -X POST "http://localhost:23333/v1/chat/completions" \
  -H "Authorization: Bearer your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-openai:gpt-4",
    "messages": [{"role": "user", "content": "Tell me a story"}],
    "stream": true
  }'
```

## Example: Using with Python

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:23333/v1",
    api_key="your-cherry-studio-api-key"
)

# List models
models = client.models.list()
for model in models:
    print(model.id)

# Chat completion
response = client.chat.completions.create(
    model="my-openai:gpt-4",
    messages=[{"role": "user", "content": "Hello!"}]
)
print(response.choices[0].message.content)

# Streaming
stream = client.chat.completions.create(
    model="my-openai:gpt-4",
    messages=[{"role": "user", "content": "Tell me a story"}],
    stream=True
)
for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```

## Supported Provider Types

The API server currently supports:
- **OpenAI-type providers** (including OpenAI, Azure OpenAI, compatible APIs)
- **Anthropic-type providers**

## Comparison with LM Studio

| Feature | Cherry Studio | LM Studio |
|---------|---------------|-----------|
| OpenAI-compatible API | ✅ | ✅ |
| Anthropic-compatible API | ✅ | ❌ |
| Multiple provider support | ✅ | ❌ (local only) |
| Cloud model support | ✅ | ❌ |
| Local model support | ✅ (via Ollama) | ✅ |
| API authentication | ✅ | ✅ |
| Swagger documentation | ✅ | ❌ |

## Troubleshooting

### Server won't start
- Check if the port is already in use by another application
- Try a different port in the settings

### Authentication errors
- Ensure you're using the correct API key from Cherry Studio settings
- Check that the Authorization header is properly formatted

### Model not found
- Verify the provider is enabled in Cherry Studio
- Ensure the model exists in the provider's model list
- Use the format `provider-id:model-id` for the model parameter

### Provider not supported
- Currently only `openai` and `anthropic` type providers are supported
- Other provider types may be added in future releases
