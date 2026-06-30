# DOS-window fix for `gateway_windows.py → restart()`

Symptom: `hermes gateway restart` (or `start()`) flashes a cmd.exe / DOS window.

Root cause: `restart()` calls `start()` after `stop()`. `start()` uses `schtasks /Run` to spawn the gateway, and `schtasks` always opens a transient console window.

Patch location: `%LOCALAPPDATA%\hermes\hermes-agent\hermes_cli\gateway_windows.py`

Fixed implementation:
```python
def restart() -> None:
    """Stop the gateway then start it again."""
    _assert_windows()
    stop()
    # Give Windows a moment to release the listening port.
    time.sleep(1.0)
    # Don't call start() — schtasks /Run flashes a DOS window. Go direct.
    running_pids = _gateway_pids()
    if running_pids:
        print(f"✓ Gateway already running (PID: {', '.join(map(str, running_pids))})")
    else:
        pid = _spawn_detached()
        _report_gateway_start(f"direct spawn (PID {pid})")
```

Tip: If the `patch` tool struggles with the multi-line string, use Python:
`python -c "import pathlib, textwrap; ..."` with `textwrap.dedent` for
reliable old/new string matching.