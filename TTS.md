# MiniMax TTS via AI Gateway

## Overview

MiniMax Text-to-Speech (TTS) works through AI Gateway via **HTTP only** (non-streaming). WebSocket streaming is **not supported** through AI Gateway's Universal WebSocket — MiniMax TTS uses a proprietary event-based WebSocket protocol that is incompatible with AI Gateway's request/response wrapper.

## Supported: HTTP TTS

**Endpoint:**
```
POST https://gateway.ai.cloudflare.com/v1/{account_id}/{gateway}/custom-minimax/v1/t2a_v2
```

**Auth:** `Authorization: Bearer {cfut_token}` (runtime token)

**Request body:**
```json
{
  "model": "speech-2.8-hd",
  "text": "Hello world, testing AI Gateway TTS.",
  "voice_setting": {
    "voice_id": "male-qn-qingse",
    "speed": 1,
    "vol": 1,
    "pitch": 0
  },
  "audio_setting": {
    "sample_rate": 32000,
    "bitrate": 128000,
    "format": "mp3",
    "channel": 1
  }
}
```

**Response:** Binary MP3 audio in `data.audio` field (hex-encoded).

**Example curl:**
```bash
curl -X POST "https://gateway.ai.cloudflare.com/v1/ACCOUNT_ID/GATEWAY/custom-minimax/v1/t2a_v2" \
  -H "Authorization: Bearer $CFUT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "speech-2.8-hd",
    "text": "Hello world",
    "voice_setting": {"voice_id": "male-qn-qingse", "speed": 1, "vol": 1, "pitch": 0},
    "audio_setting": {"sample_rate": 32000, "bitrate": 128000, "format": "mp3", "channel": 1}
  }'
```

**Python example:**
```python
import requests, json

resp = requests.post(
    f"https://gateway.ai.cloudflare.com/v1/{account_id}/{gateway}/custom-minimax/v1/t2a_v2",
    json={
        "model": "speech-2.8-hd",
        "text": "Hello world",
        "voice_setting": {"voice_id": "male-qn-qingse", "speed": 1, "vol": 1, "pitch": 0},
        "audio_setting": {"sample_rate": 32000, "bitrate": 128000, "format": "mp3", "channel": 1}
    },
    headers={"Authorization": f"Bearer {cfut_token}", "Content-Type": "application/json"}
)

audio_hex = resp.json()["data"]["audio"]
audio_bytes = bytes.fromhex(audio_hex)
with open("output.mp3", "wb") as f:
    f.write(audio_bytes)
```

## Not Supported: WebSocket Streaming

MiniMax TTS WebSocket (`wss://api.minimax.io/ws/v1/t2a_v2`) uses a proprietary event-based protocol:

1. Client sends `task_start` → server returns `task_started`
2. Client sends `task_continue` with text → server streams `task_continued` events with audio chunks
3. Client sends `task_finish` → server returns `task_finished`

AI Gateway's Universal WebSocket (`wss://gateway.ai.cloudflare.com/v1/{account}/{gateway}/`) only supports HTTP-style request/response wrapped in WebSocket frames (universal.create/universal.created). The underlying WebSocket upgrade to MiniMax's TTS endpoint fails with HTTP 500.

**Direct MiniMax TTS WebSocket works fine** — use it directly when streaming is needed:
```python
# Direct MiniMax TTS WebSocket (streaming capable)
url = "wss://api.minimax.io/ws/v1/t2a_v2"
headers = {"Authorization": f"Bearer {minimax_api_key}"}
```

## Cache Behavior

TTS responses respect the `cf-aig-cache-ttl` header. Identical TTS requests (same text, voice, model) will return cached audio.

## Rate Limiting

- MiniMax rate limit: **60 RPM** for TTS models
- AI Gateway rate limit: configurable per gateway (tested 5 req/min — well below MiniMax's limit)
- TTS quota on Token Plan: **4,000 chars/day** (Plus plan), resets at midnight UTC

## Model Support

| Model | HTTP | WebSocket | Notes |
|-------|------|-----------|-------|
| speech-2.8-hd | ✅ | ❌ | Best quality, HD voices |
| speech-2.8-turbo | ✅ | ❌ | Faster, slightly lower quality |
| speech-02-turbo | ❌ | ❌ | Not supported on Plus plan |
