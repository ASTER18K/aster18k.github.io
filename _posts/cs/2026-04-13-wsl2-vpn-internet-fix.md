---
layout: post
title: "Fixing WSL2 Internet Connectivity with GlobalProtect VPN + SSH Workaround via Cloudflare Tunnel"
category: CS
date: 2026-04-13
tags: [wsl2, vpn, globalprotect, networking, cloudflare-tunnel]
---

## Problem

When I'm on the company GlobalProtect VPN, WSL2 loses internet completely. DNS resolution still works for some reason, but every TCP connection just hangs until it times out.

## Things I tried (didn't work)

### Manual resolv.conf

```bash
# /etc/wsl.conf
[network]
generateResolvConf = false

# /etc/resolv.conf
nameserver 1.1.1.1
```

This only fixes DNS, which wasn't really broken in the first place. The actual problem is routing, so `curl` still hangs forever.

### Mirrored Mode

```ini
# %USERPROFILE%\.wslconfig
[wsl2]
networkingMode=mirrored
dnsTunneling=true
autoProxy=true
```

DNS worked again but TCP still timed out. Turns out GlobalProtect ignores traffic from the Hyper-V virtual adapter, so WSL packets never make it into the VPN tunnel.

I confirmed this by running `curl.exe google.com` on Windows (works) and `curl google.com` inside WSL (times out) at the same time.

## Fix: wsl-vpnkit

[wsl-vpnkit](https://github.com/sakai135/wsl-vpnkit) tunnels WSL2 traffic through the Windows network stack via `gvproxy`, so the packets look like normal host traffic and GlobalProtect handles them properly.

First make sure `.wslconfig` is back to NAT mode (the default):

```ini
[wsl2]
networkingMode=nat
```

If you messed with `wsl.conf` or `resolv.conf` earlier, revert them:

```bash
sudo chattr -i /etc/resolv.conf
sudo rm /etc/resolv.conf
sudo rm /etc/wsl.conf
```

Run `wsl --shutdown` and then install:

```powershell
curl.exe -L -o $env:USERPROFILE\wsl-vpnkit.tar.gz "https://github.com/sakai135/wsl-vpnkit/releases/latest/download/wsl-vpnkit.tar.gz"

wsl --import wsl-vpnkit $env:USERPROFILE\wsl-vpnkit $env:USERPROFILE\wsl-vpnkit.tar.gz --version 2
```

Then run it in a separate terminal and keep that terminal open:

```powershell
wsl.exe -d wsl-vpnkit --cd /app wsl-vpnkit
```

With that running you can open your regular WSL distro in another terminal and everything just works.

### Auto-start

```powershell
$action = New-ScheduledTaskAction -Execute "wsl.exe" -Argument "-d wsl-vpnkit --cd /app wsl-vpnkit"
$trigger = New-ScheduledTaskTrigger -AtLogOn
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -ExecutionTimeLimit 0
Register-ScheduledTask -TaskName "wsl-vpnkit" -Action $action -Trigger $trigger -Settings $settings -Description "WSL VPN connectivity"
```

Keep in mind the task only fires at login, so if you ever run `wsl --shutdown` you'll have to restart wsl-vpnkit manually.

## SSH to external servers via Cloudflare Tunnel

wsl-vpnkit gets internet working inside WSL, but the full-tunnel VPN still blocks unauthorized external SSH, and that's true on the Windows side too.

You can get around this with Cloudflare Tunnel. If your company does SSL Inspection on top of the VPN, there's an extra certificate step to deal with.

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

Then add a CNAME record in Cloudflare DNS pointing to the tunnel.

### Client side (WSL)

```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o cloudflared.deb
sudo dpkg -i cloudflared.deb
```

Add this to `~/.ssh/config`:

```
Host myserver
    HostName ssh.yourdomain.com
    User myuser
    ProxyCommand cloudflared access ssh --hostname %h
```

### SSL Inspection

If your VPN does SSL Inspection you'll hit this error:

```
tls: failed to verify certificate: x509: certificate signed by unknown authority
```

Extract the corporate CA cert and register it on the client:

```bash
echo | openssl s_client -connect ssh.yourdomain.com:443 2>/dev/null | openssl x509 -out /tmp/company-ca.crt
sudo cp /tmp/company-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

After that `ssh myserver` should connect through the tunnel.
