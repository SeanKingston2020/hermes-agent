# Gateway Crash Diagnostics — Exit Codes & Watchdog Behavior

## Observed exit codes (June 27, 2026)

| Exit code | Hex | Name | Meaning |
|-----------|-----|------|---------|
| `-1073741510` | `0xC0000139` | `STATUS_ENTRYPOINT_NOT_FOUND` | A DLL or exported function was not found at load time. The gateway process started but crashed during startup initialization. |

## What this means for the watchdog

The watchdog scheduled task (`Hermes_Gateway_Watchdog`) runs every 5 minutes and checks `hermes gateway status`. If the process is down, it attempts `hermes gateway restart`.

However:
- If the crash is a **permanent startup failure** (broken venv, Python version mismatch, corrupted DLL), `hermes gateway start` will succeed but the process immediately crashes again.
- The watchdog log (`watchdog.log`) will show repeated restart attempts.
- The scheduled task last-run result will keep showing `0xC0000139`.

## Investigation steps

1. Check the gateway log for Python errors at startup.
2. Verify Python version: `python --version` should match what the venv was built with.
3. Try starting manually: `hermes gateway start` then check status immediately.
4. Check `schtasks /Query /TN "Hermes_Gateway"` for the task record.
5. Check `watchdog.log` for restart frequency — if restarts are happening every 5 minutes, the process keeps dying.

## Common causes of 0xC0000139

- Python version changed (system Python upgraded, venv built with a different version)
- aiohttp or other native-extension package failed to load
- venv corruption