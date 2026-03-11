# Nemotron API Reference

Sources:
- [NVIDIA NIM for LLMs](https://docs.nvidia.com/nim/large-language-models/latest/)
- [OpenAI API Compatibility](https://platform.openai.com/docs/api-reference)

## Endpoints

### Base URL

```
Local NIM: http://localhost:8000/v1
NVIDIA Build: https://integrate.api.nvidia.com/v1
```

### Authentication

```python
# Local NIM (no auth required)
headers = {"Content-Type": "application/json"}

# NVIDIA Build
headers = {
    "Content-Type": "application/json",
    "Authorization": f"Bearer {api_key}"
}
```

## Chat Completions

### POST /v1/chat/completions

Generate chat completions for instruct/chat models.

**Request:**
```json
{
  "model": "nvidia/nemotron-3-nano-30b-a3b",
  "messages": [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "Explain quantum computing."}
  ],
  "temperature": 0.7,
  "max_tokens": 500,
  "top_p": 0.95,
  "frequency_penalty": 0.0,
  "presence_penalty": 0.0,
  "stream": false
}
```

**Response:**
```json
{
  "id": "chatcmpl-123",
  "object": "chat.completion",
  "created": 1234567890,
  "model": "nvidia/nemotron-3-nano-30b-a3b",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Quantum computing is..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 20,
    "completion_tokens": 150,
    "total_tokens": 170
  }
}
```

**Python:**
```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-used"
)

response = client.chat.completions.create(
    model="nvidia/nemotron-3-nano-30b-a3b",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Explain quantum computing."}
    ],
    temperature=0.7,
    max_tokens=500
)

print(response.choices[0].message.content)
```

**cURL:**
```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/nemotron-3-nano-30b-a3b",
    "messages": [
      {"role": "user", "content": "Hello!"}
    ]
  }'
```

### Streaming

**Request:**
```json
{
  "model": "nvidia/nemotron-3-nano-30b-a3b",
  "messages": [{"role": "user", "content": "Count to 10"}],
  "stream": true
}
```

**Response (Server-Sent Events):**
```
data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1234567890,"model":"nvidia/nemotron-3-nano-30b-a3b","choices":[{"index":0,"delta":{"role":"assistant","content":"1"},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1234567890,"model":"nvidia/nemotron-3-nano-30b-a3b","choices":[{"index":0,"delta":{"content":", "},"finish_reason":null}]}

data: {"id":"chatcmpl-123","object":"chat.completion.chunk","created":1234567890,"model":"nvidia/nemotron-3-nano-30b-a3b","choices":[{"index":0,"delta":{"content":"2"},"finish_reason":null}]}

data: [DONE]
```

**Python:**
```python
stream = client.chat.completions.create(
    model="nvidia/nemotron-3-nano-30b-a3b",
    messages=[{"role": "user", "content": "Count to 10"}],
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```

## Completions

### POST /v1/completions

Generate completions for base models.

**Request:**
```json
{
  "model": "nvidia/nemotron-3-nano-30b-a3b",
  "prompt": "Once upon a time",
  "max_tokens": 100,
  "temperature": 0.8,
  "top_p": 0.95,
  "n": 1,
  "stream": false,
  "logprobs": null,
  "stop": ["\n\n"]
}
```

**Response:**
```json
{
  "id": "cmpl-123",
  "object": "text_completion",
  "created": 1234567890,
  "model": "nvidia/nemotron-3-nano-30b-a3b",
  "choices": [
    {
      "text": ", in a land far away...",
      "index": 0,
      "logprobs": null,
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 4,
    "completion_tokens": 8,
    "total_tokens": 12
  }
}
```

**Python:**
```python
response = client.completions.create(
    model="nvidia/nemotron-3-nano-30b-a3b",
    prompt="Once upon a time",
    max_tokens=100,
    temperature=0.8
)

print(response.choices[0].text)
```

## Models

### GET /v1/models

List available models.

**Response:**
```json
{
  "object": "list",
  "data": [
    {
      "id": "nvidia/nemotron-3-nano-30b-a3b",
      "object": "model",
      "created": 1234567890,
      "owned_by": "nvidia"
    }
  ]
}
```

**Python:**
```python
models = client.models.list()
for model in models.data:
    print(model.id)
```

## Health Checks

### GET /v1/health/ready

Check if server is ready for requests.

**Response:**
```json
{"status": "ready"}
```

**cURL:**
```bash
curl http://localhost:8000/v1/health/ready
```

### GET /v1/health/live

Check if server is alive.

**Response:**
```json
{"status": "alive"}
```

## Parameters

### Common Parameters

**model** (string, required)
- Model identifier
- Example: `"nvidia/nemotron-3-nano-30b-a3b"`

**temperature** (float, 0-2, default: 1)
- Sampling temperature
- Higher values: more random
- Lower values: more deterministic

**max_tokens** (integer)
- Maximum tokens to generate
- Default: Model-dependent

**top_p** (float, 0-1, default: 1)
- Nucleus sampling
- Alternative to temperature

**frequency_penalty** (float, -2 to 2, default: 0)
- Penalize token frequency
- Positive: reduce repetition

**presence_penalty** (float, -2 to 2, default: 0)
- Penalize new topics
- Positive: encourage new topics

**stream** (boolean, default: false)
- Stream partial results

**stop** (string or array)
- Stop sequences
- Example: `["\n", "END"]`

**n** (integer, default: 1)
- Number of completions

### Chat-Specific Parameters

**messages** (array, required)
- Conversation history
- Format: `[{"role": "user|assistant|system", "content": "..."}]`

**tools** (array)
- Function definitions for tool calling
- OpenAI function calling format

**tool_choice** (string or object)
- Control tool selection
- Values: `"none"`, `"auto"`, `{"type": "function", "function": {"name": "..."}}`

### Completion-Specific Parameters

**prompt** (string or array, required)
- Text prompt(s)

**suffix** (string)
- Text after completion (infill)

**logprobs** (integer)
- Number of log probabilities to return

**echo** (boolean, default: false)
- Echo prompt in response

## Function Calling

### Define Functions

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get current weather",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "City name"
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"]
                    }
                },
                "required": ["location"]
            }
        }
    }
]
```

### Request with Tools

```python
response = client.chat.completions.create(
    model="nvidia/nemotron-3-nano-30b-a3b",
    messages=[{"role": "user", "content": "What's the weather in SF?"}],
    tools=tools,
    tool_choice="auto"
)

# Check for function call
if response.choices[0].message.tool_calls:
    tool_call = response.choices[0].message.tool_calls[0]
    print(tool_call.function.name)
    print(tool_call.function.arguments)
```

## Error Handling

### Error Response

```json
{
  "error": {
    "message": "Invalid request",
    "type": "invalid_request_error",
    "param": "temperature",
    "code": 400
  }
}
```

### Common Errors

**400 Bad Request**
- Invalid parameters
- Malformed JSON

**401 Unauthorized**
- Missing/invalid API key

**404 Not Found**
- Model not found
- Endpoint not found

**429 Too Many Requests**
- Rate limit exceeded

**500 Internal Server Error**
- Server error

**503 Service Unavailable**
- Model loading
- Overloaded

### Python Error Handling

```python
from openai import OpenAI, APIError, RateLimitError

try:
    response = client.chat.completions.create(
        model="nvidia/nemotron-3-nano-30b-a3b",
        messages=[{"role": "user", "content": "Hello"}]
    )
except RateLimitError:
    print("Rate limit exceeded")
except APIError as e:
    print(f"API error: {e}")
```

## Rate Limits

### NVIDIA Build API

- Rate limits vary by API key tier
- Check response headers:
  - `X-RateLimit-Limit`
  - `X-RateLimit-Remaining`
  - `X-RateLimit-Reset`

### Self-Hosted NIM

- No rate limits
- Limited by hardware capacity
- Configure batch size for throughput

## Best Practices

1. **Use appropriate temperature**: 0.7 for creativity, 0.2 for factual
2. **Set max_tokens**: Prevent runaway generation
3. **Handle streaming**: For long responses
4. **Implement retry**: With exponential backoff
5. **Cache responses**: For repeated queries
6. **Monitor usage**: Track token consumption
7. **Batch requests**: When possible
8. **Use tools**: For structured outputs
