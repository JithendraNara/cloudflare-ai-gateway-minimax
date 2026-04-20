# Tested Endpoints — MiniMax via AI Gateway

## Chat — Anthropic-Compatible Messages (Recommended)

**Endpoint:** `POST /anthropic/v1/messages`
**Provider path:** `custom-minimax/anthropic/v1/messages`
**Model:** `MiniMax-M2.7`

```bash
curl -X POST "https://gateway.ai.cloudflare.com/v1/$ACCOUNT_ID/$GATEWAY/custom-minimax/anthropic/v1/messages" \
  -H "cf-aig-authorization: Bearer $AIG_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "MiniMax-M2.7",
    "system": "You are a cricket analyst.",
    "messages": [{
      "role": "user",
      "content": [{"type": "text", "text": "Who will win IPL 2026?"}]
    }],
    "max_tokens": 200,
    "temperature": 1
  }'
```

**Response:** (truncated)
```json
{
  "id": "...",
  "type": "message",
  "role": "assistant",
  "content": [
    {"type": "thinking", "thinking": "..."},
    {"type": "text", "text": "Based on current team compositions..."}
  ],
  "usage": {
    "input_tokens": 65,
    "output_tokens": 120
  }
}
```

---

## Chat — OpenAI-Compatible

**Endpoint:** `POST /v1/chat/completions`
**Provider path:** `custom-minimax/v1/chat/completions`
**Model:** `MiniMax-M2.7`

```bash
curl -X POST "https://gateway.ai.cloudflare.com/v1/$ACCOUNT_ID/$GATEWAY/custom-minimax/v1/chat/completions" \
  -H "cf-aig-authorization: Bearer $AIG_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "MiniMax-M2.7",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 200,
    "temperature": 1,
    "reasoning_split": true
  }'
```

> MiniMax's OpenAI-compatible API can include reasoning in `message.content`. Use the Anthropic-compatible route above if you want separate thinking/text blocks.

---

## Text-to-Audio (T2A) — Speech Generation

**Endpoint:** `POST /v1/t2a_v2`
**Model:** `speech-2.8-hd` (recommended)
**Use:** Generate spoken audio from text

```bash
curl -X POST "https://gateway.ai.cloudflare.com/v1/$ACCOUNT_ID/$GATEWAY/custom-minimax/v1/t2a_v2" \
  -H "cf-aig-authorization: Bearer $AIG_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "speech-2.8-hd",
    "text": "CSK beat RCB by 5 wickets in a thrilling match.",
    "stream": false,
    "voice_setting": {
      "voice_id": "male-qn-qingse",
      "speed": 1.0,
      "vol": 1.0,
      "pitch": 0
    },
    "audio_setting": {
      "sample_rate": 32000,
      "bitrate": 128000,
      "format": "mp3",
      "channel": 1
    }
  }' \
  --output /tmp/tts_output.mp3

# Check output
ls -lh /tmp/tts_output.mp3
```

**Known Voice IDs:**
| ID | Description |
|----|-------------|
| `male-qn-qingse` | Male voice (recommended) |
| `female-tianmei` | Female voice |

**Known Models:**
| Model | Notes |
|-------|-------|
| `speech-2.8-hd` | ✅ Working — HD quality |
| `speech-02-hd` | ⚠️ Exhausted on some plans |

---

## Image Generation

**Endpoint:** `POST /v1/image_generation`
**Model:** `image-01`
**Use:** Generate images from text prompts

```bash
curl -X POST "https://gateway.ai.cloudflare.com/v1/$ACCOUNT_ID/$GATEWAY/custom-minimax/v1/image_generation" \
  -H "cf-aig-authorization: Bearer $AIG_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "image-01",
    "prompt": "A cricket stadium at night with vibrant LED lights, IPL atmosphere",
    "aspect_ratio": "16:9"
  }'
```

**Response:**
```json
{
  "created": 1713001234,
  "data": [{
    "url": "https://hailuo-image-algeng-data-us.oss-us-east-1.aliyuncs.com/..."
  }]
}
```

**Parameters:**
| Param | Type | Notes |
|-------|------|-------|
| `model` | string | `image-01` |
| `prompt` | string | Image description |
| `aspect_ratio` | string | `"1:1"` (default), `"16:9"`, `"9:16"` |

**Downloading the image:**
```bash
IMAGE_URL=$(curl -s -X POST "..." -d '...' | python3 -c "import sys,json; print(json.load(sys.stdin)['data'][0]['url'])")
curl -s "$IMAGE_URL" --output generated_image.jpg
ls -lh generated_image.jpg
```

---

## Streaming Responses

```bash
curl -X POST "https://gateway.ai.cloudflare.com/v1/$ACCOUNT_ID/$GATEWAY/custom-minimax/v1/chat/completions" \
  -H "cf-aig-authorization: Bearer $AIG_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "MiniMax-M2.7",
    "messages": [{"role": "user", "content": "Explain quantum physics in one sentence."}],
    "max_tokens": 100,
    "stream": true
  }'
```

**Response:** SSE stream with chunks like:
```
data: {"nonce":"...","choices":[{"delta":{"content":"Quantum"}}]}
data: {"nonce":"...","choices":[{"delta":{"content":" physics"}}]}
...
data: [DONE]
```

> ⚠️ Streaming responses are **not cached** — each chunk is unique.

---

## WebSocket — Non-Realtime

**URL:** `wss://gateway.ai.cloudflare.com/v1/{account}/{gateway}/custom-minimax`
**Auth:** `Authorization: Bearer $CFUT_TOKEN` (not `cf-aig-authorization`)

```javascript
const ws = new WebSocket(
  `wss://gateway.ai.cloudflare.com/v1/${ACCOUNT_ID}/${GATEWAY}/custom-minimax`,
  'chat-completions'
);

ws.send(JSON.stringify({
  type: 'universal.create',
  messages: [{ role: 'user', content: 'Hello!' }],
  model: 'MiniMax-M2.7',
  headers: { 'Authorization': 'Bearer cfut_...' }
}));

ws.on('message', (data) => {
  const msg = JSON.parse(data);
  if (msg.type === 'content') {
    process.stdout.write(msg.content);
  }
});
```

---

## Not Tested / Paid Add-ons

| Endpoint | Status |
|----------|--------|
| `/v1/embeddings` | ⚠️ Requires paid embedding plan |
| `/v1/video_generation` | Not tested |
| `/v1/music_generation` | Not tested |
| Function calling / tools | Not tested |

---

## Error Codes Reference

| Code | Message | Meaning |
|------|---------|---------|
| 0 | `success` | Request succeeded |
| 10000 | `Authentication error` | Wrong auth header (use `cf-aig-authorization`) |
| 2005 | `Unauthorized` | Token lacks permission |
| 2008 | `Invalid provider` | Provider slug wrong in URL |
| 2009 | `Unauthorized` | WebSocket auth failed |
| 2013 | `invalid params` | Missing required parameter |
| 13001 | — | Streaming timeout |
