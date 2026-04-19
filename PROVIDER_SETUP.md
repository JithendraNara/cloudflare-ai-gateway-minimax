# Provider Setup: Adding MiniMax to AI Gateway

## Overview

Cloudflare AI Gateway supports custom providers via BYOK (Bring Your Own Keys). This guide walks through adding MiniMax as a custom provider.

## Prerequisites

- Cloudflare account with AI Gateway enabled (Workers Paid plan or higher)
- MiniMax API key (with active token plan)
- Two API tokens:
  - **AI Gateway Run** (`cfut_...`) — for runtime requests through the gateway
  - **AI Gateway Edit** (`cfat_...`) — for management/metadata (optional for this guide)

## Step 1: Create an API Token (Cloudflare)

1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com/) → **My Profile** → **API Tokens**
2. Create Token → **Create Custom Token**
3. Add permissions:
   - `AI Gateway > AI Gateway Run > Edit`
   - `AI Gateway > AI Gateway Edit > Edit` (optional, for management)
4. Save the token — you'll see `cfut_...` for runtime and `cfat_...` for management

## Step 2: Add MiniMax Provider Key (BYOK)

1. Go to **AI** → **AI Gateway** → Select your gateway
2. Navigate to **Settings** → **Provider Keys**
3. Click **Add Key**:
   - Provider: `Custom`
   - Name: `minimax` (this becomes your provider slug)
   - API Key: your MiniMax API key
4. Save

## Step 3: Configure the Custom Provider

Still in gateway settings, navigate to **Providers** → **Custom Providers** → **Add Provider**:

| Field | Value | Notes |
|-------|-------|-------|
| **Provider Name** | `minimax` | Lowercase, used in URL paths |
| **base_url** | `https://api.minimax.io/anthropic` | Critical — must include `/anthropic` |
| **Auth Type** | `API Key` | Select the Provider Key added in Step 2 |

### Why `/anthropic` in the base_url?

MiniMax's API is not fully OpenAI-compatible. Standard endpoints use `/anthropic/v1/...` paths rather than `/v1/...`. By setting `base_url` to `https://api.minimax.io/anthropic`, the gateway correctly routes:

```
Gateway path:  /custom-minimax/v1/chat/completions
              ↓
base_url:     https://api.minimax.io/anthropic
              ↓
Upstream:     https://api.minimax.io/anthropic/v1/chat/completions ✅
```

## Step 4: Verify the Setup

```bash
export AIG_TOKEN="cfut_YOUR_RUN_TOKEN"
export ACCOUNT_ID="your_account_id"
export GATEWAY="your_gateway_id"

# Test chat completions
curl -s -w "\nStatus: %{http_code}\n" -X POST \
  "https://gateway.ai.cloudflare.com/v1/$ACCOUNT_ID/$GATEWAY/custom-minimax/v1/chat/completions" \
  -H "cf-aig-authorization: Bearer $AIG_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "minimax-m2.7",
    "messages": [{"role": "user", "content": "Hello, respond in 3 words."}],
    "max_tokens": 20
  }' | python3 -c "import sys,json; d=json.load(sys.stdin); print('Content:', d['choices'][0]['message']['content'])"
```

Expected output: `Content: Hello! How can I...`

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `code: 2008, Invalid provider` | Provider slug wrong in URL | Use `custom-minimax` not `minimax` |
| `login fail (401)` | base_url missing `/anthropic` | Must be `https://api.minimax.io/anthropic` |
| `unknown model 'm2.7'` | Model name format wrong | Use full name: `minimax-m2.7` |
| 401 on all requests | Wrong auth header | Use `cf-aig-authorization` not `Authorization` |

## Provider URL Formats

Once configured, two URL formats work:

### Provider-Native (Full Path Control)
```
https://gateway.ai.cloudflare.com/v1/{account}/{gateway}/custom-minimax/{provider-path}
```
Use when the endpoint path differs from OpenAI's standard `/v1/...` format.

### Unified API (OpenAI-Compatible)
```
https://gateway.ai.cloudflare.com/v1/{account}/{gateway}/compat/chat/completions
```
Model name format: `custom-minimax/minimax-m2.7`
