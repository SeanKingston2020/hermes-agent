# Hermes Agent — Cross-Machine Migration via TailScale SSH/SCP

## When to use this

Migrating a Hermes instance (skills, state, config, .env, auth) from one machine to another when both are on TailScale VPN.

## Prerequisites

- Both machines on TailScale with reachable IPs
- OpenSSH Server installed and running on TARGET machine
- SSH key from SOURCE machine authorized on TARGET

## Step 1 — Verify connectivity

```bash
# From SOURCE, verify TARGET is reachable
ping <TARGET_TAILSCALE_IP>
```

## Step 2 — Enable OpenSSH Server on TARGET (ENVY)

On TARGET (ENVY), open PowerShell as Admin:

```powershell
# Check if SSH server is installed
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'

# Install if missing:
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# Start and enable SSH server:
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

## Step 3 — Authorize SOURCE SSH Key on TARGET

On TARGET, add SOURCE's public key to `C:\Users\<user>\.ssh\authorized_keys`:

```powershell
# On TARGET, as admin:
# Create .ssh dir if missing
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.ssh"

# Add SOURCE machine's key (from SOURCE: cat ~/.ssh/id_ed25519.pub or id_rsa.pub)
# Example key for this setup:
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIL7E838TiEvcYiqUF54HNM7U6Btvtp0o86vjcNg1RHEQ sean.learn@outlook.com
```

Append the key to `C:\Users\<user>\.ssh\authorized_keys`.

## Step 4 — Test SSH

```bash
ssh <user>@<TARGET_TAILSCALE_IP> "hostname"
```

If it times out or fails, SSH server is not running on TARGET.

## Step 5 — Transfer Hermes AppData

From SOURCE machine, using SSH/SCP over TailScale:

**ENVY target user is `seank` — NOT `Sean`:**

```bash
# ENVY target
scp -r "$HOME/AppData/Local/hermes/skills" "seank@100.102.52.61:/c/Users/seank/AppData/Local/hermes/"
scp "$HOME/AppData/Local/hermes/state.db" "seank@100.102.52.61:/c/Users/seank/AppData/Local/hermes/"
scp "$HOME/AppData/Local/hermes/config.yaml" "seank@100.102.52.61:/c/Users/seank/AppData/Local/hermes/"
scp "$HOME/AppData/Local/hermes/.env" "seank@100.102.52.61:/c/Users/seank/AppData/Local/hermes/"
```

## Notes

- SOUL.md may not exist at `~/SOUL.md` — check inside `~/AppData/Local/hermes/` or skip if missing
- After transfer, on ENVY: `hermes gateway restart` to reload config
- API keys in `.env` must be present on ENVY for Telegram DM sessions to work
- If `scp` hangs, SSH daemon is not running on target — verify step 2