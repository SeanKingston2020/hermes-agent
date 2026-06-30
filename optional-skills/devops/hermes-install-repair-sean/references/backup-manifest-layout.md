# Backup Manifest Reference (June 2026)

## Standard Hermes reinstall backup layout

```
backups/hermes_reinstall_YYYYMMDD_HHMMSS/
├── BACKUP_MANIFEST.txt       ← describes contents + gotchas
├── REINSTALL_STEPS.ps1        ← restore script
├── gateway_windows.py.current ← current gateway_windows.py (DOS fix to re-apply)
├── My_AI_Agent/              ← config repo (hermes-config, hermes-skills, hermes-cron, hermes-memories, notes, scripts)
├── emily_assistant/           ← Emily skill backup
├── gh_config/                 ← GH auth status (PAT in Windows Credential Manager — must re-auth after reinstall)
└── hermes_appdata/
    ├── cron/
    ├── gateway-service/
    ├── memories/
    ├── pairing/
    ├── sessions/
    ├── skills/
    └── state-snapshots/
```

## Key notes from BACKUP_MANIFEST.txt

- GH PAT is in Windows Credential Manager; after reinstall run `gh auth login` (SSH, SeanKingston2020)
- hermes-agent repo may be shallow; DOS fix commit `e2e1b497c` may be lost — re-apply via `Rebuild-Hermes.ps1`
- TailScale may need re-auth if hostname/IP changed
- DOS fix (restart bypasses schtasks /Run, calls _spawn_detached directly) is documented in `gateway_windows.py.current`

## Manifest contents are DESTRUCTIVE on restore

Restoring `hermes_appdata/` overwrites:
- `state.db` (conversation history — sessions after backup date are lost)
- `config.yaml`
- `auth.json`
- All skills, memories, cron

Always do a pre-restore backup of current `AppData/Local/hermes/` critical files before applying.

## Verified backup timestamps found on this machine

- `hermes_reinstall_20260616_101115` — June 16 2026, 10:11:15 AM
- `hermes_reinstall_20260616_101705` — June 16 2026, 10:17:05 AM (most complete)
- `pre-restore-backup-20260621` — June 21 2026, pre-restore snapshot