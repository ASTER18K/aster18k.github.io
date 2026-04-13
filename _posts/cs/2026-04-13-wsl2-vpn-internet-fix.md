---
title: "Fixing WSL2 Internet Connectivity with GlobalProtect VPN + SSH Workaround via Cloudflare Tunnel"
date: 2026-04-13
tags: [wsl2, vpn, globalprotect, networking, cloudflare-tunnel]
---

## Problem

WSL2 + GlobalProtect VPN ⇒ no internet inside WSL. DNS works, but TCP connections just time out.

## Things I tried (didn't work)

### Manual resolv.conf

```bash
# /etc/wsl.conf
[network]
generateResolvConf = false

# /etc/resolv.conf
nameserver 1.1.1.1
```

Only fixes DNS. The actual problem is routing, so `curl` still hangs.

### Mirrored Mode

```ini
# %USERPROFILE%\.wslconfig
[wsl2]
networkingMode=mirrored
dnsTunneling=true
autoProxy=true
```

DNS worked, TCP still timed out. GlobalProtect ignores traffic from the Hyper-V virtual adapter, so WSL packets never enter the VPN tunnel.

`curl.exe` on Windows: works. `curl` in WSL: timeout. That confirmed it.

## Fix: wsl-vpnkit

[wsl-vpnkit](https://github.com/sakai135/wsl-vpnkit) tunnels WSL2 traffic through the Windows network stack via `gvproxy`. GlobalProtect handles it properly since it looks like normal host traffic.

Make sure `.wslconfig` is NAT mode (default):

```ini
[wsl2]
networkingMode=nat
```

If you messed with `wsl.conf` / `resolv.conf` earlier, revert them:

```bash
sudo chattr -i /etc/resolv.conf
sudo rm /etc/resolv.conf
sudo rm /etc/wsl.conf
```

`wsl --shutdown`, then install:

```powershell
curl.exe -L -o $env:USERPROFILE\wsl-vpnkit.tar.gz "https://github.com/sakai135/wsl-vpnkit/releases/latest/download/wsl-vpnkit.tar.gz"

wsl --import wsl-vpnkit $env:USERPROFILE\wsl-vpnkit $env:USERPROFILE\wsl-vpnkit.tar.gz --version 2
```

Run in a separate terminal:

```powershell
wsl.exe -d wsl-vpnkit --cd /app wsl-vpnkit
```

Keep it running, open WSL in another terminal. Done.

### Auto-start

```powershell
$action = New-ScheduledTaskAction -Execute "wsl.exe" -Argument "-d wsl-vpnkit --cd /app wsl-vpnkit"
$trigger = New-ScheduledTaskTrigger -AtLogOn
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -ExecutionTimeLimit 0
Register-ScheduledTask -TaskName "wsl-vpnkit" -Action $action -Trigger $trigger -Settings $settings -Description "WSL VPN connectivity"
```

After `wsl --shutdown`, you need to restart wsl-vpnkit manually since the task only triggers at login.

## SSH to external servers via Cloudflare Tunnel

wsl-vpnkit gets internet working, but full-tunnel VPN still blocks unauthorized external SSH. Same on Windows side.

Cloudflare Tunnel can get around this. If your company does SSL Inspection, you'll need to deal with the certificate issue too.

### Server side

Install `cloudflared` on the server you want to SSH into:

```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o cloudflared.deb
sudo dpkg -i cloudflared.deb

cloudflared tunnel login
cloudflared tunnel create my-ssh

cat > ~/.cloudflared/config.yml << 'EOF'
tunnel: my-ssh
credentials-file: /root/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: ssh.yourdomain.com
    service: ssh://localhost:22
  - service: http_status:404
EOF

cloudflared tunnel run my-ssh
```

Add a CNAME record in Cloudflare DNS pointing to the tunnel.

### Client side (WSL)

```bash
sudo dpkg -i cloudflared.deb
```

`~/.ssh/config`:

```
Host myserver
    HostName ssh.yourdomain.com
    User myuser
    ProxyCommand cloudflared access ssh --hostname %h
```

### SSL Inspection

If your VPN does SSL Inspection, you'll get:

```
tls: failed to verify certificate: x509: certificate signed by unknown authority
```

Extract the corporate CA cert and register it:

```bash
echo | openssl s_client -connect ssh.yourdomain.com:443 2>/dev/null | openssl x509 -out /tmp/company-ca.crt
sudo cp /tmp/company-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

Then `ssh myserver` should work.
