# Qwen3-TTS API — Endpoint Reference

Base URL (local): `http://127.0.0.1:9000`
Interactive docs: `http://127.0.0.1:9000/docs` (Swagger UI — you can test every endpoint from the browser)

Start the server:

```powershell
cd C:\Users\sak78\Qwen3TTS
.\venv\Scripts\python.exe server\app.py
```

Run the automated tests:

```powershell
.\venv\Scripts\python.exe test_endpoints.py
```

---

## Conventions

**Response format.** Every TTS endpoint accepts `"response_format"`:

- `"json"` (default) — JSON body with base64-encoded WAV files. Works for batches.
- `"wav"` — raw `audio/wav` bytes. Single text item only; convenient for `<audio src>`.

JSON response shape:

```json
{
  "sample_rate": 24000,
  "count": 2,
  "durations": [3.12, 2.87],
  "generation_seconds": 41.2,
  "audio": ["<base64 wav>", "<base64 wav>"]
}
```

**Batching.** Anywhere `text` accepts a list, `speaker` / `language` / `instruct` accept either a single value (applied to all) or a list of matching length.

**Language.** Omit it or pass `"auto"` to auto-detect. Supported: `chinese, english, french, german, italian, japanese, korean, portuguese, russian, spanish`.

**Sampling parameters.** Any TTS endpoint also accepts `max_new_tokens`, `do_sample`, `top_k`, `top_p`, `temperature`, `repetition_penalty`, `subtalker_dosample`, `subtalker_top_k`, `subtalker_top_p`, `subtalker_temperature`.

**Reference audio** (`ref_audio`) accepts a public URL, a server-local file path, a base64 WAV string, or a `data:audio/wav;base64,...` data URI. Or use the `/upload` variants to post a file directly.

> The underlying library decides "is this base64?" by checking the string contains no `/` — but standard base64 uses `/` as a symbol, so bare base64 audio is usually misread as a file path and fails. The server works around this by rewrapping bare base64 as a data URI, so all four forms work here. If you call `qwen_tts` directly elsewhere, send a data URI.

---

## Meta

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | Status, CUDA, VRAM, which models are resident |
| GET | `/v1/models` | The three models, download/load state, which endpoints each serves |
| GET | `/v1/voices` | The 9 built-in speakers with descriptions |
| GET | `/v1/languages` | Supported language names |

---

## 1. Custom voice — built-in speakers

`POST /v1/tts/custom-voice` · model: **CustomVoice**

The workhorse endpoint. 9 professional voices, optional natural-language style control.

| Field | Type | Notes |
|---|---|---|
| `text` | string \| string[] | required |
| `speaker` | string \| string[] | `vivian, serena, uncle_fu, dylan, eric, ryan, aiden, ono_anna, sohee` |
| `language` | string \| string[] | optional, defaults to auto |
| `instruct` | string \| string[] | optional style control |
| `response_format` | `"json"` \| `"wav"` | default `"json"` |

**Without an instruction prompt:**

```bash
curl -X POST http://127.0.0.1:9000/v1/tts/custom-voice \
  -H "Content-Type: application/json" \
  -d '{"text":"Hello, this is Qwen3 TTS.","speaker":"ryan","language":"english"}'
```

**With an instruction prompt:**

```bash
curl -X POST http://127.0.0.1:9000/v1/tts/custom-voice \
  -H "Content-Type: application/json" \
  -d '{"text":"I cannot believe you did that.","speaker":"vivian","language":"english","instruct":"Speak in a furious, sharp tone."}'
```

**Batch, different speaker per line:**

```json
{
  "text": ["The first line.", "The second line."],
  "speaker": ["ryan", "aiden"],
  "language": "english"
}
```

---

## 2. Voice design — invent a voice from a description

`POST /v1/tts/voice-design` · model: **VoiceDesign**

No reference audio and no preset speaker — the voice is generated from `instruct`.

| Field | Type | Notes |
|---|---|---|
| `text` | string \| string[] | required |
| `instruct` | string \| string[] | required — describes the voice |
| `language` | string \| string[] | optional |
| `response_format` | `"json"` \| `"wav"` | default `"json"` |

```bash
curl -X POST http://127.0.0.1:9000/v1/tts/voice-design \
  -H "Content-Type: application/json" \
  -d '{"text":"Welcome, traveller.","instruct":"An elderly wizard, deep gravelly voice, slow and wise.","language":"english"}'
```

Note: each call re-invents the voice, so the timbre drifts between calls. For a **consistent** character voice, register it once via `/v1/voices/designed` (section 5) and then use `/v1/tts/saved-voice`.

---

## 3. Voice clone — copy a voice from a sample

`POST /v1/tts/clone` · model: **Base**

| Field | Type | Notes |
|---|---|---|
| `text` | string \| string[] | required |
| `ref_audio` | string | required — URL, path, or base64 WAV |
| `ref_text` | string | transcript of `ref_audio`; required unless `x_vector_only_mode` |
| `x_vector_only_mode` | bool | default `false`. `true` skips the transcript, lower fidelity |
| `language` | string \| string[] | optional |

```bash
curl -X POST http://127.0.0.1:9000/v1/tts/clone \
  -H "Content-Type: application/json" \
  -d '{"text":"This is a cloned voice.","ref_audio":"https://example.com/sample.wav","ref_text":"Exact transcript of the sample.","language":"english"}'
```

**Accuracy matters:** `ref_text` must be an exact transcript of `ref_audio`. A mismatch noticeably degrades quality. 3+ seconds of clean speech is enough.

### `POST /v1/tts/clone/upload` — multipart variant

Same thing with a file upload instead of a URL. Fields are form fields: `text`, `ref_audio` (file), `ref_text`, `language`, `x_vector_only_mode`, `response_format`.

```bash
curl -X POST http://127.0.0.1:9000/v1/tts/clone/upload \
  -F "text=This is a cloned voice." \
  -F "ref_text=Exact transcript of the sample." \
  -F "language=english" \
  -F "ref_audio=@sample.wav"
```

---

## 4. Saved voices — register once, reuse forever

Cloning re-extracts speaker features on every call. Registering a voice does that work once and gives you a `voice_id`, so later calls are cheaper and the voice stays **identical** across requests. This is what your website should use for per-user voices.

### `POST /v1/voices/cloned` — register from reference audio

```json
{ "ref_audio": "https://example.com/sample.wav",
  "ref_text": "Exact transcript.",
  "x_vector_only_mode": false,
  "name": "narrator-alice" }
```

Returns:

```json
{ "voice_id": "vc_a1b2c3d4e5f6", "name": "narrator-alice", "kind": "cloned", "created_at": 1761... }
```

### `POST /v1/voices/cloned/upload` — same, via file upload

Form fields: `ref_audio` (file), `ref_text`, `x_vector_only_mode`, `name`.

### `POST /v1/voices/designed` — register an invented voice

Designs a voice from `instruct`, then persists it as a reusable `voice_id`. Uses both the VoiceDesign and Base models.

```json
{ "instruct": "A cheerful young woman with a bright, energetic tone.",
  "ref_text": "Optional sentence used to create the reference clip.",
  "language": "english",
  "name": "mascot" }
```

Returns the metadata plus `reference_audio` (base64 WAV) so you can preview the voice.

### `GET /v1/voices/saved` · `GET /v1/voices/saved/{voice_id}` · `DELETE /v1/voices/saved/{voice_id}`

List, inspect, and delete saved voices. Stored as `.pt` files in `saved_voices/`.

---

## 5. Speak with a saved voice

`POST /v1/tts/saved-voice` · model: **Base**

```json
{ "text": "Speaking with a registered voice.",
  "voice_id": "vc_a1b2c3d4e5f6",
  "language": "english" }
```

`text` may be a list for batch generation. Works with both cloned and designed voice IDs.

---

## 6. Design then clone — one-shot

`POST /v1/tts/design-then-clone`

Convenience endpoint: invents a voice, immediately builds a clone prompt from it, and speaks your text — in a single call. Returns the audio plus the `reference_audio` that was generated. Loads two models, so it is the slowest endpoint.

```json
{ "text": "The designed persona speaks this sentence.",
  "instruct": "A gruff pirate captain, weathered and loud.",
  "language": "english" }
```

If you want to reuse the persona, prefer `/v1/voices/designed` instead.

---

## 7. Codec — raw audio tokens

`POST /v1/codec/encode` — audio → discrete 12 Hz codes.

```json
{ "audio": "https://example.com/sample.wav" }
```

Returns `{ "count": 1, "shapes": [[T, 16]], "audio_codes": [...] }`.

`POST /v1/codec/decode` — codes → audio. Pass the `audio_codes` array straight back.

Useful for storing/transmitting audio compactly (12 frames per second) or for building training data. Not needed for normal TTS.

---

## Calling it from your website

```js
async function speak(text, speaker = "ryan") {
  const res = await fetch("https://<your-runpod-host>/v1/tts/custom-voice", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ text, speaker, language: "english", response_format: "wav" }),
  });
  if (!res.ok) throw new Error(await res.text());
  const url = URL.createObjectURL(await res.blob());
  new Audio(url).play();
}
```

With `response_format: "wav"` you get a blob you can feed straight to an `<audio>` element. For batches use the default `"json"` and decode each base64 string.

CORS is currently `allow_origins=["*"]` — **restrict this to your domain in [server/app.py](server/app.py) before exposing the server publicly**, and put an auth token in front of it.

---

## Configuration

| Env var | Default | Purpose |
|---|---|---|
| `QWEN_TTS_MODEL_DIR` | `C:\Users\sak78\Qwen3TTS\models` | Where the three model folders live |
| `QWEN_TTS_MAX_RESIDENT` | `1` | Models kept on the GPU at once. **Set to `3` on the A100** |
| `QWEN_TTS_ATTN` | `sdpa` | Set to `flash_attention_2` on Linux after installing flash-attn |
| `QWEN_TTS_DEVICE` | `cuda:0` | Target device |
| `QWEN_TTS_HOST` / `QWEN_TTS_PORT` | `127.0.0.1` / `9000` | Bind address |
| `QWEN_TTS_VOICE_DIR` | `C:\Users\sak78\Qwen3TTS\saved_voices` | Saved voice storage |

**On your 8 GB laptop** `MAX_RESIDENT=1` means switching between endpoint families evicts and reloads a model (~10–30 s). That is why some test calls are slow locally.

**On the A100 80 GB** set `QWEN_TTS_MAX_RESIDENT=3` — all three models stay resident (12.6 GB total), no swapping, and every endpoint is warm.

## Concurrency

All GPU work is serialized behind a single lock, so the server handles one generation at a time and queues the rest. That is deliberate: the models are not safe to call concurrently from multiple threads. To raise throughput, run several worker processes behind a load balancer on the A100 (VRAM allows it) rather than removing the lock.
