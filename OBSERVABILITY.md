# Observability — Analytics via GraphQL API

> **Prerequisite:** You need a `cfat_...` (Management) token, not a `cfut_...` (Runtime) token.

Cloudflare AI Gateway exposes a **GraphQL API** for querying usage analytics — request counts, token usage, costs, cache performance, and error rates. All data is grouped by time window, model, provider, gateway, and status code.

**GraphQL endpoint:**
```
https://api.cloudflare.com/client/v4/graphql
```

**Auth header:**
```
Authorization: Bearer cfat_YOUR_MANAGEMENT_TOKEN
```

---

## What You Can Track

| Metric | Field | Description |
|--------|-------|-------------|
| Request count | `count` | Number of requests in this group |
| Cost | `sum.cost` | Estimated cost (USD) |
| Cache hits | `sum.cachedRequests` | Requests served from cache |
| Cache misses | `sum.erroredRequests` | Requests that errored |
| Tokens in | `sum.uncachedTokensIn` | Input tokens (uncached) |
| Tokens out | `sum.uncachedTokensOut` | Output tokens (uncached) |

**Grouping dimensions** (`dimensions`):
- `datetimeMinute` — minute-level timestamp
- `datetimeHour` — hour-level timestamp
- `model` — model name (e.g., `MiniMax-M2.7`)
- `provider` — provider name (e.g., `custom-minimax`)
- `gateway` — gateway ID (e.g., `default`)
- `statusCode` — HTTP status code

---

## Example Queries

### All Requests (Last 7 Days, Top 20 Groups)

```graphql
query {
  viewer {
    accounts(filter: { accountTag: "YOUR_ACCOUNT_ID" }) {
      requests: aiGatewayRequestsAdaptiveGroups(
        limit: 20,
        filter: {
          datetimeHour_geq: "2026-04-12T00:00:00Z",
          datetimeHour_leq: "2026-04-19T23:59:59Z"
        }
      ) {
        count
        sum {
          cost
          cachedRequests
          erroredRequests
          uncachedTokensIn
          uncachedTokensOut
        }
        dimensions {
          model
          provider
          gateway
          statusCode
        }
      }
    }
  }
}
```

### Cost Breakdown by Provider and Model

```graphql
query {
  viewer {
    accounts(filter: { accountTag: "YOUR_ACCOUNT_ID" }) {
      costs: aiGatewayRequestsAdaptiveGroups(
        limit: 50,
        filter: { datetimeHour_geq: "2026-04-01T00:00:00Z" }
      ) {
        sum { cost }
        dimensions { model provider }
      }
    }
  }
}
```

### Cache Hit Rate

```graphql
query {
  viewer {
    accounts(filter: { accountTag: "YOUR_ACCOUNT_ID" }) {
      requests: aiGatewayRequestsAdaptiveGroups(
        limit: 100,
        filter: { datetimeHour_geq: "2026-04-01T00:00:00Z" }
      ) {
        sum { cachedRequests }
        dimensions { model provider }
      }
    }
  }
}
```

### Error Tracking

```graphql
query {
  viewer {
    accounts(filter: { accountTag: "YOUR_ACCOUNT_ID" }) {
      errors: aiGatewayRequestsAdaptiveGroups(
        limit: 20,
        filter: {
          datetimeHour_geq: "2026-04-01T00:00:00Z",
          statusCode_geq: 400
        }
      ) {
        count
        sum { erroredRequests }
        dimensions { statusCode model provider }
      }
    }
  }
}
```

---

## Curl Example

```bash
curl -s -X POST "https://api.cloudflare.com/client/v4/graphql" \
  -H "Authorization: Bearer cfat_YOUR_MANAGEMENT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { viewer { accounts(filter: { accountTag: \"YOUR_ACCOUNT_ID\" }) { requests: aiGatewayRequestsAdaptiveGroups(limit: 20, filter: { datetimeHour_geq: \"2026-04-12T00:00:00Z\" }) { count sum { cost cachedRequests uncachedTokensIn uncachedTokensOut } dimensions { model provider gateway statusCode } } } } }"
  }' | jq .
```

---

## Known Limitations

### Cost Tracking

- **Workers AI requests** show real costs in USD (e.g., `$0.00105605` for llama-2-7b-chat)
- **Custom provider (MiniMax via BYOK)** requests show `cost: 0` — Cloudflare does not track third-party BYOK costs. You pay MiniMax directly.

### Cache Analytics

- `aiGatewayCacheAdaptiveGroups` returns empty results even when cache is confirmed working via headers
- Cache hit/miss data IS available via `aiGatewayRequestsAdaptiveGroups` (`cachedRequests` field)
- Likely that cache analytics require a separate analytics configuration or are not yet populated for custom providers

### Error Analytics

- `aiGatewayErrorsAdaptiveGroups` returns empty — the correct GraphQL type name may differ
- Use `aiGatewayRequestsAdaptiveGroups` with `statusCode_geq: 400` filter to track errors instead

### Token Counts

- Some endpoint types (embeddings, TTS) may report `uncachedTokensIn: 0, uncachedTokensOut: 0` even when requests succeed
- Token tracking may not be implemented for all provider endpoint types

### Multiple Gateways

- If you renamed your gateway at any point, historical data appears under the old name (`my-gateway` in our case)
- Both `default` and `my-gateway` appear in analytics results

---

## Real Query Results (Sample Data)

Our account's top 10 request groups (2026-04-01 to 2026-04-19):

| Model | Provider | Gateway | Status | Requests | Cached | Cost |
|-------|----------|---------|--------|----------|--------|------|
| MiniMax-M2.7 | custom-minimax | default | 200 | 112 | 20 | $0.00 |
| llama-2-7b-chat-fp16 | workers-ai | default | 200 | 7 | 1 | $0.001056 |
| bge-large-en-v1.5 | workers-ai | default | 200 | 1 | 0 | $0.00 |
| unknown | custom-minimax | default | 401 | 9 | — | $0.00 |
| unknown | custom-minimax | default | 400 | 3 | — | $0.00 |
| embm-01 | custom-minimax | default | 2013 | 2 | — | $0.00 |

> **Note:** 401s from `custom-minimax` are from early testing with wrong tokens. 2013 from `embm-01` is MiniMax's "invalid params" response — embeddings is not in the plan.

---

## Python Helper Script

```python
#!/usr/bin/env python3
"""
Pull AI Gateway analytics and print a summary table.
Requires: requests, python-dotenv
"""

import os
import requests
from datetime import datetime, timedelta

GRAPHQL_URL = "https://api.cloudflare.com/client/v4/graphql"
ACCOUNT_ID = os.getenv("CF_ACCOUNT_ID")
TOKEN = os.getenv("CFAT_TOKEN")  # cfat_... (Management token)

def query_analytics(days=7, limit=20):
    since = (datetime.utcnow() - timedelta(days=days)).strftime("%Y-%m-%dT%H:%M:%SZ")
    query = f"""
    {{
      viewer {{
        accounts(filter: {{ accountTag: "{ACCOUNT_ID}" }}) {{
          requests: aiGatewayRequestsAdaptiveGroups(
            limit: {limit},
            filter: {{ datetimeHour_geq: "{since}" }}
          ) {{
            count
            sum {{
              cost
              cachedRequests
              erroredRequests
              uncachedTokensIn
              uncachedTokensOut
            }}
            dimensions {{
              model
              provider
              gateway
              statusCode
            }}
          }}
        }}
      }}
    }}
    """
    resp = requests.post(
        GRAPHQL_URL,
        headers={"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"},
        json={"query": query}
    )
    data = resp.json()
    groups = data.get("data", {}).get("viewer", {}).get("accounts", [{}])[0].get("requests", [])
    return groups

def print_summary(groups):
    print(f"{'Model':<25} {'Provider':<20} {'Status':<6} {'Reqs':>6} {'Cached':>6} {'Cost':>12}")
    print("-" * 80)
    for g in groups:
        d = g["dimensions"]
        s = g["sum"]
        print(f"{d['model']:<25} {d['provider']:<20} {d['statusCode']:<6} {g['count']:>6} {s['cachedRequests']:>6} ${s['cost']:>11.6f}")

if __name__ == "__main__":
    groups = query_analytics(days=7)
    print_summary(groups)
```

---

## Summary

| Feature | Status | Notes |
|---------|--------|-------|
| Request counts | ✅ Works | Grouped by model/provider/gateway/status |
| Token tracking | ⚠️ Partial | Works for chat; empty for embeddings/TTS |
| Cost tracking | ⚠️ Partial | Workers AI = real cost; BYOK = $0 |
| Cache hit tracking | ✅ Works | Via `cachedRequests` on requests groups |
| Error analytics | ⚠️ Partial | Use `statusCode_geq: 400` filter instead |
| Cache groups | ❌ Empty | `aiGatewayCacheAdaptiveGroups` returns `[]` |
| Error groups | ❌ Empty | `aiGatewayErrorsAdaptiveGroups` returns `[]` |
