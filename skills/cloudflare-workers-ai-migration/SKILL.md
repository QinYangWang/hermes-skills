---
name: cloudflare-workers-ai-migration
description: "Migrate a Cloudflare Worker from an external LLM API (OpenAI, Kimi, etc.) to Cloudflare Workers AI binding. Covers wrangler.toml changes, code refactoring, and response format differences."
---

# Migrate External LLM API to Cloudflare Workers AI

## When to use

- External LLM API is blocked by WAF, rate-limited, or unavailable from Cloudflare Workers
- You want to reduce cost and dependency on third-party API keys
- Existing Worker uses `fetch()` to call `/v1/chat/completions` and you want to switch to `env.AI.run()`

## Migration Steps

### 1. Update `wrangler.toml`

Remove external API configuration:
```toml
# DELETE these
[[secrets_store_secrets]]
binding = "KIMI_API_KEY_STORE"
store_id = "..."
secret_name = "KIMI_API_KEY"

[vars]
KIMI_BASE_URL = "https://api.kimi.com/coding/v1"
KIMI_MODEL = "kimi-for-coding"
```

Add the AI binding:
```toml
[ai]
binding = "AI"
```

> **Pitfall**: use `[ai]` (single object), **not** `[[ai]]` (array). Wrangler rejects `[[ai]]`.

### 2. Update Worker Env interface

```typescript
export interface Env {
  ASSETS: Fetcher;
  DB: D1Database;
  AI: any;               // add this
  JWT_SECRET_STORE: SecretsStoreSecret;
}
```

Remove old bindings like `KIMI_API_KEY_STORE`, `KIMI_BASE_URL`, `KIMI_MODEL` from the interface.

### 3. Replace the fetch call with `env.AI.run()`

**Before:**
```typescript
const res = await fetch(`${baseUrl}/chat/completions`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${apiKey}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    model,
    messages: [
      { role: 'system', content: systemPrompt },
      { role: 'user', content: userPrompt },
    ],
    temperature: 0.6,
  }),
});
const data = await res.json();
const html = data.choices?.[0]?.message?.content || '';
```

**After:**
```typescript
const aiRes = await env.AI.run('@cf/meta/llama-3.1-8b-instruct-fp8', {
  messages: [
    { role: 'system', content: systemPrompt },
    { role: 'user', content: userPrompt },
  ],
});
const html = aiRes?.response || '';
```

### 4. Build and deploy

```bash
npm run build
npx wrangler deploy
```

Wrangler will warn if remote config still has old vars/secrets; deploying will override them.

## Response Format Differences

| External API | Workers AI |
|--------------|------------|
| `choices[0].message.content` | `response` (string) |
| `choices[0].message.tool_calls` | `tool_calls` (array) |
| Streaming via `stream: true` | Not supported on all models; use non-streaming |
| `temperature`, `top_p`, `max_tokens` | Most models ignore these; check docs |

## Common Models

- `@cf/meta/llama-3.1-8b-instruct-fp8` — fast, good for general text generation
- `@cf/meta/llama-3.3-70b-instruct-fp8-fast` — larger, better quality
- `@cf/deepseek-ai/deepseek-r1-distill-qwen-32b` — reasoning / coding

## Pitfalls

1. **TOML syntax**: `[ai]` is a table, not an array of tables. `[[ai]]` fails validation.
2. **Model name format**: Workers AI uses `@vendor/model` names, not OpenAI-style IDs.
3. **No API key needed**: Workers AI runs on Cloudflare's edge network; no external auth required.
4. **Prompt length limits**: Workers AI models have smaller context windows than GPT-4; truncate long inputs (e.g. `rawHtml.slice(0, 15000)`).
5. **Compatibility date**: Make sure `compatibility_date` in `wrangler.toml` is recent enough (2024+).
