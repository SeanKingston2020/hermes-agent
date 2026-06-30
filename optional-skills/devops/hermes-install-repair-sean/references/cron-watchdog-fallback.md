# Gateway Watchdog Cron Fallback

## Problem
Hermes cron explicitly blocks gateway lifecycle commands (`restart`, `stop`, `kill`) to prevent restart loops (#30719). Using `hermes cron create --prompt "... gateway restart ..."` returns:

```
Blocked: cron job contains a gateway lifecycle command (restart/stop/kill).
```

## Verified workaround
Use Windows Task Scheduler or an external script to restart the gateway when needed.

### Task Scheduler pattern
1. Create a PowerShell / bash script at `~/.hermes/scripts/watchdog.ps1` (or `.sh`) that:
   - Checks `gateway.pid` for a valid process
   - Checks `gateway.log` for recent heartbeat
   - If broken: runs `hermes gateway restart`
   - Optionally sends a Telegram notification
2. Schedule it in Windows Task Scheduler to run every 5 minutes.

### Hermes cron checkout-only pattern
If you still want a Hermes cron entry, make it read-only:

```bash
hermes cron create --name "gateway-watchdog-check" \
  "*/5 * * * *" \
  "Check gateway status and Telegram connection. If down, report only. Do not restart."
```

The Hermes cron can thus alert without tripping the restart loop protection.
