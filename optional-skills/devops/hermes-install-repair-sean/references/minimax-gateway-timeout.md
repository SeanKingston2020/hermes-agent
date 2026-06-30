# MiniMax 2.7 Highspeed — Gateway Inactivity Timeout Bug

## Upstream issue
NousResearch/hermes-agent #6260 — closed in April 2026 via PR #6387.

## Symptom on our setup
- Model: `minimaxai/minimax-m2.7` via NVIDIA `/v1`
- Default `agent.gateway_timeout: 1800` (30 min)
- Default `agent.gateway_timeout_warning: 900` (15 min)
- Gateway logs show inactivity timeouts firing during active streaming sessions.
- Sessions resume only after `/reset` or new session.

## Root cause (upstream analysis)
The inactivity tracker only resets on:
1. Tool call execution
2. API call initiation
3. Receipt of a stream token

Long MiniMax API response times do not always trigger these events, so the timer expires even though the stream is active.

## Applied workaround (June 2026)
```bash
hermes config set agent.gateway_timeout 7200
hermes config set agent.gateway_timeout_warning 1800
```
Then restart the gateway.

If `hermes config set` fails, stop the gateway first — change the file lock behavior under Windows.

## Notes
- The upstream fix (PR #6387) added staged warnings before timeout escalation.
- If upgrading hermes-agent, re-check whether this config override is still needed.
