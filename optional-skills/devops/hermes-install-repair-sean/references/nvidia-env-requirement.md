# NVIDIA API Key `.env` Requirement

## Problem
After restoring Hermes from backup and setting `provider.nvidia.api_key` in `config.yaml`, Telegram DM sessions still failed with:

```
RuntimeError: Provider 'nvidia' is set in config.yaml but no API key was found. Set the NVIDIA_API_KEY environment variable, or switch to a different provider with `hermes model`.
```

## Root cause
The gateway runtime reads the NVIDIA key from `.env` as `NVIDIA_API_KEY`, not from `config.yaml`. Setting it via `hermes config set provider.nvidia.api_key` updates `config.yaml` only.

## Fix
```bash
# Verify
grep -E "^NVIDIA_API_KEY=" "$HOME/AppData/Local/hermes/.env"

# If missing, add it
echo "NVIDIA_API_KEY=nvapi-..." >> "$HOME/AppData/Local/hermes/.env"
```

Then restart the gateway:
```bash
hermes gateway restart
```

## Takeaway
`provider.nvidia.api_key` in `config.yaml` documents the intended provider, but the actual secret flow requires `NVIDIA_API_KEY` in `.env`. Always verify both after key rotation or fresh restore.
