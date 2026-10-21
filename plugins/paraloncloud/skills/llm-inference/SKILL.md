---
name: llm-inference
description: Call open models on ParalonCloud's OpenAI-compatible API from code: chat completions, embeddings, classification, the Responses and Anthropic Messages formats, batch jobs and images, with the OpenAI or Anthropic SDKs, curl, or Claude Code pointed at an open model. Use when the user wants to call or integrate a ParalonCloud model, switch an OpenAI-style app to an open model, or set up keys, models, streaming or batch on ParalonCloud.
---

# Call open models on ParalonCloud

ParalonCloud serves open models on independent GPU nodes behind an **OpenAI-compatible API**. Existing OpenAI code works by changing the base URL and the key.

- Base URL: `https://paraloncloud.com/v1`
- Auth: `Authorization: Bearer prlc_...` (create a key at https://paraloncloud.com/console, API keys)
- The key needs the **Inference** permission (on by default). `403 insufficient_scope` means it was turned off for that key.

## Pick the model at runtime

The catalog changes as nodes come and go. **Never hard-code a model list from memory**: read it.

```bash
curl https://paraloncloud.com/v1/models -H "Authorization: Bearer $PARALON_API_KEY"
```

Each entry has `id` (the `model` value to send), `type` (`chat`, `embedding`, `classification`, ...) and `max_context`. Choose by kind first, then size: a smaller chat model is faster and cheaper, a larger one better at hard reasoning and long context. Keep prompt plus reply under `max_context`.

## Endpoints

| Endpoint | For |
| --- | --- |
| `POST /v1/chat/completions` | Chat, streaming (`"stream": true`), tools, JSON output |
| `POST /v1/responses` | The OpenAI Responses format |
| `POST /v1/messages` | The Anthropic Messages format (Anthropic SDK, Claude Code) |
| `POST /v1/embeddings` | Vectors for search and RAG |
| `POST /v1/classify` | Zero-shot text classification against labels you give |
| `POST /v1/images/generations` | Images |
| `POST /v1/files`, `POST /v1/batches` | Batch jobs: a JSONL file of requests, results within 24 hours at half the token price |
| `GET /v1/models` | The live catalog |

## Code

Python, OpenAI SDK:

```python
import os
from openai import OpenAI

client = OpenAI(api_key=os.environ["PARALON_API_KEY"], base_url="https://paraloncloud.com/v1")
chat_models = [m.id for m in client.models.list().data if getattr(m, "type", "chat") == "chat"]
reply = client.chat.completions.create(
    model=chat_models[0],  # or pick by size and max_context
    messages=[{"role": "user", "content": "Summarize this in one line: ..."}],
)
print(reply.choices[0].message.content)
```

Node.js, OpenAI SDK:

```js
import OpenAI from "openai";
const client = new OpenAI({ apiKey: process.env.PARALON_API_KEY, baseURL: "https://paraloncloud.com/v1" });
const r = await client.chat.completions.create({ model: "<id from /v1/models>", messages: [{ role: "user", content: "Hello" }] });
console.log(r.choices[0].message.content);
```

curl:

```bash
curl https://paraloncloud.com/v1/chat/completions \
  -H "Authorization: Bearer $PARALON_API_KEY" -H "Content-Type: application/json" \
  -d '{"model": "<id from /v1/models>", "messages": [{"role": "user", "content": "Hello"}]}'
```

Embeddings take `{"model": "<embedding id>", "input": ["text one", "text two"]}`. Frameworks that accept an OpenAI base URL (LangChain, LlamaIndex, Vercel AI SDK, LiteLLM, n8n) work the same way.

## Claude Code on an open model

Claude Code speaks the Anthropic Messages format, which ParalonCloud answers. Set **all** the model variables to one chat model id from `/v1/models` (Claude Code sends side requests under other model names):

```bash
export ANTHROPIC_BASE_URL=https://paraloncloud.com
export ANTHROPIC_AUTH_TOKEN=$PARALON_API_KEY
export ANTHROPIC_MODEL=<chat model id>
export ANTHROPIC_SMALL_FAST_MODEL=<chat model id>
export ANTHROPIC_DEFAULT_HAIKU_MODEL=<chat model id>
export ANTHROPIC_DEFAULT_SONNET_MODEL=<chat model id>
export ANTHROPIC_DEFAULT_OPUS_MODEL=<chat model id>
claude
```

## Billing and errors

- A free key has a one-time token trial, unlocked in the Console; after that a premium key pays per token from prepaid credits. Prices: https://paraloncloud.com/pricing
- `401` wrong or revoked key; `402` trial used up or no credits (top up at https://paraloncloud.com/add-credits); `403 insufficient_scope` key without Inference; `404` model id not in `/v1/models` right now; `429` rate limit (retry with backoff).
- Keep keys in environment variables, never in client-side code or commits.

Docs: https://paraloncloud.com/docs/inference-api
