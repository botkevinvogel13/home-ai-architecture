# Home AI Infrastructure Architecture
### A Private, Powerful, Always-On AI Stack — Built in Your Home

> **Version:** 2.3 | **Status:** Active  
> **Hardware:** Minisforum MS-S1 MAX · (Phase 2: Apple Mac Studio)

---

## Table of Contents

- [Part 1: Requirements & Features](#part-1-requirements--features)
- [Part 2: System Overview](#part-2-system-overview)
- [Part 3: Bill of Materials](#part-3-bill-of-materials)
- [Part 4: Phase 1 — Core AI Stack](#part-4-phase-1--core-ai-stack)
  - [4.1 MS-S1 MAX: OS & Proxmox](#41-ms-s1-max-os--proxmox)
  - [4.2 Component Design](#42-component-design)
  - [4.3 NemoClaw + OpenClaw](#43-nemoclaw--openclaw)
  - [4.4 Ollama + Local Models](#44-ollama--local-models)
  - [4.5 Multi-Agent Configuration](#45-multi-agent-configuration)
  - [4.6 Remote Access](#46-remote-access)
  - [4.7 Security](#47-security)
- [Part 5: Phase 2 — Mac Studio Added](#part-5-phase-2--mac-studio-added)
- [Part 5.5: Phase 3 — Network-Level Security Hardening](#part-55-phase-3--network-level-security-hardening)
- [Part 5.6: Phase 4 — Home Assistant (Optional)](#part-56-phase-4--home-assistant-optional)
- [Part 5.7: Phase 5 — Media Server (Optional)](#part-57-phase-5--media-server-optional)
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

### AI-Powered Home Automation *(Phase 4 — Optional)*
- Home Assistant running on the MS-S1 MAX as a VM alongside the AI stack
- Local LLM for natural language automations and voice commands
- AI camera analysis via Frigate + Google Coral USB (passed through to the HA VM)
- Smart notifications (e.g. "a person has been at your door for 30 seconds")

### Media Server *(Phase 5 — Optional)*
- Stream video and music to any device on your network or remotely
- Hardware-accelerated transcoding on the AMD Radeon 890M
- Add any time — no effect on the AI stack

### Remote Access
- WhatsApp → your AI, from any phone, anywhere (via OpenClaw cloud relay)
- Phone → local LLM from anywhere via Tailscale VPN
- No inbound ports opened on your router — all outbound connections only

## Non-Functional Requirements

| Requirement | Approach |
|-------------|---------|
| **Privacy** | All AI inference runs locally. Cloud relay (OpenClaw gateway, Nabu Casa) are dumb pipes — they move bytes, never process content |
| **Security** | NemoClaw OpenShell sandbox isolates the AI agent. No public inbound ports. |
| **Always-on** | Proxmox auto-starts VMs/LXC on boot. Systemd services restart on failure. |
| **Router compatibility** | Designed for standard home routers including Google Nest. No VLANs required in Phase 1/2. |
| **Upgrade path** | Each phase is independently deployable. Phase 1 is fully functional on its own. |

---

# Part 2: System Overview

## Phase 1 Architecture

```
╔══════════════════════════════════════════════════════════════════════╗
║                    HOME NETWORK (192.168.86.0/24)                     ║
║                          Google Nest Router                          ║
║                                                                      ║
║  ┌───────────────────────────────────────────────────────────────┐  ║
║  │                  MS-S1 MAX (192.168.86.38)                     │  ║
║  │                  Proxmox VE — Bare Metal                      │  ║
║  │                                                               │  ║
║  │  ┌──────────────────────────────────────────────────────┐    │  ║
║  │  │  NemoClaw VM  (192.168.86.21)                         │    │  ║
║  │  │                                                      │    │  ║
║  │  │  OpenClaw agent          OpenShell sandbox           │    │  ║
║  │  │  WhatsApp gateway        Coding agent orchestration  │    │  ║
║  │  │                                                      │    │  ║
║  │  │  Main model:   Nemotron 120B (NVIDIA cloud, free)    │    │  ║
║  │  │  Worker model: Qwen 2.5 Coder 32B (local Ollama)     │    │  ║
║  │  └──────────────────────────────────────────────────────┘    │  ║
║  │                                                               │  ║
║  │  ┌──────────────────────────────────────────────────────┐    │  ║
║  │  │  Ollama LXC  (192.168.86.25)                          │    │  ║
║  │  │                                                      │    │  ║
║  │  │  Radeon 890M — full GPU, no sharing needed           │    │  ║
║  │  │  Qwen 2.5 Coder 32B  (agents, port 11434)            │    │  ║
║  │  │  phi3:mini           (fast fallback)                 │    │  ║
║  │  └──────────────────────────────────────────────────────┘    │  ║
║  └───────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════╝
                              │
                          INTERNET
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
  ┌───────┴──────┐       ┌────┴─────┐      ┌──────┴───────┐
  │ OpenClaw     │       │Tailscale │      │  NVIDIA Cloud│
  │ Cloud Gateway│       │ (VPN)    │      │  Nemotron 120B│
  │ (dumb relay) │       └──────────┘      └──────────────┘
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
║                    HOME NETWORK (192.168.86.0/24)                     ║
║                                                                      ║
║  ┌───────────────────────────────┐  ┌───────────────────────────┐  ║
║  │  MS-S1 MAX (192.168.86.38)     │  │  Mac Studio M5 Ultra      │  ║
║  │  Proxmox — Orchestration      │  │  (192.168.86.10)           │  ║
║  │                               │  │                           │  ║
║  │  NemoClaw VM ──────────────────┼─►│  Ollama                  │  ║
║  │  Ollama LXC (fallback)        │  │  70B / 120B / 200B+      │  ║
║  │                               │  │  256GB unified memory     │  ║
║  └───────────────────────────────┘  └───────────────────────────┘  ║
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
NemoClaw VM (192.168.86.21) — inside OpenShell sandbox
    │  Phase 1: HTTP → integrate.api.nvidia.com (Nemotron 120B, free)
    │  Phase 2: HTTP → 192.168.86.10:11434 (Mac Studio Ollama, local)
    ▼
Response generated
    │
    ▼
OpenClaw Cloud Gateway → WhatsApp → Your phone
```

## Network Design

```
Google Nest Router (192.168.86.1)
    │
    ├── MS-S1 MAX Proxmox host (192.168.86.38)
    │       ├── NemoClaw VM    (192.168.86.21)
    │       └── Ollama LXC     (192.168.86.25)
    │
    ├── Mac Studio — Phase 2 (192.168.86.10)
    └── Your phones, laptops, etc.
```

### Static IP Assignments

| Device | IP | How to set |
|--------|----|-----------|
| MS-S1 MAX (Proxmox) | 192.168.86.38 | Nest app → Devices → Reserve IP |
| Mac Studio (Phase 2) | 192.168.86.10 | Nest app → Devices → Reserve IP |

---

# Part 3: Bill of Materials

## Phase 1 Hardware

| Item | Model | Price | Notes |
|------|-------|-------|-------|
| **AI + Proxmox Host** | Minisforum MS-S1 MAX (AMD Ryzen AI Max+ 395, 128GB LPDDR5X, 2TB NVMe) | ~$2,920 | [Newegg](https://www.newegg.com/minisforum-barebone-systems-mini-pc-deskmini/p/2SW-002G-000W4). Ships with Windows — wipe and install Proxmox. |
| **Ethernet cable** | Cat6 | ~$10 | Wired connection to router required. |

**Phase 1 Total: ~$2,930**

## Phase 2 Additional Hardware

| Item | Model | Price | Notes |
|------|-------|-------|-------|
| **LLM Inference Server** | Apple Mac Studio M5 Ultra (256GB unified memory) | ~$4,000–5,000 | M4 Ultra (192GB) available now at ~$4,199. |
| **Ethernet cable** | Cat6 | ~$10 | — |

| Config | Incremental | Total |
|--------|------------|-------|
| + M4 Ultra Mac Studio | ~$4,200 | ~$7,130 |
| + M5 Ultra Mac Studio | ~$4,500–5,000 | ~$7,430–7,930 |

## Phase 4 Additional Hardware (Home Assistant)

| Item | Model | Price | Notes |
|------|-------|-------|-------|
| **AI Camera Accelerator** | Google Coral USB Accelerator | ~$60–80 | [coral.ai](https://coral.ai/products/accelerator). Passed through to the HA VM via Proxmox USB passthrough. |

**No additional compute hardware needed** — HA runs as a VM on the existing MS-S1 MAX.

## Phase 5 Additional Hardware (Media Server)

| Item | Model | Price | Notes |
|------|-------|-------|-------|
| **Media Storage** | WD Elements 8TB USB 3.0 | ~$120 | Or NAS. |
| **NAS (optional)** | Synology DS223 + 2× Seagate IronWolf 4TB | ~$480 | 8TB usable RAID 1. |

## Software (All Free Unless Noted)

| Software | Where | Cost |
|----------|-------|------|
| Proxmox VE | MS-S1 MAX bare metal | Free |
| Ubuntu 22.04 LTS | NemoClaw VM | Free |
| NemoClaw | NemoClaw VM | Free |
| OpenClaw | Inside NemoClaw sandbox | Subscription |
| Ollama | Ollama LXC + Mac Studio (Phase 2) | Free |
| Tailscale | MS-S1 MAX + Phone | Free (personal) |
| Docker CE | NemoClaw VM | Free |
| Home Assistant OS | HA VM on Proxmox (Phase 4) | Free |
| Nabu Casa | Cloud (Phase 4) | $6.50/mo |
| Jellyfin | Media Server LXC (Phase 5) | Free |

---

# Part 4: Phase 1 — Core AI Stack

## 4.1 MS-S1 MAX: OS & Proxmox

### Before Wiping Windows

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
IP:        192.168.86.38/24
Gateway:   192.168.86.1
DNS:       8.8.8.8
```

5. Access Proxmox web UI: `https://192.168.86.38:8006`

### Post-Install Hardening

```bash
ssh root@192.168.86.38

apt update && apt full-upgrade -y

# Switch to free community repo
rm /etc/apt/sources.list.d/pve-enterprise.list
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-community.list
apt update

# Firewall — Proxmox UI and SSH from LAN only
apt install -y ufw
ufw default deny incoming
ufw default allow outgoing
ufw allow from 192.168.86.0/24 to any port 22
ufw allow from 192.168.86.0/24 to any port 8006
ufw enable

apt install -y unattended-upgrades
dpkg-reconfigure --priority=low unattended-upgrades
```

---

## 4.2 Component Design

Phase 1 is two components on the MS-S1 MAX:

| Component | Type | vCPU | RAM | Disk | IP | Purpose |
|-----------|------|------|-----|------|----|---------|
| NemoClaw VM | VM | 4 | 16GB | 80GB | 192.168.86.21 | OpenClaw + sandbox + coding agents |
| Ollama LXC | LXC | 4 | 8GB* | 32GB | 192.168.86.25 | Local model inference |
| **Proxmox host** | — | — | ~8GB | — | 192.168.86.38 | OS overhead |
| **Model weights** | — | — | ~96GB | — | — | Unified memory pool |

> \* 8GB covers Ollama process overhead only. Model weights load into the AMD Radeon 890M's
> unified memory pool via bind-mounted DRI device — separate from the container's RAM budget.
> With NemoClaw VM and host consuming ~32GB, approximately 96GB is available for models.

### Create the NemoClaw VM

In Proxmox UI: **Create VM**

```
VM ID: 100  |  Name: nemoclaw
OS:    Ubuntu 22.04 LTS Server ISO
Disk:  80GB VirtIO SCSI
CPU:   4 cores, type: host
RAM:   16384 MB
Net:   vmbr0, VirtIO
```

Set static IP:

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    ens18:
      addresses: [192.168.86.21/24]
      routes:
        - to: default
          via: 192.168.86.1
      nameservers:
        addresses: [8.8.8.8]
```

```bash
sudo netplan apply
```

---

## 4.3 NemoClaw + OpenClaw

> NemoClaw wraps OpenClaw in a hardened sandbox — enforcing network egress policy,
> filesystem isolation, and process sandboxing via NVIDIA OpenShell.
> OpenClaw's built-in coding agent orchestration (Claude Code, Codex via ACP harness)
> runs directly inside this sandbox. No separate agent VM required.

### Prerequisites on the NemoClaw VM

```bash
ssh ubuntu@192.168.86.21

curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER && newgrp docker

curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

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

1. **NVIDIA API key** — get one free at [build.nvidia.com](https://build.nvidia.com) → any model → "Get API Key"
2. **Sandbox name** — e.g. `my-assistant`
3. **Inference provider** — NVIDIA Cloud (default)
4. **WhatsApp** — yes

### Connect WhatsApp

```bash
nemoclaw my-assistant connect

sandbox$ openclaw channels login whatsapp
# Scan QR code with WhatsApp → Settings → Linked Devices → Link a Device

sandbox$ openclaw start
```

### NemoClaw Security Layers

| Layer | What it blocks |
|-------|---------------|
| Network | Agent calling unauthorized hosts — surfaced in `openshell term` for approval |
| Filesystem | Reads/writes outside `/sandbox` and `/tmp` |
| Process | Privilege escalation, dangerous syscalls |
| Inference | All model calls routed through OpenShell gateway |

```bash
openshell term                     # Live network request monitor
nemoclaw my-assistant status       # Health check
nemoclaw my-assistant logs -f      # Live logs
nemoclaw my-assistant connect      # Shell into sandbox
```

---

## 4.4 Ollama + Local Models

> **Why LXC, not on the host?** If Ollama crashes on the Proxmox host it can freeze
> the hypervisor — taking everything down. Running it in an LXC contains crashes to
> just the LXC while Proxmox and the NemoClaw VM stay up.

### Create the Ollama LXC

In Proxmox UI: **Create CT**

```
CT ID:      200
Hostname:   ollama
Template:   Ubuntu 22.04
Disk:       32GB
CPU:        4 cores
RAM:        8192 MB
Network:    vmbr0, IP 192.168.86.25/24, Gateway 192.168.86.1
Unprivileged: YES
```

### Bind-Mount the GPU

Before starting the container:

```bash
nano /etc/pve/lxc/200.conf
```

```
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```

```bash
pct start 200
pct enter 200
```

### Install and Configure Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh

# Verify GPU is visible
ls /dev/dri/   # Should show: card0  renderD128

mkdir -p /etc/systemd/system/ollama.service.d
cat > /etc/systemd/system/ollama.service.d/override.conf << 'EOF'
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_NUM_CTX=32768"
EOF

systemctl daemon-reload
systemctl enable ollama
systemctl start ollama

# Verify GPU usage
ollama run phi3:mini "hello" --verbose
# Look for: "using GPU: AMD Radeon Graphics"
```

> **Context window:** `OLLAMA_NUM_CTX=32768` sets a 32K default for all models.
> Increase to `65536` or `131072` for 64K/128K if needed. Larger context uses more
> unified memory for the KV cache — a 32B model at 128K adds ~15–20GB on top of ~20GB
> for weights. 32K is a good starting point.

### Pull Models

```bash
ollama pull qwen2.5-coder:32b   # Coding agents
ollama pull phi3:mini            # Fast fallback
ollama list
```

### Restrict LAN Access

```bash
# On Proxmox host
iptables -A FORWARD -s 192.168.86.0/24 -d 192.168.86.25 -p tcp --dport 11434 -j ACCEPT
iptables -A FORWARD -d 192.168.86.25 -j DROP
apt install -y iptables-persistent
netfilter-persistent save
```

---

## 4.5 Multi-Agent Configuration

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
      { "id": "main", "default": true },
      {
        "id": "coder",
        "model": { "primary": "ollama/qwen2.5-coder:32b" }
      },
      {
        "id": "fast",
        "model": { "primary": "ollama/phi3:mini" }
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
        "baseUrl": "http://192.168.86.25:11434",
        "api": "ollama"
      }
    }
  }
}
```

```bash
# Allow Ollama in OpenShell policy (on NemoClaw VM host)
openshell policy add --sandbox my-assistant \
  --host 192.168.86.25 --port 11434 --protocol rest --name ollama-local
```

---

## 4.6 Remote Access

Everything uses outbound-only connections — no ports forwarded on your router.

### Tailscale: Access Your Local LLM from Anywhere

```bash
# On Proxmox host
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale ip -4   # Note your Tailscale IP (e.g. 100.64.0.1)
```

Install Tailscale on your phone (same account). Then:

```
Enchanted app (iOS) → Settings → Server URL → http://100.64.0.1:11434
```

Or access Open WebUI (run on the NemoClaw VM):

```bash
docker run -d \
  -p 8080:8080 \
  -e OLLAMA_BASE_URL=http://192.168.86.25:11434 \
  --name open-webui \
  --restart unless-stopped \
  ghcr.io/open-webui/open-webui:main
```

Access from phone: `http://100.64.0.1:8080`

---

## 4.7 Security

### Proxmox Checklist

- [ ] Proxmox UI accessible from LAN only (UFW rule in place)
- [ ] SSH accessible from LAN only
- [ ] Ollama port 11434 blocked from internet (iptables in place)
- [ ] iptables rules persistent
- [ ] Proxmox updated regularly

### NemoClaw Sandbox Checklist

- [ ] OpenShell network policy reviewed regularly via `openshell term`
- [ ] NVIDIA API key stored in `~/.nemoclaw/credentials.json`, not in code
- [ ] OpenClaw workspace does not contain sensitive credentials

---

# Part 5: Phase 2 — Mac Studio Added

## 5.1 What Changes

| | Phase 1 | Phase 2 |
|--|---------|---------|
| Orchestrator model | Nemotron 120B (NVIDIA cloud) | Local 70B–120B (Mac Studio) |
| Coding agent model | Qwen 2.5 Coder 32B | Qwen 2.5 Coder 72B |
| Parallel agents | 1–2 | 4–8 simultaneous |
| Privacy | Cloud for main agent | 100% local |
| Max model size | 32B practical | 200B+ |
| Context window | 32K default | 128K+ comfortable |

## 5.2 Mac Studio Setup

- Mac Studio M5 Ultra (256GB) — recommended; M4 Ultra (192GB) available now
- Wired ethernet to router, static IP: `192.168.86.10`

### macOS Hardening

```
FileVault → On
Firewall → On, Stealth Mode → On
Remote Login → Off
Screen Sharing → Off
```

### Install Ollama

```bash
brew install --cask ollama
```

### Configure with Extended Context

```bash
sudo nano /Library/LaunchDaemons/com.ollama.ollama.plist
```

```xml
<key>EnvironmentVariables</key>
<dict>
  <key>OLLAMA_HOST</key>
  <string>0.0.0.0:11434</string>
  <key>OLLAMA_NUM_CTX</key>
  <string>65536</string>
</dict>
```

```bash
sudo launchctl unload /Library/LaunchDaemons/com.ollama.ollama.plist
sudo launchctl load /Library/LaunchDaemons/com.ollama.ollama.plist
```

### Firewall (pf)

```bash
sudo nano /etc/pf.anchors/ollama-protect
```

```
pass in on en0 proto tcp from 192.168.86.0/24 to any port 11434
pass in on utun0 proto tcp from 100.64.0.0/10 to any port 11434
block in proto tcp to any port 11434
```

### Pull Models

```bash
ollama pull qwen2.5-coder:72b
ollama pull llama3.1:70b
ollama pull deepseek-r1:70b
ollama pull qwen2.5:7b
```

## 5.3 Switch Inference

```bash
# NemoClaw sandbox — update openclaw.json
"ollama": { "baseUrl": "http://192.168.86.10:11434" }

# Allow Mac Studio in OpenShell policy
openshell policy add --sandbox my-assistant \
  --host 192.168.86.10 --port 11434 --protocol rest --name mac-studio-ollama
```

## 5.4 Model Capabilities

| Model | RAM | Speed | Best for |
|-------|-----|-------|---------|
| qwen2.5:7b | 5GB | 200+ tok/s | Fast tasks |
| llama3.1:70b | 45GB | 55–75 tok/s | Orchestration |
| qwen2.5-coder:72b | 46GB | 45–65 tok/s | Coding agents |
| deepseek-r1:70b | 45GB | 40–60 tok/s | Complex reasoning |
| nemotron-ultra:253b | ~160GB | 20–35 tok/s | Maximum capability |

---

# Part 5.5: Phase 3 — Network-Level Security Hardening

Upgrades isolation from machine-enforced to network-enforced — a hardware layer that
holds even if a VM or OS firewall is misconfigured.

**Additional hardware:** UniFi Express (~$150) — replaces Google Nest.

## VLAN Design

| VLAN | Name | Subnet | Devices | Internet | Cross-VLAN |
|------|------|--------|---------|----------|-----------|
| 1 | Trusted | 192.168.86.0/24 | MS-S1 MAX, phones, laptops | Yes | Full |
| 20 | Inference | 192.168.2.0/24 | Mac Studio only | No | VLAN 1 → port 11434 only |
| 30 | IoT | 192.168.3.0/24 | Smart devices, cameras | No | None |

## Firewall Rules

```
Rule 1: Allow VLAN 1 → VLAN 20, port 11434
Rule 2: Block VLAN 20 → internet
Rule 3: Block VLAN 20 → VLAN 1
Rule 4: Block VLAN 30 → all
Rule 5: Allow VLAN 1 → VLAN 30, port 80/443
```

## IP Changes After Phase 3

| Device | Phase 2 IP | Phase 3 IP | VLAN |
|--------|-----------|-----------|------|
| MS-S1 MAX | 192.168.86.38 | 192.168.86.38 | 1 |
| NemoClaw VM | 192.168.86.21 | 192.168.86.21 | 1 |
| Ollama LXC | 192.168.86.25 | 192.168.86.25 | 1 |
| Mac Studio | 192.168.86.10 | **192.168.2.10** | **20** |

Update openclaw.json and OpenShell policy to `192.168.2.10` after Phase 3.

---

# Part 5.6: Phase 4 — Home Assistant (Optional)

> **Independent of Phase 2 and 3** — add at any time.
> Runs as a VM on the existing MS-S1 MAX alongside the AI stack.
> **Tradeoff:** HA shares the MS-S1 MAX with the AI stack. If Proxmox goes down
> (maintenance, crash), HA goes down too. For most home setups this is acceptable —
> you get significantly better hardware than a Pi (faster CPU, more RAM, better
> video decoding for Frigate) at the cost of that single point of failure.

## What You Need

- Google Coral USB Accelerator (~$60–80) — for AI camera detection via Frigate
- No additional compute hardware — HA runs as a VM on the MS-S1 MAX

## Create the Home Assistant VM

HAOS (Home Assistant Operating System) requires a VM — it cannot run in an LXC.

In Proxmox UI: **Create VM**

```
VM ID: 101  |  Name: home-assistant
OS:    Download HAOS qcow2 image (see below)
Disk:  32GB VirtIO SCSI
CPU:   2 cores, type: host
RAM:   4096 MB
Net:   vmbr0, VirtIO
```

### Flash HAOS

```bash
# On Proxmox host — download and import HAOS image
wget https://github.com/home-assistant/operating-system/releases/latest/download/haos_ova-*.qcow2.xz
xz -d haos_ova-*.qcow2.xz
qm importdisk 101 haos_ova-*.qcow2 local-lvm
```

In Proxmox UI: VM 101 → Hardware → Unused Disk → Edit → Add as `scsi0`, boot order to disk.

Start the VM and access HA at `http://192.168.86.30:8123` after ~5 minutes.

Set static IP in Google Nest app: **Devices → HA VM → Reserve IP → 192.168.86.30**

### Pass the Coral USB to the HA VM

Plug the Coral USB into the MS-S1 MAX, then:

```
Proxmox UI → VM 101 → Hardware → Add → USB Device
Select: Google Coral USB Accelerator
```

Coral appears inside HAOS automatically. Verify in Frigate settings.

### Add a Dedicated Ollama Instance for HA

To keep HA inference from competing with coding agent workloads, add a second
Ollama instance in the existing Ollama LXC:

```bash
pct enter 200
```

```bash
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

OLLAMA_HOST=localhost:11435 ollama pull qwen2.5:7b
OLLAMA_HOST=localhost:11435 ollama pull phi3:mini
```

Update the Proxmox host iptables to allow HA VM to reach both Ollama ports:

```bash
# On Proxmox host — update firewall rules
iptables -I FORWARD -s 192.168.86.0/24 -d 192.168.86.25 -p tcp --dport 11435 -j ACCEPT
netfilter-persistent save
```

### Install Frigate Add-on

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
Settings → Devices & Services → Add Integration → Ollama
Host: http://192.168.86.25:11435
Model: qwen2.5:7b
```

### Remote Access via Nabu Casa

```
Settings → Home Assistant Cloud → Sign In
Subscribe at nabucasa.com ($6.50/mo)
```

No port forwarding required — outbound tunnel only.

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

# Part 5.7: Phase 5 — Media Server (Optional)

> **Independent of all other phases** — add at any time.
> Runs as an LXC container alongside the Ollama LXC, sharing the GPU via bind-mount.

## Why LXC (Not a VM)

The Radeon 890M is already bind-mounted into the Ollama LXC. Full PCIe passthrough
to a VM gives exclusive GPU ownership — it would break Ollama. LXC containers share
the host kernel and support the same `/dev/dri` bind-mount, so both LXC containers
can use the GPU simultaneously for inference and transcoding.

## Create the Media Server LXC

```
CT ID:      202
Hostname:   media-server
Template:   Ubuntu 22.04
Disk:       32GB
CPU:        4 cores
RAM:        4096 MB
Network:    vmbr0, IP 192.168.86.22/24, Gateway 192.168.86.1
Unprivileged: YES
```

```bash
nano /etc/pve/lxc/202.conf
```

```
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```

```bash
pct start 202
pct enter 202

curl -fsSL https://repo.jellyfin.org/install-debuntu.sh | sudo bash
sudo systemctl enable jellyfin && sudo systemctl start jellyfin
```

Access Jellyfin: `http://192.168.86.22:8096`

Enable hardware transcoding:
```
Dashboard → Playback → Transcoding
Hardware acceleration: VAAPI
VAAPI Device: /dev/dri/renderD128
```

---

# Part 6: Performance Comparison

## Memory Bandwidth — The Bottleneck

| Hardware | Memory | Bandwidth | Implication |
|----------|--------|-----------|-------------|
| MS-S1 MAX (Radeon 890M) | 128GB LPDDR5X | ~270 GB/s | Good for ≤32B models |
| Mac Studio M5 Ultra | 256GB unified | ~800 GB/s | ~3× faster, handles 70B–200B+ |

## Tokens Per Second

| Task | Model | Phase 1 | Phase 2 | Feel |
|------|-------|---------|---------|------|
| Orchestration | Nemotron 120B (cloud) | 50–80 tok/s | — | Fast |
| Orchestration | Llama 3.1 70B (local) | 8–12 tok/s | 55–75 tok/s | Phase 1: slow; Phase 2: fast |
| Coding agent | Qwen 2.5 Coder 32B | 15–25 tok/s | 90–120 tok/s | Phase 1: readable; Phase 2: fast |
| Coding agent | Qwen 2.5 Coder 72B | 4–6 tok/s | 45–65 tok/s | Phase 1: painful; Phase 2: good |
| HA / simple | Qwen 2.5 7B | 50–70 tok/s | 200+ tok/s | Both fine |

## Model Memory (Phase 1 — ~96GB available)

| Scenario | Models | RAM used | Notes |
|----------|--------|----------|-------|
| Typical | qwen2.5-coder:32b | ~20GB | Comfortable |
| + HA (Phase 4) | + qwen2.5:7b | ~27GB | Still easy |
| Extended context | qwen2.5-coder:32b + 128K | ~40GB | Fits, leaves room |

## Is Phase 2 Worth It?

**Yes, if you:** run 3+ parallel agents, work on large codebases, want 100% local
inference, hit NVIDIA rate limits, or want 70B+ models at usable speed.

**Not yet, if you:** use this primarily as a personal assistant (Nemotron 120B
handles this well) or are satisfied with 32K context.

---

# Part 7: Testing & Validation

## Phase 1 Checks

```bash
# Ollama running
curl http://192.168.86.25:11434/api/tags

# Context window configured
curl http://192.168.86.25:11434/api/show -d '{"name":"qwen2.5-coder:32b"}' | grep num_ctx

# Ollama not reachable from internet (test on cellular)
curl --connect-timeout 5 http://YOUR_PUBLIC_IP:11434/api/tags
# Expected: timeout

# NemoClaw health
nemoclaw my-assistant status
```

## WhatsApp End-to-End Test

Send: *"What's 7 times 8?"* — should respond within 5–10 seconds.

```bash
nemoclaw my-assistant logs -f   # Watch live
```

## Phase 4 (HA) Checks

```bash
# HA Ollama instance
curl http://192.168.86.25:11435/api/tags

# Coral detected in Frigate
# Settings → Devices → Frigate → Detectors → coral should show "active"
```

## Phase 2 Checks

```bash
curl http://192.168.86.10:11434/api/tags
curl http://192.168.86.10:11434/api/show -d '{"name":"llama3.1:70b"}' | grep num_ctx
```

---

# Part 8: Maintenance

## Weekly

```bash
ollama pull qwen2.5-coder:32b
ollama pull phi3:mini

nemoclaw my-assistant status
openshell term   # Review blocked network requests

systemctl status ollama
# If Phase 4 deployed:
systemctl status ollama-ha
```

## Monthly

```bash
ssh root@192.168.86.38
apt update && apt full-upgrade -y

ssh ubuntu@192.168.86.21 "sudo apt update && sudo apt upgrade -y"

pct enter 200
apt update && apt upgrade -y
```

## Proxmox Backups

```
Datacenter → Backup → Add
Schedule: Sunday 02:00
VMs: 100 (nemoclaw)
LXC: 200 (ollama)
Mode: Snapshot, Compression: ZSTD
```

> If Phase 4 deployed: add VM 101 (home-assistant) to backup job.
> If Phase 5 deployed: add LXC 202 (media-server) to backup job.

## Phase 2: Mac Studio Model Updates

```bash
ollama list | awk 'NR>1 {print $1}' | while read model; do
  ollama pull "$model"
done
```

---

> **Quick Reference — IP Addresses**
>
> | Device | IP | Phase 3 IP | Notes |
> |--------|----|-----------|-------|
> | MS-S1 MAX (Proxmox) | 192.168.86.38 | 192.168.86.38 | Always VLAN 1 |
> | NemoClaw VM | 192.168.86.21 | 192.168.86.21 | Always VLAN 1 |
> | Ollama LXC | 192.168.86.25 | 192.168.86.25 | Always VLAN 1 |
> | Mac Studio (Phase 2) | 192.168.86.10 | 192.168.2.10 | Moves to VLAN 20 |
> | HA VM (Phase 4) | 192.168.86.30 | 192.168.86.30 | VLAN 1 |
> | Media Server LXC (Phase 5) | 192.168.86.22 | 192.168.86.22 | VLAN 1 |

---

*Architecture version: 2.3 — Phase 1 is a lean two-component AI stack (NemoClaw VM + Ollama LXC). Home Assistant moved to Phase 4 as an optional VM on MS-S1 MAX with Coral USB passthrough. Media Server is Phase 5. Raspberry Pi removed from plan.*  
*Hardware: Minisforum MS-S1 MAX + (Phase 2) Apple Mac Studio M5 Ultra*  
*Networking: Google Nest compatible — no VLANs required in Phase 1/2*
