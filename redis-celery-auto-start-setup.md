# Redis + Celery Auto-Start Setup (Podman + Windows)

> **Project Location:** `C:\Users\YOUR_USERNAME\celery-app\`
> **Auto-start:** Containers start automatically at Windows logon — no manual commands needed.
> **Created:** 2026-07-20

---

## Table of Contents
1. [What Was Set Up](#1-what-was-set-up)
2. [Project Structure](#2-project-structure)
3. [docker-compose.yml](#3-docker-composeyml)
4. [tasks.py (Celery App)](#4-taskspy-celery-app)
5. [requirements.txt](#5-requirementstxt)
6. [Dockerfile](#6-dockerfile)
7. [start-containers.ps1 (Startup Script)](#7-start-containersps1-startup-script)
8. [Windows Scheduled Task](#8-windows-scheduled-task)
9. [How Auto-Start Works](#9-how-auto-start-works)
10. [What I Did Step by Step](#10-what-i-did-step-by-step)
11. [Testing the Setup](#11-testing-the-setup)
12. [Daily Usage](#12-daily-usage)
13. [Stopping Everything](#13-stopping-everything)
14. [Rebuilding Images](#14-rebuilding-images)
15. [Disabling Auto-Start](#15-disabling-auto-start)
16. [Troubleshooting](#16-troubleshooting)

---

## 1. What Was Set Up

| Container | Image | Purpose | Port | Restart Policy |
|---|---|---|---|---|
| **redis** | `redis:7-alpine` | Celery broker + result backend | `:6379` | `unless-stopped` |
| **celery-worker** | Custom (Python 3.12) | Executes Celery tasks | — | `unless-stopped` |
| **celery-beat** | Custom (Python 3.12) | Runs periodic/scheduled tasks | — | `unless-stopped` |
| **flower** | `mher/flower:2.0` | Celery monitoring dashboard | `:5555` | `unless-stopped` |

### What each service does:

- **Redis**: Acts as the message broker. Celery tasks go through Redis. Also stores task results.
- **celery-worker**: The actual worker that picks up tasks from Redis and executes them. Scales horizontally.
- **celery-beat**: Scheduler for periodic tasks. Runs tasks automatically at set intervals (e.g., every 30 seconds in the demo).
- **Flower**: Web-based UI to monitor Celery workers, tasks, queues, and history.

---

## 2. Project Structure

```
C:\Users\YOUR_USERNAME\celery-app/
│
├── docker-compose.yml      # All 4 service definitions
├── Dockerfile              # Builds the Celery worker image
├── requirements.txt        # Python dependencies
├── tasks.py                # Celery app with sample tasks
│
├── start-containers.ps1    # PowerShell script (called by Scheduled Task)
│
└── (images built & cached by Podman, no user-visible files)
```

---

## 3. docker-compose.yml

```yaml
version: "3.8"

services:
  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    restart: unless-stopped
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  celery-worker:
    build: .
    container_name: celery-worker
    command: celery -A tasks worker --loglevel=info
    restart: unless-stopped
    depends_on:
      redis:
        condition: service_healthy
    volumes:
      - .:/app
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0

  celery-beat:
    build: .
    container_name: celery-beat
    command: celery -A tasks beat --loglevel=info
    restart: unless-stopped
    depends_on:
      redis:
        condition: service_healthy
    volumes:
      - .:/app
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0

  flower:
    image: mher/flower:2.0
    container_name: flower
    ports:
      - "5555:5555"
    restart: unless-stopped
    depends_on:
      - celery-worker
    environment:
      - CELERY_BROKER_URL=redis://redis:6379/0
      - CELERY_RESULT_BACKEND=redis://redis:6379/0

volumes:
  redis_data:
```

**Key points:**
- `restart: unless-stopped` → Container auto-restarts on crash. Containers also auto-start after Podman machine restart.
- `depends_on` with `condition: service_healthy` → Celery only starts after Redis is fully ready.
- Bind mount `.:/app` → Code changes take effect without restarting the container (development-friendly).
- Redis data persistent → `redis_data` volume ensures data survives container removal.

---

## 4. tasks.py (Celery App)

```python
import time
from celery import Celery

app = Celery(
    "tasks",
    broker="redis://redis:6379/0",
    backend="redis://redis:6379/0",
)

app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="Asia/Dhaka",
    enable_utc=True,
)


@app.task
def add(x, y):
    """Simple test task: add two numbers."""
    time.sleep(2)
    result = x + y
    print(f"{x} + {y} = {result}")
    return result


@app.task
def multiply(x, y):
    """Simple test task: multiply two numbers."""
    result = x * y
    print(f"{x} * {y} = {result}")
    return result


@app.task
def hello(name):
    """Test task that just prints a greeting."""
    print(f"Hello, {name}!")
    return f"Hello, {name}!"


@app.on_after_configure.connect
def setup_periodic_tasks(sender, **kwargs):
    """Demo: runs hello every 30 seconds."""
    sender.add_periodic_task(30.0, hello.s("world"), name="say hello every 30s")
```

**Sample tasks included:**
- `add(x, y)` — Sleeps for 2 seconds, then returns the sum
- `multiply(x, y)` — Returns the product
- `hello(name)` — Returns a greeting string
- `setup_periodic_tasks` — Automatically runs `hello("world")` every 30 seconds (via celery-beat)

---

## 5. requirements.txt

```
celery[redis]==5.4.0
redis==5.2.0
```

---

## 6. Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["celery", "-A", "tasks", "worker", "--loglevel=info"]
```

**Notes:**
- `python:3.12-slim` — lightweight Python image (~120 MB)
- Dependencies are installed from `requirements.txt`
- `CMD` is for the worker only; beat overrides the command in docker-compose.yml

---

## 7. start-containers.ps1 (Startup Script)

**File:** `C:\Users\YOUR_USERNAME\celery-app\start-containers.ps1`

```powershell
$env:Path = "C:\Program Files\RedHat\Podman;$env:USERPROFILE\.pyenv\pyenv-win\shims;$env:USERPROFILE\.local\bin;$env:PATH"
$projectDir = "C:\Users\YOUR_USERNAME\celery-app"

# 1. Make sure Podman machine is running (safe to run even if already running)
podman machine start 2>$null

# 2. Wait for Podman to be ready
Start-Sleep 3

# 3. Start all containers (Redis, Celery worker, Celery beat, Flower)
Set-Location $projectDir
podman compose up -d
```

**What this script does:**
1. Ensures Podman, podman-compose, and the docker wrapper are in PATH
2. `podman machine start` — Starts the VM (ignores error if already running)
3. Waits 3 seconds (VM boot time)
4. `podman compose up -d` — Starts all containers in the background

---

## 8. Windows Scheduled Task

**Name:** `CeleryAppContainers`

| Property | Value |
|---|---|
| Task Name | `CeleryAppContainers` |
| Trigger | At logon (user: `YOUR_USERNAME`) |
| Action | `powershell.exe -NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File C:\Users\YOUR_USERNAME\celery-app\start-containers.ps1` |
| Run level | Limited (no admin/UAC required) |
| Settings | Allow on batteries, don't stop on battery |

**How to view/modify:**
```powershell
# View task
Get-ScheduledTask -TaskName "CeleryAppContainers"

# Run manually (test)
Start-ScheduledTask -TaskName "CeleryAppContainers"

# Disable (without deleting)
Disable-ScheduledTask -TaskName "CeleryAppContainers"

# Enable again
Enable-ScheduledTask -TaskName "CeleryAppContainers"

# Delete completely
Unregister-ScheduledTask -TaskName "CeleryAppContainers" -Confirm:$false
```

---

## 9. How Auto-Start Works

```
┌─────────────────────────────────────────────────────┐
│  PC Power ON                                        │
│    ↓                                                │
│  Windows Boots                                      │
│    ↓                                                │
│  User logs in (YOUR_USERNAME)                               │
│    ↓                                                │
│  Windows Task Scheduler detects logon                │
│    ↓                                                │
│  Runs: start-containers.ps1                         │
│    ↓                                                │
│  podman machine start (WSL VM boots in ~5 sec)      │
│    ↓                                                │
│  podman compose up -d                               │
│    ├── redis :6379  ✓                               │
│    ├── celery-worker  ✓                             │
│    ├── celery-beat  ✓                               │
│    └── flower :5555  ✓                              │
│    ↓                                                │
│  ✅ Everything ready — no manual command needed     │
└─────────────────────────────────────────────────────┘
```

---

## 10. What I Did Step by Step

Here's exactly what I did to set this up:

### Step 1: Created project directory
```powershell
New-Item -ItemType Directory -Path "C:\Users\YOUR_USERNAME\celery-app" -Force
```

### Step 2: Created `docker-compose.yml`
With 4 services: Redis, celery-worker, celery-beat, Flower. All with `restart: unless-stopped`.

### Step 3: Created `tasks.py`
Celery app with `add`, `multiply`, `hello` tasks + periodic task setup.

### Step 4: Created `requirements.txt`
With `celery[redis]==5.4.0` and `redis==5.2.0`.

### Step 5: Created `Dockerfile`
Python 3.12-slim base, installs dependencies, copies code.

### Step 6: Installed `podman-compose` inside Podman VM
```powershell
podman machine ssh "sudo dnf install -y podman-compose"
```

### Step 7: Installed `podman-compose` on Windows (for `podman compose` command)
```powershell
pip install podman-compose
```

### Step 8: Built images & started containers
```powershell
cd C:\Users\YOUR_USERNAME\celery-app
podman compose up -d --build
```

### Step 9: Fixed port conflict
WSL Ubuntu had a pre-installed Redis (`redis-server`) occupying port `:6379`. Disabled it:
```bash
# Inside WSL Ubuntu
systemctl stop redis-server
systemctl disable redis-server
```
Then redeployed the containers.

### Step 10: Created startup script
`start-containers.ps1` — starts podman machine + runs compose up.

### Step 11: Registered Windows Scheduled Task
```powershell
$action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-NoProfile -WindowStyle Hidden -ExecutionPolicy Bypass -File C:\Users\YOUR_USERNAME\celery-app\start-containers.ps1"
$trigger = New-ScheduledTaskTrigger -AtLogon -User "YOUR_USERNAME"
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable
$principal = New-ScheduledTaskPrincipal -UserId "YOUR_USERNAME" -LogonType Interactive -RunLevel Limited
Register-ScheduledTask -TaskName "CeleryAppContainers" -Action $action -Trigger $trigger -Settings $settings -Principal $principal -Force
```

### Step 12: Verified everything works
```powershell
podman ps  # All 4 containers running
# Test tasks
podman exec -it celery-worker python -c "import tasks; print(tasks.add.delay(10, 20).get(timeout=10))"
# Output: 30
```

---

## 11. Testing the Setup

### Check containers are running
```powershell
podman ps
```
Expected output: 4 containers (redis, celery-worker, celery-beat, flower) all with `Up` status.

### Run a test task
```powershell
# Addition task
podman exec -it celery-worker python -c "import tasks; print(tasks.add.delay(10, 20).get(timeout=10))"
# → 30

# Multiplication task
podman exec -it celery-worker python -c "import tasks; print(tasks.multiply.delay(7, 6).get(timeout=10))"
# → 42

# Hello task
podman exec -it celery-worker python -c "import tasks; print(tasks.hello.delay('Podman').get(timeout=10))"
# → Hello, Podman!
```

### Check periodic tasks (celery-beat)
```powershell
# Wait 30 seconds after start, then check worker logs
podman logs celery-worker --tail 20
```
You should see `Hello, world!` messages every 30 seconds.

### Monitor with Flower UI
```
http://localhost:5555
```
Open in browser. Shows:
- Worker status (online/offline)
- Task history (success/failed)
- Queue depth
- Active tasks

### Check Redis is working
```powershell
# Ping Redis
podman exec redis redis-cli ping
# → PONG

# Check Redis keys
podman exec redis redis-cli keys '*'
```

---

## 12. Daily Usage

### Normal operation (auto-start)
All containers start automatically on PC boot — no action needed.

### If containers are stopped but you need them
```powershell
cd C:\Users\YOUR_USERNAME\celery-app
podman compose up -d
```

### Check logs
```powershell
podman logs celery-worker    # Worker logs
podman logs celery-beat      # Beat scheduler logs
podman logs redis            # Redis logs
podman logs flower           # Flower UI logs

# Follow logs live
podman logs -f celery-worker
```

### Restart a specific container
```powershell
podman restart celery-worker
```

### Scale workers (multiple instances)
```powershell
podman compose up -d --scale celery-worker=3
```

### Send custom task from command line
```powershell
podman exec -it celery-worker python -c "
import tasks
result = tasks.add.delay(100, 200)
print('Task ID:', result.id)
print('Result:', result.get(timeout=10))
"
```

---

## 13. Stopping Everything

### Stop containers only (keep Podman machine running)
```powershell
cd C:\Users\YOUR_USERNAME\celery-app
podman compose down
```
Auto-starts on next PC boot.

### Stop containers + Podman machine (free all memory)
```powershell
cd C:\Users\YOUR_USERNAME\celery-app
podman compose down
podman machine stop   # ← Memory usage becomes 0
```

### Stop everything AND delete data
```powershell
cd C:\Users\YOUR_USERNAME\celery-app
podman compose down -v   # -v = delete volumes (Redis data lost!)
podman machine stop
```

---

## 14. Rebuilding Images

### After changing tasks.py or requirements.txt
```powershell
cd C:\Users\YOUR_USERNAME\celery-app
podman compose up -d --build
```
This rebuilds images and updates containers. Almost zero downtime.

### Force rebuild without cache
```powershell
cd C:\Users\YOUR_USERNAME\celery-app
podman compose build --no-cache
podman compose up -d
```

---

## 15. Disabling Auto-Start

### Method 1: Disable Scheduled Task (keep everything installed)
```powershell
Disable-ScheduledTask -TaskName "CeleryAppContainers"
```
Task stays registered but won't run on trigger. To re-enable:
```powershell
Enable-ScheduledTask -TaskName "CeleryAppContainers"
```

### Method 2: Delete Scheduled Task completely
```powershell
Unregister-ScheduledTask -TaskName "CeleryAppContainers" -Confirm:$false
```
Auto-start permanently removed. Re-register if needed.

### Method 3: Delete everything
```powershell
cd C:\Users\YOUR_USERNAME\celery-app
podman compose down -v
Unregister-ScheduledTask -TaskName "CeleryAppContainers" -Confirm:$false
Remove-Item -Path "C:\Users\YOUR_USERNAME\celery-app" -Recurse -Force
```

---

## 16. Troubleshooting

### Problem: Port 6379 already in use
```powershell
# Check what's using port 6379
netstat -ano | findstr :6379

# If it's WSL Redis:
wsl -d Ubuntu -u root bash -c "systemctl stop redis-server; systemctl disable redis-server"

# Then retry:
podman compose up -d
```

### Problem: Podman machine not starting
```powershell
# Check status
podman machine list

# Force restart
podman machine stop
podman machine start

# If still fails, reset
podman machine reset
podman machine init
podman machine start
```

### Problem: "podman compose" command not found
```powershell
# Install podman-compose on Windows
pip install podman-compose

# Or install inside VM
podman machine ssh "sudo dnf install -y podman-compose"

# Verify
podman-compose --version
```

### Problem: Celery worker not connecting to Redis
```powershell
# Check if Redis is healthy
podman ps

# Check Redis logs
podman logs redis

# Check worker logs
podman logs celery-worker

# Restart both
podman restart redis celery-worker
```

### Problem: Can't access Flower at http://localhost:5555
```powershell
# Check if Flower is running
podman ps | findstr flower

# Check Flower logs
podman logs flower

# Restart Flower
podman restart flower
```

### Problem: Scheduled task not running at logon
```powershell
# Check task status
Get-ScheduledTask -TaskName "CeleryAppContainers" | Select-Object State

# Run manually
Start-ScheduledTask -TaskName "CeleryAppContainers"

# Check last run result
Get-ScheduledTask -TaskName "CeleryAppContainers" | Get-ScheduledTaskInfo
```

### Problem: Windows PATH not finding podman
```powershell
# Add Podman to PATH manually
$env:Path += ";C:\Program Files\RedHat\Podman"

# Permanent fix (already done in PowerShell profile):
# See C:\Users\YOUR_USERNAME\OneDrive\Documents\WindowsPowerShell\profile.ps1
```
