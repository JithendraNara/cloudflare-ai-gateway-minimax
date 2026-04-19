# Caching — Deep Dive

## How AI Gateway Caching Works

AI Gateway caches **identical requests** at the edge (Cloudflare's global network). When a request matches a cached entry, the response is served from cache without hitting the upstream provider.

### What Gets Cached

- **Chat completions** (text responses)
- **Image generation** responses
- Any endpoint that returns a deterministic response for identical input

### What Is NOT Cached

- Streaming responses (SSE chunks)
- Requests without the cache header opt-in
- Requests with `cf-aig-skip-cache: true`

---

## Cache Activation — Header Required

**Caching is opt-in per request.** The gateway does not cache by default. You must send:

```
cf-aig-cache-ttl: <seconds>
```

Without this header, every request goes directly to the provider.

### Minimum and Maximum TTL

| Limit | Value |
|-------|-------|
| Minimum TTL | 60 seconds |
| Maximum TTL | 1 month (2,592,000 seconds) |

---

## Cache Headers Reference

| Header | Values | Effect |
|--------|--------|--------|
| `cf-aig-cache-ttl` | Integer (seconds) | Enables cache with this TTL |
| `cf-aig-skip-cache` | `true` / `false` | Bypass the cache for this request |
| `cf-aig-cache-key` | String | Override the default cache key |

### Response Headers

| Header | Value | Meaning |
|--------|-------|---------|
| `cf-aig-cache-status` | `HIT` | Served from cache |
| `cf-aig-cache-status` | `MISS` | Fetched from provider |
| `cf-cache-status` | `DYNAMIC` | Response was not cached (normal) |

---

## Cache Key Behavior

The **cache key** is computed from:
1. Full request body (including `messages`, `model`, `max_tokens`, `temperature`, etc.)
2. Provider + model
3. Any custom `cf-aig-cache-key` header value

### Important: What Affects Cache Matching

| Parameter | Part of Cache Key? | Notes |
|-----------|-------------------|-------|
| `messages` content | ✅ Yes | Primary cache key |
| `max_tokens` | ✅ Yes | Different max = different cache entry |
| `temperature` | ✅ Yes | Different temp = different entry |
| `model` | ✅ Yes | |
| `cf-aig-cache-key` | ✅ Yes (overrides) | If set, replaces default keying |

This means:
- `max_tokens=20` and `max_tokens=100` for the same prompt → **different cache entries**
- Adding a system prompt → **different cache entry**
- Using `cf-aig-cache-key: "broadcast"` on different prompts → **different cache entries**

---

## Live Test Results

### Single Request Caching

```
Test: "Repeat this exact phrase back: test-phrase-abc123"
First request:  1.45s (MISS)
Second request: 0.17s (HIT)
Speedup: ~8.5x
```

### Multi-Turn Conversation Caching

```
Test: Full conversation with 3 messages
First request (4 msg):  4.54s (MISS)
Repeat exact request:    0.17s (HIT)
Speedup: ~27x
```

### Cross-User Cache Sharing

Same request body = same cache entry, regardless of which user sends it.

```
User A: "Who won the IPL 2024 final?"
  → MISS (1.0s) → Provider call

User B: "Who won the IPL 2024 final?" (identical body)
  → HIT (0.28s) → Served from cache
```

---

## Broadcast Pattern (High-Value Use Case)

For notifications, broadcasts, or FAQs where the same content goes to many users:

```python
import requests

AIG_TOKEN = "cfut_..."
ACCOUNT_ID = "..."
GATEWAY = "..."

def send_broadcast(prompt: str, ttl: int = 3600):
    """Send a broadcast message. First request caches, rest are instant."""
    headers = {
        "cf-aig-authorization": f"Bearer {AIG_TOKEN}",
        "cf-aig-cache-ttl": str(ttl),  # 1 hour default
        "Content-Type": "application/json",
    }

    response = requests.post(
        f"https://gateway.ai.cloudflare.com/v1/{ACCOUNT_ID}/{GATEWAY}/custom-minimax/v1/chat/completions",
        headers=headers,
        json={
            "model": "minimax-m2.7",
            "messages": [{"role": "user", "content": prompt}],
            "max_tokens": 500,
        },
    )

    cache_status = response.headers.get("cf-aig-cache-status", "UNKNOWN")
    return response.json(), cache_status

# First user → MISS (provider call)
# Users 2-1000 → HIT (cache, ~0.2s each)
```

---

## Skip Cache Pattern

Force a fresh response even if the request is cached:

```bash
curl -X POST "https://gateway.ai.cloudflare.com/v1/$ACCOUNT_ID/$GATEWAY/custom-minimax/v1/chat/completions" \
  -H "cf-aig-authorization: Bearer $AIG_TOKEN" \
  -H "cf-aig-cache-ttl: 300" \
  -H "cf-aig-skip-cache: true" \
  -H "Content-Type: application/json" \
  -d '{"model": "minimax-m2.7", "messages": [...]}'
```

Use cases:
- Real-time data queries (scores, prices)
- User-specific personalization
- Interactive conversations where context matters

---

## Custom Cache Key Pattern

Override what counts as "identical" using `cf-aig-cache-key`:

```bash
# Same prompt, but different cache keys = different entries
curl -X POST "..." \
  -H "cf-aig-cache-key: user-segment-A" \
  -d '{"model": "minimax-m2.7", "messages": [...]}'

curl -X POST "..." \
  -H "cf-aig-cache-key: user-segment-B" \
  -d '{"model": "minimax-m2.7", "messages": [...]}'
```

Use cases:
- Segment-based responses (free vs premium user)
- A/B testing cached responses
- Tenant isolation in multi-tenant apps

---

## TTL Selection Guide

| Use Case | Recommended TTL | Rationale |
|----------|---------------|-----------|
| Static FAQ | 24 hours (86400) | Content rarely changes |
| Cricket scores | 5 minutes (300) | Scores update frequently |
| IPL match predictions | 1 hour (3600) | Team news changes |
| Conversational context | 60 seconds (60) | Context-sensitive |
| User greetings | 300 seconds | Personalized per user |

---

## Observability

Check `cf-aig-cache-status` in response headers to know if a response was cached:

```python
response = requests.post(url, headers=headers, json=payload)
cache_status = response.headers.get("cf-aig-cache-status")
print(f"Cache: {cache_status}")  # "HIT" or "MISS"
```

Or in curl:

```bash
curl -i -X POST "..." ... | grep cf-aig-cache-status
```
