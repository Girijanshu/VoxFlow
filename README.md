# VoxFlow

*Multilingual AI voice generation pipeline.*

VoxFlow is a FastAPI-based text-to-speech server built around [Qwen3-TTS](https://github.com/QwenLM/Qwen3-TTS), a 1.7B-parameter voice-cloning model. It exposes a small set of HTTP endpoints for turning text into speech, cloning a voice from a short audio sample, and converting the voice of existing recordings — all without any client-side ML code. It also uses OpenAI's Whisper model internally to automatically transcribe reference audio, so no manual transcript is required when adding a new voice.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [How It Works](#how-it-works)
- [API Endpoints](#api-endpoints)
- [Quick Start](#quick-start)
- [Directory Structure](#directory-structure)
- [Usage Examples](#usage-examples)
- [Response Headers & Error Handling](#response-headers--error-handling)
- [Models](#models)
- [Requirements](#requirements)
- [Deployment Notes](#deployment-notes)
- [License](#license)

## Overview

VoxFlow wraps the Qwen3-TTS voice-cloning model in a small, stateless-feeling HTTP API. You upload (or point it at) a short reference audio clip of a voice, and the server will:

1. Trim the clip down to the most useful ~15 seconds of speech and strip leading/trailing silence.
2. Transcribe that clip automatically with Whisper, so you don't have to supply a transcript yourself.
3. Build a reusable "voice clone prompt" from the processed audio + transcript, and cache it in memory.
4. Use that cached prompt to synthesize new speech in any of the supported languages, or to convert the voice of an existing recording.

Because the voice prompt is cached per voice label after the first use, repeat requests for the same voice skip the (comparatively slow) transcription step and only pay the cost of generation.

## Features

- **Voice cloning** — Clone any voice from a short reference audio clip (a few seconds up to ~15s is used automatically).
- **Voice conversion** — Take an existing audio recording and re-render its content in a different, previously registered voice.
- **Multi-language support** — 10 languages: Chinese, English, Japanese, Korean, German, French, Russian, Portuguese, Spanish, Italian. Language detection is automatic ("Auto" mode) — you don't need to specify it per request.
- **Automatic transcription** — Uses Whisper (`base` model) to transcribe reference audio, since Qwen3-TTS itself has no built-in transcription.
- **Voice caching** — Processed reference audio, its transcription, and the derived voice-clone prompt are cached in memory per voice label, so subsequent requests for the same voice avoid repeating the Whisper transcription step.
- **Speed control** — Adjust playback speed via time-stretching (powered by `librosa`), independent of pitch.
- **Reproducible output** — Generation uses a fixed random seed, so the same text + voice + speed combination produces consistent audio across requests.
- **Format-tolerant uploads** — Accepts `wav`, `mp3`, `flac`, and `ogg` reference/input files and transparently converts them to mono 24kHz WAV internally.
- **Container-ready** — Ships with a multi-stage Dockerfile that pre-downloads all models at build time, so the runtime image starts without needing network access to Hugging Face.

## How It Works

On startup, the server loads the Qwen3-TTS model and the Whisper transcription model onto GPU (if a CUDA device is available, otherwise it falls back to CPU automatically), then runs a one-time warmup generation using the bundled demo voice so the first real request isn't slowed down by lazy initialization.

When a new voice is registered or used for the first time:

1. The reference audio is converted to mono, 24kHz WAV.
2. The server looks for a natural silence to clip the sample down to roughly 15 seconds (falling back to a hard cut if no good silence point is found), and trims leading/trailing silence.
3. Whisper transcribes the trimmed clip.
4. Qwen3-TTS builds a "voice clone prompt" from the trimmed audio and its transcript.
5. All of the above is cached in memory under the voice's label, so it's only done once per voice (until the voice is re-uploaded, which clears the cache for that label).

Every generation request then reuses the cached prompt, applies an optional speed adjustment via time-stretching, and streams the resulting WAV back to the client.

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/base_tts/` | GET | TTS with the default voice |
| `/synthesize_speech/` | GET | TTS with a specified voice |
| `/upload_audio/` | POST | Upload/register a reference voice |
| `/change_voice/` | POST | Voice conversion of an uploaded audio file |

### Endpoint Details

#### `GET /synthesize_speech/`

Generate speech from text using a specified, previously uploaded voice (or the bundled demo voice).

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `text` | string | Yes | Text to synthesize |
| `voice` | string | Yes | Voice label — matches a filename prefix in `resources/` |
| `speed` | float | No | Speech speed multiplier (default: `1.0`) |

Returns a streamed `audio/wav` response. On the first call for a given voice, expect extra latency while the reference audio is processed and transcribed; subsequent calls for the same voice reuse the cached voice prompt.

#### `GET /base_tts/`

Generate speech using the server's default voice, without needing to specify a `voice` parameter.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `text` | string | Yes | Text to synthesize |
| `speed` | float | No | Speech speed multiplier (default: `1.0`) |

#### `POST /upload_audio/`

Upload an audio file to register as a reference voice for later use with `/synthesize_speech/` or `/change_voice/`.

| Field | Type | Description |
|-------|------|-------------|
| `audio_file_label` | string | Label/name for the voice — this becomes the prefix later requests reference |
| `file` | file | Audio file (`wav`, `mp3`, `flac`, `ogg`; max 5MB) |

The uploaded file is validated by both its extension and its actual content type (via `libmagic`), then stored under `resources/` and converted to a WAV copy. If a voice with the same label was previously used and cached, uploading a new file for that label clears the old cache entry so the new audio takes effect on next use.

#### `POST /change_voice/`

Convert the voice of an existing audio file to a previously registered reference voice. The input audio's content is transcribed automatically, so you don't need to supply the text yourself.

| Field | Type | Description |
|-------|------|-------------|
| `reference_speaker` | string | Voice label to convert to (must already exist in `resources/`) |
| `file` | file | Audio file to convert |

## Quick Start

### Using Docker (Recommended)

Docker is the simplest path since the image pre-downloads and caches all models at build time — the container has no external dependencies once built.

```bash
# Build the image
docker build -t voxflow .

# Run the container
docker run --gpus all -p 7860:7860 voxflow
```

The `--gpus all` flag is optional but strongly recommended; the server will run on CPU if no GPU is available, but generation will be considerably slower.

### Local Development

```bash
# Create a conda environment
conda create -n qwen3-tts python=3.12 -y
conda activate qwen3-tts

# Install PyTorch with CUDA support
pip install torch torchaudio --index-url https://download.pytorch.org/whl/cu128

# Install the remaining Python dependencies
pip install -r requirements.txt

# Optional: install FlashAttention 2 for faster inference
pip install flash-attn --no-build-isolation

# Run the server
./start.sh
# or, equivalently:
python -m uvicorn server:app --host 0.0.0.0 --port 7860
```

The server listens on port `7860` by default. Once it's running, a quick health check is simply hitting `/base_tts/` with some text (see [Usage Examples](#usage-examples) below).

## Directory Structure

```
.
├── server.py           # Main FastAPI server
├── Dockerfile           # Multi-stage Docker build (builder + slim runtime image)
├── requirements.txt     # Python dependencies
├── start.sh              # Startup script (launches uvicorn, keeps container alive)
├── resources/            # Voice reference files
│   └── demo_speaker0.mp3 # Bundled demo voice used for warmup and examples
└── outputs/              # Generated/intermediate audio (temporary)
```

## Usage Examples

### Synthesize Speech

```bash
curl "http://localhost:7860/synthesize_speech/?text=Hello%20world&voice=demo_speaker0" \
  --output output.wav
```

### Upload a Voice

```bash
curl -X POST "http://localhost:7860/upload_audio/" \
  -F "audio_file_label=my_voice" \
  -F "file=@/path/to/voice_sample.mp3"
```

### Use an Uploaded Voice

```bash
curl "http://localhost:7860/synthesize_speech/?text=Hello%20world&voice=my_voice" \
  --output output.wav
```

### Change the Voice of Audio

```bash
curl -X POST "http://localhost:7860/change_voice/" \
  -F "reference_speaker=demo_speaker0" \
  -F "file=@/path/to/input.wav" \
  --output converted.wav
```

## Response Headers & Error Handling

Successful calls to `/synthesize_speech/` include a couple of extra response headers that can be useful for monitoring:

| Header | Description |
|--------|-------------|
| `X-Elapsed-Time` | Total server-side time (seconds) spent handling the request |
| `X-Device-Used` | Which compute device (`cuda:0` or `cpu`) generated the audio |

CORS is wide open (`allow_origins=["*"]`) so the API can be called directly from browser-based clients.

Error responses use standard HTTP status codes:

- `400` — the requested voice or reference speaker doesn't have a matching file in `resources/`
- `500` — an unexpected error occurred during processing (the response body includes a `detail` message)

`/upload_audio/` additionally validates uploads before they're written to disk, rejecting files that exceed 5MB, have an unsupported extension, or whose actual content doesn't look like audio.

## Models

| Model | Purpose | Size |
|-------|---------|------|
| `Qwen/Qwen3-TTS-12Hz-1.7B-Base` | Voice cloning TTS | 1.7B parameters |
| `openai/whisper-base` | Audio transcription | 74M parameters |

Both models are pre-downloaded during the Docker build and stored in `/root/.cache/`, so a freshly started container doesn't need to fetch anything from the network before serving its first request.

## Requirements

- **GPU**: CUDA-compatible GPU with 8GB+ VRAM recommended (the server will fall back to CPU automatically if none is available, at a significant speed cost)
- **CUDA**: 12.8+ (needed for RTX 5090 / Blackwell architecture support)
- **Python**: 3.10+

## Deployment Notes

The VoxFlow Docker image is built to work well in container-hosting environments that provide a separate network volume mount (such as RunPod):

- Application files live in `/app/server/`, not `/workspace/`, leaving `/workspace/` free for a mounted network volume.
- The container's entrypoint keeps the process alive (`sleep infinity` after launching `uvicorn` in the background), which is convenient for platforms that expect long-running containers with shell/terminal access.
- The image exposes port `7860`, matching the port `start.sh` binds `uvicorn` to.

## License

This project is distributed under the MIT License — see [`LICENSE`](LICENSE) for the full text.

It also depends on, and you should review the licenses of:
- [Qwen3-TTS](https://github.com/QwenLM/Qwen3-TTS)
- [OpenAI Whisper](https://github.com/openai/whisper)