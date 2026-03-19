# Home AI Infrastructure Architecture
### A Private, Powerful, Always-On AI Stack — Built in Your Home

> **Version:** 2.1 | **Status:** Active  
> **Hardware:** Minisforum MS-S1 MAX · Raspberry Pi 4 · (Phase 2: Apple Mac Studio)

---

## Table of Contents

- [Part 1: Requirements & Features](#part-1-requirements--features)
- [Part 2: System Overview](#part-2-system-overview)
- [Part 3: Bill of Materials](#part-3-bill-of-materials)
- [Part 4: Phase 1 — Single Machine Implementation](#part-4-phase-1--single-machine-implementation)
  - [4.1 MS-S1 MAX: OS & Proxmox](#41-ms-s1-max-os--proxmox)
  - [4.2 VM Design](#42-vm-design)
  - [4.3 NemoClaw + OpenClaw](#43-nemoclaw--openclaw)
  - [4.4 Ollama + Local Models](#44-ollama--local-models)
  - [4.5 Multi-Agent Configuration](#45-multi-agent-configuration)
  - [4.6 Media Server](#46-media-server)
  - [4.7 Home Assistant (Raspberry Pi)](#47-home-assistant-raspberry-pi)
  - [4.8 Remote Access](#48-remote-access)
  - [4.9 Security](#49-security)
- [Part 5: Phase 2 — Mac Studio Added](#part-5-phase-2--mac-studio-added)
- [Phase 3: Network-Level Security Hardening](#part-55-phase-3--network-level-security-hardening)
- [Part 6: Performance Comparison](#part-6-performance-comparison)
- [Part 7: Testing & Validation](#part-7-testing--validation)
- [Part 8: Maintenance](#part-8-maintenance)

---

# Part 1: Requirements & Features

## What This Builds

A private home AI infrastructure that gives you the capabilities of a well-staffed tech company —
running on hardware you own, with your data never leaving your home.

## Functional Requirements

### Personal AI Assistant
- Talk to your AI via WhatsApp from anywhere in the world
- AI remembers context, manages tasks, and takes autonomous action
- Responds in seconds; always on
- Fully sandboxed and secured via NVIDIA NemoClaw

### Local AI Inference
- Run large language models on your own hardware
- Phase 1: 32B models on the MS-S1 MAX integrated GPU
- Phase 2: 70B–200B+ models on the Mac Studio
- Private by default — nothing sent to external servers except via controlled cloud calls

### Coding Agents
- OpenClaw natively orchestrates coding agents (Claude Code, Codex) inside the NemoClaw sandbox
- Orchestrator agent (large, smart cloud model) breaks down tasks and delegates
- Worker agents (fast, specialized local model) execute code, write tests, open PRs
- No separate agent VM required — NemoClaw handles isolation and orchestration

### AI-Powered Home Automation
- Home Assistant with local LLM for natural language automations
- Voice commands processed entirely on-device
- AI camera analysis via Frigate + Google Coral TPU
- Smart notifications (e.g. "a person has been at your door for 30 seconds")

### Media Server
- Stream video and music to any device on your network or via remote access
- Hardware-accelerated transcoding on the AMD Radeon 890M
- Media library stored on external/NAS storage — separate from the AI stack

### Remote Access
- WhatsApp → your AI, from any phone, anywhere (via OpenClaw cloud relay)
- Phone → local LLM from anywhere via Tailscale VPN
- Home Assistant from anywhere via Nabu Casa
- No inbound ports opened on your router — all outbound connections only

## Non-Functional Requirements

| Requirement | Approach |
|-------------|---------|
| **Privacy** | All AI inference runs locally. Cloud relay (OpenClaw gateway, Nabu Casa) are dumb pipes — they move bytes, never process content |
| **Security** | NemoClaw OpenShell sandbox isolates the AI agent. No public inbound ports. |
| **Always-on** | Proxmox auto-starts VMs on boot. Systemd services restart on failure. |
| **Router compatibility** | Designed for standard home routers including Google Nest. No VLANs or managed switch required. Network isolation is handled by Proxmox internally. |
| **Upgrade path** | Phase 1 is fully functional. Phase 2 adds Mac Studio — one config change per VM, nothing else changes. |

---

# Part 2: System Overview

## Phase 1 Architecture

```
╔══════════════════════════════════════════════════════════════════════╗
║                    HOME NETWORK (192.168.1.0/24)                     ║
║                          Google Nest Router                          ║
║                                                                      ║
║  ┌───────────────────────────────────────────────────────────────┐  ║
║  │                  MS-S1 MAX (192.168.1.20)                     │  ║
║  │                  Proxmox VE — Bare Metal                      │  ║
║  │                                                               │  ║
║  │  ┌──────────────────────────────────────────────────────┐    │  ║
║  │  │  NemoClaw VM  (192.168.1.21)                         │    │  ║
║  │  │                                                      │    │  ║
║  │  │  OpenClaw agent          OpenShell sandbox           │    │  ║
║  │  │  WhatsApp gateway        Coding agent orchestration  │    │  ║
║  │  │                                                      │    │  ║
║  │  │  Main model:   Nemotron 120B (NVIDIA cloud, free)    │    │  ║
║  │  │  Worker model: Qwen 2.5 Coder 32B (local Ollama)     │    │  ║
║  │  └──────────────────────────────────────────────────────┘    │  ║
║  │                                                               │  ║
║  │  ┌──────────────────────┐  ┌──────────────────────────────┐  │  ║
║  │  │  Ollama LXC          │  │  Media Server VM             │  │  ║
║  │  │  192.168.1.25        │  │  192.168.1.22               │  │  ║
║  │  │                      │  │  Plex / Jellyfin             │  │  ║
║  │  │  Radeon 890M (40 CU) │  │  AMD HW transcoding (VAAPI)  │  │  ║
║  │  │  Qwen 2.5 Coder 32B  │  │  → external USB / NAS       │  │  ║
║  │  │  Qwen 2.5 7B (HA)    │  └──────────────────────────────┘  │  ║
║  │  │  phi3:mini (fast)    │                                    │  ║
║  │  └──────────────────────┘                                    │  ║
║  └───────────────────────────────────────────────────────────────┘  ║
║                                                                      ║
║  ┌──────────────────────┐                                           ║
║  │  Raspberry Pi 4      │                                           ║
║  │  192.168.1.30        │                                           ║
║  │  Home Assistant OS   │                                           ║
║  │  Frigate NVR         │                                           ║
║  │  Google Coral TPU    │                                           ║
║  │  → Ollama LXC        │                                           ║
║  └──────────────────────┘                                           ║
╚══════════════════════════════════════════════════════════════════════╝
                              │
                          INTERNET
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
  ┌───────┴──────┐  ┌─────────┴───────┐  ┌───────┴──────┐
  │ OpenClaw     │  │  Nabu Casa      │  │  Tailscale   │
  │ Cloud Gateway│  │  (HA remote)    │  │  (VPN relay) │
  │ (dumb relay) │  └─────────────────┘  └──────────────┘
  └───────┬──────┘
          │
  ┌───────┴──────┐
  │ WhatsApp     │
  │ (your phone) │
  └──────────────┘
```

## Phase 2 Architecture (Mac Studio Added)

```
╔══════════════════════════════════════════════════════════════════════╗
║                    HOME NETWORK (192.168.1.0/24)                     ║
║                                                                      ║
║  ┌───────────────────────────────┐  ┌───────────────────────────┐  ║
║  │  MS-S1 MAX (192.168.1.20)     │  │  Mac Studio M5 Ultra      │  ║
║  │  Proxmox — Orchestration      │  │  (192.168.1.10)           │  ║
║  │                               │  │                           │  ║
║  │  NemoClaw VM ──────────────────┼─►│  Ollama                  │  ║
║  │  Ollama LXC (fallback/HA) ────┘  │  70B / 120B / 200B+      │  ║
║  │  Media Server VM (unchanged)  │  │  models                   │  ║
║  │                               │  │                           │  ║
║  │                               │  │  256GB unified memory     │  ║
║  │                               │  │  4–8 parallel agents      │  ║
║  └───────────────────────────────┘  └───────────────────────────┘  ║
║                                                                      ║
║  ┌──────────────────────┐                                           ║
║  │  Raspberry Pi 4      │──────────────────► Mac Studio Ollama     ║
║  │  Home Assistant      │                    (larger HA models)    ║
║  └──────────────────────┘                                           ║
╚══════════════════════════════════════════════════════════════════════╝
```

## Data Flow: WhatsApp Message End-to-End

```
You (phone)
    │  WhatsApp message
    ▼
OpenClaw Cloud Gateway  (dumb relay — moves bytes, never reads content)
    │
    ▼
NemoClaw VM (192.168.1.21) — inside OpenShell sandbox
    │  Phase 1: HTTP → integrate.api.nvidia.com (Nemotron 120B, free)
    │  Phase 2: HTTP → 192.168.1.10:11434 (Mac Studio Ollama, local)
    ▼
Response generated
    │
    ▼
OpenClaw Cloud Gateway → WhatsApp → Your phone
```

## Network Design

No VLANs. No managed switch. No special router required.

```
Google Nest Router (192.168.1.1)
    │
    ├── MS-S1 MAX Proxmox host (192.168.1.20)  ← one device on LAN
    │       ├── NemoClaw VM    (192.168.1.21)
    │       ├── Ollama LXC     (192.168.1.25)
    │       └── Media Server VM (192.168.1.22)
    │
    ├── Raspberry Pi (192.168.1.30)
    ├── Mac Studio — Phase 2 (192.168.1.10)
    └── Your phones, laptops, etc.
```

### Static IP Assignments

Set DHCP reservations in Google Nest app by MAC address:

| Device | IP | How to set |
|--------|----|-----------|
| MS-S1 MAX (Proxmox) | 192.168.1.20 | Nest app → Devices → Reserve IP |
| Raspberry Pi | 192.168.1.30 | Nest app → Devices → Reserve IP |
| Mac Studio (Phase 2) | 192.168.1.10 | Nest app → Devices → Reserve IP |

VMs and LXC containers inside Proxmox get their IPs set statically in their netplan/network config.

---

# Part 3: Bill of Materials

## Phase 1 Hardware

| Item | Model | Price | Notes |
|------|-------|-------|-------|
| **AI + Proxmox Host** | Minisforum MS-S1 MAX (AMD Ryzen AI Max+ 395, 128GB LPDDR5X, 2TB NVMe) | ~$2,920 | [Newegg](https://www.newegg.com/minisforum-barebone-systems-mini-pc-deskmini/p/2SW-002G-000W4). Ships with Windows — wipe and install Proxmox. |
| **Home Automation Hub** | Raspberry Pi 4 Model B (4GB RAM) | ~$55–75 | Already owned. |
| **Camera AI Accelerator** | Google Coral USB Accelerator | ~$60–80 | [coral.ai](https://coral.ai/products/accelerator). Used by Frigate for real-time object detection. |
| **Media Storage** | WD Elements 8TB USB 3.0 External Drive | ~$120 | Or NAS (see below). |
| **MicroSD (Pi)** | Samsung Endurance 64GB | ~$15 | For Home Assistant OS. |
| **Ethernet cables** | Cat6 (2×) | ~$15 | MS-S1 MAX and Pi wired to router for reliability. |

### Optional: NAS Instead of USB Drive

| Item | Model | Price | Notes |
|------|-------|-------|-------|
| **NAS** | Synology DS223 | ~$300 | 2-bay, RAID 1. Accessible to all devices. |
| **NAS Drives** | Seagate IronWolf 4TB ×2 | ~$180 | 8TB usable with RAID 1. |

## Phase 1 Cost Summary

| Config | Cost |
|--------|------|
| Phase 1 (USB drive for media) | ~$3,170 |
| Phase 1 (NAS for media) | ~$3,550 |

## Phase 2 Additional Hardware

| Item | Model | Price | Notes |
|------|-------|-------|-------|
| **LLM Inference Server** | Apple Mac Studio M5 Ultra (256GB unified memory) | ~$4,000–5,000 | Not yet released as of early 2026; M4 Ultra (192GB) available now at ~$4,199. |
| **Ethernet cable** | Cat6 | ~$10 | Wired connection to router required for inference reliability. |

## Phase 2 Cost Summary

| Config | Incremental Cost | Total |
|--------|-----------------|-------|
| + M4 Ultra Mac Studio (192GB) | ~$4,200 | ~$7,400 |
| + M5 Ultra Mac Studio (256GB) | ~$4,500–5,000 | ~$7,700–8,200 |

## Software (All Free Unless Noted)

| Software | Where | Cost |
|----------|-------|------|
| Proxmox VE | MS-S1 MAX bare metal | Free |
| Ubuntu 22.04 LTS | VMs on Proxmox | Free |
| NemoClaw | NemoClaw VM | Free |
| OpenClaw | Inside NemoClaw sandbox | Subscription |
| Ollama | Ollama LXC + Mac Studio (Phase 2) | Free |
| Plex or Jellyfin | Media Server VM | Free (Jellyfin) / Freemium (Plex) |
| Home Assistant OS | Raspberry Pi | Free |
| Nabu Casa | Cloud | $6.50/mo |
| Tailscale | MS-S1 MAX + Phone | Free (personal) |
| Docker CE | NemoClaw VM (for coding agents) | Free |

---

# Part 4: Phase 1 — Single Machine Implementation

## 4.1 MS-S1 MAX: OS & Proxmox

### Before Wiping Windows

The MS-S1 MAX ships with Windows. Do this before wiping:

```
1. Boot into Windows
2. Go to minisforum.com → Support → MS-S1 MAX
3. Download and apply any BIOS/firmware updates
4. Restart, confirm update applied
5. Now wipe and install Proxmox
```

### Install Proxmox VE

1. Download Proxmox VE 8.2+ ISO from [proxmox.com](https://www.proxmox.com/en/downloads)
   *(Must be 8.2+ for full AMD Ryzen AI Max+ 395 kernel support)*
2. Flash to USB with Balena Etcher
3. Boot MS-S1 MAX from USB (hold **F7** during POST for boot menu)
4. Install Proxmox:

```
Hostname:  proxmox.local
IP:        192.168.1.20/24
Gateway:   192.168.1.1
DNS:       8.8.8.8
```

5. Access Proxmox web UI: `https://192.168.1.20:8006`

### Post-Install Hardening

```bash
ssh root@192.168.1.20

# Update
apt update && apt full-upgrade -y

# Switch to free community repo
rm /etc/apt/sources.list.d/pve-enterprise.list
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-community.list
apt update

# Firewall — Proxmox UI and SSH accessible from LAN only
apt install -y ufw
ufw default deny incoming
ufw default allow outgoing
ufw allow from 192.168.1.0/24 to any port 22
ufw allow from 192.168.1.0/24 to any port 8006
ufw enable

# Auto security updates
apt install -y unattended-upgrades
dpkg-reconfigure --priority=low unattended-upgrades
```

---

## 4.2 VM Design

Two VMs and one LXC container run on the MS-S1 MAX. Resource allocation from 128GB total:

| Component | vCPU | RAM | Disk | Network | Purpose |
|-----------|------|-----|------|---------|---------|
| NemoClaw VM | 4 | 16GB | 80GB | vmbr0 (LAN) | OpenClaw + NemoClaw sandbox + coding agents |
| Media Server VM | 4 | 8GB | 100GB | vmbr0 (LAN) | Plex/Jellyfin |
| Ollama LXC | 4 | 8GB* | 32GB | vmbr0 (LAN) | Model inference |
| **Proxmox host** | — | ~8GB | — | — | OS overhead |
| **Available for model weights** | — | ~88GB | — | — | Unified memory pool for Ollama |

> \* The Ollama LXC's declared 8GB RAM covers process overhead only. Model weights load
> into the AMD Radeon 890M's unified memory pool via the bind-mounted DRI device — this is
> separate from the container's declared RAM budget. With VMs and host consuming ~40GB,
> approximately 88GB of the 128GB unified pool is available for models.

> **Why no Agent Swarm VM?** OpenClaw natively orchestrates coding agents (Claude Code, Codex)
> inside the NemoClaw sandbox via its built-in ACP harness. A separate agent VM would duplicate
> this capability while adding complexity and consuming 32GB of RAM that's better used for
> larger models.

### Create Each VM

In Proxmox UI: **Create VM** — use these settings per VM:

**NemoClaw VM:**
```
VM ID: 100  |  Name: nemoclaw
OS:    Ubuntu 22.04 LTS Server ISO
Disk:  80GB VirtIO SCSI
CPU:   4 cores, type: host
RAM:   16384 MB
Net:   vmbr0, VirtIO
```

**Media Server VM:**
```
VM ID: 101  |  Name: media-server
Disk:  100GB VirtIO SCSI
CPU:   4 cores, type: host
RAM:   8192 MB
Net:   vmbr0, VirtIO
```

For each VM, install Ubuntu 22.04 and set a static IP:

```yaml
# /etc/netplan/00-installer-config.yaml (NemoClaw VM)
network:
  version: 2
  ethernets:
    ens18:
      addresses: [192.168.1.21/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8]
```

```yaml
# /etc/netplan/00-installer-config.yaml (Media Server VM)
network:
  version: 2
  ethernets:
    ens18:
      addresses: [192.168.1.22/24]
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8]
```

```bash
sudo netplan apply
```

---

## 4.3 NemoClaw + OpenClaw

> NemoClaw is NVIDIA's open-source plugin that wraps OpenClaw in a hardened sandbox.
> It intercepts all inference calls and routes them through NVIDIA OpenShell,
> enforcing network egress policy, filesystem isolation, and process sandboxing.
> OpenClaw's built-in coding agent orchestration (Claude Code, Codex via ACP harness)
> runs directly inside this sandbox — no separate agent VM required.

### Prerequisites on the NemoClaw VM

```bash
ssh ubuntu@192.168.1.21

# Docker (required by NemoClaw/OpenShell and coding agent runners)
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER && newgrp docker

# Node.js 22
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Verify
node --version   # Must be >= 20
docker --version
```

### Install NemoClaw

```bash
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash
source ~/.bashrc
```

### Run the Onboard Wizard

```bash
nemoclaw onboard
```

The wizard will ask for:
1. **NVIDIA API key** — get one free at [build.nvidia.com](https://build.nvidia.com) → any model → "Get API Key"
2. **Sandbox name** — e.g. `my-assistant`
3. **Inference provider** — select NVIDIA Cloud (default)
4. **WhatsApp** — yes

When complete:
```
Sandbox:  my-assistant (Landlock + seccomp + netns)
Model:    nvidia/nemotron-3-super-120b-a12b (NVIDIA Cloud)
```

### Connect WhatsApp

```bash
nemoclaw my-assistant connect

# Inside the sandbox:
sandbox$ openclaw channels login whatsapp
# Scan QR code with WhatsApp → Settings → Linked Devices → Link a Device

sandbox$ openclaw start
```

### NemoClaw Security Layers

| Layer | What it blocks |
|-------|---------------|
| Network | Agent calling unauthorized hosts — surfaced in `openshell term` for your approval |
| Filesystem | Reads/writes outside `/sandbox` and `/tmp` |
| Process | Privilege escalation, dangerous syscalls |
| Inference | All model calls routed through OpenShell gateway — no direct internet access |

```bash
# Monitor sandbox activity
openshell term     # Live TUI showing all network requests

# Key commands
nemoclaw my-assistant status       # Health check
nemoclaw my-assistant logs -f      # Live logs
nemoclaw my-assistant connect      # Shell into sandbox
```

---

## 4.4 Ollama + Local Models

> **Architecture note:** Running Ollama directly on the Proxmox host is an anti-pattern.
> If Ollama crashes due to a memory leak or a bad model, it can freeze the entire hypervisor —
> taking down your HA, media server, and NemoClaw VMs with it.
>
> **The fix:** Run Ollama inside a **Proxmox LXC container** with the GPU device bind-mounted
> in. Ollama gets native Radeon 890M access, but a crash stays contained to the LXC —
> Proxmox and your other VMs are unaffected.

### Step 1: Create the Ollama LXC Container

In Proxmox UI: **Create CT** (Container, not VM)

```
CT ID:      200
Hostname:   ollama
Template:   Ubuntu 22.04 (download from Proxmox template list)
Disk:       32GB
CPU:        4 cores
RAM:        8192 MB  (Ollama process overhead — models live in unified memory pool)
Network:    vmbr0, IP 192.168.1.25/24, Gateway 192.168.1.1
Unprivileged: YES
```

### Step 2: Bind-Mount the GPU into the LXC

After creating the container (do NOT start it yet):

```bash
# On Proxmox host — edit the LXC config
nano /etc/pve/lxc/200.conf
```

Add these lines to the config file:

```
# GPU device access for Ollama
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```

```bash
# Start the container
pct start 200
pct enter 200
```

### Step 3: Install Ollama Inside the LXC

```bash
# Inside the LXC (pct enter 200)

# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Verify GPU is visible
ls /dev/dri/
# Should show: card0  renderD128

# Configure Ollama to serve on LAN with extended context window support
mkdir -p /etc/systemd/system/ollama.service.d
cat > /etc/systemd/system/ollama.service.d/override.conf << 'EOF'
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_NUM_CTX=32768"
EOF

systemctl daemon-reload
systemctl enable ollama
systemctl start ollama

# Verify GPU is being used
ollama run phi3:mini "hello" --verbose
# Look for: "using GPU: AMD Radeon Graphics" in output
```

> **Context window note:** `OLLAMA_NUM_CTX=32768` sets the default context window to 32K
> tokens for all models. You can override per-request, or increase to `65536` / `131072`
> for 64K / 128K context. Larger context consumes more unified memory (KV cache) —
> a 32B model at 128K context uses ~15–20GB of KV cache on top of ~20GB for model weights.
> 32K is a good default; increase only if you regularly work with very large documents or
> codebases in a single session.

### Step 4: Pull Models

```bash
# Coding agents (primary worker model)
ollama pull qwen2.5-coder:32b

# Home Assistant automations (dedicated instance — see Step 5)
ollama pull qwen2.5:7b

# Fast fallback for simple tasks
ollama pull phi3:mini

ollama list
```

### Step 5: Dedicated Home Assistant Inference Instance

> **Why a separate instance?** All models share the 270 GB/s memory bandwidth.
> If Home Assistant triggers an automation while a coding agent is running,
> the HA model can get starved — causing smart home commands to lag.
>
> Running two Ollama instances on different ports gives HA a dedicated slot —
> always responsive, zero contention with agent work.

```bash
# Inside the Ollama LXC — create a second Ollama instance for Home Assistant

cat > /etc/systemd/system/ollama-ha.service << 'EOF'
[Unit]
Description=Ollama HA Instance
After=network.target

[Service]
ExecStart=/usr/local/bin/ollama serve
Environment="OLLAMA_HOST=0.0.0.0:11435"
Environment="OLLAMA_NUM_CTX=8192"
Environment="OLLAMA_MAX_LOADED_MODELS=1"
Environment="OLLAMA_NUM_PARALLEL=1"
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl enable ollama-ha
systemctl start ollama-ha

# Pull HA models into this instance
OLLAMA_HOST=localhost:11435 ollama pull qwen2.5:7b
OLLAMA_HOST=localhost:11435 ollama pull phi3:mini
```

Update Home Assistant to use port 11435:
```
Settings → Devices & Services → Ollama → Configure
Host: http://192.168.1.25:11435   ← dedicated HA port
```

Coding agents continue to use port 11434 (main Ollama instance).

### Model Allocation by Use Case

| Model | Instance | Port | RAM | Speed (Radeon 890M) |
|-------|----------|------|-----|---------------------|
| `qwen2.5-coder:32b` | Agents | 11434 | ~20GB | ~15–25 tok/s |
| `phi3:mini` | Agents | 11434 | ~2GB | ~100+ tok/s |
| `qwen2.5:7b` | HA dedicated | 11435 | ~5GB | ~50–70 tok/s |
| `phi3:mini` | HA dedicated | 11435 | ~2GB | ~100+ tok/s |

Total model footprint: ~29GB of the ~88GB available in the unified memory pool.

### Restrict LAN Access to Ollama LXC

```bash
# On Proxmox host — allow LAN to reach both Ollama ports, block everything else
iptables -A FORWARD -s 192.168.1.0/24 -d 192.168.1.25 -p tcp --dport 11434 -j ACCEPT
iptables -A FORWARD -s 192.168.1.0/24 -d 192.168.1.25 -p tcp --dport 11435 -j ACCEPT
iptables -A FORWARD -d 192.168.1.25 -j DROP
apt install -y iptables-persistent
netfilter-persistent save
```

---

## 4.5 Multi-Agent Configuration

OpenClaw's agent config points the main orchestrator at the NVIDIA cloud model
and coding workers at local Ollama. Coding agents (Claude Code, Codex) are spawned
directly by OpenClaw inside the NemoClaw sandbox via its built-in ACP harness.

Edit `/home/ubuntu/.openclaw/openclaw.json` inside the NemoClaw sandbox:

```bash
nemoclaw my-assistant connect
sandbox$ nano ~/.openclaw/openclaw.json
```

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "nvidia/nemotron-3-super-120b-a12b"
      }
    },
    "list": [
      {
        "id": "main",
        "default": true
      },
      {
        "id": "coder",
        "model": {
          "primary": "ollama/qwen2.5-coder:32b"
        }
      },
      {
        "id": "fast",
        "model": {
          "primary": "ollama/phi3:mini"
        }
      }
    ]
  },
  "models": {
    "providers": {
      "nvidia": {
        "baseUrl": "https://integrate.api.nvidia.com/v1",
        "api": "openai-completions"
      },
      "ollama": {
        "baseUrl": "http://192.168.1.25:11434",
        "api": "ollama"
      }
    }
  }
}
```

Allow Ollama endpoint in OpenShell network policy:

```bash
# On NemoClaw VM host (not inside sandbox)
openshell policy add --sandbox my-assistant \
  --host 192.168.1.25 --port 11434 --protocol rest --name ollama-local
```

---

## 4.6 Media Server

Plex and Jellyfin both support AMD hardware transcoding via VAAPI on Linux.

### GPU Access for Hardware Transcoding

The AMD Radeon 890M is already bind-mounted into the Ollama LXC. Full PCIe passthrough
to a VM gives that VM **exclusive** GPU ownership — it disappears from the host and LXC
entirely, breaking Ollama inference. There is no shared-passthrough option for QEMU VMs
on this GPU.

**Recommended: Convert Media Server to an LXC container.**

LXC containers share the host kernel and support the same `/dev/dri` bind-mount used by
the Ollama LXC. Both containers can use the GPU simultaneously for their respective tasks
(inference and transcoding). Jellyfin runs perfectly in an LXC with lower overhead than
a full VM.

To convert: in Proxmox, delete the Media Server VM and create a new LXC container
(Ubuntu 22.04, same IP/resources). Then add the same GPU bind-mount lines to its config:

```bash
nano /etc/pve/lxc/201.conf
```

```
# GPU device access for VAAPI transcoding (same as Ollama LXC)
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```

> **Keeping it as a VM?** You can still run Jellyfin in the VM — it will use software
> transcoding instead of hardware acceleration. For personal use with a few streams,
> software transcoding on 4 vCPUs is generally fine. Hardware transcoding only becomes
> necessary when handling many simultaneous streams or heavy 4K remuxing.

### Install Jellyfin (Free, Recommended)

```bash
ssh ubuntu@192.168.1.22

# Add Jellyfin repo
curl -fsSL https://repo.jellyfin.org/install-debuntu.sh | sudo bash

sudo systemctl enable jellyfin
sudo systemctl start jellyfin
```

Access Jellyfin: `http://192.168.1.22:8096`

### Connect Media Storage

```bash
# USB external drive
sudo mkdir -p /media/library
sudo mount /dev/sdb1 /media/library

# Make permanent
echo '/dev/sdb1 /media/library ext4 defaults 0 2' | sudo tee -a /etc/fstab
```

In Jellyfin: Add Library → point at `/media/library/Movies`, `/media/library/TV`, etc.

### Enable Hardware Transcoding in Jellyfin

```
Dashboard → Playback → Transcoding
Hardware acceleration: Video Acceleration API (VAAPI)
VAAPI Device: /dev/dri/renderD128
Enable all codec checkboxes
```

---

## 4.7 Home Assistant (Raspberry Pi)

### Flash Home Assistant OS

1. Download HAOS for Raspberry Pi 4 (64-bit) from [home-assistant.io](https://www.home-assistant.io/installation/raspberrypi)
2. Flash with Balena Etcher to microSD
3. Insert SD, connect Pi via ethernet, power on
4. Navigate to `http://homeassistant.local:8123` after ~5 minutes
5. Complete onboarding wizard

### Set Static IP

In Google Nest app: **Devices → Pi → Reserve IP → 192.168.1.30**

### Google Coral USB Setup

```bash
# Plug Coral USB into Pi
# HAOS detects it automatically — verify in Frigate settings
lsusb
# Should show: Google Inc. Coral USB Accelerator
```

### Install Frigate NVR Add-on

```
Settings → Add-ons → Add-on Store → Search "Frigate" → Install
```

```yaml
# /config/frigate.yml
detectors:
  coral:
    type: edgetpu
    device: usb

cameras:
  front_door:
    ffmpeg:
      inputs:
        - path: rtsp://YOUR_CAMERA_IP/stream
          roles: [detect, record]
    detect:
      width: 1280
      height: 720
      fps: 5
    objects:
      track: [person, car, dog]
    record:
      enabled: true
      retain:
        days: 7
```

### Connect Ollama to Home Assistant

```
Settings → Devices & Services → Add Integration → Search "Ollama"
Host: http://192.168.1.25:11435   ← dedicated HA port
Model: qwen2.5:7b
```

This creates a local LLM Conversation Agent for voice commands and AI automations.
The dedicated HA Ollama instance (port 11435) ensures smart home responses are always
fast, regardless of what coding agents are doing on the main instance (port 11434).

### Nabu Casa Remote Access

```
Settings → Home Assistant Cloud → Sign In
Subscribe at nabucasa.com ($6.50/mo)
Click Connect
```

Remote access to HA from anywhere — no port forwarding required.

### Sample AI Automation

```yaml
alias: AI Front Door Alert
trigger:
  - platform: state
    entity_id: binary_sensor.front_door_person_detected
    to: "on"
action:
  - service: conversation.process
    data:
      agent_id: conversation.ollama
      text: >
        A person was detected at the front door at {{ now().strftime('%I:%M %p') }}.
        Write a brief, friendly 1-sentence security notification.
    response_variable: ai_response
  - service: notify.mobile_app_your_phone
    data:
      title: "🚪 Front Door"
      message: "{{ ai_response.response.speech.plain.speech }}"
```

---

## 4.8 Remote Access

Everything in this setup uses **outbound-only connections**. Your Google Nest never needs
a port forwarded. From the internet's perspective, your home is a black box.

### Services & How They're Reached Remotely

| Service | Method | Router config |
|---------|--------|--------------|
| WhatsApp AI (OpenClaw) | OpenClaw cloud relay — outbound WebSocket | None |
| Home Assistant | Nabu Casa — outbound tunnel | None |
| Ollama (local LLM) | Tailscale VPN — outbound WireGuard | None |
| Jellyfin (media) | Tailscale or Jellyfin Connect | None |

---

### Tailscale: Access Your Local LLM from Anywhere

Tailscale creates an encrypted WireGuard tunnel between your devices.
Your phone can reach Ollama on your home machine as if it were sitting next to it —
over cellular, at a coffee shop, anywhere.

**Step 1: Create a Tailscale account**

Go to [tailscale.com](https://tailscale.com) and sign up. Free for personal use (up to 3 users, 100 devices).

**Step 2: Install Tailscale on the MS-S1 MAX**

```bash
# On Proxmox host (ssh root@192.168.1.20)
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
# Follow the auth URL printed in the terminal — opens browser to approve the device
```

Verify it connected:
```bash
tailscale ip -4
# Returns something like: 100.64.0.1
# This is your machine's Tailscale IP — note it down
```

**Step 3: Install Tailscale on your phone**

- **iPhone:** App Store → Tailscale → Install → Sign in with same account
- **Android:** Play Store → Tailscale → Install → Sign in with same account

Both devices will appear in your Tailscale admin console at [login.tailscale.com](https://login.tailscale.com).

**Step 4: Test the connection**

With Tailscale active on your phone (on cellular, not home WiFi):

```
Open browser on phone → http://100.64.0.1:11434/api/tags
Should return JSON list of your Ollama models
```

---

### Chat with Your Local LLM from iPhone

**Enchanted (recommended — native iOS app)**

1. Install [Enchanted](https://apps.apple.com/app/enchanted-llm/id6474268307) from App Store
2. Open Enchanted → Settings → Server URL
3. Enter: `http://100.64.0.1:11434` (your Tailscale IP)
4. Tap Connect → select a model → start chatting

**Open WebUI (browser-based — works on iPhone and Android)**

Deploy on your Media Server VM:

```bash
ssh ubuntu@192.168.1.22
docker run -d \
  -p 8080:8080 \
  -e OLLAMA_BASE_URL=http://192.168.1.25:11434 \
  --name open-webui \
  --restart unless-stopped \
  ghcr.io/open-webui/open-webui:main
```

Access via Tailscale from any browser: `http://100.64.0.1:8080`

---

### Lock Down Tailscale Access (Optional but Recommended)

In Tailscale admin console → **Access Controls**:

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["tag:phone"],
      "dst": ["tag:homeserver:11434"]
    }
  ],
  "tagOwners": {
    "tag:phone": ["autogroup:admin"],
    "tag:homeserver": ["autogroup:admin"]
  }
}
```

Tag your devices in the Tailscale admin console:
- MS-S1 MAX → `tag:homeserver`
- Your phone → `tag:phone`

---

### Phase 2: Tailscale on Mac Studio

When the Mac Studio is added, install Tailscale there too:

```bash
brew install --cask tailscale
# Open Tailscale menu bar → Log in with same account
```

Point Enchanted or Open WebUI at the Mac Studio's Tailscale IP for access to 70B+ models
from your phone — still fully private, still no port forwarding.

---

## 4.9 Security

### Network Security Checklist

- [ ] Google Nest firmware is up to date
- [ ] Nest admin password is strong (not the default)
- [ ] Guest WiFi enabled and isolated from main network
- [ ] No port forwarding rules for any AI services

### MS-S1 MAX / Proxmox Checklist

- [ ] Proxmox UI accessible from LAN only (UFW rule in place)
- [ ] SSH accessible from LAN only
- [ ] Ollama ports 11434/11435 blocked from internet (iptables rules in place)
- [ ] iptables rules persistent (iptables-persistent installed)
- [ ] Proxmox updated regularly

### NemoClaw Sandbox Checklist

- [ ] OpenShell network policy reviewed — no unnecessary hosts approved
- [ ] `openshell term` run periodically to review blocked requests
- [ ] NVIDIA API key stored in `~/.nemoclaw/credentials.json`, not in code
- [ ] OpenClaw workspace files do not contain sensitive credentials

### Raspberry Pi / Home Assistant Checklist

- [ ] HA admin account uses strong password
- [ ] HA uses Nabu Casa for remote access — no open ports
- [ ] Frigate video never uploaded to cloud
- [ ] SSH disabled on Pi unless actively needed

---

# Part 5: Phase 2 — Mac Studio Added

## 5.1 What Changes

Phase 2 adds the Mac Studio as a dedicated inference server.
**Nothing in Phase 1 is removed or modified** except the Ollama host address in config.

| | Phase 1 | Phase 2 |
|--|---------|---------|
| Orchestrator model | Nemotron 120B (NVIDIA cloud, free) | Local 70B–120B (Mac Studio) |
| Coding agent model | Qwen 2.5 Coder 32B (Radeon 890M) | Qwen 2.5 Coder 72B (Mac Studio) |
| HA model | Qwen 2.5 7B (Radeon 890M) | Qwen 2.5 7B (Mac Studio, faster) |
| Parallel agents | 1–2 before slowdown | 4–8 simultaneous |
| Privacy | Cloud for main agent | 100% local |
| Rate limits | NVIDIA free tier applies | None |
| Max model size | 32B practical | 200B+ |
| Context window | 32K default | 128K+ comfortable |

---

## 5.2 Mac Studio Setup

### Hardware Requirements

- Mac Studio M5 Ultra (256GB unified memory) — **recommended**
- Mac Studio M4 Ultra (192GB unified memory) — available now, good alternative
- Connected via **ethernet** to Google Nest router
- Static IP: 192.168.1.10 (set DHCP reservation in Nest app)

### macOS Security Hardening

```
System Settings → Privacy & Security → FileVault → Turn On
System Settings → Network → Firewall → Turn On → Enable Stealth Mode
System Settings → General → Sharing → Remote Login: OFF
System Settings → General → Sharing → Screen Sharing: OFF
```

### Install Ollama

```bash
brew install --cask ollama
```

### Configure Ollama to Serve LAN with Extended Context

```bash
# Edit Ollama launchd service
sudo nano /Library/LaunchDaemons/com.ollama.ollama.plist
```

Add environment variables:
```xml
<key>EnvironmentVariables</key>
<dict>
  <key>OLLAMA_HOST</key>
  <string>0.0.0.0:11434</string>
  <key>OLLAMA_NUM_CTX</key>
  <string>65536</string>
</dict>
```

> **Context window note:** 65536 sets the default to 64K tokens. With 192–256GB unified
> memory, the Mac Studio handles 128K context (`131072`) comfortably even on 70B models.
> The KV cache for a 70B model at 128K context is ~40–50GB — well within the Mac Studio's
> memory budget alongside the model weights (~45GB).

```bash
sudo launchctl unload /Library/LaunchDaemons/com.ollama.ollama.plist
sudo launchctl load /Library/LaunchDaemons/com.ollama.ollama.plist
```

### Restrict Access with macOS pf Firewall

```bash
sudo nano /etc/pf.anchors/ollama-protect
```

```
pass in on en0 proto tcp from 192.168.1.0/24 to any port 11434
pass in on utun0 proto tcp from 100.64.0.0/10 to any port 11434
block in proto tcp to any port 11434
```

```bash
# Add to /etc/pf.conf before final pass rule:
# anchor "ollama-protect"
# load anchor "ollama-protect" from "/etc/pf.anchors/ollama-protect"

sudo pfctl -e
sudo pfctl -f /etc/pf.conf
```

### Pull Models

```bash
# Primary coding model
ollama pull qwen2.5-coder:72b

# General reasoning / orchestration
ollama pull llama3.1:70b

# Deep reasoning / complex architecture
ollama pull deepseek-r1:70b

# Home Assistant (fast)
ollama pull qwen2.5:7b

ollama list
```

### Tailscale for Phone Access

```bash
brew install --cask tailscale
# Open Tailscale → Log in with same account as phone
```

---

## 5.3 Switch Inference

One config change per VM. No restarts beyond the config reload.

### NemoClaw VM

```bash
nemoclaw my-assistant connect
sandbox$ nano ~/.openclaw/openclaw.json
```

Change Ollama baseUrl to Mac Studio:

```json
"ollama": {
  "baseUrl": "http://192.168.1.10:11434",
  "api": "ollama"
}
```

Optionally switch main agent from NVIDIA cloud to local:

```json
"defaults": {
  "model": {
    "primary": "ollama/llama3.1:70b"
  }
}
```

Update OpenShell policy to allow Mac Studio endpoint:

```bash
# On NemoClaw VM host
openshell policy add --sandbox my-assistant \
  --host 192.168.1.10 --port 11434 --protocol rest --name mac-studio-ollama
```

### Home Assistant

```
Settings → Devices & Services → Ollama → Configure
Host: http://192.168.1.10:11434
Model: qwen2.5:7b
```

> In Phase 2, HA talks directly to the Mac Studio Ollama on port 11434. The dedicated
> HA instance on port 11435 (Ollama LXC) can be kept as a fallback or decommissioned.

---

## 5.4 New Model Capabilities in Phase 2

### Recommended Model Stack

| Agent | Phase 1 Model | Phase 2 Model | Why upgrade |
|-------|--------------|--------------|-------------|
| Orchestrator | Nemotron 120B (cloud) | llama3.1:70b or deepseek-r1:70b (local) | Private, no rate limits |
| Coder | Qwen 2.5 Coder 32B | Qwen 2.5 Coder 72B | Noticeably better at complex tasks |
| HA | Qwen 2.5 7B | Qwen 2.5 7B (same, just faster) | 200+ tok/s vs 50–70 |
| Reasoning | — | DeepSeek-R1 70B | Multi-step architectural decisions |

### Model Sizing Guide (M5 Ultra 256GB)

| Model | Size | RAM needed | Speed | Best for |
|-------|------|-----------|-------|---------|
| qwen2.5:7b | 7B | 5GB | 200+ tok/s | HA, simple tasks |
| llama3.1:70b | 70B | 45GB | 55–75 tok/s | General orchestration |
| qwen2.5-coder:72b | 72B | 46GB | 45–65 tok/s | Coding agents |
| deepseek-r1:70b | 70B | 45GB | 40–60 tok/s | Complex reasoning |
| nemotron-ultra:253b | 253B | ~160GB | 20–35 tok/s | Maximum capability |

With 256GB you can run 2–3 large models simultaneously for true parallel agent swarms.

### Context Window in Phase 2

| Model | Default context | Comfortable max | KV cache at max |
|-------|----------------|----------------|-----------------|
| qwen2.5:7b | 32K | 128K | ~5GB |
| llama3.1:70b | 64K | 128K | ~40GB |
| qwen2.5-coder:72b | 32K | 128K | ~42GB |
| deepseek-r1:70b | 64K | 128K | ~40GB |

---

# Part 5.5: Phase 3 — Network-Level Security Hardening

## Why Phase 3

Phase 1 isolation is handled by Proxmox internally.
Phase 2 adds the Mac Studio protected by its own `pf` firewall.

Phase 3 upgrades isolation from **machine-enforced** to **network-enforced** — a separate
layer of defense that operates even if a VM or OS-level control is bypassed.

## Additional Hardware Required

| Item | Model | Price | Notes |
|------|-------|-------|-------|
| **VLAN-capable router** | UniFi Express | ~$150 | Replaces Google Nest. Router + managed switch in one. |

## VLAN Design

| VLAN | Name | Subnet | Devices | Internet | Cross-VLAN |
|------|------|--------|---------|----------|-----------|
| 1 | Trusted | 192.168.1.0/24 | MS-S1 MAX, Pi, phones, laptops | Yes | Full |
| 20 | Inference | 192.168.2.0/24 | Mac Studio only | No | VLAN 1 → port 11434 only |
| 30 | IoT | 192.168.3.0/24 | Smart devices, cameras | No | None |

## UniFi Express Setup

### Create VLANs

```
UniFi Dashboard → Networks → Create New Network

Network: Inference
VLAN ID: 20
Subnet: 192.168.2.1/24
DHCP: Enabled

Network: IoT
VLAN ID: 30
Subnet: 192.168.3.1/24
DHCP: Enabled
```

### Assign Devices to VLANs

```
UniFi Dashboard → Devices → [device] → Port → Network

Mac Studio  → VLAN 20 (Inference)
MS-S1 MAX   → VLAN 1 (Trusted)
Pi          → VLAN 1 (Trusted)
IoT devices → VLAN 30
```

### Firewall Rules

```
UniFi Dashboard → Firewall & Security → Rules

Rule 1: Allow VLAN 1 → VLAN 20, port 11434 (Ollama on Mac Studio)
Rule 2: Block VLAN 20 → internet (Mac Studio never touches WAN)
Rule 3: Block VLAN 20 → VLAN 1 (inference can't initiate connections back)
Rule 4: Block VLAN 30 → all (IoT can't reach anything)
Rule 5: Allow VLAN 1 → VLAN 30, port 80/443 (you can manage IoT devices)
```

### IP Assignments After Phase 3

| Device | Phase 2 IP | Phase 3 IP | VLAN |
|--------|-----------|-----------|------|
| MS-S1 MAX (Proxmox host) | 192.168.1.20 | 192.168.1.20 | 1 (unchanged) |
| NemoClaw VM | 192.168.1.21 | 192.168.1.21 | 1 (unchanged) |
| Media Server VM | 192.168.1.22 | 192.168.1.22 | 1 (unchanged) |
| Ollama LXC | 192.168.1.25 | 192.168.1.25 | 1 (unchanged) |
| Raspberry Pi | 192.168.1.30 | 192.168.1.30 | 1 (unchanged) |
| Mac Studio | 192.168.1.10 | **192.168.2.10** | **20 (Inference)** |

Update the config reference to Mac Studio's new IP:

```bash
# NemoClaw sandbox:
nemoclaw my-assistant connect
sandbox$ nano ~/.openclaw/openclaw.json
# Update ollama baseUrl to http://192.168.2.10:11434
```

Update OpenShell policy:

```bash
openshell policy add --sandbox my-assistant \
  --host 192.168.2.10 --port 11434 --protocol rest --name mac-studio-ollama-v2
```

Update Home Assistant:

```
Settings → Devices & Services → Ollama → Configure
Host: http://192.168.2.10:11434
```

## What Phase 3 Adds Over Phase 2

| Security control | Phase 2 | Phase 3 |
|-----------------|---------|---------|
| Mac Studio internet access | Blocked by macOS pf | Blocked by router (hardware) |
| IoT device isolation | None | VLAN 30, fully isolated |
| Traffic visibility | None | UniFi dashboard, per-device |
| Defense if macOS pf misconfigured | Unprotected | Router still blocks it |

---

# Part 6: Performance Comparison

## Why Performance Differs: The Memory Bandwidth Constraint

Token generation speed is limited by **memory bandwidth**, not compute.
Every token generated requires reading the entire model's weights from memory.

| Hardware | Memory | Bandwidth | Implication |
|----------|--------|-----------|-------------|
| MS-S1 MAX (Radeon 890M) | 128GB LPDDR5X | ~270 GB/s | Good for smaller models, limited parallelism |
| Mac Studio M5 Ultra | 256GB unified | ~800 GB/s | ~3× faster per token, handles multiple large models |

---

## Tokens Per Second by Task

| Task | Model | Phase 1 | Phase 2 | Feel |
|------|-------|---------|---------|------|
| Orchestration | Nemotron 120B (cloud) | 50–80 tok/s | — | Fast |
| Orchestration | Llama 3.1 70B (local) | 8–12 tok/s | 55–75 tok/s | Phase 1: slow; Phase 2: fast |
| Coding agent | Qwen 2.5 Coder 32B | 15–25 tok/s | 90–120 tok/s | Phase 1: readable; Phase 2: fast |
| Coding agent | Qwen 2.5 Coder 72B | 4–6 tok/s | 45–65 tok/s | Phase 1: painful; Phase 2: good |
| HA automations | Qwen 2.5 7B | 50–70 tok/s | 200+ tok/s | Both fine; Phase 2 near-instant |
| Simple tasks | phi3:mini (3.8B) | 150+ tok/s | 300+ tok/s | Both fast |

> **What do these speeds feel like?**
> - 5–10 tok/s → watching words appear slowly, like dial-up
> - 15–25 tok/s → readable, like fast typing speed
> - 50–80 tok/s → feels instant for short responses
> - 100+ tok/s → effectively instant for everything

---

## Concurrent Models: What Fits Simultaneously

### Phase 1 — MS-S1 MAX (128GB, ~88GB available for models)

| Scenario | Models loaded | RAM used | Concurrent agents | Notes |
|----------|--------------|----------|------------------|-------|
| Minimal | phi3:mini only | ~2GB | 2–3 | Fast but low quality |
| Typical | qwen2.5-coder:32b + qwen2.5:7b (HA) | ~27GB | 1–2 agents | Good balance |
| Heavy | qwen2.5-coder:32b + 32K context | ~38GB | 1–2 agents | Comfortable |
| Extended context | qwen2.5-coder:32b + 128K context | ~40GB | 1 agent | Max context, still fits |

### Phase 2 — Mac Studio M5 Ultra (256GB)

| Scenario | Models loaded | RAM used | Concurrent agents | Notes |
|----------|--------------|----------|------------------|-------|
| Standard | qwen2.5-coder:72b + llama3.1:70b + qwen2.5:7b | ~97GB | 4–5 | Comfortable |
| Full swarm | 4× qwen2.5-coder:32b instances | ~82GB | 4 simultaneous | Each at full speed |
| Maximum | nemotron-ultra:253b alone | ~160GB | 1–2 | Largest available model |
| Long context | llama3.1:70b at 128K | ~85GB | 3–4 | Model + KV cache |

---

## Is Phase 2 Worth It?

**Yes, if you:**
- Run 3+ coding agents in parallel regularly
- Work on complex multi-file or multi-repo codebases
- Want 100% private inference (nothing to cloud)
- Hit NVIDIA rate limits on heavy agentic workflows
- Want 70B+ models at usable speed
- Want 64K–128K context windows as a comfortable default

**Not yet, if you:**
- Use this primarily as a personal assistant (Nemotron 120B handles this well)
- Run single coding agent tasks occasionally
- Are satisfied with 32K context for your workloads

**Recommendation:** Live in Phase 1 for 2–3 months.
If you find yourself waiting on inference, wanting more parallel agents, or bumping into
context length limits, that's your signal.

---

# Part 7: Testing & Validation

## Phase 1 Health Checks

```bash
# Ollama serving on LAN
curl http://192.168.1.25:11434/api/tags
# Expected: JSON list of installed models

# HA Ollama instance
curl http://192.168.1.25:11435/api/tags
# Expected: JSON list of HA models

# From Raspberry Pi — HA can reach Ollama HA instance
curl http://192.168.1.25:11435/api/tags
# Expected: same JSON

# Ollama NOT reachable from internet (test on cellular)
curl --connect-timeout 5 http://YOUR_PUBLIC_IP:11434/api/tags
# Expected: timeout / connection refused

# NemoClaw sandbox health
nemoclaw my-assistant status
# Expected: Sandbox running, inference connected

# Verify context window setting
curl http://192.168.1.25:11434/api/show -d '{"name":"qwen2.5-coder:32b"}' | grep num_ctx
# Expected: 32768 (or your configured value)
```

## End-to-End WhatsApp Test

1. Send WhatsApp message to linked number: *"What's 7 times 8?"*
2. Should receive response within 5–10 seconds
3. Watch NemoClaw logs: `nemoclaw my-assistant logs -f`
4. Confirm model used: `openclaw nemoclaw status --json`

## Phase 2 Additional Checks

```bash
# Mac Studio Ollama serving on LAN
curl http://192.168.1.10:11434/api/tags

# Context window on Mac Studio
curl http://192.168.1.10:11434/api/show -d '{"name":"llama3.1:70b"}' | grep num_ctx
# Expected: 65536 (or your configured value)

# Mac Studio Ollama NOT on internet (test from cellular)
curl --connect-timeout 5 http://YOUR_PUBLIC_IP:11434/api/tags
# Expected: timeout

# NemoClaw using Mac Studio
openclaw nemoclaw status --json
# Confirm endpoint shows 192.168.1.10
```

---

# Part 8: Maintenance

## Weekly

```bash
# Update Ollama models (pulls latest versions)
ollama pull qwen2.5-coder:32b
ollama pull qwen2.5:7b

# Check NemoClaw sandbox
nemoclaw my-assistant status
openshell term   # Review any blocked network requests

# Check both Ollama instances are running
systemctl status ollama
systemctl status ollama-ha
```

## Monthly

```bash
# Update Proxmox host
ssh root@192.168.1.20
apt update && apt full-upgrade -y

# Update all VMs
for ip in 192.168.1.21 192.168.1.22; do
  ssh ubuntu@$ip "sudo apt update && sudo apt upgrade -y"
done

# Update Ollama LXC
pct enter 200
apt update && apt upgrade -y

# Rotate NVIDIA API key (if concerned)
# build.nvidia.com → API Keys → Generate New
# Update: nemoclaw onboard
```

## Proxmox Backups

```
Datacenter → Backup → Add
Schedule: Sunday 02:00
Storage: local
VMs: 100 (nemoclaw), 101 (media-server)
LXC: 200 (ollama)
Mode: Snapshot
Compression: ZSTD
```

## Phase 2: Mac Studio Model Updates

```bash
# Update all models
ollama list | awk 'NR>1 {print $1}' | while read model; do
  echo "Updating $model..."
  ollama pull "$model"
done
```

---

> **Quick Reference — IP Addresses**
>
> | Device | Phase 1/2 IP | Phase 3 IP | Notes |
> |--------|-------------|-----------|-------|
> | Google Nest Router | 192.168.1.1 | replaced by UniFi | — |
> | MS-S1 MAX (Proxmox host) | 192.168.1.20 | 192.168.1.20 | VLAN 1 |
> | NemoClaw VM | 192.168.1.21 | 192.168.1.21 | VLAN 1 |
> | Media Server VM | 192.168.1.22 | 192.168.1.22 | VLAN 1 |
> | Ollama LXC | 192.168.1.25 | 192.168.1.25 | VLAN 1 |
> | Raspberry Pi (HA) | 192.168.1.30 | 192.168.1.30 | VLAN 1 |
> | Mac Studio (Phase 2) | 192.168.1.10 | 192.168.2.10 | VLAN 20 (Inference) |

---

*Architecture version: 2.1 — Removed Agent Swarm VM (OpenClaw handles agent orchestration natively). Fixed GPU sharing. Added context window configuration. Cleaned Phase 3 tables.*
*Hardware: Minisforum MS-S1 MAX + Raspberry Pi 4 + (Phase 2) Apple Mac Studio M5 Ultra*
*Networking: Google Nest compatible — no VLANs, no managed switch required in Phase 1/2*
