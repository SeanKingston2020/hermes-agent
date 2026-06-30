# Fork Sync — Update Cloned Fork from Installed Copy

## When this happens

After a Hermes update, the installed copy in `AppData\Local\hermes\hermes-agent` is newer than the cloned fork at `~\hermes-agent`. The installer updates AppData; it does NOT update the fork.

Symptom: `hermes --version` and `hermes gateway status` work (they use AppData), but the fork is behind.

## Sync steps

**1. Find the installed version's commit SHA:**
```bash
cd ~/AppData/Local/hermes/hermes-agent
git log --oneline -1   # e.g. 5937b9519
```

**2. Reset the fork to that commit:**
```bash
cd ~/hermes-agent
git remote set-url origin https://github.com/NousResearch/hermes-agent.git  # ensure correct remote
git reset --hard <SHA>
# e.g. git reset --hard 5937b9519
```

**3. Push to your GitHub fork:**
```bash
git push origin main
```
If push times out (large diff, SSH link), run in background:
```bash
git push origin main 2>&1 &
# or use terminal tool with background=true + notify_on_complete=true
```

## Push timeout workaround

SSH push hangs on large diffs. Options:
- Background it: `terminal(background=true, notify_on_complete=true, timeout=600)`
- Switch to HTTPS: `git remote set-url origin https://github.com/<user>/<repo>.git`
- Verify auth works: `ssh -T git@github.com` — if it returns "successfully authenticated", SSH is fine, the diff is just large

## Verification
After push, your fork on GitHub should show the same commit SHA as the installed copy.

## Gateway state note

A running gateway (shown via `hermes gateway status`) uses the AppData copy. The fork is for development/GitHub sync — keeping them in sync is cosmetic/integrity, not a gateway availability requirement.