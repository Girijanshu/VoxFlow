# Qwen3-TTS Server - RunPod Deployment

A text-to-speech server with voice cloning, powered by Qwen3-TTS.



## Requirements

- **Pod with CUDA 12.8**

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/base_tts/` | GET | TTS with default English voice |
| `/synthesize_speech/` | GET | TTS with specified voice |
| `/upload_audio/` | POST | Upload reference voice |
| `/change_voice/` | POST | Voice conversion |

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

## Notes

- Server files are in `/app/server/` (not `/workspace/`)
- `/workspace/` is free for network volumes
- Supports 10 languages: Chinese, English, Japanese, Korean, German, French, Russian, Portuguese, Spanish, Italian
