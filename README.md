# Home AI Infrastructure Architecture
### A Private, Powerful, Always-On AI Stack — Built in Your Home

> **Version:** 3.0 | **Status:** Active  
> **Hardware:** Minisforum MS-S1 MAX · (Phase 2: Apple Mac Studio)

---

## Table of Contents

- [Part 1: Requirements & Features](#part-1-requirements--features)
- [Part 2: System Overview](#part-2-system-overview)
- [Part 3: Bill of Materials](#part-3-bill-of-materials)
- [Part 4: Phase 1 — Core AI Stack](#part-4-phase-1--core-ai-stack)
  - [4.1 MS-S1 MAX: OS & Proxmox](#41-ms-s1-max-os--proxmox)
  - [4.2 Component Design](#42-component-design)
  - [4.3 Create the NemoClaw VM](#43-create-the-nemoclaw-vm)
    - [GPU Passthrough for Strix Halo](#step-3b-gpu-passthrough-for-the-nemoclaw-vm-strix-halo--amd-ryzen-ai-max)
  - [4.4 Ubuntu Setup Inside the VM](#44-ubuntu-setup-inside-the-vm)
  - [4.5 NemoClaw + OpenClaw](#45-nemoclaw--openclaw)
  - [4.6 Ollama + Local Models](#46-ollama--local-models)
  - [4.7 Multi-Agent Configuration](#47-multi-agent-configuration)
  - [4.8 Remote Access](#48-remote-access)
  - [4.9 Security Checklist](#49-security-checklist)
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
| **Privacy** | All AI inference runs locally. Cloud relay (OpenClaw gateway) are dumb pipes — they move bytes, never process content |
| **Security** | NemoClaw OpenShell sandbox isolates the AI agent. No public inbound ports. |
| **Always-on** | Proxmox auto-starts VMs/LXC on boot. Systemd services restart on failure. |
| **Router compatibility** | Designed for standard home routers including Google Nest. No VLANs required in Phase 1/2. |
| **Upgrade path** | Each phase is independently deployable. Phase 1 is fully functional on its own. |

---

# Part 2: System Overview

## Phase 1 Architecture

```
╔══════════════════════════════════════════════════════════════════════╗
║                    HOME NETWORK (192.168.86.0/24)                    ║
║                          Google Nest Router                          ║
║                                                                      ║
║  ┌────────────────────────────────────────────────────────────────┐  ║
║  │                  MS-S1 MAX (192.168.86.38)                     │  ║
║  │                  Proxmox VE — Bare Metal                       │  ║
║  │                                                                │  ║
║  │  ┌────────────────────────────────────────────────────────┐    │  ║
║  │  │  NemoClaw VM  (192.168.86.21)                          │    │  ║
║  │  │                                                        │    │  ║
║  │  │  OpenClaw agent          OpenShell sandbox             │    │  ║
║  │  │  WhatsApp gateway        Coding agent orchestration    │    │  ║
║  │  │                                                        │    │  ║
║  │  │  Main model:   Nemotron 120B (NVIDIA cloud, free)      │    │  ║
║  │  │  Worker model: Qwen 2.5 Coder 32B (local Ollama)       │    │  ║
║  │  └────────────────────────────────────────────────────────┘    │  ║
║  │                                                                │  ║
║  │  ┌────────────────────────────────────────────────────────┐    │  ║
║  │  │  Ollama LXC  (192.168.86.26)                           │    │  ║
║  │  │                                                        │    │  ║
║  │  │  Radeon 890M — full GPU, no sharing needed             │    │  ║
║  │  │  Qwen 2.5 Coder 32B  (agents, port 11434)              │    │  ║
║  │  │  phi3:mini           (fast fallback)                   │    │  ║
║  │  └────────────────────────────────────────────────────────┘    │  ║
║  └────────────────────────────────────────────────────────────────┘  ║
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
║                    HOME NETWORK (192.168.86.0/24)                    ║
║                                                                      ║
║  ┌─────────────────────────────────┐  ┌───────────────────────────┐  ║
║  │  MS-S1 MAX (192.168.86.38)      │  │  Mac Studio M5 Ultra      │  ║
║  │  Proxmox — Orchestration        │  │  (192.168.86.10)          │  ║
║  │                                 │  │                           │  ║
║  │  NemoClaw VM ──────────────────►│  │  Ollama                   │  ║
║  │  Ollama LXC (fallback)          │  │  70B / 120B / 200B+       │  ║
║  │                                 │  │  256GB unified memory     │  ║
║  └─────────────────────────────────┘  └───────────────────────────┘  ║
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
    │       └── Ollama LXC     (192.168.86.26)
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
| **AI + Proxmox Host** | Minisforum MS-S1 MAX (AMD Ryzen AI Max+ 395, 128GB LPDDR5X, 2TB NVMe) | ~$2,920 | Ships with Windows — wipe and install Proxmox |
| **Ethernet cable** | Cat6 | ~$10 | Wired connection to router required |
| **USB drive (8GB+)** | Any brand | ~$10 | For Proxmox installer |

**Phase 1 Total: ~$2,940**

## Phase 2 Additional Hardware

| Item | Model | Price | Notes |
|------|-------|-------|-------|
| **LLM Inference Server** | Apple Mac Studio M5 Ultra (256GB unified memory) | ~$4,000–5,000 | M4 Ultra (192GB) available now at ~$4,199 |
| **Ethernet cable** | Cat6 | ~$10 | — |

## Phase 4 Additional Hardware (Home Assistant)

| Item | Model | Price | Notes |
|------|-------|-------|-------|
| **AI Camera Accelerator** | Google Coral USB Accelerator | ~$60–80 | Passed through to HA VM via Proxmox USB passthrough |

## Phase 5 Additional Hardware (Media Server)

| Item | Model | Price | Notes |
|------|-------|-------|-------|
| **Media Storage** | WD Elements 8TB USB 3.0 | ~$120 | Or NAS |
| **NAS (optional)** | Synology DS223 + 2× Seagate IronWolf 4TB | ~$480 | 8TB usable RAID 1 |

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

## Accounts to Create Before You Start

You'll need these before beginning installation. Create them now so you have the credentials ready:

| Account | URL | Why |
|---------|-----|-----|
| **NVIDIA Build** | [build.nvidia.com](https://build.nvidia.com) | Free API key for Nemotron 120B (the main AI model) |
| **OpenClaw** | [openclaw.ai](https://openclaw.ai) | Subscription for the AI agent platform |
| **Tailscale** | [tailscale.com](https://tailscale.com) | Free VPN for remote access to local LLM |

---

# Part 4: Phase 1 — Core AI Stack

## 4.1 MS-S1 MAX: OS & Proxmox

### Step 1: Firmware Update (Do This First)

Before wiping Windows, apply any BIOS/firmware updates. This is important — firmware
updates often can't be applied after switching OS.

```
1. Boot into Windows (shipped state)
2. Go to minisforum.com → Support → MS-S1 MAX → Downloads
3. Download and run any BIOS or firmware update tools listed
4. Restart, confirm update applied in BIOS (hold DEL during POST)
5. Now proceed to wipe and install Proxmox
```

### Step 2: Download Proxmox VE ISO

On your laptop/desktop:

1. Go to [proxmox.com/en/downloads](https://www.proxmox.com/en/downloads)
2. Download **Proxmox VE 8.2** or newer
   *(Must be 8.2+ for full AMD Ryzen AI Max+ 395 kernel support)*
3. Verify the SHA256 checksum shown on the download page:
   ```bash
   # macOS
   shasum -a 256 proxmox-ve_*.iso

   # Windows (PowerShell)
   Get-FileHash proxmox-ve_*.iso -Algorithm SHA256
   ```

### Step 3: Flash to USB

1. Download [Balena Etcher](https://etcher.balena.io) (free, works on Mac/Windows/Linux)
2. Open Etcher → Flash from file → select the Proxmox ISO
3. Select your USB drive
4. Click Flash
5. When done, eject the USB

### Step 4: Boot and Install Proxmox

1. Plug the USB into the MS-S1 MAX
2. Power on and immediately hold **F7** to get the boot menu
   *(If F7 doesn't work, try DEL to enter BIOS and set USB as first boot device)*
3. Select the USB drive from the boot menu
4. At the Proxmox installer, choose **Install Proxmox VE (Graphical)**
5. Accept the EULA
6. Target disk: select the 2TB NVMe — click **Next**
7. Location: set your country/timezone — click **Next**
8. Set a strong root password and enter your email — click **Next**
9. Network configuration:

```
Hostname:  proxmox.local
IP:        192.168.86.38/24
Gateway:   192.168.86.1
DNS:       8.8.8.8
```

5. Access Proxmox web UI: `https://192.168.86.38:8006`

### Step 5: Reserve the Static IP

Before the machine reboots and comes up on the network:
- Open the **Google Home app** → Wi-Fi → Devices → find the MS-S1 MAX → Reserve IP → set to `192.168.86.38`

### Step 6: Access Proxmox Web UI

On your laptop, open a browser and go to:
```
https://192.168.86.38:8006
```

You'll get a self-signed certificate warning — click through it. Log in with:
```
Username: root
Password: [the password you set during install]
```

You'll see a "No valid subscription" popup — click OK. You don't need a subscription.

### Step 7: Post-Install Hardening

Open a terminal and SSH into the Proxmox host:

```bash
ssh root@192.168.86.38

Run all of these:

```bash
# Update everything
apt update && apt full-upgrade -y

# Switch to the free community repo (removes subscription nag on apt updates)
rm /etc/apt/sources.list.d/pve-enterprise.list
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-community.list
apt update

# Firewall — restrict Proxmox UI and SSH to LAN only
apt install -y ufw
ufw default deny incoming
ufw default allow outgoing
ufw allow from 192.168.86.0/24 to any port 22
ufw allow from 192.168.86.0/24 to any port 8006
ufw enable

# Auto security updates
apt install -y unattended-upgrades
dpkg-reconfigure --priority=low unattended-upgrades
# Select "Yes" when prompted
```

Verify the firewall is active:
```bash
ufw status
# Should show: Status: active, with rules for ports 22 and 8006
```

---

## 4.2 Component Design

Phase 1 is two components on the MS-S1 MAX:

| Component | Type | vCPU | RAM | Disk | IP | Purpose |
|-----------|------|------|-----|------|----|---------|
| NemoClaw VM | VM | 4 | 16GB | 80GB | 192.168.86.21 | OpenClaw + sandbox + coding agents |
| Ollama LXC | LXC | 4 | 8GB* | 32GB | 192.168.86.26 | Local model inference |
| **Proxmox host** | — | — | ~8GB | — | 192.168.86.38 | OS overhead |
| **Model weights** | — | — | ~96GB | — | — | Unified memory pool |

> \* 8GB covers Ollama process overhead only. Model weights load into the AMD Radeon 890M's
> unified memory pool via bind-mounted DRI device — separate from the container's RAM budget.
> With NemoClaw VM and host consuming ~32GB, approximately 96GB is available for models.

---

## 4.3 Create the NemoClaw VM

> **Why a VM and not an LXC?**
> NemoClaw uses NVIDIA OpenShell for sandboxing. OpenShell requires a full Linux kernel with
> strict process and namespace isolation — it enforces network egress policy, filesystem
> containment, and syscall filtering at a level that LXC containers can't provide (LXC shares
> the host kernel and its namespace separation is weaker). A VM gives OpenShell a clean,
> isolated kernel boundary it can fully control. This is the same reason the NemoClaw README
> spins up a lightweight Linux VM on macOS instead of using containers.

### Step 1: Download the Ubuntu ISO into Proxmox

In the Proxmox web UI:

1. In the left panel, click **local (proxmox)** → **ISO Images**
2. Click **Download from URL**
3. Paste this URL:
   ```
   https://releases.ubuntu.com/22.04/ubuntu-22.04.3-live-server-amd64.iso
   ```
4. Click **Query URL** — Proxmox will fill in the filename
5. Click **Download** — wait for it to complete (about 1–2 minutes)

Alternatively, if you prefer to verify the checksum manually first:
```bash
# On the Proxmox host SSH session:
cd /var/lib/vz/template/iso/
wget https://releases.ubuntu.com/22.04/ubuntu-22.04.3-live-server-amd64.iso
# Verify checksum (find expected hash at releases.ubuntu.com/22.04/SHA256SUMS)
sha256sum ubuntu-22.04.3-live-server-amd64.iso
```

### Step 2: Create the VM

In the Proxmox web UI, click **Create VM** (top right):

**General tab:**
```
Node:    proxmox
VM ID:   100
Name:    nemoclaw
```

**OS tab:**
```
Storage:        local
ISO image:      ubuntu-22.04.3-live-server-amd64.iso
Type:           Linux
Version:        6.x - 2.6 Kernel
```

**System tab:**
```
Graphic card:   Default
Machine:        q35
BIOS:           OVMF (UEFI)
Add EFI Disk:   ✓ checked  →  Storage: local-lvm
SCSI Controller: VirtIO SCSI Single
Qemu Agent:     ✓ checked
```

**Disks tab:**
```
Bus/Device:   SCSI  |  scsi0
Storage:      local-lvm
Disk size:    80
Cache:        Write back
Discard:      ✓ checked  (enables TRIM for the SSD)
```

**CPU tab:**
```
Sockets:  1
Cores:    4
Type:     host    ← important: gives VM access to AMD AI instructions
```

**Memory tab:**
```
Memory (MiB):  16384
Balloon:       ✓ checked  (allows unused RAM to be reclaimed by host)
```

**Network tab:**
```
Bridge:   vmbr0
Model:    VirtIO (paravirtualized)
Firewall: ✓ checked
```

Click **Finish**. The VM is created but not started yet.

### Step 3: Start the VM and Open the Console

1. In the left panel, click **100 (nemoclaw)**
2. Click **Start** (top right)
3. Click **Console** — a noVNC window opens showing the Ubuntu installer

---

### Step 3b: GPU Passthrough for the NemoClaw VM (Strix Halo / AMD Ryzen AI Max)

> **Do this before installing Ubuntu** if you want the NemoClaw VM to use the AMD
> Radeon 890M / 8060S directly for local inference (instead of routing through the
> Ollama LXC). Skip this section if you're using NVIDIA cloud inference only.
>
> **Known limitations on Strix Halo:**
> - The iGPU can only be passed through to a **Windows VM once per host boot** (the "reset bug").
>   Linux VMs are more forgiving on Proxmox 9+ with recent kernels (6.15+).
> - Dynamic VRAM allocation is unstable — set a fixed VRAM amount in the BIOS and never
>   change it at the OS level.
> - You must supply your own extracted VBIOS — someone else's dump may not work.

#### 1. Blacklist AMD GPU Drivers on the Proxmox Host

The host must not load the AMD GPU drivers, otherwise it will grab the GPU before VFIO
can claim it and the VM will hang on boot.

SSH into the Proxmox host:

```bash
ssh root@192.168.86.38
```

Create the blacklist file:

```bash
cat > /etc/modprobe.d/pve-blacklist.conf << 'EOF'
blacklist radeon
blacklist amdgpu
blacklist snd_hda_intel
EOF
```

Tell VFIO which PCI IDs to claim. Find your GPU's IDs first:

```bash
lspci -nn | grep -i amd
# Look for lines like:
# c5:00.0 VGA compatible controller [0300]: Advanced Micro Devices ... [1002:1586]
# c5:00.1 Audio device [0403]: Advanced Micro Devices ... [1002:1640]
```

Create the VFIO config with those IDs:

```bash
cat > /etc/modprobe.d/vfio.conf << 'EOF'
options vfio-pci ids=1002:1586,1002:1640 disable_vga=1
EOF
```

> Replace `1002:1586,1002:1640` with the IDs from your `lspci` output if they differ.

Rebuild the initramfs and set the kernel cmdline:

```bash
# Add initcall_blacklist to prevent framebuffer from claiming the GPU at boot
echo "$(cat /etc/kernel/cmdline) initcall_blacklist=sysfb_init" > /etc/kernel/cmdline

update-initramfs -u -k all
proxmox-boot-tool refresh
```

Reboot the Proxmox host:

```bash
reboot
```

After reboot, verify VFIO claimed the GPU (not the host driver):

```bash
lspci -k | grep -A3 "VGA\|Audio" | grep "Kernel driver in use"
# Should show: vfio-pci
# NOT: amdgpu or radeon
```

#### 2. Extract Your VBIOS

The Strix Halo iGPU requires a VBIOS ROM file injected at passthrough time. Without it,
the VM will hang during boot. You must extract it from your specific machine — a VBIOS
from another machine may not work.

```bash
# On the Proxmox host — find the GPU's PCI address
lspci -nn | grep "VGA" | grep AMD
# e.g.: c5:00.0 VGA ...

# Extract the VBIOS (replace c5:00.0 with your address)
cd /usr/share/kvm/
echo 1 > /sys/bus/pci/devices/0000:c5:00.0/rom
cat /sys/bus/pci/devices/0000:c5:00.0/rom > vbios.bin
echo 0 > /sys/bus/pci/devices/0000:c5:00.0/rom

# Verify the file exists and has content
ls -lh vbios.bin
# Should be ~128KB to ~512KB — if it's 0 bytes, something went wrong
```

> If the `rom` file doesn't exist, your GPU may not expose it at that path. In that case,
> download a VBIOS extraction tool (`amdvbflash`) or find a pre-extracted VBIOS for your
> exact board revision from the Strix Halo community (strixhalo.wiki has some for the
> Bosgame M5 and EVO-X2).

#### 3. Add PCI Devices to the VM — Use Separate Entries, Not "All Functions"

> **Important:** Do NOT use the "All Functions" checkbox in the Proxmox UI for Strix Halo.
> It doesn't let you set `rombar=0` on the audio device individually, and passing the audio
> device with rombar enabled causes boot hangs. Add each PCI function as a separate entry.

SSH into the Proxmox host and edit the VM config directly:

```bash
nano /etc/pve/qemu-server/100.conf
```

Find the PCI addresses for the iGPU, its audio device, and the USB controller in the
same IOMMU group:

```bash
# List IOMMU groups to find what's grouped with your GPU
find /sys/kernel/iommu_groups/ -type l | sort -V | while read f; do
  printf "Group $(basename $(dirname $f)): "
  lspci -nns $(basename $f)
done | grep -A5 "$(lspci -nn | grep VGA | grep AMD | awk '{print $1}')"
```

Add these lines to `/etc/pve/qemu-server/100.conf` (adjust PCI addresses to match your output):

```
# GPU passthrough — Strix Halo iGPU
# Replace c5:00.X with the actual PCI addresses from your lspci output
hostpci0: 0000:c5:00.0,pcie=1,romfile=vbios.bin,x-vga=1
hostpci1: 0000:c5:00.1,pcie=1,rombar=0
hostpci2: 0000:c5:00.4,pcie=1,rombar=0
```

| Setting | Meaning |
|---------|---------|
| `romfile=vbios.bin` | Injects the extracted VBIOS — required for Strix Halo |
| `x-vga=1` | Marks this as the primary GPU — required, VM hangs without it |
| `rombar=0` on audio/USB | Hides the ROM bar — prevents boot hangs on those devices |
| Separate entries | Allows different settings per function — "All Functions" checkbox can't do this |

Also ensure the VM uses `q35` machine type and `host` CPU type (already set in Step 2 above).

#### 4. Start the VM

```bash
qm start 100
```

Watch the console via Proxmox web UI. The VM should POST and reach the Ubuntu installer
(or boot Ubuntu if already installed). If it hangs:

- **Black screen / no POST:** VBIOS is missing or wrong — re-extract or try a different dump
- **Hangs at "Loading initial ramdisk":** Host drivers weren't blacklisted properly — re-check `/etc/modprobe.d/pve-blacklist.conf` and rebuild initramfs
- **GPU not visible inside VM:** Run `lspci` inside the VM — if the AMD GPU doesn't appear, check that VFIO claimed it on the host (`lspci -k | grep vfio`)

#### 5. Inside the VM: Kernel Requirements

Once Ubuntu is installed inside the VM, ensure you're on a recent enough kernel:

```bash
# Inside the NemoClaw VM
uname -r
# Needs to be 6.15 or newer for Strix Halo amdgpu driver support
```

If the kernel is older, upgrade:

```bash
sudo apt update && sudo apt install -y linux-generic-hwe-22.04
sudo reboot
```

After reboot, verify the GPU is visible:

```bash
lspci | grep -i amd
# Should show the Radeon GPU

# Check amdgpu loaded
lsmod | grep amdgpu

# Check render node exists
ls /dev/dri/
# Should show: card0  renderD128
```

---

## 4.4 Ubuntu Setup Inside the VM

### Step 1: Ubuntu Installer

The Ubuntu Server installer will walk you through these screens. Use arrow keys and Enter to navigate.

**Language:** English → Enter

**Keyboard:** Detect keyboard layout, or select yours manually → Done

**Type of install:** Ubuntu Server *(not minimized)* → Done

**Network connections:** You'll see `ens18` — leave it on DHCP for now. We'll set the static IP after install.
→ Done

**Configure proxy:** Leave blank → Done

**Ubuntu archive mirror:** Leave default → Done → wait for it to test connectivity

**Guided storage configuration:**
```
Use an entire disk: ✓ selected
[the 80GB VirtIO disk should be selected]
Set up this disk as an LVM group: ✓ checked
```
→ Done → Continue (on the confirmation screen)

**Profile setup:**
```
Your name:           [your name, e.g. Kevin]
Your server's name:  nemoclaw
Pick a username:     ubuntu
Choose a password:   [strong password — save this]
Confirm password:    [same]
```
→ Done

**Ubuntu Pro:** Skip for now → Continue

**SSH Setup:**
```
Install OpenSSH server: ✓ checked   ← critical — enables SSH access
Import SSH identity:    No
```
→ Done

**Featured server snaps:** Don't select anything → Done

The installer will now run. This takes about 5–10 minutes. When it finishes, you'll see **"Reboot Now"** — click it.

When the VM reboots, the console will show a login prompt:
```
nemoclaw login:
```

Log in with `ubuntu` and the password you set.

### Step 2: Install the QEMU Guest Agent

The guest agent lets Proxmox communicate cleanly with the VM (shutdown, IP reporting, etc.):

```bash
sudo apt update
sudo apt install -y qemu-guest-agent
sudo systemctl enable qemu-guest-agent
sudo systemctl start qemu-guest-agent
```

### Step 3: Set a Static IP

First, find your actual interface name — it varies by machine:

```bash
ip link show
# Look for something like: ens18, enp6s18, eth0
# It will NOT be lo (that's loopback)
```

> **Note:** The interface is commonly `ens18` but on some hardware (including the MS-S1 MAX)
> it shows up as `enp6s18`. Use whatever `ip link show` reports — replace `ens18` in all
> commands below with your actual interface name.

If the interface has no IP yet (no inet address), bring it up and grab a DHCP lease first so you can SSH in:

```bash
sudo ip link set enp6s18 up   # use your interface name
sudo dhclient enp6s18
ip addr show enp6s18
# Should now show a DHCP IP — use this to SSH in from your laptop
```

Now set the static IP:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Replace the entire file contents with (substituting your interface name):

```yaml
network:
  version: 2
  ethernets:
    enp6s18:        # ← replace with your interface name from ip link show
      addresses: [192.168.86.21/24]
      routes:
        - to: default
          via: 192.168.86.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
```

Apply it:

```bash
sudo netplan apply
```

Verify the new IP is active:

```bash
ip addr show enp6s18   # use your interface name
# Should show: inet 192.168.86.21/24
```

### Step 4: Verify SSH Access from Your Laptop

Open a terminal on your laptop:

```bash
ssh ubuntu@192.168.86.21
```

If it connects and shows the Ubuntu prompt, you're good. You can close the noVNC console in Proxmox and work entirely from SSH from here on.

> **Tip:** Set up SSH key authentication so you don't need a password every time:
> ```bash
> # On your laptop — generate a key if you don't have one
> ssh-keygen -t ed25519 -C "your_email@example.com"
>
> # Copy your public key to the VM
> ssh-copy-id ubuntu@192.168.86.21
>
> # Now test — should log in without a password
> ssh ubuntu@192.168.86.21
> ```

### Step 5: Basic System Hardening

```bash
# Update all packages
sudo apt update && sudo apt full-upgrade -y

# Install useful tools
sudo apt install -y curl wget git htop unzip ufw

# Firewall — allow SSH only from your LAN
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.86.0/24 to any port 22
sudo ufw --force enable

# Auto security updates
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
# Select "Yes" when prompted

# Set timezone
sudo timedatectl set-timezone America/Chicago
# Adjust to your timezone. List options: timedatectl list-timezones
```

### Step 6: Install Prerequisites for NemoClaw

```bash
# Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Verify Docker works
docker run hello-world
# Should print: "Hello from Docker!"

# Node.js 22
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Verify versions
node --version    # Must be >= 20 (will show v22.x.x)
docker --version  # Any recent version is fine
```

---

## 4.5 NemoClaw + OpenClaw

> NemoClaw wraps OpenClaw in a hardened sandbox — enforcing network egress policy,
> filesystem isolation, and process sandboxing via NVIDIA OpenShell.
> OpenClaw's built-in coding agent orchestration (Claude Code, Codex via ACP harness)
> runs directly inside this sandbox. No separate agent VM required.

### Before You Start: Get Your Credentials

You need two things before running the installer:

**1. NVIDIA API Key (free)**
- Go to [build.nvidia.com](https://build.nvidia.com)
- Click any model → click **"Get API Key"**
- Sign in with (or create) an NVIDIA developer account — it's free
- Copy the key — looks like `nvapi-xxxxxxxxxxxxxxxxxxxx`

**2. OpenClaw** — set up during the NemoClaw onboard wizard, no separate account needed beforehand

### Step 1: Install NemoClaw

SSH into the NemoClaw VM:

```bash
ssh ubuntu@192.168.86.21

curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER && newgrp docker

curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

node --version   # Must be >= 20
docker --version
```

Install NemoClaw:

```bash
curl -fsSL https://www.nvidia.com/nemoclaw.sh | bash
source ~/.bashrc

# Verify it installed
nemoclaw --version
```

### Step 2: Run the Onboard Wizard

```bash
nemoclaw onboard
```

The wizard will ask for:

1. **NVIDIA API key** — paste the `nvapi-xxx` key you copied above
2. **Sandbox name** — type `my-assistant` (or any name you like, no spaces)
3. **OpenClaw account** — enter your openclaw.ai email and password
4. **Inference provider** — select `NVIDIA Cloud (Nemotron 120B)` — this is the default
5. **WhatsApp** — select `Yes`

The wizard creates your sandbox and pulls the OpenClaw container. This takes 2–5 minutes.

### Step 3: Connect WhatsApp

```bash
nemoclaw my-assistant connect
```

This drops you into the sandbox shell. Your prompt will change to something like `sandbox$`.

Inside the sandbox:

```bash
sandbox$ openclaw channels login whatsapp
```

A QR code will appear in the terminal.

On your phone:
```
WhatsApp → Settings → Linked Devices → Link a Device → scan the QR code
```

After scanning, you'll see a confirmation in the terminal. Then:

```bash
sandbox$ openclaw start
```

Your AI assistant is now live. Send a WhatsApp message — it should respond within 10 seconds.

### Step 4: Configure Auto-Start on Boot

Exit the sandbox (`exit` or Ctrl+D), then back on the VM:

```bash
# Create a systemd service so NemoClaw starts automatically on VM boot
sudo tee /etc/systemd/system/nemoclaw.service > /dev/null << 'EOF'
[Unit]
Description=NemoClaw AI Sandbox
After=docker.service network-online.target
Requires=docker.service

[Service]
Type=simple
User=ubuntu
ExecStart=/usr/local/bin/nemoclaw my-assistant start --foreground
ExecStop=/usr/local/bin/nemoclaw my-assistant stop
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable nemoclaw
sudo systemctl start nemoclaw

# Check it's running
sudo systemctl status nemoclaw
```

### Step 5: NemoClaw Management Commands

```bash
nemoclaw my-assistant status       # Health check
nemoclaw my-assistant logs -f      # Live logs (Ctrl+C to exit)
nemoclaw my-assistant connect      # Shell into sandbox
nemoclaw my-assistant stop         # Stop the sandbox
nemoclaw my-assistant start        # Start the sandbox
openshell term                     # Live network request monitor
```

### NemoClaw Security Layers

| Layer | What it blocks |
|-------|---------------|
| Network | Agent calling unauthorized hosts — surfaced in `openshell term` for approval |
| Filesystem | Reads/writes outside `/sandbox` and `/tmp` |
| Process | Privilege escalation, dangerous syscalls |
| Inference | All model calls routed through OpenShell gateway |

---

## 4.6 Ollama + Local Models

> **Why LXC, not on the host?** If Ollama crashes on the Proxmox host it can freeze
> the hypervisor — taking everything down. Running it in an LXC contains crashes to
> just the LXC while Proxmox and the NemoClaw VM stay up.

### Step 1: Download the Ubuntu LXC Template

In the Proxmox web UI:

1. In the left panel, click **local (proxmox)** → **CT Templates**
2. Click **Templates** button
3. Search for `ubuntu-22.04` → select **ubuntu-22.04-standard** → click **Download**
4. Wait for the download to complete

### Step 2: Create the Ollama LXC

In the Proxmox web UI, click **Create CT** (top right):

**General tab:**
```
CT ID:      200
Hostname:   ollama
Template:   Ubuntu 22.04
Disk:       32GB
CPU:        4 cores
RAM:        8192 MB
Network:    vmbr0, IP 192.168.86.26/24, Gateway 192.168.86.1
Unprivileged: YES
```

**Template tab:**
```
Storage:   local
Template:  ubuntu-22.04-standard_*.tar.zst
```

**Disks tab:**
```
Storage:  local-lvm
Disk size: 32
```

**CPU tab:**
```
Cores: 4
```

**Memory tab:**
```
Memory:  8192
Swap:    2048
```

**Network tab:**
```
Name:     eth0
Bridge:   vmbr0
IPv4:     Static
IPv4/CIDR: 192.168.86.26/24
Gateway:  192.168.86.1
```

**DNS tab:**
```
DNS servers: 8.8.8.8
```

**Confirm tab:** Review, then click **Finish**

> **Important:** Do NOT start the container yet — you need to add the GPU bind-mount first.

### Step 3: Bind-Mount the GPU

SSH into the Proxmox host:

```bash
ssh root@192.168.86.38
nano /etc/pve/lxc/200.conf
```

Add these lines at the bottom of the file:

```
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```

Save and exit (`Ctrl+O`, Enter, `Ctrl+X`).

Also enable "unprivileged" mode with nesting (required for GPU access):

```bash
# In the Proxmox web UI:
# 200 (ollama) → Options → Features → Edit → check "Nesting"
# Or via CLI:
echo "features: nesting=1" >> /etc/pve/lxc/200.conf
```

### Step 4: Start the LXC and Install Ollama

```bash
# Start the container
pct start 200

# Enter it
pct enter 200
```

You're now inside the Ollama LXC as root. Run:

```bash
# Update and install tools
apt update && apt full-upgrade -y
apt install -y curl wget

# Verify the GPU is visible
ls /dev/dri/
# Must show: card0  renderD128
# If this directory is empty, the bind-mount didn't work — double-check /etc/pve/lxc/200.conf

# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh
```

### Step 5: Configure Ollama

Configure Ollama to listen on all interfaces (so the NemoClaw VM can reach it) and set the context window:

```bash
mkdir -p /etc/systemd/system/ollama.service.d
cat > /etc/systemd/system/ollama.service.d/override.conf << 'EOF'
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_NUM_CTX=32768"
EOF

systemctl daemon-reload
systemctl enable ollama
systemctl restart ollama

# Verify it's running
systemctl status ollama
# Should show: Active: active (running)
```

> **Context window:** `OLLAMA_NUM_CTX=32768` sets a 32K token default for all models.
> Increase to `65536` for 64K if needed. Larger context uses more unified memory for the
> KV cache — 32K is a good starting point for 32B models.

### Step 6: Verify GPU Usage

```bash
# Pull a small model first for a quick test
ollama pull phi3:mini

# Run it with verbose output and check for GPU
ollama run phi3:mini "say hello" --verbose
```

In the output, look for a line containing:
```
using GPU: AMD Radeon Graphics
```

If you see it, the GPU is working. If you see `using CPU`, the bind-mount isn't working — revisit Step 3.

### Step 7: Pull the Main Models

```bash
# This takes a while — 32B model is ~20GB
ollama pull qwen2.5-coder:32b    # Primary coding agent model

# phi3 already pulled in Step 6 — fast fallback
ollama list
# Should show both models
```

### Step 8: Restrict LAN Access (Security)

Back on the Proxmox host, restrict Ollama so it's only accessible from the NemoClaw VM and Proxmox host (not from your phone, laptop, etc.):

```bash
# On Proxmox host
iptables -A FORWARD -s 192.168.86.0/24 -d 192.168.86.26 -p tcp --dport 11434 -j ACCEPT
iptables -A FORWARD -d 192.168.86.26 -j DROP
apt install -y iptables-persistent
netfilter-persistent save
```

Verify from the NemoClaw VM that Ollama is reachable:

```bash
ssh ubuntu@192.168.86.21
curl http://192.168.86.26:11434/api/tags
# Should return JSON with your model list
```

---

## 4.7 Multi-Agent Configuration

This sets up three agents: the main orchestrator (uses NVIDIA cloud Nemotron 120B),
a coding agent (uses local Qwen 2.5 Coder), and a fast fallback (uses local phi3).

```bash
# Connect into the NemoClaw sandbox
nemoclaw my-assistant connect

# Edit the OpenClaw config
sandbox$ nano ~/.openclaw/openclaw.json
```

Replace the contents with:

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
        "baseUrl": "http://192.168.86.26:11434",
        "api": "ollama"
      }
    }
  }
}
```

Save and exit.

Allow Ollama in the OpenShell network policy (exit sandbox first):

```bash
sandbox$ exit

# On the NemoClaw VM host
openshell policy add --sandbox my-assistant \
  --host 192.168.86.26 --port 11434 --protocol rest --name ollama-local
```

Restart the sandbox to apply config:

```bash
nemoclaw my-assistant stop
nemoclaw my-assistant start
```

---

## 4.8 Remote Access

Everything uses outbound-only connections — no ports forwarded on your router.

### Tailscale: Access Your Local LLM from Anywhere

Install Tailscale **on the Ollama LXC** (not the Proxmox host) — this gives your phone
direct access to port 11434 via its own Tailscale IP.

```bash
# On Proxmox host
pct enter 200

# Inside Ollama LXC
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
tailscale ip -4   # Note the Tailscale IP (e.g. 100.64.0.2)
```

Install Tailscale on your phone (same Tailscale account). Then point your app directly at the LXC:

```
Enchanted app (iOS) → Settings → Server URL → http://100.64.0.2:11434
```

Or run Open WebUI on the NemoClaw VM for a browser-based chat UI:

```bash
# On the NemoClaw VM (ssh ubuntu@192.168.86.21)
docker run -d \
  -p 8080:8080 \
  -e OLLAMA_BASE_URL=http://192.168.86.26:11434 \
  --name open-webui \
  --restart unless-stopped \
  ghcr.io/open-webui/open-webui:main
```

Then install Tailscale on the NemoClaw VM too, and access Open WebUI from your phone:

```bash
# On NemoClaw VM
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
tailscale ip -4   # e.g. 100.64.0.3
```

```
Browser on phone → http://100.64.0.3:8080
```

---

## 4.9 Security Checklist

### Proxmox Host
- [ ] Proxmox UI accessible from LAN only (`ufw status` shows rules for port 8006)
- [ ] SSH accessible from LAN only (`ufw status` shows rule for port 22)
- [ ] Ollama port 11434 blocked except from NemoClaw VM (`iptables -L FORWARD`)
- [ ] iptables rules persisted (`netfilter-persistent save` was run)
- [ ] Unattended upgrades enabled

### NemoClaw VM
- [ ] UFW enabled, SSH from LAN only
- [ ] OpenShell network policy reviewed (`openshell term`)
- [ ] NVIDIA API key not stored in code — only in `~/.nemoclaw/credentials.json`
- [ ] OpenClaw workspace doesn't contain plaintext passwords

### Ollama LXC
- [ ] Only accessible from NemoClaw VM and Proxmox host
- [ ] Not reachable from your phone/laptop (test: `curl --connect-timeout 5 http://192.168.86.26:11434/api/tags` from your laptop — should fail)

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
System Settings → Privacy & Security → FileVault → Turn On
System Settings → Network → Firewall → Turn On
System Settings → Network → Firewall → Options → Enable Stealth Mode
System Settings → General → Sharing → Remote Login → Off
System Settings → General → Sharing → Screen Sharing → Off
```

### Install Ollama

```bash
brew install --cask ollama
```

### Configure with Extended Context and LAN Binding

```bash
sudo nano /Library/LaunchDaemons/com.ollama.ollama.plist
```

Find the `<key>EnvironmentVariables</key>` section (or add it if missing) and set:

```xml
<key>EnvironmentVariables</key>
<dict>
  <key>OLLAMA_HOST</key>
  <string>0.0.0.0:11434</string>
  <key>OLLAMA_NUM_CTX</key>
  <string>65536</string>
</dict>
```

Apply the config:

```bash
sudo launchctl unload /Library/LaunchDaemons/com.ollama.ollama.plist
sudo launchctl load /Library/LaunchDaemons/com.ollama.ollama.plist
```

### Firewall: Allow LAN Access to Ollama Only

```bash
sudo nano /etc/pf.anchors/ollama-protect
```

```
pass in on en0 proto tcp from 192.168.86.0/24 to any port 11434
pass in on utun0 proto tcp from 100.64.0.0/10 to any port 11434
block in proto tcp to any port 11434
```

```bash
echo 'anchor "ollama-protect" from "/etc/pf.anchors/ollama-protect"' \
  | sudo tee -a /etc/pf.conf
sudo pfctl -f /etc/pf.conf
sudo pfctl -e
```

### Pull Models

```bash
ollama pull qwen2.5-coder:72b    # Primary coding agent (~46GB)
ollama pull llama3.1:70b          # Orchestration (~45GB)
ollama pull deepseek-r1:70b       # Complex reasoning (~45GB)
ollama pull qwen2.5:7b            # Fast tasks (~5GB)
```

## 5.3 Switch OpenClaw to Mac Studio

```bash
# NemoClaw sandbox — update openclaw.json
"ollama": { "baseUrl": "http://192.168.86.10:11434" }

openshell policy add --sandbox my-assistant \
  --host 192.168.86.10 --port 11434 --protocol rest --name mac-studio-ollama
```

Restart:

```bash
nemoclaw my-assistant stop
nemoclaw my-assistant start
```

## 5.4 Model Capabilities

| Model | RAM | Speed (M5 Ultra) | Best for |
|-------|-----|-------|---------|
| qwen2.5:7b | 5GB | 200+ tok/s | Fast tasks, HA |
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
| Ollama LXC | 192.168.86.26 | 192.168.86.26 | 1 |
| Mac Studio | 192.168.86.10 | **192.168.2.10** | **20** |

After Phase 3, update openclaw.json and OpenShell policy to use `192.168.2.10`.

---

# Part 5.6: Phase 4 — Home Assistant (Optional)

> **Independent of Phase 2 and 3** — add at any time.
> Runs as a VM on the existing MS-S1 MAX alongside the AI stack.

## What You Need

- Google Coral USB Accelerator (~$60–80) — for AI camera detection via Frigate
- No additional compute hardware

## Create the Home Assistant VM

HAOS requires a VM — it cannot run in an LXC.

### Step 1: Download and Import the HAOS Image

SSH into the Proxmox host:

```bash
# On Proxmox host — download and import HAOS image
HAOS_VER=$(curl -s https://api.github.com/repos/home-assistant/operating-system/releases/latest \
  | grep '"tag_name"' | cut -d'"' -f4)
wget "https://github.com/home-assistant/operating-system/releases/download/${HAOS_VER}/haos_ova-${HAOS_VER}.qcow2.xz"
xz -d "haos_ova-${HAOS_VER}.qcow2.xz"
qm importdisk 101 "haos_ova-${HAOS_VER}.qcow2" local-lvm
```

### Step 2: Create the VM

Start the VM and access HA at `http://192.168.86.30:8123` after ~5 minutes.

Set static IP in Google Nest app: **Devices → HA VM → Reserve IP → 192.168.86.30**

### Pass the Coral USB to the HA VM

Plug the Coral USB into the MS-S1 MAX, then:

**General tab:**
```
VM ID: 101  |  Name: home-assistant
```

**OS tab:**
```
Do not use any media   ← we'll import the disk manually
```

**System tab:**
```
Machine:    q35
BIOS:       OVMF (UEFI)
Add EFI Disk: ✓  →  Storage: local-lvm
```

**Disks tab:**
```
Delete the default disk (we'll import the HAOS disk below)
```

**CPU tab:**
```
Cores: 2  |  Type: host
```

**Memory tab:**
```
Memory: 4096
```

**Network tab:**
```
Bridge: vmbr0  |  Model: VirtIO
```

Click **Finish**.

### Step 3: Import the HAOS Disk

```bash
# On Proxmox host — import the qcow2 image as VM 101's disk
qm importdisk 101 /tmp/haos_ova-*.qcow2 local-lvm
```

In the Proxmox web UI:
1. Click **101 (home-assistant)** → **Hardware**
2. You'll see **Unused Disk 0** — click it → click **Edit**
3. Set Bus/Device to **SCSI** → **scsi0** → click **Add**
4. Click **Options** → **Boot Order** → drag `scsi0` to the top → click **OK**

### Step 4: Start Home Assistant

```bash
pct start 101   # or click Start in the web UI
```

Wait 3–5 minutes for first boot. Then open:
```
http://192.168.86.30:8123
```

*(Proxmox will report the HAOS VM's IP via the guest agent once HA is running)*

Complete the HA onboarding wizard in the browser.

Set static IP: Google Home app → Wi-Fi → Devices → find the HA VM → Reserve IP → `192.168.86.30`

### Step 5: Pass the Coral USB to the HA VM

Plug the Coral USB into the MS-S1 MAX. Then in the Proxmox web UI:

```
101 (home-assistant) → Hardware → Add → USB Device
Device: [select the Google Coral USB Accelerator from the list]
```

Click **Add**. The Coral appears inside HAOS automatically. Verify in Frigate settings.

### Step 6: Add a Dedicated Ollama Instance for HA

To keep HA inference from competing with coding agent workloads, add a second Ollama
instance on a different port inside the existing Ollama LXC:

```bash
pct enter 200

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

# Pull small models for HA use
OLLAMA_HOST=localhost:11435 ollama pull qwen2.5:7b
OLLAMA_HOST=localhost:11435 ollama pull phi3:mini

# Verify
curl http://localhost:11435/api/tags
```

Allow the HA VM to reach the HA Ollama port on the Proxmox host:

```bash
# On Proxmox host — update firewall rules
iptables -I FORWARD -s 192.168.86.0/24 -d 192.168.86.26 -p tcp --dport 11435 -j ACCEPT
netfilter-persistent save
```

### Step 7: Install Frigate Add-on

In Home Assistant:
```
Settings → Add-ons → Add-on Store → search "Frigate" → Install
```

Basic config (`/config/frigate.yml`):

```yaml
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

### Step 8: Connect Ollama to Home Assistant

```
Settings → Devices & Services → Add Integration → Ollama
Host: http://192.168.86.26:11435
Model: qwen2.5:7b
```

### Step 9: Remote Access via Nabu Casa

```
Settings → Home Assistant Cloud → Sign In
```

Subscribe at [nabucasa.com](https://www.nabucasa.com) ($6.50/mo). No port forwarding required — outbound tunnel only.

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

In the Proxmox web UI, click **Create CT**:

```
CT ID:      202
Hostname:   media-server
Password:   [set root password]
Template:   ubuntu-22.04-standard_*.tar.zst
Disk:       32GB  (local-lvm)
CPU:        4 cores
RAM:        4096 MB
Network:    vmbr0, IP 192.168.86.22/24, Gateway 192.168.86.1
Unprivileged: YES
```

Click **Finish** — do NOT start yet.

Add the GPU bind-mount:

```bash
ssh root@192.168.86.38
nano /etc/pve/lxc/202.conf
```

Add at the bottom:

```
features: nesting=1
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```

Start and install Jellyfin:

```bash
pct start 202
pct enter 202

apt update
curl -fsSL https://repo.jellyfin.org/install-debuntu.sh | sudo bash
systemctl enable jellyfin && systemctl start jellyfin
```

Access Jellyfin: `http://192.168.86.22:8096`

Enable hardware transcoding in Jellyfin:
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

---

# Part 7: Testing & Validation

## Phase 1 Checks

Run these after completing Phase 1 to confirm everything is wired up correctly.

```bash
# Ollama running
curl http://192.168.86.26:11434/api/tags

# Context window configured
curl http://192.168.86.26:11434/api/show -d '{"name":"qwen2.5-coder:32b"}' | grep num_ctx

# 3. Ollama NOT reachable from internet (run this on cellular data, not WiFi)
curl --connect-timeout 5 http://YOUR_PUBLIC_IP:11434/api/tags
# Expected: connection timeout — good, it's locked down

# 4. NemoClaw sandbox health
nemoclaw my-assistant status
# Expected: Running, all green

# 5. OpenShell network policy
openshell term
# Review any blocked requests — approve Ollama LXC traffic if prompted
```

## WhatsApp End-to-End Test

Send a WhatsApp message to yourself: *"What's 7 times 8?"*

Expected: response within 5–10 seconds.

Watch live logs if it's not responding:

```bash
nemoclaw my-assistant logs -f
```

## Phase 4 (Home Assistant) Checks

```bash
# HA Ollama instance
curl http://192.168.86.26:11435/api/tags

# Frigate Coral detector is active
# HA UI → Settings → Devices → Frigate → Detectors → coral → Active
```

## Phase 2 (Mac Studio) Checks

```bash
curl http://192.168.86.10:11434/api/tags
curl http://192.168.86.10:11434/api/show -d '{"name":"llama3.1:70b"}' | grep num_ctx
```

---

# Part 8: Maintenance

## Weekly

```bash
# Update Ollama models (LXC)
pct enter 200
ollama pull qwen2.5-coder:32b
ollama pull phi3:mini

# Check NemoClaw health
nemoclaw my-assistant status

# Review network requests the sandbox has made
openshell term

# Check service status
systemctl status ollama
systemctl status nemoclaw   # On the NemoClaw VM
```

If Phase 4 is deployed:
```bash
systemctl status ollama-ha
```

## Monthly

```bash
ssh root@192.168.86.38
apt update && apt full-upgrade -y

ssh ubuntu@192.168.86.21 "sudo apt update && sudo apt upgrade -y"

# Update Ollama LXC
pct enter 200
apt update && apt upgrade -y

# Update NemoClaw itself
nemoclaw update
```

## Proxmox Backups

In the Proxmox web UI:
```
Datacenter → Backup → Add
Schedule:      Sunday 02:00
Storage:       local
VMs:           100 (nemoclaw)
LXC:           200 (ollama)
Mode:          Snapshot
Compression:   ZSTD
```

> If Phase 4 deployed: add VM 101 (home-assistant) to the backup job.
> If Phase 5 deployed: add LXC 202 (media-server) to the backup job.

## Phase 2: Update All Mac Studio Models

```bash
# SSH into Mac Studio or run in Terminal
ollama list | awk 'NR>1 {print $1}' | while read model; do
  echo "Updating $model..."
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
> | Ollama LXC | 192.168.86.26 | 192.168.86.26 | Always VLAN 1 |
> | Mac Studio (Phase 2) | 192.168.86.10 | 192.168.2.10 | Moves to VLAN 20 |
> | HA VM (Phase 4) | 192.168.86.30 | 192.168.86.30 | VLAN 1 |
> | Media Server LXC (Phase 5) | 192.168.86.22 | 192.168.86.22 | VLAN 1 |

---

*Architecture version: 3.0 — Full step-by-step instructions added for Ubuntu ISO download,
VM/LXC creation, Ubuntu installer walkthrough, SSH setup, NemoClaw onboarding with
credential prerequisites, and OpenClaw subscription setup. Phase 1 is a lean
two-component AI stack (NemoClaw VM + Ollama LXC). Raspberry Pi removed from plan.*  
*Hardware: Minisforum MS-S1 MAX + (Phase 2) Apple Mac Studio M5 Ultra*  
*Networking: Google Nest compatible — no VLANs required in Phase 1/2*
