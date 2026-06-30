---
name: hermes-install-repair
description: Install, reinstall, restore, and repair Hermes Agent on Windows. Covers pre-flight audits, backup restoration (skills, state snapshots, config), gateway fixes, and applying post-install patches.
---

# Hermes Agent — Install / Reinstall / Repair

## When to use this skill

- Fresh Hermes Agent install on Windows
- Restoring a backup after reinstall
- Applying known patches (DOS window fix, gateway_start issues)
- Migrating skills/memories from another machine or backup
- Recovering from broken gateway / config state

## Timezone

Sean's machine is in **MST (UTC-7)**. When scheduling or interpreting cron times, note whether the system/Hermes scheduler runs in local time or UTC. Cron expressions are typically UTC unless explicitly stated otherwise.

## Machine Naming (Sean's setup)

| Machine | OS | User | AppData path | TailScale IP | Hermes CLI |
|---|---|---|---|---|---|
| KingstonAI-Device | Win 10 (Dell) | Sean.Learn | `C:\Users\Sean.Learn\AppData\Local\hermes` | 100.112.121.41 | Full CLI — supports `/exec` for shell commands |
| ENVY | Win 11 | seank | `C:\Users\seank\AppData\Local\hermes` | 100.102.52.61 | Restricted CLI — NO shell commands, no `/exec` |

**Critical path difference:** Dell path uses `Sean.Learn`, ENVY uses `seank`. All restore/copy operations must use the correct user folder. Do NOT assume paths are the same between machines.

Always check what is already in place before running install scripts.

NOTE: On this Windows host, the `terminal` tool executes commands through **Git Bash / MSYS**, NOT PowerShell. PowerShell builtins and cmdlets such as `Test-Path`, `Get-ChildItem`, `Select-Object`, `Test-Path`, and `$env:VARNAME` syntax will **fail** in that shell. Use POSIX equivalents or PowerShell explicitly via `pwsh`:

```powershell
# In the Hermes terminal tool, prefer POSIX/Bash-compatible:
git --version; pwsh --version; gh --version; tailscale --version
[ -d "$HOME/hermes-agent" ] && echo "hermes-agent exists" || echo "missing"
[ -d "$HOME/My_AI_Agent" ] && echo "My_AI_Agent exists" || echo "missing"
hermes --version
[ -d "$HOME/AppData/Local/hermes" ] && echo "hermes home exists" || echo "missing"
```

Or, when you really need PowerShell cmdlets, explicitly invoke `pwsh` with PowerShell syntax.

Skip any step where prerequisites are already met.

## Full appdata restore from numbered backup folder

A common backup form is `backups/hermes_reinstall_YYYYMMDD_HHMMSS/hermes_appdata/`.
These are meant to be restored with a pre-flight backup of current state.

Recommended sequence:
1. `hermes gateway stop`
2. Back up current critical state files:
   `Copy-Item "$env:LOCALAPPDATA\\hermes\\state.db" <safe-backup-dir>`
   `Copy-Item "$env:LOCALAPPDATA\\hermes\\config.yaml" <safe-backup-dir>`
   `Copy-Item "$env:LOCALAPPDATA\\hermes\\auth.json" <safe-backup-dir>`
3. Restore:
   `Copy-Item "$BACKUP\\hermes_appdata\\*" "$env:LOCALAPPDATA\\hermes\\" -Recurse -Force`
4. Post-restore check: confirm `$env:LOCALAPPDATA\\hermes\\hermes-agent\\hermes_cli\\gateway_windows.py`
   still contains the DOS-window `restart()` fix (call to `_spawn_detached()`).
   If it does not, re-apply the fix before starting the gateway.
## Backup restore order

### Verified pattern (Windows, Hermes already installed)
Pre-flight check revealed:
- `hermes-agent` fork already present at `~/hermes-agent`
- `AppData\\Local\\hermes\\hermes-agent` also present (installed copy)
- DOS-window fix may already be present in the installed copy (commit `e2e1b497c`); verify before patching

Recommended restore sequence:
1. Stop gateway: `hermes gateway stop`
2. Back up current critical files: `state.db`, `config.yaml`, `auth.json` to `~/backups/pre-restore-YYYYMMDD/`
3. Restore appdata: `cp -r <BACKUP>/hermes_appdata/* "$HOME/AppData/Local/hermes/"`
4. Verify DOS fix in the installed copy: `grep -n "def restart" AppData/Local/hermes/hermes-agent/hermes_cli/gateway_windows.py`
5. Start gateway: `hermes gateway start`

This avoids touching the in-use cloned `~/hermes-agent` unless the `hermes_appdata\hermes-agent\` stub needs refresh.

### 1. Skills (safe, non-destructive)
```powershell
$BACKUP = "C:\Users\...\backups\hermes_reinstall_...\hermes_appdata\skills"
$TARGET = "$env:LOCALAPPDATA\hermes\skills"
Remove-Item $TARGET -Recurse -Force   # or cp -r after ensuring no lock files
Copy-Item "$BACKUP\*" $TARGET -Recurse -Force
```

### 2. State snapshot (DESTRUCTIVE — ask user first)
Overwrites current `state.db`, `config.yaml`, `auth.json`, etc.
```powershell
$SNAP = "C:\Users\...\backups\...\hermes_appdata\state-snapshots\20260612-182646-pre-update"
Copy-Item "$SNAP\*" "$env:LOCALAPPDATA\hermes\" -Recurse -Force
```

**Always warn the user:** restoring a state snapshot replaces current `state.db` and destroys any sessions created after the snapshot date. The same warning applies to a full `hermes_appdata` restore: it overwrites `state.db`, so keep a separate pre-restore backup if you need to recover recent sessions.

## DOS-window fix (gateway_windows.py → restart())

Symptom: `hermes gateway restart` or `schtasks /Run` flashes a DOS/cmd window.

Fix `restart()` in `%LOCALAPPDATA%\hermes\hermes-agent\hermes_cli\gateway_windows.py`:

- **BEFORE:** `restart()` calls `stop()`, then `start()`, which internally uses `schtasks /Run` (pop-up).
- **AFTER:** `restart()` calls `stop()`, then `_spawn_detached()` directly. No `start()`, no `schtasks /Run`.

```python
def restart() -> None:
    """Stop the gateway then start it again."""
    _assert_windows()
    stop()
    time.sleep(1.0)
    running_pids = _gateway_pids()
    if running_pids:
        print(f"✓ Gateway already running (PID: {', '.join(map(str, running_pids))})")
    else:
        pid = _spawn_detached()
        _report_gateway_start(f"direct spawn (PID {pid})")
```

## Applying patches

The built-in `patch` tool can fail if old_string/new_string quoting is awkward.
Fallback: use `python` (or `python3` if available) with `pathlib` + `textwrap.dedent` for reliable multi-line replacement, or use `%ProgramData%\...` style path quoting in PowerShell.

## Fresh ENVY Install Configuration Sequence

ENVY (Win 11, `seank`) has a **restricted Hermes CLI** — no shell access, no `/exec`, no `!`, no file commands. The CLI is for messaging only.

**SOUL.md placement on ENVY:**
- Dell clones the fork to `hermes-agent\docker\SOUL.md` on ENVY
- But Hermes looks for SOUL.md at `AppData\Local\hermes\SOUL.md`
- If the fork lands in `hermes-agent\docker\`, it must be copied to `AppData\Local\hermes\SOUL.md`
- The stash path is `C:\Users\seank\AppData\Local\hermes\hermes-agent\`

**If ENVY CLI has no shell access, use the stash:**
1. Git clone your fork to `C:\Users\seank\hermes-agent` (from the Dell session, not ENVY CLI)
2. Copy `hermes-agent\docker\SOUL.md` → `AppData\Local\hermes\SOUL.md`
3. If git clone worked in ENVY CLI but the file ended up elsewhere, check `AppData\Local\hermes\hermes-agent\`

## Fork-first setup for ENVY

**Critical:** ENVY's Hermes install clones from **upstream NousResearch by default**, NOT from the user's fork. If the user wants their Kinger persona on ENVY, the fork must be set as the primary remote **before** SOUL.md is placed. Even with SOUL.md in the correct `AppData\Local\hermes\SOUL.md` path, the gateway reads SOUL.md from the git repo working tree — and the upstream repo has the default Hermes persona.

**Verified symptom from June 30 2026 session:**
- SOUL.md was correctly copied to `AppData\Local\hermes\SOUL.md` on ENVY
- Gateway restart still showed default Hermes persona despite correct SOUL.md placement
- Root cause: `git remote -v` in the `hermes-agent` directory showed `https://github.com/NousResearch/hermes-agent`
- Fix: change remote to user's fork (`SeanKingston2020/hermes-agent`), then `git pull` to bring in the custom SOUL.md, then restart gateway

**Sequence for ENVY SOUL.md setup:**
1. From a RDP session or other access, clone user's fork to `C:\Users\seank\hermes-agent`
2. In ENVY's Hermes CLI, run `/exec git remote set-url origin https://github.com/SeanKingston2020/hermes-agent.git`
3. Run `/exec git pull` to pull the Kinger SOUL.md from the fork
4. Copy `hermes-agent\docker\SOUL.md` → `AppData\Local\hermes\SOUL.md`
5. Stop and restart the gateway — persona should now load from fork

**If ENVY CLI has no shell access** (no `/exec`, no `!`, no file commands): use RDP file explorer to edit `C:\Users\seank\hermes-agent\.git\config` and change the remote URL from upstream to user's fork. Then copy SOUL.md to `AppData\Local\hermes\SOUL.md` via file explorer.

## Post-restore gateway lifecycle

```powershell
hermes gateway stop
# restore files ...
hermes gateway start
hermes gateway status
```

If `start` says "already running", that is good — PID reuse is fine.

## Gateway watchdog (Windows Task Scheduler)

Hermes cron blocks gateway lifecycle commands (`restart/stop/kill`) to prevent restart loops (#30719). Auto-restart must therefore be done outside Hermes.

Recommended: Windows Task Scheduler with a PowerShell watchdog.

**Watchdog script template** (`~/AppData/Local/hermes/scripts/gateway-watchdog.ps1`):
```powershell
$ErrorActionPreference = "SilentlyContinue"
$LogFile = "$env:LOCALAPPDATA\hermes\logs\watchdog.log"
$Hermes = "$env:LOCALAPPDATA\hermes\hermes-agent\venv\Scripts\hermes.exe"

function Write-Log {
    param([string]$Msg)
    "$(Get-Date -f 'yyyy-MM-dd HH:mm:ss') - $Msg" | Out-File -Append -FilePath $LogFile
}

Write-Log "Watchdog check"
$status = & $Hermes gateway status 2>&1
if ($LASTEXITCODE -ne 0 -or $status -match "No gateway process detected") {
    Write-Log "Gateway down. Restarting..."
    $restart = & $Hermes gateway restart 2>&1
    Write-Log "Restart result: $restart"
} else {
    Write-Log "Gateway healthy."
}
```

**Register in Windows Task Scheduler** (run from PowerShell, or use `pwsh -Command` from Git Bash because `schtasks` path translation fails in MSYS):
```powershell
schtasks /Create /TN "Hermes_Gateway_Watchdog" /TR "powershell -ExecutionPolicy Bypass -WindowStyle Hidden -File 'C:\\Users\\...\\AppData\\Local\\hermes\\scripts\\gateway-watchdog.ps1'" /SC MINUTE /MO 5 /F
```
This is the verified pattern as of June 2026. The script logs to `watchdog.log` so you can audit restarts.

**Hermes cron fallback (read-only alert):**
If you still want a Hermes cron entry, make it read-only:
```bash
hermes cron create --name "gateway-watchdog-check" \
  --deliver telegram \
  "*/5 * * * *" \
  "Check gateway status and Telegram connection. If down, report only. Do not restart."
- [references/windows-restore-findings.md](references/windows-restore-findings.md) — discovered backup layout notes and the shell-boundary pitfall in this environment
- [references/rdp-tailnet-setup.md](references/rdp-tailnet-setup.md) — enabling RDP on this machine and connecting from another Win11 machine via TailScale
- [references/voice-tts-stt-local.md](references/voice-tts-stt-local.md) — voice record key (Ctrl+B), STT provider=local (no cloud), auto_tts settings, and local TTS options (piper/neutts) for zero-cost CPU-friendly speech

## External auth checklist

| Service | Recovery step |
|---|---|
| GitHub (gh CLI) | `gh auth login` — SSH, SeanKingston2020 |
| PAT | Re-enter into Windows Credential Manager or `hermes config set provider.github.token` |
| Nous subscription | `hermes config set provider.nous.api_key` |
| NVIDIA API | `hermes config set provider.nvidia.api_key` **AND** write `NVIDIA_API_KEY=...` into `.env`. The gateway reads the key from `.env`, not `config.yaml`. |
| Telegram | Restore/set `TELEGRAM_BOT_TOKEN`, `TELEGRAM_ALLOWED_USERS`, and `TELEGRAM_HOME_CHANNEL` in `.env`. `hermes config set channels.telegram.bot_token` may fail because nested platform keys do not map cleanly to env var names; inspect or update `.env` directly when needed. |
| TailScale | `tailscale up --operator=$env:USERNAME` |

- **RDP PortNumber registry ghost:** After enabling RDP, the service may still not listen on 3389 if a stale `PortNumber` value exists in the registry. `Get-ItemProperty` returns nothing/empty, `netstat` shows no listener, and RDP clients fail silently. Fix: `Remove-ItemProperty` on the `PortNumber` key under `HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp`, then `Restart-Service TermService`.

## Pitfalls

- **Multi-machine path mismatch:** KingstonAI-Device uses `C:\Users\Sean.Learn\...`, ENVY uses `C:\Users\seank\...`. Never assume the same path works across machines. SOUL.md on Dell is at `C:\Users\Sean.Learn\hermes-agent\docker\SOUL.md`; on ENVY it must go to `C:\Users\seank\hermes-agent\docker\SOUL.md`.
- **Windows path separators:** Python scripts in this folder must use `r'C:\\Users\\...'` raw strings.
- **Python alias:** `python3` may invoke the Microsoft Store stub; use `python` (Python 3.11) on this machine.
- **Shell boundary in this environment:** The `terminal` tool here runs bash/MSYS, so PowerShell-exclusive cmdlets such as `Test-Path`, `Get-ChildItem`, and `Select-Object` will fail with "command not found". Use bash/POSIX equivalents (`ls`, `find`, `head`, `wc`, `sort`, etc.), or explicitly invoke `pwsh -Command "..."` if a PowerShell-specific operation is truly needed.
- **Backup discovery:** Hermes backups live in `$env:USERPROFILE\\backups`; the folder name contains a timestamp `_HHMMSS` (e.g. `hermes_reinstall_20260616_101705`). Search for `*705` or `*backups*` to locate. The user may describe it as "ends in 705" — the number is the `_SS` portion of the timestamp, not a separate identifier. Inspect layout with `ls backups/` before touching `state.db` or `config.yaml`.
- **Copy-Item wildcards:** `cp -r "$BACKUP\\*" $TARGET` fails silently if `$BACKUP\\*` does not expand; verify with `Test-Path` first.
- **Gateway lock files:** Do not delete `gateway.lock` or `gateway.pid` manually while gateway is running; stop via `hermes gateway stop` first.
- **`hermes config set` file lock:** If the gateway is running, `hermes config set` can fail with `PermissionError` (WinError 5, Access Denied) because `config.yaml` is held open. Stop the gateway before changing config, then restart.
- **`hermes config set` for `.env`-mapped keys:** Some nested keys (for example `channels.telegram.bot_token`) do not survive the key-to-env-var conversion and can raise `ValueError` or time out. When `config set` fails, read or edit `.env` directly for secrets and platform env vars, then verify with `grep` on `.env` and `config.yaml`.
- **Skill restore:** Some existing Hermes installs ship with empty skill stubs. Wiping and re-copying from backup avoids empty-folder fallback problems.
- **API key in `.env` vs `config.yaml`:** `hermes config set provider.nvidia.api_key` writes to `config.yaml`, but the gateway runtime reads the key from `.env` as `NVIDIA_API_KEY`. If only `config.yaml` is updated, Telegram/DM sessions will throw `RuntimeError: Provider 'nvidia' is set in config.yaml but no API key was found`. Always verify the key exists in `.env` after setting it.

- [references/multi-machine-migration.md](references/multi-machine-migration.md) — cross-machine migration (Windows → Windows via AnyViewer or SCP), machine naming map (KingstonAI-Device / ENVY), SOUL.md copy path
- [references/soul-md-backup.md](references/soul-md-backup.md) — SOUL.md persona file, git backup workflow, delta install, hourly cron job setup
- [references/gateway-crash-exit-codes.md](references/gateway-crash-exit-codes.md) — STATUS_ENTRYPOINT_NOT_FOUND (0xC0000139) crash diagnosis
- [references/backup-manifest-layout.md](references/backup-manifest-layout.md) — standard backup folder layout, key manifest notes, verified timestamps found on this machine

- [references/dos-window-fix.md](references/dos-window-fix.md) — exact patch snippet for `restart()` in `gateway_windows.py`
- **ENVY CLI restriction:** ENVY's Hermes CLI may be a restricted/chat-only variant with no shell access. If `/exec`, `!`, `shell`, and similar commands are not recognized, use the stash or pull from fork from a different access method (RDP + file explorer). Do not assume shell commands work on ENVY.
- [references/cross-machine-migration.md](references/cross-machine-migration.md) — SSH/SCP transfer of Hermes appdata between TailScale machines (Dell ↔ ENVY)
- [references/state-snapshot-restore.md](references/state-snapshot-restore.md) — destructive-restore checklist and safe command sequences
- [references/windows-restore-findings.md](references/windows-restore-findings.md) — discovered backup layout notes and the shell-boundary pitfall in this environment
- [references/telegram-env-recovery.md](references/telegram-env-recovery.md) — Telegram `.env` keys and fallback when `config set` fails for nested platform paths
- [references/cron-watchdog-fallback.md](references/cron-watchdog-fallback.md) — Hermes cron blocks gateway lifecycle commands; use Task Scheduler or read-only cron instead
- [references/watchdog-script.md](references/watchdog-script.md) — verified PowerShell watchdog script and Task Scheduler registration steps
- [references/nvidia-env-requirement.md](references/nvidia-env-requirement.md) — NVIDIA API key must be present in `.env` as `NVIDIA_API_KEY` for Telegram DM sessions to work
- [references/minimax-gateway-timeout.md](references/minimax-gateway-timeout.md) — MiniMax 2.7 highspeed inactivity timeout bug and applied workaround
