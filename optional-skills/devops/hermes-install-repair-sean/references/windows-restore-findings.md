# Windows Restore Findings

## Session context
This note captures the recovery pattern used in the June 16 backup restore session so future Windows reinstall/repair sessions can skip rediscovery.

## Verified locations
- `$env:USERPROFILE\\backups`
- Example restore source: `backups\\hermes_reinstall_20260616_101705\\hermes_appdata\\`
- Additional source: `backups\\...\\My_AI_Agent\\` (config repo clone)
- State snapshot source: `backups\\...\\hermes_appdata\\state-snapshots\\20260612-182646-pre-update\\`
- Live skills target: `$env:LOCALAPPDATA\\hermes\\skills`

## Shell boundary pitfall
The terminal tool here executes bash/MSYS. PowerShell-only cmdlets such as `Test-Path`, `Get-ChildItem`, and `Select-Object` fail. Use:
- Bash equivalents for inspection: `find`, `ls`, `head`, `wc`, `sort`
- Explicit PowerShell invocation when needed: `pwsh -Command "..."`

## Restore sequence
1. Stop gateway and inspect backups: `hermes gateway stop`
2. Pre-restore safety backup: copy `state.db`, `config.yaml`, `auth.json` to a safe dir
3. Full appdata restore: copy from `backups\\...\\hermes_appdata\\*` into `$env:LOCALAPPDATA\\hermes\\`
4. Verify DOS-window fix in `gateway_windows.py`
5. Start gateway: `hermes gateway start`

## Notes
- Avoid `Test-Path` for validation checks in this environment.
- Confirm `.md` sources before copying into a live environment.
- Pre-restore safety backup prevents losing recent sessions if `state.db` will be overwritten.
