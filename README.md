# Cloudflare AI Gateway + MiniMax Custom Provider

> **Use any AI provider behind Cloudflare's AI Gateway** — get caching, rate limiting, observability, and unified billing on top of MiniMax (or any OpenAI-compatible API).

[![Cloudflare](https://img.shields.io/badge/Cloudflare-AI%20Gateway-orange)](https://developers.cloudflare.com/ai-gateway/)
[![MiniMax](https://img.shields.io/badge/MiniMax-API-blue)](https://www.minimaxi.com/developers)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue)](LICENSE)

## What This Repo Covers

This repo documents the full setup and integration of **MiniMax** as a custom provider in **Cloudflare AI Gateway**, including:

- ✅ Adding a custom provider (MiniMax) via the Cloudflare Dashboard
- ✅ Routing: Unified API (`/compat`) vs Provider-native endpoints
- ✅ Caching: Per-request headers, cache key strategies, broadcast optimization
- ✅ Observability: Analytics, logging, cost tracking
- ✅ Live test results with latency and cache behavior data
- ✅ AI agent integration (production usage patterns)

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        Your App                             │
│                   (AI Agent, CLI, App, etc.)                │
└─────────────────────┬───────────────────────────────────────┘
                      │ HTTP/REST
                      ▼
┌─────────────────────────────────────────────────────────────┐
│              Cloudflare AI Gateway                          │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │  Cache   │  │  Rate    │  │  Log &   │  │  Dynamic  │  │
│  │  Layer   │  │  Limit   │  │Analytics │  │  Routing  │  │
│  └──────────┘  └──────────┘  └──────────┘  └───────────┘  │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Custom Provider: MiniMax               │    │
│  │              (via Provider Key — BYOK)             │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTPS
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    MiniMax API                               │
│           (Chat, T2A, Image Gen, Video, Music)              │
└─────────────────────────────────────────────────────────────┘
```

## Quick Start

### 1. Add MiniMax as a Custom Provider

1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com/) → **AI** → **AI Gateway**
2. Select your gateway → **Settings** → **Provider Keys** → **Add Key**
3. Provider: `Custom`, Name: `minimax` (or any slug)
4. Paste your MiniMax API key
5. Save

### 2. Configure the Provider

In the same gateway settings, add the custom provider:

| Field | Value |
|-------|-------|
| **Provider Name** | `minimax` |
| **base_url** | `https://api.minimax.io` |
| **Auth Type** | `API Key` (key stored via Provider Key above) |

> **Important:** keep the MiniMax API key in Cloudflare BYOK. Application requests authenticate to Cloudflare with `cf-aig-authorization`; do not send your MiniMax key from the app.
> The custom provider `base_url` should be the API root. Include `/anthropic/...` or `/v1/...` in the gateway request path.

### 3. Make Your First Request

```bash
# Using cfut_ token (AI Gateway Run permission)
export AIG_TOKEN="cfut_YOUR_RUN_TOKEN"
export ACCOUNT_ID="your_account_id"
export GATEWAY="your_gateway_id"

# Anthropic-compatible Messages endpoint (recommended for MiniMax-M2.7)
curl -X POST "https://gateway.ai.cloudflare.com/v1/$ACCOUNT_ID/$GATEWAY/custom-minimax/anthropic/v1/messages" \
  -H "cf-aig-authorization: Bearer $AIG_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "MiniMax-M2.7",
    "system": "You are concise.",
    "messages": [{
      "role": "user",
      "content": [{"type": "text", "text": "Return exactly: gateway-ok"}]
    }],
    "max_tokens": 50
  }'
```

### 4. Use the Unified API (OpenAI-compatible)

```bash
curl -X POST "https://gateway.ai.cloudflare.com/v1/$ACCOUNT_ID/$GATEWAY/compat/chat/completions" \
  -H "cf-aig-authorization: Bearer $AIG_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "custom-minimax/MiniMax-M2.7",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

> MiniMax's OpenAI-compatible API may include `<think>` / reasoning content in `message.content`. For cleaner separation of thinking and final text, prefer the Anthropic-compatible Messages route above.

## Key Endpoints Tested

| Endpoint | Method | Model | Status |
|----------|--------|-------|--------|
| `/anthropic/v1/messages` | POST | `MiniMax-M2.7` | ✅ Working — recommended chat route |
| `/v1/chat/completions` | POST | `MiniMax-M2.7` | ✅ Working — OpenAI-compatible, may include reasoning in content |
| `/v1/t2a_v2` | POST | `speech-2.8-hd` | ✅ Working |
| `/v1/image_generation` | POST | `image-01` | ✅ Working |
| `/v1/embeddings` | POST | `embo-01` | ⚠️ Requires paid plan |
| `/v1/video_generation` | POST | — | Not tested |
| `/v1/music_generation` | POST | — | Not tested |

## Caching — Live Test Results

Caching is **per-request opt-in** via HTTP headers. No gateway-level toggle needed.

### Test Results

| Scenario | First Request | Cached Request | Speedup |
|----------|--------------|-----------------|---------|
| Identical prompt | 1.45s (MISS) | 0.17s (HIT) | **~8.5x faster** |
| Multi-turn conversation | 4.5s (MISS) | 0.17s (HIT) | **~26x faster** |

### Required Headers

```
cf-aig-cache-ttl: 300       # Enable cache, TTL in seconds (min: 60, max: 1 month)
cf-aig-skip-cache: true    # Bypass cache for this request
cf-aig-cache-key: my-key    # Override cache key
```

### Broadcast Pattern (e.g., IPL Score Alert)

```python
# User A asks "CSK vs RCB score" → cached
# User B asks "CSK vs RCB score" → served from cache instantly
# 1000 users ask same question → 1 MiniMax call, 999 cache hits

headers = {
    "cf-aig-authorization": f"Bearer {AIG_TOKEN}",
    "cf-aig-cache-ttl": "3600",  # 1 hour cache
}
```

## What's Covered in Detail

| Document | What It Covers |
|----------|---------------|
| [PROVIDER_SETUP.md](PROVIDER_SETUP.md) | Step-by-step custom provider configuration |
| [ROUTING.md](ROUTING.md) | Unified API vs Provider-native, dynamic routing |
| [CACHING.md](CACHING.md) | Deep dive: headers, TTL, keys, broadcast patterns |
| [ENDPOINTS.md](ENDPOINTS.md) | All tested MiniMax endpoints with example requests |
| [INTEGRATION.md](INTEGRATION.md) | AI agent integration patterns |
| [TESTING.md](TESTING.md) | Raw test data, methodology, latency benchmarks |
| [TTS.md](TTS.md) | Text-to-Speech via AI Gateway — HTTP works, WebSocket streaming not supported |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System diagrams and data flow |
| [OBSERVABILITY.md](OBSERVABILITY.md) | GraphQL analytics API — usage stats, costs, cache, errors |
| [RATE_LIMITING.md](RATE_LIMITING.md) | Live test results — fixed vs sliding window, 429 behavior, per-request overrides |

## Two Auth Systems (Critical)

| System | Auth Header | Token Type |
|--------|-------------|------------|
| AI Gateway proxy | `cf-aig-authorization: Bearer` | `cfut_...` (Runtime) |
| Cloudflare REST API | `Authorization: Bearer` | `cfat_...` (Management) |
| MiniMax upstream | Stored in Cloudflare BYOK | MiniMax API key |

> ⚠️ Runtime requests to AI Gateway use `cf-aig-authorization`. `Authorization` is reserved for Cloudflare management API calls, and the MiniMax provider key should be stored in Cloudflare BYOK.

## Environment Variables

See [`.env.example`](.env.example) for all environment variables used in this setup.

```bash
# Copy and fill in your values
cp .env.example .env
```

## References

This guide is based on and expands the official Cloudflare AI Gateway documentation:

- [Cloudflare AI Gateway Docs](https://developers.cloudflare.com/ai-gateway/) — Official documentation
- [Cloudflare AI Gateway API](https://developers.cloudflare.com/api/resources/ai-gateway/) — REST API reference
- [Custom Provider Setup](https://developers.cloudflare.com/ai-gateway/providers/) — Adding third-party providers
- [Caching in AI Gateway](https://developers.cloudflare.com/ai-gateway/caching/) — Official caching docs
- [MiniMax API Docs](https://www.minimaxi.com/developers) — MiniMax API reference

## License

MIT — free to use, modify, and share.
