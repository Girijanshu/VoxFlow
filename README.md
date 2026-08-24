# VoxFlow

*Multilingual AI voice generation pipeline.*

A FastAPI text-to-speech server built on [Qwen3-TTS](https://github.com/QwenLM/Qwen3-TTS) (1.7B-parameter voice cloning) with Whisper for automatic reference-audio transcription. Give it a short voice sample and it clones the voice, transcribes it automatically, and caches the result so repeat requests skip re-transcription.

## Features

- **Voice cloning** from a short reference clip (auto-trimmed to ~15s)
- **Voice conversion** of existing recordings to a registered voice
- **10 languages** — Chinese, English, Japanese, Korean, German, French, Russian, Portuguese, Spanish, Italian (auto-detected)
- **Automatic transcription** via Whisper — no manual transcript needed
- **Voice caching** — processed audio + prompt cached per voice label after first use
- **Speed control** via time-stretching, independent of pitch
- **Reproducible output** — fixed seed for consistent generation
- **Format-tolerant** — accepts wav/mp3/flac/ogg, converts internally
- **Container-ready** — models pre-downloaded at Docker build time

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/base_tts/` | GET | TTS with the default voice |
| `/synthesize_speech/` | GET | TTS with a specified voice |
| `/upload_audio/` | POST | Upload/register a reference voice |
| `/change_voice/` | POST | Voice conversion of an uploaded audio file |

<details>
<summary>Endpoint parameters</summary>

**`GET /synthesize_speech/`** — `text` (required), `voice` (required, matches a filename prefix in `resources/`), `speed` (optional, default `1.0`). Returns streamed `audio/wav`.

**`GET /base_tts/`** — `text` (required), `speed` (optional, default `1.0`). Uses the default voice.

**`POST /upload_audio/`** — `audio_file_label` (string), `file` (wav/mp3/flac/ogg, max 5MB). Validated by extension and content type; re-uploading a label clears its cache.

**`POST /change_voice/`** — `reference_speaker` (must already exist in `resources/`), `file` (audio to convert). Input is auto-transcribed.

</details>

## Quick Start

**Docker (recommended)** — models are baked into the image at build time:

```bash
docker build -t voxflow .
docker run --gpus all -p 7860:7860 voxflow
```

**Local:**

```bash
conda create -n qwen3-tts python=3.12 -y && conda activate qwen3-tts
pip install torch torchaudio --index-url https://download.pytorch.org/whl/cu128
pip install -r requirements.txt
pip install flash-attn --no-build-isolation  # optional, faster inference

./start.sh   # or: python -m uvicorn server:app --host 0.0.0.0 --port 7860
```

Serves on port `7860` by default.

## Usage Examples

```bash
# Synthesize speech
curl "http://localhost:7860/synthesize_speech/?text=Hello%20world&voice=demo_speaker0" --output output.wav

# Register a voice
curl -X POST "http://localhost:7860/upload_audio/" -F "audio_file_label=my_voice" -F "file=@/path/to/voice_sample.mp3"

# Convert a voice
curl -X POST "http://localhost:7860/change_voice/" -F "reference_speaker=demo_speaker0" -F "file=@/path/to/input.wav" --output converted.wav
```

## Directory Structure

```
.
├── server.py            # FastAPI server
├── Dockerfile            # Multi-stage build
├── requirements.txt
├── start.sh
├── resources/             # Voice reference files
└── outputs/               # Generated/temp audio
```

## Notes

- Responses from `/synthesize_speech/` include `X-Elapsed-Time` and `X-Device-Used` headers; CORS is wide open.
- Errors: `400` for an unmatched voice/reference speaker, `500` for unexpected failures.
- **Models**: `Qwen/Qwen3-TTS-12Hz-1.7B-Base` (TTS, 1.7B params) + `openai/whisper-base` (transcription, 74M params), pre-cached in the image.
- **Requirements**: CUDA GPU with 8GB+ VRAM recommended (CPU fallback works but is slow), CUDA 12.8+, Python 3.10+.
- **Deployment**: app files live in `/app/server/` (not `/workspace/`) so a network volume can be mounted separately; container stays alive via `sleep infinity` after launching `uvicorn`.

## License

MIT — see [`LICENSE`](LICENSE). Also review the licenses of [Qwen3-TTS](https://github.com/QwenLM/Qwen3-TTS) and [OpenAI Whisper](https://github.com/openai/whisper).
