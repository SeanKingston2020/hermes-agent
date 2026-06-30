# RDP via TailScale — This Machine (Win 10) as RDP Host

## Machine Info (this host)
- Hostname: `kingstonai-device-1` (Dell Inspiron 1750, Win 10)
- TailScale IP: `100.112.121.41`
- Faster Win11 machine (KINGSTON-ENVY): `100.102.52.61` — online and connected to same TailScale network

## Enable RDP (full steps)

```powershell
# 1. Enable RDP
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' -Name 'fDenyTSConnections' -Value 0 -Force

# 2. Allow RDP through firewall
Enable-NetFirewallRule -DisplayGroup 'Remote Desktop'

# 3. Enable RDP to get help (allows RDC to connect)
Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' -Name 'fAllowToGetHelp' -Value 1 -Force
```

## Add user to Remote Desktop Users

```powershell
Add-LocalGroupMember -Group 'Remote Desktop Users' -Member 'Sean.Learn'
```
This is required or the user will be rejected at login even when RDP is otherwise working.

## Known issue: RDP service running but NOT listening on 3389

Symptom: `TermService` shows Running, firewall rules are enabled, `fDenyTSConnections=0`, but `netstat` shows nothing on port 3389 and RDP connections fail.

### Diagnosis steps

1. Check if PortNumber registry key is present (non-default blocks listening):
   ```powershell
   Get-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name 'PortNumber'
   ```
   - If it returns nothing or errors, the key may be stale/missing — remove it and let Windows rebuild the default.

2. Remove PortNumber to force default 3389:
   ```powershell
   Remove-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name 'PortNumber'
   ```

3. Restart the service:
   ```powershell
   Restart-Service TermService
   ```

4. Verify:
   ```powershell
   netstat -ano | Select-String '3389'
   # Should show: TCP 0.0.0.0:3389 LISTENING
   ```

## CRITICAL: Machine restart required after enabling RDP

Even when all registry settings are correct, the RDP listener may still not bind to port 3389 on a freshly configured Windows 10 machine. This is a known Windows issue — registry changes for RDP don't fully take effect until a **full restart** of the machine.

Symptoms before restart:
- `Get-NetTCPConnection -State Listen` shows no binding on port 3389
- `Test-NetConnection localhost -Port 3389` returns `TcpTestSucceeded: False`
- RDP clients immediately fail to connect

**Fix:** Restart the machine:
```powershell
Restart-Computer
# or
shutdown /r /t 30
```

After restart, verify with `netstat -ano | Select-String '3389'` — should show `TCP 0.0.0.0:3389 LISTENING`.

## Windows Home limitation

Windows **Home** editions cannot host RDP sessions (no RDP server). They can only *connect* to RDP hosts.

Workarounds: upgrade to Win11 Pro, or use an alternative like VNC / Parsec / RustDesk.

## Connect from Win11

1. Remote Desktop Connection → Computer: `100.112.121.41`
2. Username: `Sean.Learn` (Windows account on this machine)
3. Authenticate with Windows password

## Notes

- Works over TailScale VPN — no port forwarding or public IP needed
- Both machines must be on the same TailScale tailnet
- Inference always runs on this machine's CPU — RDP only changes which keyboard/screen you use