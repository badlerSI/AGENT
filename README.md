# AGENT - Personal GPU-Powered AI API

This repository documents how to use your private AI infrastructure for building agents. Your backend runs on an **NVIDIA RTX 6000 Blackwell (98GB VRAM)** serving **Nemotron Super 49B** - NVIDIA's latest reasoning model with full tool-use support.

## Quick Start

### Your API Endpoints

| Endpoint | URL |
|----------|-----|
| **Chat Completions** | `https://si.tools/agent/chat` |
| **Text Completions** | `https://si.tools/agent/complete` |
| **List Models** | `https://si.tools/agent/models` |
| **Health Check** | `https://si.tools/agent/health` |

### Model Name
```
nemotron-super
```

---

## Usage Examples

### Basic Chat Request (curl)

```bash
curl -X POST https://si.tools/agent/chat \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nemotron-super",
    "messages": [
      {"role": "user", "content": "What is 2+2?"}
    ],
    "max_tokens": 100
  }'
```

### JavaScript/Bun

```javascript
const response = await fetch("https://si.tools/agent/chat", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    model: "nemotron-super",
    messages: [{ role: "user", content: "Hello!" }],
    max_tokens: 500
  })
});

const data = await response.json();
console.log(data.choices[0].message.content);
```

### Python

```python
import requests

response = requests.post("https://si.tools/agent/chat", json={
    "model": "nemotron-super",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 500
})

print(response.json()["choices"][0]["message"]["content"])
```

### Using OpenAI SDK (Python)

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://si.tools/agent",
    api_key="not-needed"  # No auth required
)

response = client.chat.completions.create(
    model="nemotron-super",
    messages=[{"role": "user", "content": "Hello!"}],
    max_tokens=500
)

print(response.choices[0].message.content)
```

### Using OpenAI SDK (Node.js/Bun)

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://si.tools/agent",
  apiKey: "not-needed"
});

const response = await client.chat.completions.create({
  model: "nemotron-super",
  messages: [{ role: "user", content: "Hello!" }],
  max_tokens: 500
});

console.log(response.choices[0].message.content);
```

---

## Tool Use / Function Calling

Nemotron Super supports tool use with the Hermes format. Here's how to use it:

```javascript
const response = await fetch("https://si.tools/agent/chat", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    model: "nemotron-super",
    messages: [
      { role: "user", content: "What's the weather in San Francisco?" }
    ],
    tools: [
      {
        type: "function",
        function: {
          name: "get_weather",
          description: "Get the current weather for a location",
          parameters: {
            type: "object",
            properties: {
              location: {
                type: "string",
                description: "City name"
              }
            },
            required: ["location"]
          }
        }
      }
    ],
    tool_choice: "auto",
    max_tokens: 500
  })
});

const data = await response.json();
const message = data.choices[0].message;

if (message.tool_calls) {
  console.log("Tool call:", message.tool_calls[0].function);
} else {
  console.log("Response:", message.content);
}
```

---

## Building a Simple Agent (Bun)

Here's a minimal agent loop that can call tools:

```typescript
// agent.ts - Run with: bun agent.ts "Your task here"

const TOOLS = [
  {
    type: "function",
    function: {
      name: "shell",
      description: "Run a shell command and return output",
      parameters: {
        type: "object",
        properties: { cmd: { type: "string", description: "Command to run" } },
        required: ["cmd"]
      }
    }
  }
];

async function chat(messages: any[], useTools = true) {
  const res = await fetch("https://si.tools/agent/chat", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "nemotron-super",
      messages,
      tools: useTools ? TOOLS : undefined,
      max_tokens: 2000
    })
  });
  return (await res.json()).choices[0].message;
}

async function runTool(name: string, args: any): Promise<string> {
  if (name === "shell") {
    const proc = Bun.spawn(["bash", "-c", args.cmd]);
    return await new Response(proc.stdout).text();
  }
  return `Unknown tool: ${name}`;
}

async function agent(task: string) {
  const messages = [
    { role: "system", content: "You are a helpful agent. Use tools when needed." },
    { role: "user", content: task }
  ];

  while (true) {
    const response = await chat(messages);
    messages.push(response);

    if (response.tool_calls?.length) {
      for (const tc of response.tool_calls) {
        const args = JSON.parse(tc.function.arguments);
        console.log(`[Tool: ${tc.function.name}]`, args);
        const result = await runTool(tc.function.name, args);
        console.log(`[Result]`, result.slice(0, 200));
        messages.push({ role: "tool", tool_call_id: tc.id, content: result });
      }
    } else {
      console.log("\nAgent:", response.content);
      break;
    }
  }
}

agent(Bun.argv[2] || "List the files in the current directory");
```

Run it:
```bash
bun agent.ts "What files are in my home directory?"
```

---

## API Reference

### POST /agent/chat

OpenAI-compatible chat completions endpoint.

**Request Body:**
```json
{
  "model": "nemotron-super",
  "messages": [
    {"role": "system", "content": "You are helpful."},
    {"role": "user", "content": "Hello!"}
  ],
  "max_tokens": 500,
  "temperature": 0.7,
  "tools": [],
  "tool_choice": "auto"
}
```

**Response:**
```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "model": "nemotron-super",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! How can I help you?"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 10,
    "completion_tokens": 8,
    "total_tokens": 18
  }
}
```

### POST /agent/complete

Text completions endpoint (non-chat).

### GET /agent/models

List available models.

### GET /agent/health

Health check endpoint.

---

## Hardware

Your API is powered by:

- **GPU**: NVIDIA RTX PRO 6000 Blackwell (98GB VRAM)
- **Model**: Nemotron Super 49B (NVFP4 quantized)
- **Context Length**: 16,384 tokens
- **Concurrent Requests**: 4 max
- **Features**: Tool use enabled (Hermes parser)

---

## Architecture

```
Your MacBook
     |
     v (HTTPS)
https://si.tools/agent/*
     |
     v (Cloudflare Tunnel)
Mariposa03 (your server)
     |
     v (Caddy reverse proxy)
vLLM on port 9100
     |
     v
Nemotron Super 49B on RTX 6000 Blackwell
```

---

## Troubleshooting

### "Method Not Allowed"
You're making a GET request. Use POST for `/agent/chat` and `/agent/complete`.

### Slow responses
The model is large (49B parameters). First request may take a few seconds to warm up. Subsequent requests are faster.

### Connection refused
Check if the server is running: `curl https://si.tools/agent/health`

---

## For Claude Code on MacBook

When using Claude Code on your MacBook and it needs to make LLM calls, use:

```javascript
// Instead of OpenAI:
const response = await fetch("https://si.tools/agent/chat", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    model: "nemotron-super",
    messages: messages,
    max_tokens: 2000
  })
});
```

This routes through your own GPU infrastructure instead of paying for API calls.

---

## Workshop Notes

For the "Build Your Own Agent" workshop, when they say:
- "Use GLM" or "Use Claude API" → Use `https://si.tools/agent/chat` with model `nemotron-super`
- "Set your API key" → Not needed, leave blank or use any string
- "Set base URL" → `https://si.tools/agent`

Your 98GB Blackwell is more powerful than most cloud APIs and it's free (you own it).
