# Home AI Infrastructure Architecture
### A Private, Powerful, Always-On AI Stack — Built in Your Home

> **Version:** 1.0 | **Status:** Reference Architecture  
> **Audience:** Part 1–2 for anyone; Part 3–11 for engineers and AI agents

---

## Table of Contents

- [Part 1: What Is This And Why Does It Matter](#part-1-what-is-this-and-why-does-it-matter)
- [Part 2: System Architecture Overview](#part-2-system-architecture-overview)
- [Part 3: Bill of Materials](#part-3-bill-of-materials)
- [Part 4: Network Design](#part-4-network-design)
- [Part 5: Mac Studio Setup](#part-5-mac-studio-setup)
- [Part 6: Raspberry Pi 4 Setup — Home Assistant](#part-6-raspberry-pi-4-setup--home-assistant)
- [Part 7: x86 Proxmox Setup](#part-7-x86-proxmox-setup)
- [Part 8: Remote Access Setup](#part-8-remote-access-setup)
- [Part 9: Security Hardening Checklist](#part-9-security-hardening-checklist)
- [Part 10: Testing & Validation](#part-10-testing--validation)
- [Part 11: Maintenance & Growth Path](#part-11-maintenance--growth-path)

---

# Part 1: What Is This And Why Does It Matter

## The Short Version

This is a blueprint for building your own private AI infrastructure at home — one that gives you the capabilities of a well-staffed tech company, running entirely on hardware you own, with your data never leaving your walls.

You get:
- A **personal AI assistant** you can talk to via WhatsApp from anywhere in the world
- **Home automation** that understands context and speaks plain English
- **Autonomous coding agents** that can build software while you sleep
- **Local ChatGPT-quality AI** accessible from your phone, anywhere, without sending your conversations to a corporation

All of it is private. All of it is yours.

---

## Why Own Your Infrastructure?

When you use ChatGPT, Claude, or Google Gemini, your conversations are sent to their servers. You're trusting them with your questions, your projects, your habits, and increasingly your most sensitive professional work. You're also paying per-token — and those bills grow fast when you're running agents at scale.

With this setup:

- **Your data never leaves your home network.** Conversations stay between you and your hardware.
- **No per-token billing.** Run as many queries as you want. The cost is electricity.
- **No rate limits.** Spin up 20 coding agents at once. Nobody's throttling you.
- **No model deprecations.** You control which models you run and when you upgrade.
- **Resilient to internet outages.** Most functions work locally even if your ISP goes down.

This isn't about distrust of AI companies — it's about ownership, control, and economics at scale.

---

## What You Can Actually Do With This

### Talk to your AI from anywhere
Send a WhatsApp message: *"Summarize my emails from today and flag anything urgent."*  
Your AI assistant wakes up on a server in your home, reads your emails, and responds — all within seconds. No cloud AI involved. No data shared.

### Home that thinks
Your home cameras, powered by AI object detection, know the difference between your dog and a stranger. When an unfamiliar person lingers at your door for 30 seconds, your home messages you. When your kids arrive home, the lights adjust. When you say "good night," a dozen devices respond intelligently.

### Coding agents that work for you
Describe a feature. Go to sleep. Wake up to a pull request. Your coding agents can clone repos, write code, run tests, and open PRs — completely autonomously, running on your own hardware, using your own AI models.

### Local AI from your phone
When you're away from home, a secure tunnel (Tailscale VPN) lets your phone reach your home AI as if you were sitting next to it. Use apps like Enchanted or Open WebUI to chat with your local models. No subscription. No data leaving your control.

---

## Simple Analogy: What Each Component Does

Think of this like a small company:

| Component | The Analogy | What It Actually Does |
|-----------|-------------|----------------------|
| **Mac Studio M5 Ultra** | The Brain | Runs all AI inference. Every question gets processed here. |
| **Raspberry Pi 4** | The Building Manager | Runs your smart home. Knows who's home, controls devices, watches cameras. |
| **Proxmox Machine (OpenClaw VM)** | The Executive Assistant | Your personal AI — takes your requests, decides what to do, delegates tasks. |
| **Proxmox Machine (Agent Swarm VM)** | The Dev Team | Autonomous coding agents. Get tasks, build things, report back. |
| **Tailscale** | The Private Tunnel | Secure encrypted connection between your phone and home, anywhere in the world. |
| **Nabu Casa** | The Home Hotline | Secure cloud bridge for your home automation, no ports to open. |
| **WhatsApp Gateway** | The Intercom | How you talk to your AI from your phone, via the app you already use. |

---

## Privacy Model in Plain English

Every piece of AI processing in this system happens on hardware sitting in your home. When you ask your assistant a question via WhatsApp:

1. Your message travels to OpenClaw's cloud gateway (just a relay — it doesn't store or process content)
2. The gateway delivers it to your home server
3. Your home server sends it to your Mac Studio for AI processing
4. The Mac Studio generates a response
5. The response travels back through the same relay to your phone

The cloud components (WhatsApp, OpenClaw gateway) are **dumb pipes** — they move bytes, not meaning. The intelligence lives at home.

---

# Part 2: System Architecture Overview

## Full System Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                           HOME NETWORK (192.168.1.0/24)                         ║
║                                                                                  ║
║  ┌─────────────────────────────┐     ┌─────────────────────────────────────┐    ║
║  │     MAC STUDIO M5 ULTRA     │     │        PROXMOX HOST (MS-01)         │    ║
║  │     192.168.1.10            │     │        192.168.1.20                 │    ║
║  │                             │     │                                     │    ║
║  │  ┌───────────────────────┐  │     │  ┌─────────────────────────────┐   │    ║
║  │  │  Ollama LLM Server    │  │◄────┼──│  VM1: OpenClaw              │   │    ║
║  │  │  0.0.0.0:11434 (pf)   │  │     │  │  192.168.1.21               │   │    ║
║  │  └───────────────────────┘  │◄────┼──│  4 vCPU / 8GB RAM / 64GB   │   │    ║
║  │  ┌───────────────────────┐  │     │  └─────────────────────────────┘   │    ║
║  │  │  Tailscale (exit)     │  │     │                                     │    ║
║  │  └───────────────────────┘  │     │  ┌─────────────────────────────┐   │    ║
║  │                             │◄────┼──│  VM2: Agent Swarm           │   │    ║
║  └─────────────────────────────┘     │  │  192.168.20.10              │   │    ║
║             ▲                        │  │  8+ vCPU / 48GB+ RAM / 500GB│   │    ║
║             │                        │  │  Docker: Claude Code, Codex  │   │    ║
║             │                        │  └─────────────────────────────┘   │    ║
║  ┌──────────┴──────────────┐         └─────────────────────────────────────┘    ║
║  │  RASPBERRY PI 4         │                                                     ║
║  │  192.168.30.10          │                                                     ║
║  │  (VLAN 30)              │                                                     ║
║  │                         │                                                     ║
║  │  ┌─────────────────┐    │                                                     ║
║  │  │ Home Assistant  │    │                                                     ║
║  │  │ Frigate NVR     │    │                                                     ║
║  │  │ Google Coral TPU│    │                                                     ║
║  │  └─────────────────┘    │                                                     ║
║  └─────────────────────────┘                                                     ║
║                                                                                  ║
║  ┌─────────────────────────────────────────────────────────────────────────┐    ║
║  │                        HOME ROUTER / SWITCH                              │    ║
║  │           VLAN 1: Main (Mac Studio, Proxmox)                            │    ║
║  │           VLAN 20: Agent Swarm (isolated)                                │    ║
║  │           VLAN 30: IoT / Home Assistant                                  │    ║
║  └──────────────────────────────────┬──────────────────────────────────────┘    ║
╚═════════════════════════════════════╪════════════════════════════════════════════╝
                                      │
                                  INTERNET
                                      │
              ┌───────────────────────┼────────────────────────┐
              │                       │                         │
    ┌─────────┴────────┐   ┌──────────┴──────────┐  ┌─────────┴────────┐
    │  OpenClaw Cloud  │   │   Nabu Casa Cloud   │  │  Tailscale       │
    │  Gateway (relay) │   │   (HA remote access)│  │  Coordination    │
    └─────────┬────────┘   └──────────┬──────────┘  └─────────┬────────┘
              │                       │                         │
    ┌─────────┴────────┐   ┌──────────┴──────────┐  ┌─────────┴────────┐
    │  WhatsApp        │   │  HA Mobile App      │  │  Phone (VPN)     │
    │  (your phone)    │   │  (your phone)       │  │  → Mac Studio    │
    └──────────────────┘   └─────────────────────┘  └──────────────────┘
```

---

## Network Topology Detail

```
                          ┌────────────────────┐
                          │    HOME ROUTER     │
                          │  (VLAN-capable)    │
                          │                    │
                VLAN 1 ───┤ Port 1-4 (trunk)  ├─── VLAN 20 (Agent Swarm)
          (Main/Trusted)  │                    │
                          │ Port 5-8 (IoT)    ├─── VLAN 30 (IoT/HA)
                          └────────────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │               │
           ┌────────┴──┐  ┌───────┴──────┐  ┌────┴──────┐
           │ Mac Studio│  │ Proxmox Host │  │  Pi 4     │
           │ VLAN 1    │  │ VLAN 1       │  │  VLAN 30  │
           │ .10       │  │ .20          │  │  .30      │
           └───────────┘  └──────────────┘  └───────────┘
                               │
                    ┌──────────┴──────────┐
                    │                     │
           ┌────────┴──┐         ┌────────┴──┐
           │OpenClaw VM│         │Agent Swarm│
           │ VLAN 1    │         │ VLAN 20   │
           │ .21       │         │ 20.10     │
           └───────────┘         └───────────┘
```

---

## Data Flow: WhatsApp Message End-to-End

```
You (phone)
    │
    │  1. Send WhatsApp message: "What's the weather like?"
    ▼
WhatsApp Servers
    │
    │  2. Webhook delivery (HTTPS)
    ▼
OpenClaw Cloud Gateway
    │
    │  3. Relay to your home (outbound WebSocket, no inbound port needed)
    ▼
OpenClaw VM (192.168.1.21)
    │
    │  4. Parse request, build LLM prompt
    ▼
Mac Studio Ollama (192.168.1.10:11434)
    │
    │  5. LLM inference (100% local, never leaves home)
    ▼
OpenClaw VM (192.168.1.21)
    │
    │  6. Receive response, format reply
    ▼
OpenClaw Cloud Gateway
    │
    │  7. Relay response back to WhatsApp
    ▼
WhatsApp Servers
    │
    │  8. Deliver to your phone
    ▼
You (phone) — receive response, typically in 2-5 seconds
```

---

## Data Flow: Remote LLM from Phone via Tailscale

```
You (phone, away from home)
    │
    │  1. Open Enchanted / Open WebUI app
    │     Configured: http://100.x.x.x:11434 (Tailscale IP)
    ▼
Tailscale Network (encrypted WireGuard tunnel)
    │
    │  2. Encrypted traffic routes through Tailscale coordination server
    │     (only coordinates — never sees content)
    ▼
Mac Studio Tailscale Interface
    │
    │  3. Ollama receives request on LAN IP (via Tailscale)
    ▼
Ollama LLM Inference
    │
    │  4. Response travels back through same tunnel
    ▼
You (phone) — full local model response, private
```

---

## Component Role Summary

| Component | Hardware | OS/Software | Role | Network |
|-----------|----------|-------------|------|---------|
| **LLM Inference Server** | Mac Studio M5 Ultra | macOS + Ollama | Runs all AI models; serves inference to everything on LAN | VLAN 1 · 192.168.1.10 |
| **Home Automation Hub** | Raspberry Pi 4 (4GB) | Home Assistant OS | Smart home control, Frigate camera AI, HA automations | VLAN 30 · 192.168.30.10 |
| **AI Orchestrator** | Proxmox VM1 | Ubuntu 22.04 + OpenClaw | Personal AI assistant, WhatsApp gateway, task orchestration | VLAN 1 · 192.168.1.21 |
| **Agent Swarm** | Proxmox VM2 | Ubuntu 22.04 + Docker | Autonomous coding agents (Claude Code, Codex, OpenHands) | VLAN 20 · 192.168.20.10 |
| **Remote Access (HA)** | Cloud | Nabu Casa | Encrypted cloud relay for HA mobile app | Internet |
| **Remote Access (LLM)** | Cloud | Tailscale | WireGuard VPN for phone → Mac Studio | Internet |
| **Remote Access (AI)** | Cloud | OpenClaw Gateway | WebSocket relay for WhatsApp → OpenClaw | Internet |

---

# Part 3: Bill of Materials

## Hardware

| Item | Model | Est. Price | Notes |
|------|-------|-----------|-------|
| **LLM Server** | Apple Mac Studio M5 Ultra (256GB unified memory) | ~$4,000–5,000 | Not yet released as of early 2026; pre-order/watch Apple.com. M4 Ultra (192GB) available now as alternative (~$4,199). |
| **Proxmox Host** | Minisforum MS-01 (Intel i9-12900H, 64GB DDR5, 1TB NVMe) | ~$500–600 | [minisforum.com](https://minisforum.com/). Upgrade to 128GB RAM ($150) for larger swarms. |
| **Home Automation Hub** | Raspberry Pi 4 Model B (4GB RAM) | ~$55–75 | [raspberrypi.com](https://www.raspberrypi.com/). Include official USB-C power supply. |
| **AI Accelerator (Camera)** | Google Coral USB Accelerator | ~$60–80 | [coral.ai](https://coral.ai/products/accelerator). Used by Frigate for real-time object detection. |
| **Storage (Pi)** | Samsung Endurance microSD 64GB (or 128GB) | ~$15–20 | Or use USB SSD for better reliability. |
| **Storage (Agent Swarm extra)** | Samsung 870 EVO 1TB SATA SSD | ~$80 | Additional disk for Agent Swarm VM if Proxmox host only has 1TB. |
| **Networking** | UniFi Express or TP-Link ER605 + TL-SG108E | ~$80–150 | Need VLAN-capable router + managed switch. UniFi Express does both. |
| **USB Hub (Pi)** | Anker 4-Port USB 3.0 Hub | ~$15 | For Coral TPU + other peripherals. |
| **MicroSD Reader** | Anker USB-C SD Card Reader | ~$12 | For flashing Pi SD card. |

### Hardware Totals

| Tier | Config | Est. Total |
|------|--------|-----------|
| **Minimum Viable** | M4 Ultra Mac Studio + 64GB Proxmox + Pi 4 | ~$4,800 |
| **Recommended** | M5 Ultra Mac Studio + 64GB Proxmox + Pi 4 + Coral | ~$5,700 |
| **Full Power** | M5 Ultra + 128GB Proxmox + Pi 4 + Coral + UniFi | ~$6,100 |

---

## Software (All Free Unless Noted)

| Software | Where It Runs | Cost | Link |
|----------|--------------|------|------|
| **Ollama** | Mac Studio | Free | [ollama.com](https://ollama.com) |
| **Home Assistant OS** | Raspberry Pi 4 | Free | [home-assistant.io](https://www.home-assistant.io) |
| **Nabu Casa** | Cloud subscription | $6.50/mo | [nabucasa.com](https://www.nabucasa.com) |
| **Frigate NVR** | Home Assistant (add-on) | Free | [frigate.video](https://frigate.video) |
| **Proxmox VE** | x86 host | Free (community) | [proxmox.com](https://www.proxmox.com) |
| **Ubuntu 22.04 LTS** | Proxmox VMs | Free | [ubuntu.com](https://ubuntu.com) |
| **OpenClaw** | OpenClaw VM | Subscription | [openclaw.ai](https://openclaw.ai) |
| **Docker CE** | Agent Swarm VM | Free | [docker.com](https://docker.com) |
| **Claude Code CLI** | Agent Swarm VM | API key needed | [anthropic.com](https://anthropic.com) |
| **Codex CLI** | Agent Swarm VM | API key needed | [openai.com](https://openai.com) |
| **OpenHands** | Agent Swarm VM | Free (self-hosted) | [github.com/All-Hands-AI/OpenHands](https://github.com/All-Hands-AI/OpenHands) |
| **Tailscale** | Mac Studio + Phone | Free (personal) | [tailscale.com](https://tailscale.com) |
| **Balena Etcher** | Workstation (setup only) | Free | [balena.io/etcher](https://www.balena.io/etcher) |
| **Enchanted** (iOS) | iPhone | Free | App Store |
| **Open WebUI** | Any browser / Docker | Free | [openwebui.com](https://openwebui.com) |

---

# Part 4: Network Design

## IP Address Scheme

Use static IPs for all infrastructure devices — DHCP is fine for phones/laptops.

| Device | Hostname | Static IP | VLAN | Notes |
|--------|----------|-----------|------|-------|
| Home Router | router.local | 192.168.1.1 | 1 | Gateway |
| Mac Studio | macstudio.local | 192.168.1.10 | 1 | LLM inference server |
| Proxmox Host | proxmox.local | 192.168.1.20 | 1 | Hypervisor management |
| OpenClaw VM | openclaw.local | 192.168.1.21 | 1 | AI orchestrator |
| Agent Swarm VM | agents.local | 192.168.20.10 | 20 | Isolated agent network |
| Raspberry Pi 4 | homeassistant.local | 192.168.30.10 | 30 | HA hub |
| Managed Switch | switch.local | 192.168.1.2 | 1 | If separate from router |
| DHCP Range | — | 192.168.1.100–200 | 1 | Phones, laptops, guests |

---

## VLAN Design

| VLAN ID | Name | Subnet | Purpose | Internet Access | LAN Access |
|---------|------|--------|---------|----------------|-----------|
| 1 | Main/Trusted | 192.168.1.0/24 | Mac Studio, Proxmox host, OpenClaw VM | Yes | Full |
| 20 | Agent Swarm | 192.168.20.0/24 | Agent Swarm VM only | Yes (git/repos) | Mac Studio only |
| 30 | IoT / Home Automation | 192.168.30.0/24 | Raspberry Pi, cameras, smart devices | No (Nabu Casa exception) | Mac Studio only |
| 40 | Guest | 192.168.40.0/24 | Guest WiFi | Yes | None |

> **Why isolate?** If an agent in the swarm is compromised or misbehaves, it cannot reach OpenClaw, cannot pivot to your main network, and cannot touch your home automation. Containment by default.

---

## Router/Switch Configuration (UniFi Example)

### Create VLANs in UniFi

```
Networks → Add Network:
  Name: Agent Swarm
  VLAN ID: 20
  Subnet: 192.168.20.1/24
  DHCP: Enabled (range .100-.200)

Networks → Add Network:
  Name: IoT/HA
  VLAN ID: 30
  Subnet: 192.168.30.1/24
  DHCP: Enabled
```

### Port Profiles (for managed switch)

```
Port 1-4:   All (trunk) — uplink to router
Port 5:     VLAN 1 (untagged) — Mac Studio
Port 6:     VLAN 1 (untagged) — Proxmox Host
Port 7:     VLAN 30 (untagged) — Raspberry Pi
Port 8:     VLAN 1 (untagged) — spare
```

> For Proxmox: connect a single trunk port (tagged VLANs 1+20) so Proxmox can present different VLANs to different VMs.

---

## Firewall Rules

### Inter-VLAN Rules (Router/Firewall)

| Rule | Source | Destination | Port | Action | Reason |
|------|--------|-------------|------|--------|--------|
| 1 | VLAN 20 (Agent Swarm) | 192.168.1.10 | 11434 | ALLOW | Agents need LLM |
| 2 | VLAN 20 (Agent Swarm) | VLAN 1 | Any | DENY | Agents can't reach OpenClaw |
| 3 | VLAN 20 (Agent Swarm) | Internet | 80,443 | ALLOW | Agents need git/package access |
| 4 | VLAN 30 (IoT/HA) | 192.168.1.10 | 11434 | ALLOW | HA needs LLM |
| 5 | VLAN 30 (IoT/HA) | VLAN 1 | Any | DENY | IoT can't reach main network |
| 6 | VLAN 30 (IoT/HA) | Internet | Any | DENY | IoT devices stay local |
| 7 | VLAN 1 | 192.168.1.21 | 22,443 | ALLOW | Management access to OpenClaw |
| 8 | VLAN 1 | 192.168.20.10 | 22 | ALLOW | Management access to agents |
| 9 | Any | 192.168.1.10 | 11434 | DENY | Default: block LLM from internet |

### WAN Rules (Internet → Home)

| Rule | Source | Destination | Action | Reason |
|------|--------|-------------|--------|--------|
| Block all inbound | Internet | Any | DENY | No inbound ports opened |
| Allow established | — | — | ALLOW | Return traffic for outbound connections |

> **There is no port forwarding in this architecture.** All remote access (OpenClaw, HA, Tailscale) uses outbound connections or encrypted overlays. Your home is a black box from the internet's perspective.

---

# Part 5: Mac Studio Setup

## Prerequisites
- Mac Studio M5 Ultra (or M4 Ultra as current alternative)
- macOS Sequoia or later
- Connected to home network via Ethernet (not WiFi — reliability matters for inference)
- Assigned static IP: 192.168.1.10

---

## Step 1: macOS Security Hardening

### Enable FileVault (Full Disk Encryption)
```
System Settings → Privacy & Security → FileVault → Turn On FileVault
```
Save the recovery key in a password manager (not iCloud).

### Enable macOS Firewall
```
System Settings → Network → Firewall → Turn On
```

Then open Firewall Options and set:
```
Block all incoming connections: OFF (we'll manage per-app)
Automatically allow signed software to receive incoming connections: OFF
Enable stealth mode: ON
```

### Configure Automatic Security Updates
```
System Settings → General → Software Update → Automatic Updates
Enable: Security responses and system files
Enable: Install macOS updates (optional but recommended)
```

### Set Screen Lock
```
System Settings → Lock Screen:
  Require password: immediately
  Screen saver after: 5 minutes
```

### Disable Remote Login / Sharing (unless you need it)
```
System Settings → General → Sharing:
  Remote Login: OFF (unless you use SSH for management)
  Screen Sharing: OFF
  Remote Management: OFF
```

---

## Step 2: Install Ollama

```bash
# Download and install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Verify installation
ollama --version
```

---

## Step 3: Configure Ollama Binding and Firewall Protection

By default, Ollama binds to `127.0.0.1` (localhost only). We need it accessible on all interfaces (`0.0.0.0`) so that both LAN clients and Tailscale (remote access) can reach it. Security is provided by three layered controls instead of the bind address:

1. **macOS `pf` firewall** — only LAN (192.168.1.0/24) and Tailscale (100.64.0.0/10) can connect to port 11434
2. **Router/firewall** — no port 11434 forwarding to the internet
3. **Tailscale ACLs** — only `tag:phone` can reach port 11434 remotely

### Create/edit Ollama launchd service config

```bash
# Check current Ollama service file location
cat /Library/LaunchDaemons/com.ollama.ollama.plist
```

Edit the plist to set the `OLLAMA_HOST` environment variable:

```bash
sudo nano /Library/LaunchDaemons/com.ollama.ollama.plist
```

Add or modify the `EnvironmentVariables` key:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.ollama.ollama</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/local/bin/ollama</string>
    <string>serve</string>
  </array>
  <key>EnvironmentVariables</key>
  <dict>
    <key>OLLAMA_HOST</key>
    <string>0.0.0.0:11434</string>
  </dict>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
</dict>
</plist>
```

```bash
# Reload the service
sudo launchctl unload /Library/LaunchDaemons/com.ollama.ollama.plist
sudo launchctl load /Library/LaunchDaemons/com.ollama.ollama.plist
```

### Add macOS pf Firewall Rules to Protect Ollama

Ollama listens on all interfaces, but `pf` restricts who can actually connect:

```bash
# Create a pf anchor for Ollama protection
sudo nano /etc/pf.anchors/ollama-protect
```

Add the following content to `/etc/pf.anchors/ollama-protect`:

```
# Allow LAN access to Ollama
pass in on en0 proto tcp from 192.168.1.0/24 to any port 11434
# Allow Tailscale interface access to Ollama
pass in on utun0 proto tcp from 100.64.0.0/10 to any port 11434
# Block everything else to Ollama
block in proto tcp to any port 11434
```

> **Note:** The Tailscale interface name is typically `utun0` or `utun1`. Verify with `ifconfig | grep utun` while Tailscale is connected. Use the interface that shows a `100.x.x.x` address. Update `utun0` above if needed.

```bash
# Load the anchor (add to /etc/pf.conf):
sudo nano /etc/pf.conf
# Add the following two lines before the final "pass" line:
#   anchor "ollama-protect"
#   load anchor "ollama-protect" from "/etc/pf.anchors/ollama-protect"

# Enable and reload pf
sudo pfctl -e
sudo pfctl -f /etc/pf.conf

# Make pf load on boot
sudo defaults write /Library/Preferences/com.apple.alf globalstate -int 1
```

### Verify Ollama Binding and pf Rules

```bash
# Verify Ollama is listening on all interfaces
sudo lsof -i :11434
# Should show: ollama ... TCP *:11434 (LISTEN)

# Verify pf rules are loaded
sudo pfctl -sr | grep 11434

# Test from LAN (should work):
curl http://192.168.1.10:11434/api/tags

# Test from Tailscale IP (should work when connected to Tailscale):
curl http://100.64.0.x:11434/api/tags

# Test from internet/cellular (should FAIL):
curl --connect-timeout 5 http://YOUR_PUBLIC_IP:11434/api/tags
```

---

## Step 4: macOS Application Firewall Rules

Add Ollama to the firewall allowlist:

```bash
# Allow Ollama through Application Firewall
/usr/libexec/ApplicationFirewall/socketfilterfw --add /usr/local/bin/ollama
/usr/libexec/ApplicationFirewall/socketfilterfw --unblockapp /usr/local/bin/ollama

# Verify
/usr/libexec/ApplicationFirewall/socketfilterfw --listapps
```

> **Note:** macOS Application Firewall controls inbound connections by app. The primary defense for access control is the `pf` firewall (Step 3 above) which restricts connections by source IP. The router/VLAN firewall rules and Tailscale ACLs provide additional layers.

---

## Step 5: Local DNS / Hosts File

Add friendly names for your infrastructure:

```bash
sudo nano /etc/hosts
```

Add:
```
192.168.1.10   macstudio.local macstudio
192.168.1.20   proxmox.local proxmox
192.168.1.21   openclaw.local openclaw
192.168.20.10  agents.local agents
192.168.30.10  homeassistant.local homeassistant
```

---

## Step 6: Pull AI Models

```bash
# General purpose — fast, good quality
ollama pull llama3.1:8b

# More capable, slower
ollama pull llama3.1:70b

# Coding specialist (recommended for Agent Swarm)
ollama pull qwen2.5-coder:32b

# Vision model (for analyzing camera images)
ollama pull llava:13b

# Lightweight for HA automations (fast response)
ollama pull phi3:mini

# List installed models
ollama list
```

### Model Recommendations by Use Case

| Use Case | Recommended Model | VRAM (approx) | Notes |
|----------|-----------------|--------------|-------|
| Daily chat / assistant | llama3.1:8b | 6GB | Fast, capable |
| Complex reasoning | llama3.1:70b | 45GB | Slower, much better |
| Coding tasks | qwen2.5-coder:32b | 20GB | Best local coding model |
| Home automation | phi3:mini | 2GB | Very fast for simple tasks |
| Image analysis | llava:13b | 8GB | Camera feeds, screenshots |
| Embeddings/search | nomic-embed-text | 500MB | For semantic search |

> With 256GB unified memory on M5 Ultra, you can run multiple models simultaneously.

---

## Step 7: Test Ollama from LAN

From another machine on the same network:

```bash
# Test health endpoint
curl http://192.168.1.10:11434/api/tags

# Test inference
curl http://192.168.1.10:11434/api/generate \
  -d '{"model":"llama3.1:8b","prompt":"Say hello.","stream":false}'

# Test that it's NOT reachable from internet
# (do this from a phone on cellular, not WiFi)
curl --connect-timeout 5 http://YOUR_PUBLIC_IP:11434/api/tags
# Expected: connection timeout / refused
```

---

# Part 6: Raspberry Pi 4 Setup — Home Assistant

## Step 1: Flash Home Assistant OS

### Requirements
- Raspberry Pi 4 (4GB RAM)
- 64GB+ microSD card (or USB SSD for better reliability)
- Balena Etcher installed on your workstation

```bash
# Download the official HAOS image
# Go to: https://www.home-assistant.io/installation/raspberrypi
# Download: Home Assistant OS for Raspberry Pi 4 (64-bit)
# File: haos_rpi4-64-XX.X.img.xz
```

1. Open Balena Etcher
2. Click **Flash from file** → select the downloaded `.img.xz`
3. Click **Select target** → choose your microSD card
4. Click **Flash** → wait for completion + verification
5. Insert microSD into Pi 4
6. Connect Pi 4 to network via Ethernet (required for initial setup)
7. Power on Pi 4

---

## Step 2: Initial Home Assistant Setup

```
# Wait ~5 minutes for first boot
# Navigate to: http://homeassistant.local:8123
# (or http://192.168.30.10:8123)

# Follow onboarding:
# 1. Create admin account
# 2. Set location (for sunrise/sunset automations)
# 3. HA will auto-discover devices on your network
```

### Assign Static IP to Pi

In your router, create a DHCP reservation for the Pi's MAC address → 192.168.30.10

Or in HA: **Settings → System → Network → IPv4** → set static IP.

---

## Step 3: Nabu Casa Remote Access

```
# In Home Assistant:
Settings → Home Assistant Cloud → Sign In / Create Account

# Subscribe at: https://www.nabucasa.com ($6.50/mo)
# After subscribing, click "Connect" in HA Cloud settings

# This creates an encrypted tunnel — NO port forwarding needed
# Your HA instance gets a unique URL like:
# https://xxxxxxxxxxxxxxxx.ui.nabu.casa
```

Download the **Home Assistant** app on your phone and log in with your Nabu Casa account. Remote access is now working.

---

## Step 4: Google Coral USB TPU Setup

The Coral USB Accelerator dramatically accelerates Frigate's object detection.

```bash
# SSH into Pi (or use HA terminal add-on)
# Settings → Add-ons → Terminal & SSH

# Install Coral USB drivers (done inside HA OS terminal)
# HAOS handles this automatically when Coral is detected
# Just plug in the Coral USB and HA/Frigate will use it

# Verify Coral is detected:
lsusb
# Should show: Global Unichip Corp. or Google Inc. Coral USB Accelerator
```

> In Frigate config, you'll reference the Coral as the detector. See below.

---

## Step 5: Frigate NVR Installation

### Install Frigate Add-on

```
Settings → Add-ons → Add-on Store → Search "Frigate"
Install: Frigate NVR
Start after installation: Yes
Show in sidebar: Yes
```

### Configure Frigate

Edit the Frigate configuration. Create `/config/frigate.yml`:

```yaml
# /config/frigate.yml

mqtt:
  enabled: false  # Set to true if you use MQTT

detectors:
  coral:
    type: edgetpu
    device: usb

cameras:
  front_door:
    ffmpeg:
      inputs:
        - path: rtsp://camera-ip/stream  # Replace with your camera's RTSP URL
          roles:
            - detect
            - record
    detect:
      width: 1280
      height: 720
      fps: 5
    record:
      enabled: true
      retain:
        days: 7
        mode: motion
    motion:
      mask:
        - 0,0,0,200,300,200,300,0  # Adjust mask to exclude static areas
    objects:
      track:
        - person
        - car
        - dog
        - cat

# Snapshots storage
snapshots:
  enabled: true
  retain:
    default: 14

# Recording storage (adjust path as needed)
record:
  output_args:
    record: -f segment -segment_time 10 -segment_format mp4 -reset_timestamps 1 -strftime 1 -c copy -an
```

Restart Frigate add-on after saving config.

---

## Step 6: Ollama Integration in Home Assistant

Install the Ollama integration to enable AI-powered automations:

```
Settings → Devices & Services → Add Integration
Search: "Ollama"
```

Configure:
```
Host: http://192.168.1.10:11434
Model: phi3:mini  (fast for automations)
```

This creates a **Conversation Agent** that can process natural language commands and trigger HA actions.

---

## Step 7: Recommended HA Integrations

| Integration | Purpose |
|-------------|---------|
| **Ollama** | Local LLM for AI automations |
| **Frigate** | AI camera/NVR with object detection |
| **Google Cast** | Control Google speakers/displays |
| **Apple TV** | Presence detection + media control |
| **Z-Wave JS** | Z-Wave device control (with Z-Wave stick) |
| **Zigbee2MQTT** | Zigbee device control (with Zigbee stick) |
| **Météo-France / Met.no** | Weather for automations |
| **Mobile App** | Phone presence, notifications |
| **Local Calendar** | Schedule-based automations |
| **Assist Pipeline** | Voice assistant using local LLM |

---

## Step 8: Sample AI Automation

This automation uses the local Ollama LLM to generate a contextual notification when an unknown person is detected at the front door:

```yaml
# Configuration → Automations → Create Automation → YAML mode

alias: AI Door Alert
description: "LLM-enhanced front door notification"
trigger:
  - platform: state
    entity_id: binary_sensor.front_door_person_detected
    to: "on"
condition: []
action:
  - service: conversation.process
    data:
      agent_id: conversation.ollama
      text: >
        A person was just detected at the front door at {{ now().strftime('%I:%M %p') }}.
        Write a brief, friendly 1-sentence security notification for a homeowner.
    response_variable: llm_response
  - service: notify.mobile_app_your_phone
    data:
      title: "🚪 Front Door"
      message: >
        {{ llm_response.response.speech.plain.speech
           | default('Motion detected at front door at ' + now().strftime('%I:%M %p')) }}
mode: single
```

> **Note:** Replace `notify.mobile_app_your_phone` with your actual phone's notification service name, found under **Settings → Devices & Services → Mobile App**. The `| default(...)` fallback ensures the automation works even if the LLM response format changes.

---

# Part 7: x86 Proxmox Setup

## Hardware Selection Rationale

The Proxmox machine runs two VMs continuously. Requirements:

- **CPU:** Needs enough cores to give 4 to OpenClaw VM + 8+ to Agent Swarm VM
- **RAM:** 8GB for OpenClaw + 48GB+ for agents (agents are memory-hungry) = 64GB minimum, 128GB preferred
- **Storage:** 64GB OS + 64GB OpenClaw VM + 500GB+ for Agent Swarm = 1TB+ NVMe
- **Form Factor:** Small, quiet, always-on

### Recommended: Minisforum MS-01

| Spec | Value |
|------|-------|
| CPU | Intel Core i9-12900H (14 cores, 20 threads) |
| RAM | 64GB DDR5 (upgradeable to 128GB) |
| Storage | 1TB NVMe PCIe 4.0 (add 2nd NVMe for agent storage) |
| Network | 2.5GbE + 10GbE (SFP+) |
| Size | 197 × 172 × 55mm |
| Price | ~$500–600 (64GB config) |
| Link | [minisforum.com/ms-01](https://minisforum.com/) |

**Alternative:** Beelink SEi12 Pro or Intel NUC 13 Pro (similar specs).

**Upgrade path:** 128GB RAM variant of MS-01 (~$700) for running larger agent swarms.

---

## Step 1: Install Proxmox VE

### Create Bootable USB

```bash
# Download Proxmox VE ISO: https://www.proxmox.com/en/downloads
# Flash to USB (on Mac/Linux):
sudo dd if=proxmox-ve_8.x.iso of=/dev/sdX bs=4M status=progress

# On Windows: use Rufus or Balena Etcher
```

### Proxmox Installation

1. Boot MS-01 from USB (spam DEL or F7 during POST for boot menu)
2. Select **Install Proxmox VE (Graphical)**
3. Accept EULA
4. **Target disk:** Select 1TB NVMe (this will be erased)
5. **Country/Timezone/Keyboard:** Set appropriately
6. **Root password:** Use a strong password. Save it.
7. **Network config:**
   - Management interface: `enp2s0` (or whichever is 2.5GbE)
   - Hostname: `proxmox.local`
   - IP: `192.168.1.20/24`
   - Gateway: `192.168.1.1`
   - DNS: `192.168.1.1`
8. Review → **Install**
9. Remove USB when prompted, system will reboot

Access Proxmox web UI: `https://192.168.1.20:8006` (accept self-signed cert)

---

## Step 2: Post-Install Hardening

```bash
# SSH into Proxmox
ssh root@192.168.1.20

# Update system
apt update && apt full-upgrade -y

# Remove enterprise repo (requires subscription), add community repo
rm /etc/apt/sources.list.d/pve-enterprise.list
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-community.list
apt update

# Disable subscription nag (optional)
sed -i.bak "s/data.status !== 'Active'/false/g" \
  /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js

# Install useful tools
apt install -y htop iotop curl wget git ufw

# Configure UFW firewall
ufw default deny incoming
ufw default allow outgoing
ufw allow from 192.168.1.0/24 to any port 22   # SSH from LAN only
ufw allow from 192.168.1.0/24 to any port 8006 # Proxmox UI from LAN only
ufw enable

# Set up automatic security updates
apt install -y unattended-upgrades
dpkg-reconfigure --priority=low unattended-upgrades
```

---

## Step 3: VM Network Design

Proxmox uses Linux bridges. We'll create two bridges:

- `vmbr0` — VLAN 1 (trusted/main) — for OpenClaw VM
- `vmbr1` — VLAN 20 (agent swarm, isolated) — for Agent Swarm VM

In Proxmox UI: **Node → System → Network → Create → Linux Bridge**

```
Bridge 1 (vmbr0):
  Name: vmbr0
  Bridge ports: enp2s0 (your main NIC)
  Comment: Main LAN / VLAN 1

Bridge 2 (vmbr1):
  Name: vmbr1
  Bridge ports: (none — internal only, or tag on switch as VLAN 20)
  Comment: Agent Swarm / VLAN 20
```

Or via CLI:

```bash
# /etc/network/interfaces — add:

auto vmbr1
iface vmbr1 inet static
    address 192.168.20.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    comment AgentSwarm-VLAN20
```

```bash
# Apply network changes
ifreload -a
```

---

## Step 4: Create OpenClaw VM

In Proxmox UI: **Create VM**

```
General:
  VM ID: 100
  Name: openclaw

OS:
  ISO: Upload Ubuntu 22.04 LTS server ISO
  (Upload via Datacenter → Storage → local → Upload)

System:
  Machine: q35
  BIOS: OVMF (UEFI)
  Add EFI Disk: yes (1MB)

Disk:
  Bus: VirtIO SCSI
  Storage: local-lvm
  Size: 64GB

CPU:
  Sockets: 1
  Cores: 4
  Type: host (for best performance)

Memory:
  RAM: 8192 MB (8GB)

Network:
  Bridge: vmbr0
  Model: VirtIO
  VLAN Tag: (none — this is native VLAN 1)

Confirm → Finish
```

### Install Ubuntu 22.04 on OpenClaw VM

Start VM → Open console → Follow Ubuntu installer:

```
# During install:
Network: configure DHCP first, set static after
Hostname: openclaw
Username: ubuntu (or your preference)
Install OpenSSH: YES

# After install — set static IP:
sudo nano /etc/netplan/00-installer-config.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens18:
      addresses:
        - 192.168.1.21/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [192.168.1.1, 8.8.8.8]
```

```bash
sudo netplan apply
```

---

## Step 5: Create Agent Swarm VM

```
General:
  VM ID: 101
  Name: agent-swarm

OS:
  ISO: Ubuntu 22.04 LTS (same ISO)

Disk:
  Bus: VirtIO SCSI
  Size: 500GB

CPU:
  Cores: 8 (or all remaining — i9-12900H has 14 cores, leave 4 for system + OpenClaw)
  Type: host

Memory:
  RAM: 49152 MB (48GB)  — adjust based on total RAM minus OpenClaw's 8GB

Network:
  Bridge: vmbr1  ← ISOLATED BRIDGE
  Model: VirtIO
```

Set static IP:

```yaml
# /etc/netplan/00-installer-config.yaml
network:
  version: 2
  ethernets:
    ens18:
      addresses:
        - 192.168.20.10/24
      routes:
        - to: default
          via: 192.168.20.1  # Proxmox host acts as gateway
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

### Enable IP Forwarding on Proxmox (for Agent Swarm internet access)

```bash
# On Proxmox host:
echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf
sysctl -p

# NAT for Agent Swarm internet access
iptables -t nat -A POSTROUTING -s 192.168.20.0/24 -o vmbr0 -j MASQUERADE

# FORWARD rules — ORDER MATTERS: specific rules before broad ones

# 1. Allow established/related connections (important for stateful traffic)
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 2. Allow Agent Swarm to reach ONLY Mac Studio port 11434
iptables -A FORWARD -s 192.168.20.0/24 -d 192.168.1.10 -p tcp --dport 11434 -j ACCEPT

# 3. Block Agent Swarm from reaching OpenClaw VM (before broad internet ACCEPT)
iptables -A FORWARD -s 192.168.20.0/24 -d 192.168.1.21 -j DROP

# 4. Block Agent Swarm from rest of trusted LAN (before broad internet ACCEPT)
iptables -A FORWARD -s 192.168.20.0/24 -d 192.168.1.0/24 -j DROP

# 5. Allow Agent Swarm internet access (git, npm, pip, package registries)
iptables -A FORWARD -s 192.168.20.0/24 -o vmbr0 -p tcp -m multiport --dports 80,443 -j ACCEPT

# 6. Drop everything else from Agent Swarm
iptables -A FORWARD -s 192.168.20.0/24 -j DROP

# Persist rules
apt install -y iptables-persistent
netfilter-persistent save
```

```bash
# Verify rules are correct
iptables -L FORWARD -n -v --line-numbers
# Rules should appear in the order above
```

### Verify Proxmox Is Routing Correctly Between Networks

```bash
# On Proxmox host:

# Check both bridges exist and are up
ip addr show vmbr0   # Should show 192.168.1.20/24
ip addr show vmbr1   # Should show 192.168.20.1/24

# Check IP forwarding is enabled
cat /proc/sys/net/ipv4/ip_forward
# Expected: 1

# Check NAT rule is active
iptables -t nat -L POSTROUTING -n -v
# Should show MASQUERADE rule for 192.168.20.0/24

# Check FORWARD rules are correct and in right order
iptables -L FORWARD -n -v --line-numbers
```

### Verify Agent Swarm VM Can Reach Mac Studio

```bash
# From Agent Swarm VM (after iptables rules are set):
ip route show
# Should show: default via 192.168.20.1

traceroute 192.168.1.10
# Should hop through 192.168.20.1 then reach 192.168.1.10

curl http://192.168.1.10:11434/api/tags
# Should return model list JSON
```

---

## Step 6: Install OpenClaw on OpenClaw VM

```bash
# SSH into OpenClaw VM
ssh ubuntu@192.168.1.21

# Install Node.js 22
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Install OpenClaw
# Refer to official installation docs at: https://openclaw.ai/docs/install
# The following commands reflect the current installation method as of this writing.
# If these fail, check the latest instructions at the URL above.

# Method 1: Direct install (most common)
curl -fsSL https://get.openclaw.ai | bash

# After installation, verify:
openclaw --version

# Initialize OpenClaw (interactive setup — follow prompts):
# This will ask you to:
# 1. Authenticate with your OpenClaw account
# 2. Link your WhatsApp number (scan QR code)
# 3. Configure your workspace
openclaw init

# Set Ollama as the LLM backend:
openclaw config set llm.provider ollama
openclaw config set llm.host http://192.168.1.10:11434
openclaw config set llm.model llama3.1:8b

# Start OpenClaw as a background service:
openclaw start

# Enable auto-start on system boot:
openclaw service install
openclaw service start

# Verify OpenClaw is running and connected:
openclaw status
# Expected output: Connected | WhatsApp: linked | LLM: ollama@192.168.1.10:11434
```

> **Note:** If any of the above commands fail, run `openclaw help` and consult https://openclaw.ai/docs for the current installation procedure. OpenClaw's CLI interface may update over time.

---

## Step 7: Install Docker and Agent Frameworks on Agent Swarm VM

```bash
# SSH into Agent Swarm VM
ssh ubuntu@192.168.20.10

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu
newgrp docker

# Verify Docker
docker run hello-world

# Install Node.js (for Claude Code, Codex CLIs)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Install Python (for OpenHands and tooling)
sudo apt install -y python3 python3-pip python3-venv

# Install Claude Code CLI
npm install -g @anthropic-ai/claude-code

# Configure Claude Code to use local Ollama
# Claude Code supports OpenAI-compatible APIs via environment variables
export ANTHROPIC_API_KEY="ollama"  # Dummy value
export ANTHROPIC_BASE_URL="http://192.168.1.10:11434/v1"

# Or add to /etc/environment permanently:
echo 'ANTHROPIC_BASE_URL=http://192.168.1.10:11434/v1' | sudo tee -a /etc/environment

# Note: Claude Code works best with Anthropic's actual API for complex coding tasks.
# For local model use, OpenHands or direct Ollama API calls are often better.
# You may want to keep the real ANTHROPIC_API_KEY for Claude Code
# and use local Ollama for lighter agent tasks via OpenHands.

# Install OpenAI Codex CLI
npm install -g @openai/codex

# Set environment variables for agent API keys
# (Add to /etc/environment or per-user .bashrc)
echo 'ANTHROPIC_API_KEY=your_anthropic_key_here' | sudo tee -a /etc/environment
echo 'OPENAI_API_KEY=your_openai_key_here' | sudo tee -a /etc/environment

# Configure agents to use local Ollama on Mac Studio
echo 'OLLAMA_HOST=http://192.168.1.10:11434' | sudo tee -a /etc/environment

# For OpenAI-compatible apps pointing to local Ollama:
echo 'OPENAI_BASE_URL=http://192.168.1.10:11434/v1' | sudo tee -a /etc/environment
echo 'OPENAI_API_KEY=ollama' | sudo tee -a /etc/environment  # dummy key

source /etc/environment
```

### Install OpenHands (via Docker)

```bash
# OpenHands self-hosted setup
mkdir -p ~/openhands
cd ~/openhands

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
services:
  openhands:
    image: ghcr.io/all-hands-ai/openhands:main
    ports:
      - "3000:3000"
    environment:
      - SANDBOX_RUNTIME_CONTAINER_IMAGE=ghcr.io/all-hands-ai/runtime:main
      - LLM_API_KEY=ollama
      - LLM_BASE_URL=http://192.168.1.10:11434
      - LLM_MODEL=ollama/qwen2.5-coder:32b
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ~/.openhands-state:/.openhands-state
    restart: unless-stopped
EOF

docker compose up -d

# Access OpenHands UI at: http://192.168.20.10:3000
```

---

# Part 8: Remote Access Setup

## Tailscale: Remote LLM Access from Phone

### Install Tailscale on Mac Studio

```bash
# Download and install Tailscale for macOS from:
# https://tailscale.com/download/macos

# Or via Homebrew:
brew install --cask tailscale

# Open Tailscale from menu bar → Log In
# Authenticate with your Tailscale account
```

### Install Tailscale on iPhone/Android

1. Download **Tailscale** from App Store or Google Play
2. Sign in with the same account used on Mac Studio
3. Enable VPN when away from home

### Verify Both Devices Are Connected

In Tailscale admin console (`login.tailscale.com`):
- Both your Mac Studio and phone should appear as connected nodes
- Note the Tailscale IP assigned to Mac Studio (e.g., `100.64.0.1`)

---

## Configure Tailscale ACLs

In Tailscale admin console → **Access Controls**:

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["tag:phone"],
      "dst": ["tag:mac-studio:11434"]
    }
  ],
  "tagOwners": {
    "tag:phone": ["autogroup:admin"],
    "tag:mac-studio": ["autogroup:admin"]
  },
  "hosts": {
    "mac-studio": "100.64.0.1"
  }
}
```

Tag your devices in the Tailscale console:
- Mac Studio → Tag: `mac-studio`
- Phone → Tag: `phone`

This ensures **only your phone** can reach Ollama via Tailscale. Even if someone else joins your Tailscale network, they cannot reach port 11434.

> **Ollama Binding Note:** Ollama is configured to listen on all interfaces (`0.0.0.0`) with access controlled by the macOS `pf` firewall (configured in Part 5, Step 3). The Tailscale ACLs provide an additional layer — only your tagged phone device is allowed through.

---

## Test Remote LLM from Phone

### Option 1: Enchanted (iOS, recommended)

1. Install **Enchanted** from App Store
2. Settings → Server URL: `http://100.64.0.x:11434` (your Mac Studio's Tailscale IP)
3. Connect → select a model
4. Chat — works like ChatGPT, 100% local

### Option 2: Open WebUI (any browser)

```bash
# Deploy Open WebUI on Agent Swarm VM (accessible via Tailscale if you add it there)
# Or run it on Mac Studio directly:

docker run -d \
  -p 8080:8080 \
  -e OLLAMA_BASE_URL=http://192.168.1.10:11434 \
  --name open-webui \
  --restart unless-stopped \
  ghcr.io/open-webui/open-webui:main
```

Access at `http://100.64.0.x:8080` via Tailscale.

---

## WhatsApp + OpenClaw Remote Access

No setup required beyond the OpenClaw installation. Here's how it works:

1. OpenClaw, when started, connects **outbound** to OpenClaw's cloud gateway via WebSocket
2. The gateway holds a persistent connection to your home OpenClaw instance
3. Your WhatsApp number is linked during OpenClaw's `init` setup
4. Messages you send to the designated number → gateway → your home → Mac Studio → response back

**Security properties:**
- No inbound ports are opened on your router
- The cloud gateway is a dumb relay — it cannot read message content (encrypted in transit)
- If your home OpenClaw instance is offline, messages queue until it reconnects

---

## Nabu Casa HA Remote Access Testing

```bash
# Verify from HA Settings → Home Assistant Cloud:
# Status should show: "Connected"
# Remote UI URL: https://xxxxxxxx.ui.nabu.casa

# Test from phone (on cellular, NOT home WiFi):
# Open HA app → tap the external URL
# Should load your dashboard

# Test from browser:
# Navigate to your .ui.nabu.casa URL
# Log in → full HA functionality remotely
```

---

# Part 9: Security Hardening Checklist

## Router / Network

- [ ] Router firmware is up to date
- [ ] Default admin credentials changed (not `admin/admin`)
- [ ] Remote router management disabled (no WAN access to admin UI)
- [ ] WPS disabled (vulnerability to brute-force)
- [ ] UPnP disabled (devices cannot auto-open ports)
- [ ] Guest WiFi network isolated from main LAN
- [ ] VLAN 20 (Agent Swarm) cannot reach VLAN 1 except Mac Studio:11434
- [ ] VLAN 30 (IoT/HA) cannot reach VLAN 1 except Mac Studio:11434
- [ ] No port forwarding rules for Mac Studio, Proxmox, or Pi
- [ ] DNS-over-HTTPS enabled on router (optional but recommended)
- [ ] Log outbound connections (for anomaly detection)

## Mac Studio

- [ ] FileVault enabled (full disk encryption)
- [ ] macOS Firewall enabled with stealth mode
- [ ] Ollama bound to `0.0.0.0:11434` with macOS `pf` firewall restricting access to LAN + Tailscale only
- [ ] Remote Login (SSH) disabled unless actively used
- [ ] Screen Sharing disabled
- [ ] Remote Management disabled
- [ ] Automatic security updates enabled
- [ ] Screen lock set to immediate after sleep
- [ ] No port forwarding on router to port 11434
- [ ] Verify Ollama NOT reachable from WAN (test from cellular)
- [ ] Tailscale ACLs restrict port 11434 to `tag:phone` only

## Proxmox Host

- [ ] Root password is strong (20+ chars, unique)
- [ ] Proxmox web UI accessible from LAN only (UFW rule)
- [ ] SSH accessible from LAN only
- [ ] Enterprise repo disabled (no unnecessary outbound subscription checks)
- [ ] iptables rules in place: Agent Swarm cannot reach OpenClaw VM
- [ ] iptables rules persistent (iptables-persistent installed)
- [ ] Proxmox host updated regularly
- [ ] VMs do not share storage in ways that allow cross-VM access

## Agent Swarm VM

- [ ] Docker daemon not exposed on network (unix socket only)
- [ ] Agent API keys stored as environment variables, not in code
- [ ] No sensitive credentials in docker-compose files in git
- [ ] VM network isolated to vmbr1 (VLAN 20)
- [ ] Cannot reach OpenClaw VM (verified with ping/curl test)
- [ ] Can reach Mac Studio port 11434 (required for inference)

## Raspberry Pi / Home Assistant

- [ ] HA admin account uses strong password
- [ ] HA uses Nabu Casa for remote access (not open ports)
- [ ] Pi not directly accessible from internet
- [ ] HA integrations use local devices only (no cloud where possible)
- [ ] SSH disabled on Pi if not needed
- [ ] Frigate only processes video locally (no cloud upload)
- [ ] Camera RTSP streams not accessible outside VLAN 30

## Network Monitoring (Recommended)

```bash
# Install ntopng on Proxmox for traffic visibility:
apt install -y ntopng

# Or use pi-hole on a separate VM/container for:
# - DNS-based ad/tracking blocking
# - DNS query logging (anomaly detection)

# Regularly review:
# - Outbound connections from Agent Swarm VM
# - Any unexpected connections to/from Mac Studio
# - HA add-on network activity
```

---

# Part 10: Testing & Validation

## Component Health Checks

### Mac Studio / Ollama

```bash
# From Mac Studio itself:
ollama list        # Should show installed models
ollama ps          # Running models

# From OpenClaw VM (should work):
curl http://192.168.1.10:11434/api/tags
# Expected: JSON list of models

# From Agent Swarm VM (should work):
curl http://192.168.1.10:11434/api/tags
# Expected: JSON list of models

# From phone on CELLULAR (should FAIL):
curl --connect-timeout 5 http://YOUR_PUBLIC_IP:11434/api/tags
# Expected: curl: (28) Connection timeout
# If you get JSON back, STOP — your LLM is exposed to the internet
```

### Proxmox VMs

```bash
# Verify OpenClaw VM is running:
ssh ubuntu@192.168.1.21 "systemctl status openclaw"

# Verify Agent Swarm VM is running:
ssh ubuntu@192.168.20.10 "docker ps"

# Verify inter-VM isolation (from Agent Swarm VM):
ssh ubuntu@192.168.20.10
ping 192.168.1.21    # Should FAIL (OpenClaw VM)
curl http://192.168.1.10:11434/api/tags  # Should SUCCEED (Mac Studio)
curl http://google.com                    # Should SUCCEED (internet)
```

### Home Assistant

```bash
# Verify HA is reachable locally:
curl -s http://192.168.30.10:8123 | grep -o "<title>[^<]*</title>"
# Expected: <title>Home Assistant</title>

# Verify Nabu Casa connection:
# HA Settings → Cloud → Status: Connected

# Verify Frigate + Coral:
# HA Sidebar → Frigate → Detectors tab → should show "edgetpu" with inference ms
```

---

## Security Verification Tests

### Test 1: LLM Not Internet-Accessible

```bash
# Turn off home WiFi on phone, use cellular only
# Open browser or Termius app on phone:
curl --connect-timeout 10 http://YOUR_HOME_PUBLIC_IP:11434/api/tags

# PASS: Connection refused or timed out
# FAIL: You receive JSON — immediately check Ollama binding and firewall rules
```

### Test 2: Agent Swarm Cannot Reach OpenClaw

```bash
# From Agent Swarm VM:
ssh ubuntu@192.168.20.10
curl --connect-timeout 3 http://192.168.1.21:8080  # OpenClaw port
# Expected: curl: (28) Connection timed out — PASS
```

### Test 3: Agent Swarm CAN Reach LLM

```bash
# From Agent Swarm VM:
curl http://192.168.1.10:11434/api/generate \
  -d '{"model":"phi3:mini","prompt":"hello","stream":false}'
# Expected: JSON with response field — PASS
```

---

## End-to-End Test: WhatsApp → OpenClaw → Mac Studio

1. **Send a WhatsApp message** to your linked number:
   > "What is 7 times 8?"

2. **Watch OpenClaw logs** on OpenClaw VM:
   ```bash
   ssh ubuntu@192.168.1.21
   openclaw logs --follow
   # Should show: incoming message, LLM request to 192.168.1.10:11434, response sent
   ```

3. **Verify you receive a response** on WhatsApp within 5-10 seconds.

4. **Confirm LLM was used locally:**
   ```bash
   # On Mac Studio:
   ollama ps
   # Should show model loaded and active during/after the request
   ```

---

## End-to-End Test: HA Automation Using Local LLM

1. **Trigger the AI Door Alert automation manually:**
   ```
   HA → Developer Tools → Services
   Service: automation.trigger
   Entity: automation.ai_door_alert
   ```

2. **Check HA logs:**
   ```
   Settings → System → Logs → Filter: "conversation"
   # Should show Ollama conversation request and response
   ```

3. **Verify phone notification received** with AI-generated text (not a static string).

---

# Part 11: Maintenance & Growth Path

## Regular Maintenance Tasks

### Weekly
```bash
# Update Ollama models
ollama pull llama3.1:8b     # Gets latest version if available
ollama pull qwen2.5-coder:32b

# Check for unused models
ollama list
ollama rm model-name:tag    # Remove models you're not using

# Update Agent Swarm Docker images
ssh ubuntu@192.168.20.10
docker compose pull          # In each compose project directory
docker compose up -d

# Check HA for updates
# HA → Settings → System → Updates
```

### Monthly
```bash
# Update Proxmox
ssh root@192.168.1.20
apt update && apt full-upgrade -y

# Update OpenClaw VM
ssh ubuntu@192.168.1.21
sudo apt update && sudo apt upgrade -y

# Update Agent Swarm VM
ssh ubuntu@192.168.20.10
sudo apt update && sudo apt upgrade -y

# Review Proxmox backup status (set up backups if not done)
# Datacenter → Backup → Add scheduled backup for both VMs
```

### Quarterly
- Rotate API keys (Anthropic, OpenAI)
- Review Tailscale ACLs — remove stale devices
- Review HA integrations — disable unused ones
- Audit firewall rules — still appropriate?
- Check disk usage on all VMs

---

## Scaling the Agent Swarm

### Add More Agents

```bash
# On Agent Swarm VM, each agent is a Docker container
# Scale specific agents via docker-compose:

cd ~/openhands
docker compose scale openhands=3  # Run 3 parallel OpenHands instances

# Or add new agent frameworks:
# SWE-agent, AutoGPT, CrewAI, etc.
# All point to http://192.168.1.10:11434 for inference
```

### When 64GB RAM Isn't Enough

1. Upgrade Proxmox host to 128GB RAM (~$150 for 2x64GB DDR5 kit)
2. Increase Agent Swarm VM memory allocation in Proxmox
3. Alternatively, add a second Proxmox machine and use Proxmox Cluster

---

## When to Upgrade Pi 4 to Pi 5

Upgrade when:
- [ ] Frigate struggles to process multiple camera streams simultaneously
- [ ] HA automations feel sluggish (>2s response)
- [ ] You want to run heavier local ML workloads on the Pi itself
- [ ] You add 4+ cameras and notice CPU peaking

**Pi 5 advantages for this use case:**
- 2–3× faster CPU
- Better USB bandwidth (Coral USB performs better)
- PCIe interface (can add M.2 HAT for local NVMe storage)

Migration: Flash new HAOS image on Pi 5 SD card, restore from HA backup, swap hardware.

---

## K3s Upgrade Path (Multi-Node Proxmox Cluster)

If you add a second Proxmox machine and want to run agents at larger scale:

```bash
# Install K3s on first Proxmox VM (server)
curl -sfL https://get.k3s.io | sh -

# Get node token
cat /var/lib/rancher/k3s/server/node-token

# Join second node (agent)
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.1.21:6443 \
  K3S_TOKEN=<node-token> sh -

# Deploy OpenHands as K3s deployment
kubectl apply -f openhands-deployment.yaml

# Horizontal scaling
kubectl scale deployment openhands --replicas=5
```

K3s makes it easy to:
- Distribute agent workloads across multiple machines
- Auto-restart failed agents
- Rolling updates with zero downtime
- Resource quotas per agent type

---

## Model Updates with Ollama

```bash
# Check which models are outdated
ollama list
# Compare with: https://ollama.com/library

# Update a specific model (pulls latest tag)
ollama pull llama3.1:8b
ollama pull qwen2.5-coder:32b

# Update all models (script)
ollama list | awk 'NR>1 {print $1}' | while read model; do
  echo "Updating $model..."
  ollama pull "$model"
done

# Remove old versions (Ollama auto-manages this, but to force cleanup)
ollama list  # Find old versions by digest
ollama rm old-model:tag
```

---

## OpenClaw Updates

```bash
# Update OpenClaw
ssh ubuntu@192.168.1.21

# Check current version
openclaw --version

# Update (method depends on installation)
npm update -g openclaw
# or
openclaw update

# Restart after update
openclaw restart

# If breaking changes: check changelog at openclaw.ai/changelog
# Before updating, take a Proxmox snapshot of the OpenClaw VM:
# In Proxmox UI: VM 100 → Snapshots → Take Snapshot
```

---

## Backup Strategy

```bash
# Proxmox VM backups (set up scheduled backups)
# Datacenter → Backup → Add

Schedule: Sunday 02:00
Storage: local (or external USB if attached)
VMs: 100 (openclaw), 101 (agent-swarm)
Mode: Snapshot
Compression: ZSTD

# Home Assistant backups
# Settings → System → Backups → Add Automatic Backup
# Frequency: Daily
# Keep: 7 days
# (Use Nabu Casa for off-site backup: HA Cloud → Backup)

# Ollama models don't need backup — re-pull from ollama.com
# But model customizations/Modelfiles should be saved:
ls ~/.ollama/models/
```

---

*Document maintained by: OpenClaw Home AI Stack*  
*Last updated: 2026*  
*Architecture version: 1.0*

---

> **Quick Reference — IP Addresses**
> 
> | Device | IP |
> |--------|----|
> | Router | 192.168.1.1 |
> | Mac Studio (Ollama) | 192.168.1.10 |
> | Proxmox Host | 192.168.1.20 |
> | OpenClaw VM | 192.168.1.21 |
> | Agent Swarm VM | 192.168.20.10 |
> | Raspberry Pi / HA | 192.168.30.10 |
