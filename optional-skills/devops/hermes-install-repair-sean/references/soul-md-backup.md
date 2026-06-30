# SOUL.md — Persona File Git Backup Pattern

## What is SOUL.md

Located at `hermes-agent/docker/SOUL.md`. Defines the agent's personality, name, tone, and interests. Loaded fresh every message — no restart needed after edits.

## Backup workflow

When SOUL.md is edited during a session, push to the user's fork for durability:

```bash
cd ~/hermes-agent
git add docker/SOUL.md
git commit -m "Update SOUL.md: $(date '+%Y-%m-%d %H:%M')"
git push origin main
```

## Automated hourly backup with Delta

For users who want delta-style diffs on auto-backup:

### 1. Install Delta (git diff viewer with syntax highlighting)

```bash
winget install -e --id dandavison.delta --silent
```

Delta installs to `%LOCALAPPDATA%\Microsoft\WinGet\Links\delta.exe`.

### 2. Create backup script

`~/.hermes/scripts/soul-backup.sh`:

```bash
#!/bin/bash
REPO="$HOME/hermes-agent"
SOUL="$REPO/docker/SOUL.md"
LOG="$HOME/AppData/Local/hermes/logs/soul-backup.log"

cd "$REPO" || exit 1

git fetch origin main 2>/dev/null
LOCAL=$(git rev-parse @)
REMOTE=$(git rev-parse origin/main 2>/dev/null)

[ "$LOCAL" = "$REMOTE" ] && exit 0  # silent if synced

git pull origin main >> "$LOG" 2>&1

# Add delta to PATH (winget install location)
export PATH="$HOME/AppData/Local/Microsoft/WinGet/Links:$PATH"

if git diff --stat origin/main -- "$SOUL" | grep -q "SOUL.md"; then
    echo "=== SOUL.md changes detected ===" >> "$LOG"
    git diff origin/main -- "$SOUL" | delta >> "$LOG" 2>&1
fi

git add "$SOUL"
git diff --cached --quiet && exit 0

git commit -m "Auto-backup SOUL.md: $(date '+%Y-%m-%d %H:%M')" >> "$LOG" 2>&1
git push origin main >> "$LOG" 2>&1
echo "Backup completed: $(date)" >> "$LOG"
```

### 3. Register hourly cron job

```bash
# Script must live in ~/.hermes/scripts/ (not ~/AppData/Local/hermes/scripts/)
hermes cron create \
  --name "SOUL.md Hourly Backup" \
  --no-agent \
  --script soul-backup.sh \
  --schedule "0 * * * *"
```

- Script path is relative to `~/.hermes/scripts/` — do NOT use absolute paths.
- `--no-agent` = script output delivered verbatim; no LLM involved.
- Schedule is cron syntax (UTC by default; adjust for user's timezone).

## Key paths (this machine)

| Item | Path |
|---|---|
| SOUL.md | `~/hermes-agent/docker/SOUL.md` |
| Backup script | `~/.hermes/scripts/soul-backup.sh` |
| Log file | `~/AppData/Local/hermes/logs/soul-backup.log` |
| Delta (winget) | `~/.local/bin/delta` / `~/AppData/Local/Microsoft/WinGet/Links/delta` |