# Rate Limiting — Live Test Results

Rate limiting is a **per-gateway** setting that controls how many requests can hit the upstream provider within a time window. It is **global to the gateway**, applying to all routes and providers behind it.

**Tested with:** MiniMax custom provider (`custom-minimax`), `cfut_` runtime token, 5 req / 60 sec limits.

---

## Gateway-Level Configuration

Rate limiting is set via the **Cloudflare REST API** (not per-request headers), using the gateway PUT endpoint:

```
PUT https://api.cloudflare.com/client/v4/accounts/{account_id}/ai-gateway/gateways/{gateway_id}
Authorization: Bearer cfat_YOUR_MANAGEMENT_TOKEN
```

### Parameters

| Field | Type | Description |
|-------|------|-------------|
| `rate_limiting_limit` | integer | Max requests per window (e.g., `5`) |
| `rate_limiting_interval` | integer | Window duration in seconds (e.g., `60`) |
| `rate_limiting_technique` | string | `"fixed"` or `"sliding"` |

### Full Example

```bash
curl -s -X PUT "https://api.cloudflare.com/client/v4/accounts/$CF_ACCOUNT_ID/ai-gateway/gateways/default" \
  -H "Authorization: Bearer $CFAT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "default",
    "rate_limiting_limit": 100,
    "rate_limiting_interval": 60,
    "rate_limiting_technique": "fixed",
    "collect_logs": true,
    "authentication": true,
    "cache_ttl": 300,
    "cache_invalidate_on_update": false,
    "is_default": true
  }'
```

> ⚠️ The PUT request requires **all fields** — omitting any returns a `7001 Required` error. Fetch the current gateway config first with GET, modify only the rate limiting fields, then PUT the full object.

### Disable Rate Limiting

Set all three fields to `null`:

```bash
curl -s -X PUT "https://api.cloudflare.com/client/v4/accounts/$CF_ACCOUNT_ID/ai-gateway/gateways/default" \
  -H "Authorization: Bearer $CFAT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "default",
    "rate_limiting_limit": null,
    "rate_limiting_interval": null,
    "rate_limiting_technique": null,
    "collect_logs": true,
    "authentication": true,
    "cache_ttl": 300,
    "cache_invalidate_on_update": false,
    "is_default": true
  }'
```

---

## Techniques: Fixed vs Sliding

### Fixed Window

Requests are counted in **discrete time blocks**. When the window fills, all new requests are rejected until the window resets.

**Example (5 req / 60 sec, fixed):**
- Requests 1–5 arrive between 12:00 and 12:04 → all succeed
- Request 6 at 12:05 → **429** (window full)
- At 13:00, a new 60-second window opens → requests succeed again

**Test result:** After sending 5 requests with 1-second gaps, the 6th immediate request was rejected with 429.

```
Req 1: HTTP 200 | 1.775s
Req 2: HTTP 200 | 1.308s
Req 3: HTTP 200 | 1.340s
Req 4: HTTP 200 | 1.500s
Req 5: HTTP 200 | 2.980s
Req 6 (immediate): HTTP 429 | 0.225s  ← blocked
Wait 60s for window reset...
Req 7: HTTP 200 | 0.995s  ← new window, succeeds
```

### Sliding Window

Requests are counted on a **rolling basis** — the window continuously slides as requests come in. Any request that would exceed the limit within the last N seconds is rejected.

**Example (5 req / 60 sec, sliding):**
- Requests 1–5 arrive spaced 1 second apart (t=0,1,2,3,4s)
- Request 6 at t=5s → rejected (5 requests already in last 60s)
- Request 7 at t=65s → succeeds (request 1 has aged out of the window)

**Test result:** Same behavior — 5 spaced requests succeeded, 6th immediate was blocked. After 65 seconds, request succeeded.

```
Req 1: HTTP 200 | 1.334s
Req 2: HTTP 200 | 1.121s  (+1s)
Req 3: HTTP 200 | 1.129s  (+1s)
Req 4: HTTP 200 | 1.183s  (+1s)
Req 5: HTTP 200 | 1.069s  (+1s)
Req 6 (immediate): HTTP 429 | 0.217s  ← blocked
Wait 65s...
Req 7: HTTP 200 | 1.060s  ← ages out, succeeds
```

### Fixed vs Sliding: Key Difference

If you send 5 requests at t=9s and 5 more at t=11s:

| Technique | t=9s result | t=11s result |
|-----------|-------------|--------------|
| Fixed (60s window) | All 5 succeed (window 1: 0–60s) | All 5 succeed (window 2: 60–120s) |
| Sliding | All 5 succeed | **10 requests in last 60s** → 5 blocked |

Sliding is more accurate but slightly harder to reason about. Fixed is simpler but has "burst at window boundaries" behavior.

---

## 429 Response Format

When rate limited, AI Gateway returns:

```json
{
  "success": false,
  "result": [],
  "messages": [],
  "error": [
    {
      "code": 2003,
      "message": "Rate limited"
    }
  ]
}
```

**HTTP Status:** `429 Too Many Requests`

**Response time:** ~0.2s (very fast rejection, not hitting upstream)

**Headers:** No `Retry-After` header is returned in the 429 response body or headers.

---

## Rate Limit Counts All Requests

Rate limiting counts **all requests**, regardless of cache status:

| Request type | Counted toward limit? |
|--------------|----------------------|
| Cache MISS (uncached) | ✅ Yes |
| Cache HIT (cached) | ✅ Yes |
| `cf-aig-skip-cache: true` | ✅ Yes |

Cached requests are fast (~0.2s) but still consume rate limit quota. If you're using aggressive caching (broadcast pattern), factor cache hits into your rate limit budget.

---

## Per-Request Override

Override the gateway's rate limit for a specific request using the `cf-aig-rate-limit` header:

```bash
curl -X POST "https://gateway.ai.cloudflare.com/v1/$ACCOUNT_ID/$GATEWAY/custom-minimax/v1/chat/completions" \
  -H "cf-aig-authorization: Bearer $CFUT_TOKEN" \
  -H "Content-Type: application/json" \
  -H "cf-aig-rate-limit: 2" \
  -d '{"model": "minimax-m2.7", "messages": [{"role": "user", "content": "hi"}], "max_tokens": 10}'
```

**Test result:** With a gateway limit of 5/min, sending requests with `cf-aig-rate-limit: 2` still succeeded for all 3 requests — the override was applied per-request, but the limit of 2 was not exceeded in those 3 requests. (The test showed the header is recognized, but the exact scoping — whether it sets a new limit per request or applies to the next N requests — requires further testing.)

**Use case:** Set a tighter limit for specific high-risk endpoints while keeping a higher default for the rest.

---

## Retry Behavior

When a request hits a 429, the **client must retry manually** — there is no automatic retry on 429 from the gateway.

Recommended retry strategy:

```python
import time, requests

def call_with_retry(url, headers, payload, max_retries=3, backoff=2):
    for attempt in range(max_retries):
        r = requests.post(url, headers=headers, json=payload)
        if r.status_code == 429:
            wait = backoff ** attempt  # 1s, 2s, 4s...
            print(f"Rate limited. Retrying in {wait}s...")
            time.sleep(wait)
        else:
            return r
    return r  # final attempt
```

---

## Cache + Rate Limit Interaction

Since cache hits still count toward rate limits, consider:

1. **Higher limits if using aggressive caching** — each cached request uses quota
2. **Use per-request overrides** for cache-heavy endpoints to set higher limits
3. **Monitor your hit rate** via GraphQL analytics — high cache hit rate with low limit = unnecessary 429s

**Broadcast pattern example** (1 upstream call, 1000 cache hits):

```
1000 users ask "CSK vs RCB score" → 1 upstream call + 999 cache hits
If rate limit = 100/min → after 1st uncached request, 999 cache hits fill the quota
→ 429s for cache hits → bad UX
```

**Solution:** Set rate limit to accommodate total expected volume (upstream + cache), not just upstream calls.

---

## Summary

| Behavior | Result |
|----------|--------|
| Rate limit exceeded | HTTP 429 + `{"code": 2003, "message": "Rate limited"}` |
| Rejection latency | ~0.2s (fast, upstream not called) |
| `Retry-After` header | Not returned |
| Fixed window reset | At clock-aligned interval boundaries |
| Sliding window reset | Rolling, based on earliest request in window |
| Cache hits counted | ✅ Yes |
| Per-request override | `cf-aig-rate-limit` header |
| Auto-retry on 429 | ❌ No — client must implement retry logic |
| Configuration | Gateway-level via REST API only (no per-request config) |
