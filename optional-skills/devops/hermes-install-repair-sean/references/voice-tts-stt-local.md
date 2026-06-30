# Voice Activation & Local TTS/STT Settings

## Voice Record Key
- **Record Key:** `Ctrl+B` — hold to record, release to send
- Config: `voice.record_key: ctrl+b`

## Speech-to-Text (STT) — Mic Input
- `stt.enabled: true`
- `stt.provider: local` — mic audio processed locally via Whisper. **No cloud required.**
- `stt.local.model: base` — small model, CPU-friendly for this Dell

## Text-to-Speech (TTS) — spoken responses
- `voice.auto_tts: true` — responses spoken aloud automatically
- `voice.beep_enabled: true` — audio feedback when recording starts/stops
- `voice.silence_threshold: 200`
- `voice.silence_duration: 3.0` — stops recording after 3s of silence
- `voice.max_recording_seconds: 120`

## Local TTS Options (zero cloud cost, CPU-friendly)

### Piper (lightweight, recommended for this machine)
```yaml
tts:
  provider: piper
  piper:
    voice: en_US-lessac-medium
```
Piper runs entirely on CPU, no GPU needed.

### Neutts (small GGUF model)
```yaml
tts:
  provider: neutts
  neutts:
    model: neuphonic/neutts-air-q4-gguf
    device: cpu
```
Runs a quantized GGUF model locally. Heavier than Piper but better quality.

## Enable STT in config.yaml (minimal)
```yaml
stt:
  enabled: true
  provider: local
  local:
    model: base      # small Whisper, CPU-friendly
    language: ''      # auto-detect

voice:
  record_key: ctrl+b
  max_recording_seconds: 120
  auto_tts: true
  beep_enabled: true
  silence_threshold: 200
  silence_duration: 3.0
```

## Status Check
```bash
hermes status
# Look for: "Speech-to-text  ✓ included by subscription, not currently selected"
# Nous subscription includes STT; local provider overrides cloud.
```