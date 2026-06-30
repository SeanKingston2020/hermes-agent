# Telegram Env Recovery

## Session finding (June 21 restore)
`hermes config set channels.telegram.bot_token ...` fails in this Hermes build because nested platform keys cannot be converted to a valid uppercase env var name.

What actually stores Telegram credentials:
- `TELEGRAM_BOT_TOKEN`
- `TELEGRAM_ALLOWED_USERS`
- `TELEGRAM_HOME_CHANNEL`

Recovery command:
```bash
grep -E "^TELEGRAM_" "$HOME/AppData/Local/hermes/.env"
```

Edit `.env` directly when `config set` rejects the key, then restart the gateway.
