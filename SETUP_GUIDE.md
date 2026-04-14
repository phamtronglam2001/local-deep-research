# Complete Setup Guide: Local Deep Research + SearXNG on Windows with Podman

This guide walks you through setting up Local Deep Research and SearXNG on Windows using Podman and WSL2, from scratch.

## Table of Contents

1. [Prerequisites & Installation](#prerequisites--installation)
2. [Understanding the Architecture](#understanding-the-architecture)
3. [Configuring LM Studio](#configuring-lm-studio)
4. [Setting Up Environment Variables](#setting-up-environment-variables)
5. [Running the Containers](#running-the-containers)
6. [Accessing the Applications](#accessing-the-applications)
7. [Troubleshooting](#troubleshooting)

---

## Prerequisites & Installation

### Step 1: Install WSL2 (Windows Subsystem for Linux)

WSL2 is a lightweight Linux environment that runs on Windows. Podman will use WSL2 as its backend.

```powershell
# Open PowerShell as Administrator and run:
wsl --install
```

This installs WSL2 and Ubuntu by default. Restart your computer when prompted.

**What is WSL2?**
- A lightweight Linux virtual machine that runs natively on Windows
- Podman containers run inside WSL2, not directly on Windows
- WSL2 gets its own IP address (example: `172.31.252.236`)
- Windows is reachable from WSL2 via a gateway IP (example: `172.31.240.1`)

### Step 2: Install Podman for Windows

Podman is a containerization tool similar to Docker.

```powershell
# Using Chocolatey (if you have it installed)
choco install podman

# Or download from: https://github.com/containers/podman/releases
# and follow the installer
```

After installation, initialize the Podman machine:

```powershell
podman machine init
podman machine start
```

**Note:** On Windows, Podman uses WSL2 to run containers. The `podman machine` is actually a WSL2 virtual machine.

### Step 3: Verify Installation

```powershell
podman --version
podman ps  # Should show an empty list of containers
```

### Step 4: Install LM Studio (on Windows host)

Download and install LM Studio from: https://lmstudio.ai/

1. Install LM Studio on your Windows machine (not in WSL2)
2. Download a model (e.g., "Qwen 3 4B Thinking")
3. Start the Local Server in LM Studio
4. Verify it's running on `localhost:1234`

---

## Understanding the Architecture

This is crucial to understand before configuring networking.

```
┌─────────────────────────────────────────────────────────────┐
│ WINDOWS HOST                                                │
│ ┌──────────────────────────────────────────────────────────┐│
│ │ LM Studio (localhost:1234)  [Your Windows machine]       ││
│ └──────────────────────────────────────────────────────────┘│
│                                                              │
│ ┌──────────────────────────────────────────────────────────┐│
│ │ WSL2 ENVIRONMENT (Linux VM)                              ││
│ │ Gateway: 172.31.240.1 (reaches back to Windows)          ││
│ │ WSL IP: 172.31.252.236 (accessible from Windows)         ││
│ │                                                           ││
│ │ ┌─────────────────────────────────────────────────────┐ ││
│ │ │ Podman Network: "ldr-network"                       │ ││
│ │ │                                                     │ ││
│ │ │ ┌──────────────────┐  ┌──────────────────────────┐ │ ││
│ │ │ │ SearXNG          │  │ Deep Research            │ ││ │
│ │ │ │ Port: 8080       │  │ Port: 5000               │ ││ │
│ │ │ │ Internal DNS:    │  │ Internal DNS: deep-      │ ││ │
│ │ │ │ "searxng:8080"   │  │ research:5000            │ ││ │
│ │ │ └──────────────────┘  └──────────────────────────┘ ││ │
│ │ │                                                     │ ││
│ │ │ Can reach each other by name (DNS resolution)      │ ││
│ │ └─────────────────────────────────────────────────────┘ ││
│ │                                                           ││
│ │ Can reach Windows host at: 172.31.240.1:1234            ││
│ └──────────────────────────────────────────────────────────┘│
│                                                              │
│ Port Forwarding (Windows → WSL2):                           │
│ - Port 3000 (Windows) → 5000 (Deep Research container)     │
│ - Port 8888 (Windows) → 8080 (SearXNG container)           │
└─────────────────────────────────────────────────────────────┘
```

### Key IP Addresses and Their Meanings

| Address | Purpose | Accessible From |
|---------|---------|-----------------|
| `localhost:1234` | LM Studio on Windows | Windows machine only |
| `172.31.240.1:1234` | Windows gateway (from WSL2 perspective) | Containers inside WSL2 |
| `172.31.252.236` | WSL2 machine IP | Windows host and internet |
| `searxng:8080` | Container hostname on shared network | Other containers on same network |
| `deep-research:5000` | Container hostname on shared network | Other containers on same network |

---

## Configuring LM Studio

### Step 1: Verify LM Studio is Running

```powershell
# Check if LM Studio is listening on port 1234
Test-NetConnection -ComputerName localhost -Port 1234
```

Expected output:
```
TcpTestSucceeded : True
```

### Step 2: Get the Windows Gateway IP (for WSL2 containers)

Containers inside WSL2 need to know which IP to use to reach Windows.

```powershell
# Find the WSL2 gateway IP (usually 172.31.240.1, but may vary)
wsl ip route | Select-String "default"
```

Expected output:
```
default via 172.31.240.1 dev eth0 proto kernel
```

**The gateway IP is `172.31.240.1`** in this example. Note yours in case it's different.

---

## Setting Up Environment Variables

### Step 1: Locate the .env File

Navigate to the Local Deep Research directory and open `.env`:

```powershell
cd d:\CodeApp\local-deep-research
notepad .env
```

### Step 2: Configure for LM Studio

Edit the file to match your setup. Here's the template:

```bash
# Local Deep Research Configuration for Windows + WSL2

# Web Server Settings
LDR_WEB_HOST=0.0.0.0
LDR_WEB_PORT=5000
LDR_WEB_USE_HTTPS=false

# LM Studio Settings
LDR_LLM_PROVIDER=lmstudio
LDR_LLM_LMSTUDIO_URL=http://172.31.240.1:1234/v1
LDR_LLM_MODEL=qwen3-4b-thinking-2507
```

### Key Configuration Explained

- **`LDR_LLM_LMSTUDIO_URL=http://172.31.240.1:1234/v1`**
  - `172.31.240.1` = Windows gateway IP (reachable from containers in WSL2)
  - `1234` = Port where LM Studio is running on Windows
  - `/v1` = API endpoint version

- **`LDR_LLM_MODEL=qwen3-4b-thinking-2507`**
  - Must match a model you've downloaded in LM Studio
  - You can verify available models by accessing: `http://localhost:1234/v1/models` from your Windows browser

### Step 3: Save the File

Save and close the `.env` file.

---

## Running the Containers

### Step 1: Create a Shared Network

First, create a network that both containers will share. This allows them to communicate with each other using container names.

```powershell
podman network create ldr-network
```

### Step 2: Start SearXNG Container

```powershell
podman run -d --name searxng `
  --network ldr-network `
  -p 8888:8080 `
  docker.io/searxng/searxng:latest
```

**What this does:**
- `-d` = Run in background (detached mode)
- `--name searxng` = Container name (used for DNS resolution)
- `--network ldr-network` = Connect to the shared network
- `-p 8888:8080` = Forward Windows port 8888 to container port 8080
- `docker.io/searxng/searxng:latest` = Image to run

### Step 3: Start Deep Research Container

```powershell
cd d:\CodeApp\local-deep-research

podman run -d --name deep-research `
  --network ldr-network `
  -p 3000:5000 `
  --env-file .env `
  -v deep-research:/data `
  -e LDR_DATA_DIR=/data `
  docker.io/localdeepresearch/local-deep-research
```

**What this does:**
- `--network ldr-network` = Connect to the same network as SearXNG
- `-p 3000:5000` = Forward Windows port 3000 to container port 5000
- `--env-file .env` = Load environment variables from .env file
- `-v deep-research:/data` = Create persistent data volume
- `-e LDR_DATA_DIR=/data` = Set data directory inside container

### Step 4: Verify Both Containers Are Running

```powershell
podman ps
```

Expected output:
```
CONTAINER ID  IMAGE                                           COMMAND     STATUS         PORTS
abc123def...   docker.io/searxng/searxng:latest               Up 1 min   0.0.0.0:8888->8080/tcp  searxng
xyz789pqr...   docker.io/localdeepresearch/local-deep-research  ldr-web  Up 1 min   0.0.0.0:3000->5000/tcp  deep-research
```

### Step 5: Test Container Connectivity

Verify that Deep Research can reach SearXNG through the shared network:

```powershell
podman exec deep-research python3 -c "import urllib.request; urllib.request.urlopen('http://searxng:8080').read(); print('✓ SearXNG is reachable')"
```

Expected output:
```
✓ SearXNG is reachable
```

---

## Accessing the Applications

### Find Your WSL2 IP Address

The containers are running in WSL2. You access them from Windows using the WSL2 IP address.

```powershell
wsl ip addr | Select-String "inet\s" | Select-Object -First 2
```

Expected output:
```
    inet 127.0.0.1/8 scope host lo
    inet 172.31.252.236/20 brd 172.31.255.255 scope global eth0
```

Your WSL2 IP is `172.31.252.236` (the global IP, not the loopback).

### Access Deep Research

Open your web browser and navigate to:

```
http://172.31.252.236:3000
```

You should see the Local Deep Research web interface.

### Access SearXNG

Open your web browser and navigate to:

```
http://172.31.252.236:8888
```

You should see the SearXNG search interface.

### Configure Deep Research to Use SearXNG

1. In Deep Research web UI, go to **Settings**
2. Find the **Search Engine** or **SearXNG** configuration
3. Set the SearXNG instance URL to: `http://searxng:8080`
   - Use the container name `searxng` (not the IP address)
   - Use the internal container port `8080` (not `8888`)
4. Save the settings

Now Deep Research will use SearXNG for searches.

---

## Troubleshooting

### Issue: "Unable to conduct research without a search engine"

**Solution:** 
- Verify SearXNG URL in settings is set to `http://searxng:8080`
- Make sure both containers are on the same network: `podman ps`
- Check that SearXNG container is running: `podman logs searxng`

### Issue: "Connection error" when researching (LM Studio)

**Symptoms:** Research starts but fails with connection error

**Solution:**
1. Verify LM Studio is running on Windows: `Test-NetConnection -ComputerName localhost -Port 1234`
2. Check `.env` file has correct LM Studio URL: `http://172.31.240.1:1234/v1`
3. Verify the model name in `.env` matches a model in LM Studio
4. Test from container: `podman exec deep-research python3 -c "import urllib.request; urllib.request.urlopen('http://172.31.240.1:1234/v1/models').read()"`

### Issue: Cannot access http://172.31.252.236:3000 from browser

**Possible causes:**

1. **Wrong IP address**
   ```powershell
   wsl ip addr | Select-String "inet\s"
   ```
   Use the correct WSL IP (should be 172.31.x.x, not 127.0.0.1)

2. **Container not running**
   ```powershell
   podman ps
   ```
   Both containers should show "Up"

3. **Port forwarding issue**
   ```powershell
   netstat -ano | findstr "3000"
   ```
   Should show port 3000 listening

### Issue: Containers can't reach each other

**Solution:**
Verify both containers are on the same network:

```powershell
podman inspect deep-research | Select-String "Networks"
podman inspect searxng | Select-String "Networks"
```

Both should show `"Networks": "ldr-network"`. If not, recreate them with the correct network flag.

### Issue: WSL2 IP address changes

WSL2 IPs can change when you restart. To see the current IP:

```powershell
wsl ip addr | Select-String "inet\s" | Select-Object -First 2
```

Then update your browser bookmarks or use this IP in any application that needs it.

---

## Useful Commands Reference

### View Logs

```powershell
# Deep Research logs
podman logs -f deep-research

# SearXNG logs
podman logs -f searxng

# Follow logs in real-time (Ctrl+C to stop)
```

### Stop Containers

```powershell
podman stop deep-research searxng
```

### Remove Containers

```powershell
podman rm deep-research searxng
```

### List All Containers

```powershell
podman ps -a
```

### Check Network

```powershell
podman network ls
podman network inspect ldr-network
```

### Get Container IP Inside Network

```powershell
podman inspect deep-research --format='{{json .NetworkSettings.Networks}}'
```

---

## Summary

You now have:

✅ WSL2 running a Linux environment on Windows  
✅ Podman running containers inside WSL2  
✅ LM Studio configured on Windows (reachable from containers)  
✅ SearXNG container running for search functionality  
✅ Deep Research container running for research tasks  
✅ Shared network allowing containers to communicate  
✅ Port forwarding set up for browser access  

**Access your applications:**
- Deep Research: `http://172.31.252.236:3000`
- SearXNG: `http://172.31.252.236:8888`

**Container-to-container URLs (internal only):**
- SearXNG from Deep Research: `http://searxng:8080`
- Deep Research to Windows LM Studio: `http://172.31.240.1:1234/v1`

Enjoy your local research setup!
