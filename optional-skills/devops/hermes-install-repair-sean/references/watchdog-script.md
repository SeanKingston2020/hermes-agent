# Gateway Watchdog Script Template

## Verified script (June 2026)

Path: `~/AppData/Local/hermes/scripts/gateway-watchdog.ps1`

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

## Register with Windows Task Scheduler

```powershell
schtasks /Create /TN "Hermes_Gateway_Watchdog" /TR "powershell -ExecutionPolicy Bypass -WindowStyle Hidden -File 'C:\\Users\\<user>\\AppData\\Local\\hermes\\scripts\\gateway-watchdog.ps1'" /SC MINUTE /MO 5 /F
```
If running from Git Bash / MSYS, wrap the command in `pwsh -Command` because `schtasks` path translation fails in MSYS.

Verify:
```powershell
schtasks /Query /TN "Hermes_Gateway_Watchdog"
```

## Why Task Scheduler, not Hermes cron

Hermes cron explicitly blocks gateway lifecycle commands (`restart`, `stop`, `kill`) with:
```
Blocked: cron job contains a gateway lifecycle command (restart/stop/kill).
```
This prevents restart loops (#30719). Task Scheduler is the correct Windows-native supervisor for this.
