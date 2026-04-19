# Hermes Agent Integration

## Overview

This document shows how Hermes (the personal AI agent) uses AI Gateway with MiniMax in production.

## Current Architecture

```
Hermes (Python)
    │
    ├─► MiniMax Direct ──► MiniMax API
    │       (voice/TTS, fast path)
    │
    └─► AI Gateway ──────► Custom Provider: MiniMax
            (chat, caching, observability)
```

## Integration Pattern: Cached Chat

For IPL briefings, news alerts, and any broadcast-style content:

```python
import requests
from typing import Optional

class AIGatewayClient:
    def __init__(self, account_id: str, gateway_id: str, aig_token: str):
        self.base_url = f"https://gateway.ai.cloudflare.com/v1/{account_id}/{gateway_id}"
        self.token = aig_token

    def chat(
        self,
        messages: list,
        model: str = "minimax-m2.7",
        max_tokens: int = 500,
        cache_ttl: Optional[int] = None,
        skip_cache: bool = False,
    ) -> tuple[str, str]:
        """Returns (response_text, cache_status)"""
        headers = {
            "cf-aig-authorization": f"Bearer {self.token}",
            "Content-Type": "application/json",
        }

        if cache_ttl:
            headers["cf-aig-cache-ttl"] = str(cache_ttl)
        if skip_cache:
            headers["cf-aig-skip-cache"] = "true"

        payload = {
            "model": model,
            "messages": messages,
            "max_tokens": max_tokens,
        }

        response = requests.post(
            f"{self.base_url}/custom-minimax/v1/chat/completions",
            headers=headers,
            json=payload,
            timeout=30,
        )

        cache_status = response.headers.get("cf-aig-cache-status", "UNKNOWN")
        data = response.json()

        return data["choices"][0]["message"]["content"], cache_status


# Usage for broadcast content (e.g., IPL score briefing)
client = AIGatewayClient(
    account_id=os.environ["CF_ACCOUNT_ID"],
    gateway_id=os.environ["CF_GATEWAY_ID"],
    aig_token=os.environ["AIG_TOKEN"],
)

# First user gets fresh response (MISS), next 999 users get cached (HIT)
content, cache = client.chat(
    messages=[{"role": "user", "content": "CSK vs RCB score update"}],
    cache_ttl=300,  # 5 minute cache for scores
)
print(f"Cache: {cache}, Response: {content}")
```

## Integration Pattern: TTS via AI Gateway

```python
def text_to_speech(text: str, voice_id: str = "male-qn-qingse") -> bytes:
    """Generate speech via AI Gateway T2A endpoint."""
    response = requests.post(
        f"{base_url}/custom-minimax/v1/t2a_v2",
        headers={
            "cf-aig-authorization": f"Bearer {aig_token}",
            "Content-Type": "application/json",
        },
        json={
            "model": "speech-2.8-hd",
            "text": text,
            "stream": False,
            "voice_setting": {
                "voice_id": voice_id,
                "speed": 1.0,
                "vol": 1.0,
                "pitch": 0,
            },
            "audio_setting": {
                "sample_rate": 32000,
                "bitrate": 128000,
                "format": "mp3",
                "channel": 1,
            },
        },
        timeout=30,
    )
    return response.content  # raw MP3 bytes
```

## Cron Job: Daily IPL Briefing

```python
# Cron: Daily at 8 PM ET
CRON_SCHEDULE = "0 20 * * *"  # 8 PM daily

def generate_ipl_briefing() -> dict:
    """Generate daily IPL briefing with caching."""
    # Build messages
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": "Give me today's IPL match previews and predictions."},
    ]

    # First request: MISS (hits MiniMax), cached for 1 hour
    # All subsequent identical requests: HIT
    content, cache_status = client.chat(
        messages=messages,
        cache_ttl=3600,  # 1 hour
        max_tokens=800,
    )

    return {
        "content": content,
        "cache": cache_status,
        "generated_at": datetime.now().isoformat(),
    }
```

## Environment Variables

```bash
# AI Gateway
CF_ACCOUNT_ID=your_account_id
CF_GATEWAY_ID=your_gateway_id
AIG_TOKEN=cfut_your_runtime_token
```

## When to Use Each Path

| Use Case | Path | Why |
|----------|------|-----|
| IPL briefing, news | AI Gateway + cache | Broadcast optimization |
| Real-time scores | AI Gateway, skip cache | Fresh data required |
| TTS voice output | AI Gateway T2A | Same caching benefits |
| Quick questions | MiniMax Direct | Faster, no gateway overhead |
| Streaming responses | AI Gateway | Real-time token delivery |

## Error Handling

```python
import time

def chat_with_retry(messages, max_retries=3):
    for attempt in range(max_retries):
        try:
            return client.chat(messages)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limited
                time.sleep(2 ** attempt)
                continue
            raise
    raise Exception("Max retries exceeded")
```

## Metrics to Track

| Metric | How to Get |
|--------|-----------|
| Cache hit rate | Count `HIT` / total requests |
| Latency | Response time per request |
| Token usage | From response `usage` field |
| Error rate | Non-200 responses / total |
