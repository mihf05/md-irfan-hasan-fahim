# Docker to Podman: Complete Migration Guide
---

## Table of Contents
1. [Docker vs Podman](#1-docker-vs-podman)
2. [Why Switch?](#2-why-switch)
3. [Installation](#3-installation)
4. [Basic Commands](#4-basic-commands)
5. [Podman Machine Management](#5-podman-machine-management)
6. [Docker vs Podman Command Mapping](#6-docker-vs-podman-command-mapping)
7. [Working with Containers & Images](#7-working-with-containers--images)
8. [Docker Compose with Podman](#8-docker-compose-with-podman)
9. [Port Mapping & Networking](#9-port-mapping--networking)
10. [Volume & Data Management](#10-volume--data-management)
11. [Known Issues & Workarounds](#11-known-issues--workarounds)
12. [Resource Usage Comparison](#12-resource-usage-comparison)
13. [Useful Tips](#13-useful-tips)
14. [Appendix: All Commands Reference](#14-appendix-all-commands-reference)

---

## 1. Docker vs Podman

| Feature | Docker Desktop | Podman |
|---|---|---|
| **Architecture** | Client-Server (daemon always running) | Daemonless (fork/exec model) |
| **Background Service** | `dockerd` always runs (~2-4 GB RAM) | No background daemon |
| **Startup** | Auto-starts with Windows | Manual (`podman machine start`) |
| **Memory Usage** | 2-4 GB fixed | ~500 MB when running, 0 when stopped |
| **CLI Syntax** | `docker <command>` | `podman <command>` (99% same) |
| **Docker Compatibility** | Native | Built-in (`podman` can alias as `docker`) |
| **Rootless Mode** | Limited | Default (more secure) |
| **Pod Support** | No native | Built-in (`podman pod`) |
| **License** | Proprietary (free for small teams) | Open Source (Apache 2.0) |
| **VM Technology** | Hyper-V / WSL 2 | WSL 2 |
| **GUI** | Built-in Dashboard | Optional (Podman Desktop) |

### Pros & Cons

**Docker Desktop Pros:**
- GUI dashboard for containers/images
- One-click setup
- Docker Compose built-in
- Larger ecosystem / community
- Kubernetes integration built-in

**Docker Desktop Cons:**
- Always runs in background — eats 2-4 GB RAM even when idle
- Heavy installation (~1 GB+)
- License restrictions for large enterprises
- Slower startup
- WSL 2 disk space usage grows over time

**Podman Pros:**
- Daemonless — no background service, zero memory when not in use
- Lightweight (~200 MB install)
- Fully open source
- Rootless by default (more secure)
- Built-in pod support (Kubernetes-like)
- Drop-in Docker replacement (same CLI)
- Faster container startup
- Can map to Docker socket for compatibility

**Podman Cons:**
- No official GUI (Podman Desktop available but optional)
- Docker Compose support requires extra step (`podman-compose` or `podman compose`)
- Slightly different networking model
- Smaller ecosystem
- Windows support newer than Docker Desktop

---

## 2. Why Switch?

**Main reason:** Docker Desktop consumes 2-4 GB RAM continuously — even when you're not running any container. Podman uses ~500 MB when a container is running, and **0 MB when stopped**.

Your system will feel noticeably faster after switching.

---

## 3. Installation

### 3.1 Uninstall Docker Desktop

```powershell
# Via Settings → Apps → Docker Desktop → Uninstall
# OR via command:
& "C:\Program Files\Docker\Docker\Docker Desktop Installer.exe" "uninstall"
```

### 3.2 Install Podman

```powershell
winget install redhat.podman
```

This installs Podman to `C:\Program Files\RedHat\Podman\`.

### 3.3 Initialize & Start Podman Machine

```powershell
# Initialize VM (one time only)
podman machine init

# Start the VM (do this when you need containers)
podman machine start
```

### 3.4 Set Up Docker CLI Alias

**For PowerShell** (adds to profile automatically):

```powershell
# Check profile path
$PROFILE.CurrentUserAllHosts

# Add these lines to your profile:
$env:Path += ";C:\Program Files\RedHat\Podman"
function docker { podman @args }
```

**For cmd.exe** (batch wrapper):

A wrapper `docker.cmd` was created at: `%USERPROFILE%\.local\bin\docker.cmd`

Contents:
```batch
@echo off
podman %*
```

This directory was added to your User PATH. Restart cmd.exe for it to take effect.

---

## 4. Basic Commands

```powershell
# === Machine Management ===
podman machine start          # Start Podman VM
podman machine stop           # Stop Podman VM (frees all memory)
podman machine list           # List all machines
podman machine info           # Show VM details
podman machine restart        # Restart the VM

# === Container Lifecycle ===
podman run hello-world                     # Run a test container
podman run -d --name myapp nginx            # Run nginx in background
podman run -it ubuntu bash                  # Interactive shell
podman ps                                   # List running containers
podman ps -a                                # List all containers
podman stop <container>                     # Stop a container
podman start <container>                    # Start a stopped container
podman restart <container>                  # Restart a container
podman rm <container>                       # Remove a container
podman rm -f <container>                    # Force remove running container

# === Container Logs & Exec ===
podman logs <container>                     # View container logs
podman logs -f <container>                  # Follow logs live
podman exec -it <container> bash            # Shell into running container
podman top <container>                      # Processes inside container

# === Images ===
podman images                               # List all images
podman pull nginx                           # Pull an image
podman rmi <image>                          # Remove an image
podman rmi -a                               # Remove all images
podman build -t myimage .                   # Build image from Dockerfile
podman tag <image> <new-tag>               # Tag an image
podman save -o myimage.tar <image>          # Export image to tar
podman load -i myimage.tar                  # Import image from tar

# === System ===
podman info                                 # Show system info
podman version                              # Show version
podman system df                            # Show disk usage
podman system prune                         # Clean unused data
podman system prune -a                      # Aggressive cleanup
```

---

## 5. Podman Machine Management

```powershell
# --- STARTUP & SHUTDOWN ---
podman machine start        # Start (VM boots in ~5 seconds)
podman machine stop         # Stop (frees ALL memory, zero RAM usage)

# --- STATUS ---
podman machine list         # Show all machines & their status
podman machine info         # Detailed machine info (CPU, RAM, disk)

# --- CONFIGURATION ---
podman machine set --cpus 4 --memory 4096   # Change CPU/RAM allocation
podman machine set --rootful                # Switch to rootful mode (for ports < 1024)
podman machine set --rootless               # Switch back to rootless mode

# --- MAINTENANCE ---
podman machine ssh          # SSH into the VM
podman machine inspect      # Inspect machine configuration
podman machine rm           # Remove a machine
podman machine reset        # Factory reset (removes everything)

# --- SUGGESTED WORKFLOW ---
# When you need containers:
podman machine start && podman run nginx

# When done:
podman machine stop     # Zero memory usage again
```

### Rootless vs Rootful

| Mode | Default | When to use |
|---|---|---|
| **Rootless** (default) | Yes | Normal use, ports > 1024, more secure |
| **Rootful** | Switch with `--rootful` | Need ports < 1024 (e.g., port 80, 443) |

---

## 6. Docker vs Podman Command Mapping

| Docker | Podman | Notes |
|---|---|---|
| `docker run` | `podman run` | Same flags work |
| `docker ps` | `podman ps` | Identical |
| `docker images` | `podman images` | Identical |
| `docker build` | `podman build` | Same |
| `docker pull` | `podman pull` | Same |
| `docker push` | `podman push` | Same |
| `docker exec` | `podman exec` | Same |
| `docker logs` | `podman logs` | Same |
| `docker stop` | `podman stop` | Same |
| `docker rm` | `podman rm` | Same |
| `docker rmi` | `podman rmi` | Same |
| `docker-compose up` | `podman-compose up` | Or `podman compose up` |
| `docker system prune` | `podman system prune` | Same |
| `docker login` | `podman login` | Same |
| `docker network ls` | `podman network ls` | Same |
| `docker volume ls` | `podman volume ls` | Same |
| `docker inspect` | `podman inspect` | Same |

> **99% of `docker` commands work with `podman` without any change.**

---

## 7. Working with Containers & Images

### Running Containers

```powershell
# Web server with port mapping
podman run -d -p 8080:80 --name webserver nginx
# Access: http://localhost:8080

# With environment variables
podman run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=secret --name mysql mysql:8

# Restart policy
podman run -d --restart unless-stopped --name app nginx

# Resource limits
podman run -d --memory=512m --cpus=0.5 --name limited nginx

# Remove automatically when stopped
podman run --rm -it ubuntu bash
```

### Building Images

```powershell
# From Dockerfile in current directory
podman build -t myapp:latest .

# With build args
podman build --build-arg VERSION=1.0 -t myapp:latest .

# From a different directory
podman build -t myapp -f /path/to/Dockerfile /path/to/context
```

### Image Management

```powershell
# Search images
podman search nginx

# Pull specific version
podman pull nginx:alpine

# Show image details
podman inspect nginx

# Show image history
podman history nginx
```

---

## 8. Docker Compose with Podman

### Method 1: Podman's built-in compose (recommended)

```powershell
# Install the compose subcommand (one time)
podman machine ssh
sudo dnf install -y podman-compose
exit

# Then use:
podman compose up -d
podman compose down
podman compose logs -f
podman compose ps
```

> **Note:** This requires the podman-machine VM to be running.

### Method 2: Using docker-compose with DOCKER_HOST

```powershell
# Install docker-compose normally, then:
$env:DOCKER_HOST = "npipe:////./pipe/docker_engine"
docker-compose up -d
```

(Works because Podman forwards the Docker API socket.)

### Sample docker-compose.yml that works with Podman:

```yaml
version: "3.8"
services:
  web:
    image: nginx
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: secret
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

---

## 9. Port Mapping & Networking

### Port Forwarding

```powershell
# Map host port 8080 to container port 80
podman run -d -p 8080:80 nginx

# Map multiple ports
podman run -d -p 8080:80 -p 3000:3000 nginx

# UDP port
podman run -d -p 5353:53/udp --name dns coredns

# Random host port (host picks a free port)
podman run -d -p 80 nginx
```

### Networks

```powershell
# List networks
podman network ls

# Create a custom network
podman network create mynetwork

# Run container on a network
podman run -d --network mynetwork --name app nginx

# Connect running container to network
podman network connect mynetwork <container>

# Disconnect
podman network disconnect mynetwork <container>

# Remove network
podman network rm mynetwork

# Inspect network
podman network inspect mynetwork
```

### Port Mapping Issue (Important!)

Podman in WSL 2 cannot directly access container ports from Windows unless:
- The `podman machine` is running (it handles port forwarding)
- Or you use `localhost` from Windows (not from WSL)

This works:
```
http://localhost:8080    (from Windows browser)
```

This does NOT work (common Docker Desktop confusion):
```
http://192.168.x.x:8080  (WSL IP — Podman doesn't expose this)
```

---

## 10. Volume & Data Management

### Volumes

```powershell
# Create a named volume
podman volume create mydata

# List volumes
podman volume ls

# Inspect volume
podman volume inspect mydata

# Remove volume
podman volume rm mydata

# Remove unused volumes
podman volume prune

# Mount a volume
podman run -d -v mydata:/data --name app nginx
```

### Bind Mounts

```powershell
# Mount host directory into container
podman run -d -v C:\project:/app:Z --name app nginx

# Read-only mount
podman run -d -v C:\config:/etc/config:ro,Z --name app nginx

# Mount with SELinux context
# :Z = shared, :z = private
```

> **Note:** With Podman on WSL 2, the host path is relative to the WSL VM, not Windows directly. Use paths from the WSL perspective, or use named volumes for simpler cross-platform compatibility.

---

## 11. Known Issues & Workarounds

### Issue 1: `docker` command not found in cmd.exe
**Fix:** A `docker.cmd` wrapper was created at `%USERPROFILE%\.local\bin\docker.cmd`. Restart cmd.exe or run:
```cmd
set PATH=%PATH%;%USERPROFILE%\.local\bin
```

### Issue 2: Port < 1024 not accessible (e.g., port 80, 443)
**Fix:** Switch to rootful mode:
```powershell
podman machine set --rootful
podman machine stop
podman machine start
```

### Issue 3: Container can't access the internet
**Fix:** The WSL VM needs DNS. Try:
```powershell
podman machine ssh
sudo sh -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
exit
podman machine stop
podman machine start
```

### Issue 4: Disk space growing
**Fix:** WSL 2 VHDX files can grow large. Compact them:
```powershell
# Inside WSL:
podman system prune -a

# Then from Windows:
wsl --shutdown
diskpart
# select vdisk file="%USERPROFILE%\AppData\Local\WSL\ext4.vhdx"
# compact vdisk
# exit
```

### Issue 5: Docker compose not working
**Fix:** Use `podman compose` instead:
```powershell
podman compose up -d
```
Or set `DOCKER_HOST` environment variable:
```powershell
$env:DOCKER_HOST = "npipe:////./pipe/docker_engine"
docker-compose up -d
```

### Issue 6: Cannot access container from another machine on network
**Fix:** Podman on Windows + WSL doesn't natively expose containers to the LAN. Workaround options:
- Use SSH tunneling
- Use `podman machine ssh -L 8080:localhost:8080`
- Use nginx reverse proxy on Windows

### Issue 7: File permission issues with bind mounts
**Fix:** Files created inside container may have wrong permissions on Windows. Use named volumes instead of bind mounts where possible:
```powershell
podman volume create appdata
podman run -d -v appdata:/app nginx
```

---

## 12. Resource Usage Comparison

| Scenario | Docker Desktop | Podman |
|---|---|---|
| **Idle (no containers)** | 2-4 GB RAM, ~50 processes | 0 MB RAM, 0 processes |
| **1 container running** | 2.5-4.5 GB RAM | ~500 MB RAM |
| **5 containers running** | 3-6 GB RAM | ~1-1.5 GB RAM |
| **CPU overhead** | Moderate | Low |
| **Startup time** | 30-60 seconds | 3-5 seconds (`podman machine start`) |
| **Install size** | ~1.5 GB | ~200 MB |
| **WSL disk usage** | 10-30 GB (grows over time) | ~2-3 GB |

### Memory Saving Tips with Podman

```powershell
# Stop VM when not using containers
podman machine stop

# Start only when needed
podman machine start

# Limit VM resources
podman machine set --memory 2048 --cpus 2

# Auto-stop on idle (manual — no built-in auto-stop)
# Just remember to run: podman machine stop
```

---

## 13. Useful Tips

### Tip 1: One-liner to start and run

```powershell
podman machine start && podman run --rm -p 8080:80 nginx
```

### Tip 2: Shell function for auto-start

Add to PowerShell profile:
```powershell
function drun {
    $running = podman machine list --format "{{.State}}"
    if ($running -ne "Running") { podman machine start }
    podman run @args
}
```

### Tip 3: Clean up everything

```powershell
podman system prune -a --volumes
```

### Tip 4: Reduce Podman machine memory

```powershell
# Reduce to 1 GB RAM
podman machine set --memory 1024
podman machine stop
podman machine start
```

### Tip 5: Use Podman as a direct Docker replacement

Set this environment variable permanently in your system:
```
DOCKER_HOST=npipe:////./pipe/docker_engine
```

This makes any Docker CLI tool (docker-compose, etc.) talk to Podman automatically.

### Tip 6: Check if Podman is running

```powershell
podman machine list
# Look for podman-machine-default → Running
```

### Tip 7: See what's eating disk

```powershell
podman system df
```

### Tip 8: Speed up builds with cache

```powershell
podman build --layers -t myapp .
```

---

## 14. Appendix: All Commands Reference

### Machine Commands

| Command | Description |
|---|---|
| `podman machine init` | Initialize a new VM (first time only) |
| `podman machine start` | Start the VM |
| `podman machine stop` | Stop the VM |
| `podman machine restart` | Restart the VM |
| `podman machine list` | List all machines |
| `podman machine info` | Show machine details |
| `podman machine inspect` | Inspect machine config |
| `podman machine ssh` | SSH into the VM |
| `podman machine set` | Change machine settings |
| `podman machine rm` | Remove a machine |
| `podman machine reset` | Factory reset |
| `podman machine cp` | Copy files to/from VM |

### Container Commands

| Command | Description |
|---|---|
| `podman run` | Create & start a container |
| `podman create` | Create a container (without starting) |
| `podman start` | Start a stopped container |
| `podman stop` | Stop a running container |
| `podman restart` | Restart a container |
| `podman pause` | Pause container processes |
| `podman unpause` | Unpause container processes |
| `podman kill` | Kill container (SIGKILL) |
| `podman rm` | Remove container |
| `podman rm -f` | Force remove running container |
| `podman ps` | List running containers |
| `podman ps -a` | List all containers |
| `podman exec` | Run command in running container |
| `podman logs` | View container logs |
| `podman logs -f` | Follow logs |
| `podman top` | Show container processes |
| `podman inspect` | Show container metadata |
| `podman stats` | Live resource usage |
| `podman port` | List port mappings |
| `podman rename` | Rename a container |
| `podman update` | Update container config |
| `podman wait` | Wait for container to exit |
| `podman attach` | Attach to container |
| `podman commit` | Create image from container |
| `podman cp` | Copy files to/from container |
| `podman diff` | Show filesystem changes |
| `podman export` | Export filesystem as tar |
| `podman import` | Import tar as filesystem |

### Image Commands

| Command | Description |
|---|---|
| `podman images` | List images |
| `podman pull` | Pull image from registry |
| `podman push` | Push image to registry |
| `podman rmi` | Remove image |
| `podman build` | Build image from Dockerfile |
| `podman tag` | Tag an image |
| `podman save` | Save image to tar |
| `podman load` | Load image from tar |
| `podman search` | Search registry for images |
| `podman history` | Show image history |
| `podman inspect` | Show image details |
| `podman login` | Login to registry |
| `podman logout` | Logout from registry |
| `podman manifest` | Manage manifest lists |

### Pod Commands (Podman-only feature)

| Command | Description |
|---|---|
| `podman pod create` | Create a pod |
| `podman pod list` | List pods |
| `podman pod inspect` | Inspect pod |
| `podman pod stats` | Show pod resource usage |
| `podman pod start` | Start pod |
| `podman pod stop` | Stop pod |
| `podman pod restart` | Restart pod |
| `podman pod rm` | Remove pod |
| `podman pod ps` | List containers in pod |

### Network Commands

| Command | Description |
|---|---|
| `podman network ls` | List networks |
| `podman network create` | Create a network |
| `podman network rm` | Remove a network |
| `podman network connect` | Connect container to network |
| `podman network disconnect` | Disconnect container |
| `podman network inspect` | Inspect network |
| `podman network prune` | Remove unused networks |

### Volume Commands

| Command | Description |
|---|---|
| `podman volume ls` | List volumes |
| `podman volume create` | Create a volume |
| `podman volume rm` | Remove a volume |
| `podman volume inspect` | Inspect a volume |
| `podman volume prune` | Remove unused volumes |

### System Commands

| Command | Description |
|---|---|
| `podman info` | System-wide info |
| `podman version` | Show version |
| `podman system df` | Disk usage |
| `podman system prune` | Clean unused data |
| `podman system prune -a` | Aggressive cleanup |
| `podman system events` | Stream system events |

### Compose Commands

| Command | Description |
|---|---|
| `podman compose up` | Start services |
| `podman compose up -d` | Start in background |
| `podman compose down` | Stop & remove services |
| `podman compose ps` | List service containers |
| `podman compose logs` | View service logs |
| `podman compose exec` | Run in service container |
| `podman compose build` | Build service images |
| `podman compose pull` | Pull service images |
| `podman compose restart` | Restart services |
| `podman compose stop` | Stop services |
| `podman compose rm` | Remove stopped containers |

---

## Quick Reference Card

```powershell
# === DAILY WORKFLOW ===

# Morning — start containers
podman machine start
podman compose up -d

# Work — normal docker/podman commands
podman ps
podman logs app
podman exec -it app bash

# Evening — clean up
podman compose down
podman machine stop   # ← ZERO memory usage
```

> **Bottom line:** Use `podman` instead of `docker` (99% same syntax).  
> Remember `podman machine start` before using containers and `podman machine stop` after.  
> If you prefer the `docker` command, the alias from this guide makes it work transparently.
