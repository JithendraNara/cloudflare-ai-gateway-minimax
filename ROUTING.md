# Routing — Unified API vs Provider-Native

## Two Ways to Call Through AI Gateway

### 1. Unified API (`/compat/`)

OpenAI-compatible endpoint. Works with any OpenAI SDK or client.

```
https://gateway.ai.cloudflare.com/v1/{account}/{gateway}/compat/chat/completions
```

**Model format:** `custom-{provider-slug}/{model-name}`

```bash
curl -X POST "https://gateway.ai.cloudflare.com/v1/$ACCOUNT_ID/$GATEWAY/compat/chat/completions" \
  -H "cf-aig-authorization: Bearer $AIG_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "custom-minimax/MiniMax-M2.7",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

**Pros:**
- Drop-in replacement for OpenAI
- Works with existing SDKs
- Easy provider switching

**Cons:**
- Assumes OpenAI-compatible response format
- Some MiniMax-specific params may not translate

### 2. Provider-Native (`/custom-{slug}/`)

Direct access to the provider's native API format.

```
https://gateway.ai.cloudflare.com/v1/{account}/{gateway}/custom-{provider-slug}/{provider-path}
```

```bash
curl -X POST "https://gateway.ai.cloudflare.com/v1/$ACCOUNT_ID/$GATEWAY/custom-minimax/v1/chat/completions" \
  -H "cf-aig-authorization: Bearer $AIG_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "MiniMax-M2.7",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 50
  }'
```

**Pros:**
- Full access to provider-specific features
- Correct parameter names (MiniMax uses `texts` not `input` for embeddings)
- Better for non-OpenAI providers

**Cons:**
- Provider-specific client/SDK needed
- No automatic format translation

---

## Which to Use?

| Scenario | Recommended |
|----------|------------|
| General chat with MiniMax-M2.7 | `/custom-minimax/anthropic/v1/messages` |
| OpenAI SDK compatibility | `/custom-minimax/v1/chat/completions` or `/compat/` |
| MiniMax-specific params (T2A, image gen) | `/custom-minimax/` |
| Streaming responses | `/custom-minimax/` |
| Multi-modal (image, audio) | `/custom-minimax/` |
| Simple chat with standard params | `/compat/` |

---

## Dynamic Routing

AI Gateway supports **Dynamic Routing** — visual or JSON-based routing logic.

### What You Can Do

- **Conditional routing:** Route based on user plan, request metadata, headers
- **A/B testing:** Split traffic between providers/models
- **Fallback:** If primary fails, automatically try backup
- **Rate/Budget limits:** Per-key quotas

### Example: MiniMax Primary → Workers AI Fallback

```json
{
  "route": "support",
  "version": 1,
  "nodes": [
    {
      "id": "start",
      "type": "start"
    },
    {
      "id": "primary",
      "type": "model",
      "provider": "custom-minimax",
      "model": "MiniMax-M2.7"
    },
    {
      "id": "fallback",
      "type": "model",
      "provider": "workers-ai",
      "model": "@cf/meta/llama-3.3-70b-instruct-fp8-fast"
    },
    {
      "id": "end",
      "type": "end"
    }
  ],
  "edges": [
    {"from": "start", "to": "primary"},
    {"from": "primary", "to": "end", "condition": "success"},
    {"from": "primary", "to": "fallback", "condition": "error"},
    {"from": "fallback", "to": "end"}
  ]
}
```

> **Note:** Workers AI fallback would incur additional costs per Cloudflare's pricing.

### Using Dynamic Routes

```bash
# Instead of specifying model, use the route name
curl -X POST "https://gateway.ai.cloudflare.com/v1/$ACCOUNT_ID/$GATEWAY/compat/chat/completions" \
  -H "cf-aig-authorization: Bearer $AIG_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "dynamic/support",
    "messages": [{"role": "user", "content": "Help me!"}]
  }'
```

---

## Authentication Headers

| Token Type | Header | Use For |
|-----------|--------|---------|
| `cfut_...` (Runtime) | `cf-aig-authorization: Bearer` | AI Gateway proxy requests |
| `cfat_...` (Management) | `Authorization: Bearer` | Cloudflare REST API (gateways, logs) |

> ⚠️ Never use `Authorization: Bearer` on AI Gateway proxy endpoints — only `cf-aig-authorization`.

---

## Rate Limiting

Configure per-key rate limits via the dashboard or API:

```bash
# Set a rate limit via API
curl -X PUT "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/gateways/$GATEWAY/rate-limit" \
  -H "Authorization: Bearer $CFAT_TOKEN" \
  -d '{
    "requests": 60,
    "period": 60
  }'
```

This limits each API key to 60 requests per minute.
