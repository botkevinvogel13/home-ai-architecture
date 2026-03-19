# Home AI Infrastructure Architecture
### A Private, Powerful, Always-On AI Stack — Built in Your Home

> **Version:** 2.0 | **Status:** Active  
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
  - [5.1 What Changes](#51-what-changes)
  - [5.2 Mac Studio Setup](#52-mac-studio-setup)
  - [5.3 Switch Inference](#53-switch-inference)
  - [5.4 New Model Capabilities](#54-new-model-capabilities)
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
- Orchestrator agent (large, smart model) breaks down tasks and delegates
- Worker agents (fast, specialized model) execute code, write tests, open PRs
- Phase 1: 1–2 parallel agents at ~15–25 tok/s
- Phase 2: 4–8 parallel agents at ~90–120 tok/s each

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
| **Security** | NemoClaw OpenShell sandbox isolates the AI agent. Agent Swarm VM isolated via internal Proxmox bridge. No public inbound ports. |
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
║  │  ┌──────────────────────┐  ┌──────────────────────────────┐  │  ║
║  │  │  NemoClaw VM         │  │  Agent Swarm VM              │  │  ║
║  │  │  192.168.1.21        │  │  internal only (no LAN IP)   │  │  ║
║  │  │                      │  │                              │  │  ║
║  │  │  OpenClaw agent      │  │  Coding agents               │  │  ║
║  │  │  OpenShell sandbox   │  │  Claude Code / Codex         │  │  ║
║  │  │  WhatsApp gateway    │  │  OpenHands                   │  │  ║
║  │  │                      │  │  → Ollama (local)            │  │  ║
║  │  │  Main model:         │  │  → Qwen 2.5 Coder 32B        │  │  ║
║  │  │  Nemotron 120B cloud │  │                              │  │  ║
║  │  │  Worker model:       │  └──────────────────────────────┘  │  ║
║  │  │  Qwen 2.5 Coder 32B  │                                    │  ║
║  │  └──────────────────────┘  ┌──────────────────────────────┐  │  ║
║  │                            │  Media Server VM             │  │  ║
║  │  ┌──────────────────────┐  │  192.168.1.22               │  │  ║
║  │  │  Ollama (host)       │  │  Plex / Jellyfin             │  │  ║
║  │  │  Radeon 890M (40 CU) │  │  AMD HW transcoding         │  │  ║
║  │  │  Qwen 2.5 Coder 32B  │◄─┤  → external USB / NAS       │  │  ║
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
║  │  → Ollama on MS-S1   │                                           ║
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
║  │  Agent Swarm VM ───────────────┼─►│  70B / 120B / 200B+      │  ║
║  │  Media Server VM (unchanged)  │  │  models                   │  ║
║  │                               │  │                           │  ║
║  │  Ollama still runs locally    │  │  256GB unified memory     │  ║
║  │  (fallback / small/fast tasks)│  │  4–8 parallel agents      │  ║
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
All isolation is handled internally by Proxmox.

```
Google Nest Router (192.168.1.1)
    │
    ├── MS-S1 MAX Proxmox host (192.168.1.20)  ← one device on LAN
    │       └── VMs with LAN IPs get DHCP reservations from Nest
    │
    ├── Raspberry Pi (192.168.1.30)
    ├── Mac Studio — Phase 2 (192.168.1.10)
    └── Your phones, laptops, etc.

Inside Proxmox (never touches the router):
    vmbr1 (internal bridge)
        └── Agent Swarm VM (192.168.20.10)  ← no external IP, fully isolated
```

### Static IP Assignments

Set DHCP reservations in Google Nest app by MAC address:

| Device | IP | How to set |
|--------|----|-----------| 
| MS-S1 MAX (Proxmox) | 192.168.1.20 | Nest app → Devices → Reserve IP |
| Raspberry Pi | 192.168.1.30 | Nest app → Devices → Reserve IP |
| Mac Studio (Phase 2) | 192.168.1.10 | Nest app → Devices → Reserve IP |

VMs inside Proxmox (NemoClaw VM, Media Server VM) get their IPs assigned by the Proxmox DHCP
or set statically in each VM's netplan config.

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
| Ollama | MS-S1 MAX host + Mac Studio (Phase 2) | Free |
| Plex or Jellyfin | Media Server VM | Free (Jellyfin) / Freemium (Plex) |
| Home Assistant OS | Raspberry Pi | Free |
| Nabu Casa | Cloud | $6.50/mo |
| Tailscale | MS-S1 MAX + Phone | Free (personal) |
| Docker CE | Agent Swarm VM | Free |

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

Three VMs run on the MS-S1 MAX. Resource allocation from 128GB total:

| VM | vCPU | RAM | Disk | Network | Purpose |
|----|------|-----|------|---------|---------|
| NemoClaw VM | 4 | 16GB | 80GB | vmbr0 (LAN) | OpenClaw + NemoClaw sandbox |
| Agent Swarm VM | 8 | 32GB | 200GB | vmbr1 (internal only) | Coding agents |
| Media Server VM | 4 | 8GB | 100GB | vmbr0 (LAN) | Plex/Jellyfin |
| **Proxmox host** | — | ~8GB | — | — | OS overhead + Ollama |
| **Available for Ollama** | 4 | ~64GB | — | — | Model inference |

> Ollama runs on the **Proxmox host** (bare metal), not inside a VM, so the AMD Radeon 890M
> GPU is directly accessible without GPU passthrough complexity.

### Create Internal Bridge for Agent Swarm

In Proxmox UI: **Node → System → Network → Create → Linux Bridge**

```
Name:         vmbr1
IP:           192.168.20.1/24
Bridge ports: (none — internal only)
Comment:      Agent Swarm internal network
```

Enable NAT so Agent Swarm can reach the internet (for git, packages) but not LAN:

```bash
# On Proxmox host:
echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf
sysctl -p

# NAT for Agent Swarm internet access
iptables -t nat -A POSTROUTING -s 192.168.20.0/24 -o vmbr0 -j MASQUERADE

# Block Agent Swarm from reaching LAN (except Ollama port)
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -s 192.168.20.0/24 -d 192.168.1.20 -p tcp --dport 11434 -j ACCEPT
iptables -A FORWARD -s 192.168.20.0/24 -d 192.168.1.0/24 -j DROP
iptables -A FORWARD -s 192.168.20.0/24 -p tcp -m multiport --dports 80,443 -j ACCEPT
iptables -A FORWARD -s 192.168.20.0/24 -j DROP

apt install -y iptables-persistent
netfilter-persistent save
```

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

**Agent Swarm VM:**
```
VM ID: 101  |  Name: agent-swarm
Disk:  200GB VirtIO SCSI
CPU:   8 cores, type: host
RAM:   32768 MB
Net:   vmbr1, VirtIO   ← internal only
```

**Media Server VM:**
```
VM ID: 102  |  Name: media-server
Disk:  100GB VirtIO SCSI
CPU:   4 cores, type: host
RAM:   8192 MB
Net:   vmbr0, VirtIO
```

For each VM, install Ubuntu 22.04 and set a static IP:

```yaml
# /etc/netplan/00-installer-config.yaml (NemoClaw VM example)
network:
  version: 2
  ethernets:
    ens18:
      addresses: [192.168.1.21/24]      # 192.168.1.22 for media, 192.168.20.10 for agent swarm
      routes:
        - to: default
          via: 192.168.1.1              # 192.168.20.1 for agent swarm
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

### Prerequisites on the NemoClaw VM

```bash
ssh ubuntu@192.168.1.21

# Docker (required by NemoClaw/OpenShell)
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

Ollama runs on the **Proxmox host** (not a VM) for direct GPU access.

```bash
# On Proxmox host (ssh root@192.168.1.20)

# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Bind to all interfaces so VMs and LAN devices can reach it
mkdir -p /etc/systemd/system/ollama.service.d
cat > /etc/systemd/system/ollama.service.d/override.conf << 'EOF'
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
EOF

systemctl daemon-reload
systemctl enable ollama
systemctl restart ollama
```

### Pull Models

```bash
# Coding agents (primary worker model)
ollama pull qwen2.5-coder:32b

# Home Assistant automations (fast, lightweight)
ollama pull qwen2.5:7b

# Very fast fallback for simple tasks
ollama pull phi3:mini

# Verify
ollama list
```

### Model Allocation by Use Case

| Model | Use | RAM needed | Speed (Radeon 890M) |
|-------|-----|-----------|---------------------|
| `qwen2.5-coder:32b` | Coding agents | ~20GB | ~15–25 tok/s |
| `qwen2.5:7b` | Home Assistant | ~5GB | ~50–70 tok/s |
| `phi3:mini` | Fast responses | ~2GB | ~100+ tok/s |

Total model footprint: ~27GB of the 128GB unified memory pool.

### Restrict LAN Access to Ollama

Ollama has no auth. Restrict with iptables so only your devices can reach port 11434:

```bash
# On Proxmox host — allow LAN + VMs, block everything else
iptables -A INPUT -s 192.168.1.0/24 -p tcp --dport 11434 -j ACCEPT
iptables -A INPUT -s 192.168.20.0/24 -p tcp --dport 11434 -j ACCEPT
iptables -A INPUT -p tcp --dport 11434 -j DROP
netfilter-persistent save
```

---

## 4.5 Multi-Agent Configuration

OpenClaw's agent config points the main orchestrator at the NVIDIA cloud model
and coding workers at local Ollama.

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
        "baseUrl": "http://192.168.1.20:11434",
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
  --host 192.168.1.20 --port 11434 --protocol rest --name ollama-local
```

### Agent Swarm VM Setup

```bash
ssh ubuntu@192.168.20.10

# Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER && newgrp docker

# Node.js
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs python3 python3-pip

# Claude Code CLI
npm install -g @anthropic-ai/claude-code

# OpenAI Codex CLI
npm install -g @openai/codex

# Point all agents at local Ollama
echo 'OLLAMA_HOST=http://192.168.1.20:11434' | sudo tee -a /etc/environment
echo 'OPENAI_BASE_URL=http://192.168.1.20:11434/v1' | sudo tee -a /etc/environment
echo 'OPENAI_API_KEY=ollama' | sudo tee -a /etc/environment
source /etc/environment
```

### OpenHands (Self-Hosted Coding Agent UI)

```bash
# On Agent Swarm VM
mkdir -p ~/openhands && cd ~/openhands
cat > docker-compose.yml << 'EOF'
services:
  openhands:
    image: ghcr.io/all-hands-ai/openhands:main
    ports:
      - "3000:3000"
    environment:
      - LLM_API_KEY=ollama
      - LLM_BASE_URL=http://192.168.1.20:11434
      - LLM_MODEL=ollama/qwen2.5-coder:32b
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ~/.openhands-state:/.openhands-state
    restart: unless-stopped
EOF
docker compose up -d
# Access: http://192.168.1.20:3000 (via Proxmox port forward) or directly
```

---

## 4.6 Media Server

Plex and Jellyfin both support AMD hardware transcoding via VAAPI on Linux.

### GPU Passthrough to Media Server VM

For hardware transcoding, the AMD Radeon 890M needs to be accessible inside the VM.
Use Proxmox's VAAPI device passthrough (simpler than full GPU passthrough):

```bash
# On Proxmox host — find the render device
ls /dev/dri/
# Should show: card0, renderD128

# In Proxmox UI: Media Server VM → Hardware → Add → PCI Device
# Select: AMD Radeon 890M
# Check: All Functions, Primary GPU: NO
```

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
Host: http://192.168.1.20:11434
Model: qwen2.5:7b
```

This creates a local LLM Conversation Agent for voice commands and AI automations.

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

### Tailscale: Access Ollama from Phone

```bash
# On Proxmox host
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Install Tailscale on phone → sign in with same account.

Your phone now reaches Ollama at `http://100.x.x.x:11434` (Tailscale IP) from anywhere.

**Enchanted app (iOS):** Settings → Server URL → `http://100.x.x.x:11434`
**Open WebUI:** Deploy on Media Server VM and access via Tailscale IP.

### No Port Forwarding Needed

| Service | Remote Access Method | Router Config Needed |
|---------|---------------------|---------------------|
| WhatsApp AI | OpenClaw cloud relay (outbound WS) | None |
| Home Assistant | Nabu Casa (outbound) | None |
| Ollama from phone | Tailscale VPN (outbound) | None |
| Jellyfin | Tailscale or Jellyfin Connect | None |

Your router never needs an inbound port opened.

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
- [ ] Ollama port 11434 blocked from internet (iptables rule in place)
- [ ] Agent Swarm VM cannot reach NemoClaw VM or other LAN devices (iptables DROP rule)
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
# Or: curl -fsSL https://ollama.com/install.sh | sh
```

### Configure Ollama to Serve LAN

```bash
# Edit Ollama launchd service
sudo nano /Library/LaunchDaemons/com.ollama.ollama.plist
```

Add environment variable for host binding:
```xml
<key>EnvironmentVariables</key>
<dict>
  <key>OLLAMA_HOST</key>
  <string>0.0.0.0:11434</string>
</dict>
```

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
# Load anchor (add to /etc/pf.conf before final pass rule):
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

# List installed
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

Change Ollama baseUrl from `192.168.1.20` to `192.168.1.10`:

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

### Agent Swarm VM

```bash
# Update environment
sudo sed -i 's/192.168.1.20/192.168.1.10/g' /etc/environment
source /etc/environment
# Restart any running agent services
```

### Home Assistant

```
Settings → Devices & Services → Ollama → Configure
Host: http://192.168.1.10:11434
Model: qwen2.5:7b
```

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

---

---

# Part 5.5: Phase 3 — Network-Level Security Hardening

## Why Phase 3

Phase 1 isolation is handled by Proxmox internally — the agent swarm has no LAN IP and
can't reach the Mac Studio. Phase 2 adds the Mac Studio protected by its own `pf` firewall.

Phase 3 upgrades isolation from **machine-enforced** to **network-enforced** — a separate
layer of defense that operates even if a VM or OS-level control is bypassed.

The trigger for Phase 3 is adding the Mac Studio (Phase 2). At that point you have two
physical machines on your LAN, and belt-and-suspenders security means the router itself
enforces what can talk to what — independent of any software on either machine.

## Additional Hardware Required

| Item | Model | Price | Notes |
|------|-------|-------|-------|
| **VLAN-capable router** | UniFi Express | ~$150 | Replaces Google Nest. Router + managed switch in one. VLAN, firewall rules, traffic monitoring. |

The UniFi Express is the cleanest single-device upgrade. It replaces your Google Nest entirely
and adds full VLAN support, per-device firewall rules, and a monitoring dashboard.

## What Changes

Your Google Nest is replaced by the UniFi Express. Everything else — MS-S1 MAX, Pi,
Mac Studio — stays identical. You just plug them into the UniFi instead of the Nest.

## VLAN Design

| VLAN | Name | Subnet | Devices | Internet | Cross-VLAN |
|------|------|--------|---------|----------|-----------|
| 1 | Trusted | 192.168.1.0/24 | MS-S1 MAX, Mac Studio, Pi, phones, laptops | Yes | Full |
| 20 | Inference | 192.168.2.0/24 | Mac Studio only | No | VLAN 1 → port 11434 only |
| 30 | IoT | 192.168.3.0/24 | Smart devices, cameras | No | None |

> **Mac Studio on VLAN 20 (Inference):** No internet access at all. Can only receive
> connections on port 11434 from VLAN 1 devices. Even if macOS pf firewall were misconfigured,
> the router blocks everything else at the network level. Belt and suspenders.

> **IoT on VLAN 30:** Smart plugs, lights, cameras, etc. are completely isolated from your
> AI stack. A compromised IoT device cannot reach Ollama, OpenClaw, or any LAN services.

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

Mac Studio → VLAN 20 (Inference)
MS-S1 MAX  → VLAN 1 (Trusted)
Pi         → VLAN 1 (Trusted)   [or VLAN 30 if you want HA isolated]
IoT devices → VLAN 30
```

### Firewall Rules

```
UniFi Dashboard → Firewall & Security → Rules

Rule 1: Allow VLAN 1 → VLAN 20, port 11434 (Ollama)
Rule 2: Block VLAN 20 → internet (Mac Studio never touches WAN)
Rule 3: Block VLAN 20 → VLAN 1 (inference can't initiate connections back)
Rule 4: Block VLAN 30 → all (IoT can't reach anything)
Rule 5: Allow VLAN 1 → VLAN 30, port 80/443 (you can manage IoT devices)
```

### Update IP Assignments

Move Mac Studio to VLAN 20 subnet:

| Device | Old IP | New IP | VLAN |
|--------|--------|--------|------|
| MS-S1 MAX | 192.168.1.20 | 192.168.1.20 | 1 (unchanged) |
| NemoClaw VM | 192.168.1.21 | 192.168.1.21 | 1 (unchanged) |
| Media Server VM | 192.168.1.22 | 192.168.1.22 | 1 (unchanged) |
| Raspberry Pi | 192.168.1.30 | 192.168.1.30 | 1 (unchanged) |
| Mac Studio | 192.168.1.10 | **192.168.2.10** | **20 (Inference)** |

Update the single config reference to Mac Studio's new IP:

```bash
# On each VM that talks to Mac Studio:
sudo sed -i 's/192.168.1.10/192.168.2.10/g' /etc/environment
source /etc/environment
```

```bash
# NemoClaw sandbox:
nemoclaw my-assistant connect
sandbox$ nano ~/.openclaw/openclaw.json
# Update ollama baseUrl to http://192.168.2.10:11434
```

## What Phase 3 Adds Over Phase 2

| Security control | Phase 2 | Phase 3 |
|-----------------|---------|---------|
| Agent swarm isolation | Proxmox internal bridge | Proxmox + router VLAN |
| Mac Studio internet access | Blocked by macOS pf | Blocked by router (hardware) |
| IoT device isolation | None | VLAN 30, fully isolated |
| Traffic visibility | None | UniFi dashboard, per-device |
| Defense if macOS pf misconfigured | Unprotected | Router still blocks it |

## Network Monitoring

UniFi provides a live dashboard showing traffic per device, blocked connections, and alerts.

```
UniFi Dashboard → Insights → Traffic
→ See exactly what each device is sending/receiving
→ Alert on unexpected outbound connections from Mac Studio or MS-S1 MAX
```

This is the security posture of a small business — appropriate for a setup handling
autonomous agents with internet access and sensitive personal data.

---


# Part 6: Performance Comparison

## Tokens Per Second

| Task | Phase 1 | Phase 2 | Delta |
|------|---------|---------|-------|
| Orchestration (main agent) | 50–80 tok/s (cloud) | 55–75 tok/s (local 70B) | Similar speed, private |
| Coding agent | 15–25 tok/s (32B local) | 45–65 tok/s (72B local) | **2–4× faster and smarter** |
| HA automations | 50–70 tok/s (7B local) | 200+ tok/s (7B on M5 Ultra) | **3–4× faster** |
| Parallel agents | 1–2 before queuing | 4–8 simultaneous | **Swarm scale** |

## Coding & Agentic Workflow Quality

| Capability | Phase 1 | Phase 2 |
|-----------|---------|---------|
| Single file edits | Excellent | Excellent |
| Multi-file refactoring | Good | Excellent |
| System architecture decisions | Adequate (32B limit) | Strong (70B reasoning) |
| Long context (200K+ tokens) | Degraded at 32B | Solid at 70B+ |
| Parallel agent swarms | 1–2 agents | 4–8 agents |
| Privacy | Main agent uses cloud | 100% on-premises |
| Rate limits | NVIDIA free tier | None |

## Is Phase 2 Worth It?

**Yes, if you:**
- Run 3+ coding agents in parallel regularly
- Work on complex multi-file or multi-repo codebases
- Want 100% private inference (nothing to cloud)
- Hit NVIDIA rate limits on heavy agentic workflows

**Not yet, if you:**
- Use this primarily as a personal assistant
- Run single coding agent tasks occasionally
- Are satisfied with Phase 1 performance

**Recommendation:** Live in Phase 1 for 2–3 months. If you find yourself waiting on inference
or hitting rate limits, that's the signal to add the Mac Studio.

---

# Part 7: Testing & Validation

## Phase 1 Health Checks

```bash
# Ollama serving on LAN
curl http://192.168.1.20:11434/api/tags
# Expected: JSON list of installed models

# From Raspberry Pi — HA can reach Ollama
curl http://192.168.1.20:11434/api/tags
# Expected: same JSON

# Ollama NOT reachable from internet (test on cellular)
curl --connect-timeout 5 http://YOUR_PUBLIC_IP:11434/api/tags
# Expected: timeout / connection refused

# Agent Swarm cannot reach NemoClaw VM
ssh ubuntu@192.168.20.10
curl --connect-timeout 3 http://192.168.1.21:18789
# Expected: timeout (blocked by iptables)

# Agent Swarm CAN reach Ollama
curl http://192.168.1.20:11434/api/tags
# Expected: model list

# NemoClaw sandbox health
nemoclaw my-assistant status
# Expected: Sandbox running, inference connected
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

# Update Agent Swarm Docker images
ssh ubuntu@192.168.20.10
docker compose pull && docker compose up -d
```

## Monthly

```bash
# Update Proxmox host
ssh root@192.168.1.20
apt update && apt full-upgrade -y

# Update all VMs
for ip in 192.168.1.21 192.168.1.22 192.168.20.10; do
  ssh ubuntu@$ip "sudo apt update && sudo apt upgrade -y"
done

# Rotate NVIDIA API key (if concerned)
# build.nvidia.com → API Keys → Generate New
# Update: nemoclaw onboard
```

## Proxmox Backups

```
Datacenter → Backup → Add
Schedule: Sunday 02:00
Storage: local
VMs: 100 (nemoclaw), 101 (agent-swarm), 102 (media-server)
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
> | Device | IP |
> |--------|----|
> | Google Nest Router | 192.168.1.1 |
> | MS-S1 MAX (Proxmox host) | 192.168.1.20 |
> | NemoClaw VM | 192.168.1.21 |
> | Media Server VM | 192.168.1.22 |
> | Agent Swarm VM | 192.168.20.10 (internal) |
> | Raspberry Pi (HA) | 192.168.1.30 |
> | Mac Studio (Phase 2) | 192.168.1.10 |

---

*Architecture version: 2.0 — Coherent single-document rewrite*
*Hardware: Minisforum MS-S1 MAX + Raspberry Pi 4 + (Phase 2) Apple Mac Studio M5 Ultra*
*Networking: Google Nest compatible — no VLANs, no managed switch required*
