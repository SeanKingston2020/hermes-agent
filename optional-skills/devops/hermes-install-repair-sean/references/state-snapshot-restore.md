# State snapshot restore guidelines

Backup folders inside `hermes_appdata/state-snapshots/` contain a full snapshot of `state.db`, `config.yaml`, `auth.json`, `gateway.lock`, etc.

Files overwritten:
- `state.db` (sessions / memories)
- `config.yaml`
- `auth.json`
- `gateway.lock`, `gateway.pid`
- `processes.json`
- any other file at the root of `AppData\Local\hermes`

Required warning: State snapshot restore is destructive for anything created after the snapshot date. Always ask the user for confirmation before restoring.

Safe restore sequence:
1. `hermes gateway stop`
2. `cp -r "$SNAP\*" "$env:LOCALAPPDATA\hermes\" -Recurse -Force`
3. `hermes gateway start`

Skills restore (non-destructive):
```powershell
Remove-Item "$env:LOCALAPPDATA\hermes\skills" -Recurse -Force
Copy-Item "$BACKUP\skills\*" "$env:LOCALAPPDATA\hermes\skills" -Recurse -Force
```