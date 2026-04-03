# Using Segnog with Zhipu AI (Z.AI)

[Zhipu AI](https://z.ai/) (Z.AI) provides an OpenAI-compatible Chat Completion API,
making it a drop-in provider for Segnog's LLM backend.

## API Endpoints

Z.AI exposes **two separate base URLs** — choose the one that matches your use-case:

| Endpoint | Base URL | Notes |
|---|---|---|
| **General** | `https://api.z.ai/api/paas/v4/` | Standard chat completions, tool use, web search |
| **Coding Plan** | `https://api.z.ai/api/coding/paas/v4/` | Subscription-based coding endpoint (see [Coding Plan](#coding-plan)) |

Both endpoints share the same request/response schema and accept the same API key.

## Available Models

| Model | Tier | Max Output | Notes |
|---|---|---|---|
| `glm-5` | Flagship | 128K | Highest capability, reasoning-enabled |
| `glm-5-turbo` | Flagship | 128K | Faster variant of GLM-5 |
| `glm-4.7` | High | 128K | Strong all-round model |
| `glm-4.7-flash` | Mid | 128K | Fast and economical |
| `glm-4.7-flashx` | Mid | 128K | Extended flash variant |
| `glm-4.6` | Mid | 128K | Solid general-purpose |
| `glm-4.5` | Mid | 96K | Stable workhorse |
| `glm-4.5-air` | Light | 96K | Lightweight variant |
| `glm-4.5-flash` | Light | 96K | Fastest in 4.5 series |

## Quick Start

### 1. Get an API key

Sign up at [z.ai](https://z.ai/) and create a key on the
[API Keys page](https://z.ai/manage-apikey/apikey-list).

### 2. Configure your `.env`

```env
OPENROUTER_API_KEY=your-openrouter-key    # for embeddings
LLM_API_KEY=your-zai-api-key              # Z.AI key
LLM_BASE_URL=https://api.z.ai/api/paas/v4/
LLM_MODEL=glm-5
```

### 3. Start Segnog

```bash
docker compose up -d
```

## Reasoning / Thinking Mode

GLM-5, GLM-5-Turbo, GLM-4.7, and GLM-4.5 series support chain-of-thought
reasoning. When thinking is enabled, the response includes a
`reasoning_content` field on the assistant message — exactly the same field
name that Segnog already captures for metacognition traces.

### How Z.AI controls thinking

Z.AI uses a `thinking` object in the request body:

```json
{
  "model": "glm-5",
  "messages": [...],
  "thinking": {
    "type": "enabled"
  }
}
```

| Parameter | Values | Default |
|---|---|---|
| `thinking.type` | `enabled` / `disabled` | `enabled` |
| `thinking.clear_thinking` | `true` / `false` | `true` |

> **Note:** GLM-5 and GLM-5-Turbo think compulsorily when `type` is `enabled`.
> GLM-4.6 and GLM-4.5 decide automatically whether to think.

### Compatibility with Segnog

Segnog's `llm_call()` currently passes `reasoning_split` + `reasoning_effort`
via `extra_body` (the MiniMax convention). Z.AI uses a different schema
(`thinking.type`). However, in practice:

- **GLM-5 always reasons by default** — no extra parameters needed.
- Segnog already captures `reasoning_content` from the response (line 138 of
  `client.py`), so reasoning traces flow into the metacognition buffer
  automatically.
- The `<think>...</think>` regex fallback also catches inline reasoning blocks
  that some GLM-4.5V models may emit.

**No code changes are required** for basic reasoning support with GLM-5.

## Coding Plan

Z.AI offers a [Coding Plan](https://docs.z.ai/devpack/overview) subscription
that gives higher rate limits and quota for coding-oriented workloads.

### Key differences

| Feature | Standard API | Coding Plan |
|---|---|---|
| Base URL | `https://api.z.ai/api/paas/v4/` | `https://api.z.ai/api/coding/paas/v4/` |
| Billing | Pay-per-token | Monthly subscription (from $10/mo) |
| Models | All models | GLM-5.1, GLM-5, GLM-5-Turbo, GLM-4.7, GLM-4.6, GLM-4.5, GLM-4.5-Air |
| Rate limits | Standard | Higher limits, 5-hour rolling quota |
| Extras | — | Web Search MCP, Web Reader MCP, Vision |

### Using the Coding Plan endpoint with Segnog

Simply change `LLM_BASE_URL` in your `.env`:

```env
LLM_BASE_URL=https://api.z.ai/api/coding/paas/v4/
LLM_MODEL=glm-5
```

Everything else stays the same — same API key, same request format.

## Tool Use / Function Calling

Z.AI supports OpenAI-compatible function calling with up to 128 tools.
Segnog's existing tool-call handling works without modification.

## Web Search (built-in)

Z.AI has a built-in `web_search` tool type that can be passed in the `tools`
array. When used, the response includes a `web_search` array with search
results (title, content, link, publish date). This is a Z.AI-specific
extension not currently used by Segnog but could be useful for future
enrichment pipelines.

## Critical Notes & Known Quirks

### 1. Embeddings Model Constraint
While the Chinese domestic Zhipu API (`open.bigmodel.cn`) offers `embedding-3` and `embedding-2`, **the global Z.AI API does not currently expose an embedding endpoint.**

Because Segnog requires embeddings to power vector search on the graph, **you cannot use your Z.AI API key for embeddings**. You MUST configure a separate provider (like OpenAI or OpenRouter) for embeddings in your configuration:

```env
EMBEDDING_API_KEY=sk-your-openai-api-key-here
EMBEDDING_BASE_URL=https://api.openai.com/v1
EMBEDDING_MODEL=text-embedding-3-small
```

### 2. Knowledge Extraction (DSPy) Patch
Segnog uses DSPy internally in a background worker to extract structured knowledge. By default, DSPy forces `response_format: {"type": "json_object"}`. Z.AI strictly rejects this format flag and throws an `Invalid API parameter` error. 

To resolve this, Segnog's `DirectJSONAdapter` (`src/memory_service/intelligence/llm/dspy_adapter.py`) has been explicitly patched to bypass `json_object` whenever the model family `glm` is detected. GLM handles the JSON production purely from the system prompt.

## Switching to a Different Provider

The env vars are provider-neutral. To switch away from Z.AI:

```env
# OpenAI
LLM_API_KEY=sk-...
LLM_BASE_URL=https://api.openai.com/v1
LLM_MODEL=gpt-4o-mini

# MiniMax (original default)
LLM_API_KEY=...
LLM_BASE_URL=https://api.minimax.io/v1
LLM_MODEL=MiniMax-M2.7

# DeepSeek
LLM_API_KEY=...
LLM_BASE_URL=https://api.deepseek.com/v1
LLM_MODEL=deepseek-chat
```

## Reference

- [Z.AI API Introduction](https://docs.z.ai/api-reference/introduction)
- [Chat Completion API](https://docs.z.ai/api-reference/llm/chat-completion)
- [Coding Plan Overview](https://docs.z.ai/devpack/overview)
- [Error Codes](https://docs.z.ai/api-reference/api-code)
- [Rate Limits](https://z.ai/manage-apikey/rate-limits)
