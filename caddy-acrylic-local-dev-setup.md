# Caddy + Acrylic DNS — Local Dev Setup Guide

## What is this?

A full guide on running **Caddy** as a local reverse proxy with **Acrylic DNS Proxy** so you never have to edit `hosts` file again.

Used for: running multiple local websites with custom domain names (`.local`, `.dev`, `.test`, `.lan`, `.gay`) on different ports, all accessible via nice URLs like `http://irfan.local` or `http://myapp.local`.

---

## What's installed already?

| Software | Status |
|---|---|
| **Caddy v2.11.4** | Installed at `C:\Users\mihf0\AppData\Local\Microsoft\WindowsApps\caddy.exe` |
| **Acrylic DNS Proxy** | Installed at `C:\Program Files (x86)\Acrylic DNS Proxy\` — Service is **Running** |
| **Windows DNS** | Ethernet adapter DNS set to `127.0.0.1` (points to Acrylic) |

---

## How it all works (big picture)

```
Browser: http://irfan.local
        ↓
Windows asks DNS → Acrylic (127.0.0.1) → "irfan.local = 127.0.0.1"
        ↓
Browser connects to 127.0.0.1:80
        ↓
Caddy sees Host header "irfan.local" → proxies to localhost:3000
```

Acrylic handles **DNS resolution** (name → IP).

Caddy handles **port routing** (domain → port).

---

## Part 1: Acrylic DNS Setup (Done)

AcrylicHosts.txt is at:

```
C:\Program Files (x86)\Acrylic DNS Proxy\AcrylicHosts.txt
```

### Current entries added:

```
127.0.0.1 >teseraos.local >teseraos.lan >teseraos.dev >teseraos.gay >teseraos.test
127.0.0.1 irfan.local
```

The `>` means "this domain AND all subdomains". So `>teseraos.local` covers:

- teseraos.local
- api.teseraos.local
- backend.teseraos.local
- anything.teseraos.local

### How to add more domains later:

Open **AcrylicUI.exe** (GUI) from:

```
C:\Program Files (x86)\Acrylic DNS Proxy\AcrylicUI.exe
```

Or edit `AcrylicHosts.txt` directly (as admin), then restart service:

```powershell
Restart-Service AcrylicDNSProxySvc
```

### Pros of Acrylic DNS

| Pro | Con |
|---|---|
| Never touch hosts file again | Need to restart service after edit |
| GUI available (AcrylicUI.exe) | One extra software to install |
| Supports wildcards (`*`, `>`) | Uses port 53 locally |
| Caches DNS for speed | |
| Works system-wide (all browsers, tools) | |

### Alternative: Windows hosts file (no extra software)

Edit `C:\Windows\System32\drivers\etc\hosts` as admin:

```
127.0.0.1 irfan.local
127.0.0.1 teseraos.local
127.0.0.1 api.teseraos.local
```

**Problem:** No wildcard support. Every subdomain needs a separate line. No GUI. Need admin each time.

---

## Part 2: Caddy Setup

### What is Caddy?

A web server that:
- Reverse proxies (domain → localhost:port)
- Auto HTTPS (not needed for local dev, use `tls internal` instead)
- Reads a simple config file called **Caddyfile**

### Basic Caddyfile structure

Create a file named `Caddyfile` (no extension) anywhere:

```
mydomain.local {
    reverse_proxy localhost:3000
}
```

### How to run Caddy

**From same folder as Caddyfile:**
```powershell
cd D:\projects\myproject
caddy run
```

**From anywhere with explicit config path:**
```powershell
caddy run --config "D:\projects\myproject\Caddyfile"
```

**Stop Caddy:**
```powershell
caddy stop
```

### Full example Caddyfile

```
# Admin API (optional, for monitoring)
{
    admin 0.0.0.0:22020
}

# Backend API on port 13001
backend.teseraos.local, api.teseraos.local,
backend.teseraos.lan, api.teseraos.lan,
backend.teseraos.dev, api.teseraos.dev,
backend.teseraos.gay, api.teseraos.gay,
backend.teseraos.test, api.teseraos.test {
    reverse_proxy localhost:13001
    tls internal
}

# Main frontend on port 13000
teseraos.local, *.teseraos.local,
teseraos.lan, *.teseraos.lan,
teseraos.dev, *.teseraos.dev,
teseraos.gay, *.teseraos.gay,
teseraos.test, *.teseraos.test {
    reverse_proxy localhost:13000
    tls internal
}

# Your custom domain on port 3000
irfan.local {
    reverse_proxy localhost:3000
}
```

### What `tls internal` does

Caddy normally tries to get SSL certs from Let's Encrypt (internet). Since `.local` and friends are local-only, `tls internal` tells Caddy to generate a self-signed cert locally. Your browser will show a warning — click "Advanced → Proceed" or add the cert once.

### Pros of Caddy

| Pro | Con |
|---|---|
| Super simple config syntax | Self-signed cert warning in browser |
| Auto HTTPS (for real domains) | No built-in GUI |
| Built-in reverse proxy | |
| Auto-restart on config change (with `--watch`) | |
| Single binary, no dependencies | |

### Alternative: nginx

```nginx
server {
    listen 80;
    server_name irfan.local;
    location / {
        proxy_pass http://localhost:3000;
    }
}
```

**Problem:** nginx config is more complex. No auto HTTPS. No auto-reload.

---

## Part 3: Quick reference — all commands

### Acrylic DNS

```powershell
# Restart Acrylic (after editing hosts)
Restart-Service AcrylicDNSProxySvc

# Start
Start-Service AcrylicDNSProxySvc

# Stop
Stop-Service AcrylicDNSProxySvc

# Check status
Get-Service AcrylicDNSProxySvc

# Open GUI
& "C:\Program Files (x86)\Acrylic DNS Proxy\AcrylicUI.exe"
```

### Caddy

```powershell
# Run with Caddyfile in current folder
caddy run

# Run with custom Caddyfile path
caddy run --config "D:\path\to\Caddyfile"

# Run in background as a service (Windows)
caddy run  # keep terminal open, or use:

# Stop Caddy
caddy stop

# Check version
caddy version

# Serve static files (no Caddyfile needed)
caddy file-server --listen :8080

# Simple reverse proxy (no Caddyfile needed)
caddy reverse-proxy --from irfan.local --to localhost:3000
```

### DNS test commands

```powershell
# Check resolution
Resolve-DnsName irfan.local
Resolve-DnsName teseraos.local
Resolve-DnsName api.teseraos.local

# Or use nslookup
nslookup irfan.local 127.0.0.1
```

---

## Part 4: Adding a new project step by step

Say you have a React app on port 5173:

1. **Add DNS entry** (if not already covered by wildcard):

   Edit `AcrylicHosts.txt` as admin:
   ```
   127.0.0.1 myapp.local
   ```

   Then:
   ```powershell
   Restart-Service AcrylicDNSProxySvc
   ```

2. **Add Caddy entry** in your Caddyfile:

   ```
   myapp.local {
       reverse_proxy localhost:5173
   }
   ```

3. **Run Caddy** (or restart if already running):

   ```powershell
   caddy run
   ```

4. Open `http://myapp.local` in browser.

---

## Part 5: Troubleshooting

### "Site can't be reached" / DNS not working

```powershell
# Check Acrylic is running
Get-Service AcrylicDNSProxySvc

# Check DNS is set to 127.0.0.1
Get-DnsClientServerAddress -AddressFamily IPv4

# Test resolution
Resolve-DnsName irfan.local
```

### Caddy not proxying

```powershell
# Check if Caddy is running and listening
netstat -ano | findstr :80
netstat -ano | findstr :443

# Run Caddy in foreground to see logs
caddy run
```

### Port 80 is already in use (by IIS, Skype, etc.)

Change Caddy's HTTP port in Caddyfile:

```
{
    http_port 8080
    https_port 8443
}

irfan.local {
    reverse_proxy localhost:3000
}
```

Then access: `http://irfan.local:8080`

### "tls: internal" gives cert warning in browser

This is normal for local dev. Click "Advanced" → "Proceed to ..." in Chrome/Edge. Or use HTTP only (remove `tls internal` line) — Caddy will serve on port 80.

---

## Summary

```
┌─────────────┐     ┌──────────────┐     ┌───────┐     ┌──────────┐
│ Browser     │ ──▶ │ Acrylic DNS  │ ──▶ │ Caddy │ ──▶ │ Your App │
│ irfan.local │     │ → 127.0.0.1  │     │ → :80 │     │ :3000    │
└─────────────┘     └──────────────┘     └───────┘     └──────────┘
```

**One-time setup done ✅**
- Acrylic running ✅
- DNS pointing to Acrylic ✅
- Domains resolving to 127.0.0.1 ✅

**You just need to:**
1. Create/write a Caddyfile
2. Run `caddy run`
3. Done
