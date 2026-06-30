# Cross-Machine Hermes Migration (Windows)

## When to use this

Migrating Hermes from one Windows machine to another — or syncing state between two machines — when SSH is not available or impractical.

## Prerequisites

- Both machines on the same network (or TailScale VPN)
- File transfer capability (AnyViewer, network share, USB stick)
- Target machine already has Hermes installed (clean or existing)

## Migration Sequence

### 1. Source machine: identify files to copy

```bash
# Critical migration files
ls $HOME/AppData/Local/hermes/state.db
ls $HOME/AppData/Local/hermes/config.yaml
ls $HOME/AppData/Local/hermes/auth.json
ls $HOME/AppData/Local/hermes/.env
ls -d $HOME/AppData/Local/hermes/skills/*/

# SOUL.md (if custom persona exists)
ls $HOME/hermes-agent/docker/SOUL.md
```

### 2. Source path reference

```
C:\Users\<source_user>\AppData\Local\hermes\
C:\Users\<source_user>\hermes-agent\docker\SOUL.md
```

### 3. Target path reference

```
C:\Users\<target_user>\AppData\Local\hermes\
C:\Users\<target_user>\hermes-agent\docker\SOUL.md
```

### 4. File transfer (AnyViewer pattern)

1. Connect source machine via AnyViewer (or preferred remote desktop)
2. Open **File Transfer** in AnyViewer
3. Navigate to source: `C:\Users\<source_user>\AppData\Local\hermes\`
4. Copy to target: `C:\Users\<target_user>\AppData\Local\hermes\`
5. Overwrite existing files when prompted
6. Copy `SOUL.md` to `C:\Users\<target_user>\hermes-agent\docker\`

### 5. Verify on target machine

```bash
hermes --version
hermes gateway status
dir C:\Users\<target_user>\AppData\Local\hermes\
```

### 6. Restart gateway to pick up state

```bash
hermes gateway stop
hermes gateway start
hermes gateway status
```

## SOUL.md Note

SOUL.md lives at `hermes-agent/docker/SOUL.md` — NOT in AppData. If the source machine has a custom persona, it must be copied separately from the AppData migration.

Default Hermes persona = no SOUL.md needed on target (defaults to Hermes if missing).

## Source vs Target Path Reference

| Machine | User | Full AppData path |
|---|---|---|
| KingstonAI-Device (Dell) | Sean.Learn | `C:\Users\Sean.Learn\AppData\Local\hermes` |
| ENVY (Win 11) | seank | `C:\Users\seank\AppData\Local\hermes` |

**Always use the correct user folder — never `Sean.Learn` when targeting ENVY and vice versa.**

## SSH Alternative

If OpenSSH Server is installed on target machine, SCP works over TailScale:

```bash
# From source machine (Dell / KingstonAI-Device)
# ENVY target: user=seank, path uses "seank" NOT "Sean.Learn"
scp -r ~/AppData/Local/hermes/* "seank@100.102.52.61:/c/Users/seank/AppData/Local/hermes/"
scp ~/hermes-agent/docker/SOUL.md "seank@100.102.52.61:/c/Users/seank/hermes-agent/docker/"
```

OpenSSH Server install (target machine, PowerShell as Admin):
```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

## Machine Naming (Sean's setup)

| Machine | OS | TailScale IP | Notes |
|---|---|---|---|
| KingstonAI-Device | Win 10 (Dell) | 100.112.121.41 | Primary agent (Kinger) |
| ENVY | Win 11 | 100.102.52.61 | Secondary / faster machine |

Both run independent Hermes instances with separate state, skills, and memory.