# Phase 3: Home Assistant on Wammy

Oct 23, 2026 · @J

**Status:** 🟢 Done

## What I built

Home Assistant OS (HAOS) running as a VM on Proxmox: a self-hosted smart home hub that connects devices, runs automations and keeps control on my own network instead of each brand's cloud.

- HAOS VM with a static IP in the static zone, using Pi-hole for DNS
- 2FA, updates and a first backup before adding anything
- Companion app on my phone in "Most secure" connection mode
- Network printer integrated, with a low-ink notification automation

## Why a VM, not an LXC

HAOS is a complete appliance operating system. It manages its own add-ons, updates and container runtime, so it expects a whole machine to itself. A VM is the officially supported way to run it on Proxmox and keeps it isolated from my other services.

## The VM

I downloaded the current stable `haos_ova` image and built the VM with `qm` (the VM equivalent of `pct`):

```
qm create $ID --name homeassistant --machine q35 --bios ovmf --ostype l26 \
  --cores 2 --memory 4096 --cpu host \
  --scsihw virtio-scsi-pci --net0 virtio,bridge=vmbr0 \
  --efidisk0 local-lvm:0,efitype=4m,pre-enrolled-keys=0 \
  --agent enabled=1 --onboot 1
qm disk import $ID haos_ova-<version>.qcow2 local-lvm
qm set $ID --scsi0 local-lvm:vm-$ID-disk-1,discard=on,ssd=1 --boot order=scsi0
qm resize $ID scsi0 32G
qm start $ID
```

| Option | Why |
| --- | --- |
| `--bios ovmf` + `--efidisk0` | HAOS needs UEFI firmware |
| `pre-enrolled-keys=0` | Secure Boot off; HAOS isn't signed for it |
| `--cpu host` | Passes the real CPU's features through for performance |
| `virtio` | Fast paravirtualised disk and network drivers |
| `--agent enabled=1` | Proxmox can see the VM's IP and shut it down cleanly |
| `--onboot 1` | Starts automatically with the host |
| `discard=on` | Freed space is returned to thin-provisioned storage |

Resources: 2 cores, 4 GB RAM, 32 GB disk. I checked headroom first with `free -h` and `pvesm status`.

I looked up the current stable version from Home Assistant's published version file instead of hardcoding one:

```
curl -s https://raw.githubusercontent.com/home-assistant/version/master/stable.json | grep '"ova"' | cut -d '"' -f 4
```

## Security choices

- **Static IP** in the static zone, with Pi-hole as DNS, so Home Assistant's own lookups are filtered and logged.
- **2FA** (authenticator app) on the owner account, and a non-obvious username.
- **Least privilege:** the first account is the owner; anyone else gets a separate non-admin account.
- **Backup before changes:** a local backup before adding any devices. Off-machine backups wait for Phase 2.
- **Companion app in "Most secure" mode:** Home Assistant uses plain `http://` on my LAN, so the app only connects when it can confirm I'm on my home Wi-Fi. It won't send my session over unencrypted HTTP on a network that just looks similar. Remote access will come through Tailscale (Phase 4), not open ports.
- **Local over cloud where possible:** local integrations keep working without the internet and don't send device activity to a vendor.

## Devices

| Device | Result |
| --- | --- |
| Network printer | Added through the IPP integration: ink levels and status as sensors |
| Govee bulb (H6008) | No LAN Control option, so cloud-only. Left on Alexa for now; enabled Device Safety to lock it to my account |
| Alexa | Deferred. Controlling Home Assistant from Alexa needs either a paid cloud subscription or exposing Home Assistant to the internet |

For cloud-only devices, the community store (HACS) has custom integrations, but they run unreviewed inside Home Assistant with full access. I'm treating that as a supply-chain risk to take on deliberately, not as a quick fix.

## First automation

**When** the printer's ink sensor drops below 10%, **then** send a notification to my phone. Tested with **Run actions**, which skips the trigger and runs the action immediately.

## Troubleshooting: the missing web interface

After setting the static IP, the web interface stopped loading for hours. I worked through it layer by layer:

1. **Network layer:** `ping` replied, and `ip neigh` showed the MAC address matched the VM, so there was no IP conflict.
2. **Service layer:** `curl -v` to port 8123 returned **Connection refused**: the machine was reachable but nothing was listening on that port.
3. **Application:** `core info` in the HAOS console showed `port: 80`. Home Assistant was serving on port 80, not the usual 8123.
4. `curl -sI` on port 80 returned `405 Method Not Allowed`, which proved the web server was up. The interface loaded at the plain address with no port.

Lesson: **a working ping with "connection refused" means right machine, wrong port or service down.** The next step is to check which port the service actually uses, not to keep retrying the expected one.

## Gotchas and lessons

- **The netmask was entered twice.** Typing `/24` in the IP field while the netmask field also set it produced `/24/24`, which Home Assistant rejected. Enter the bare IP when there's a separate netmask field.
- **iOS Local Network permission.** The Companion app can't reach anything on the LAN until it's allowed under the phone's Local Network privacy setting.
- **iPhone auto-capitalisation** broke the login. Paste credentials from the password manager instead of typing them.
- **iOS only reveals the Wi-Fi name to apps with Precise Location set to Always**, which the app needs to know it's at home.
- **Multiple access points with different Wi-Fi names** looked like leaving home, so the app kept disconnecting. Every home network name has to be listed in the app, or the extender should use the same name as the router.
- **Saved public Wi-Fi networks are an evil-twin risk.** Phones auto-join any network with a saved name, so I removed old public networks.

## Security concepts

- **Defence in depth:** 2FA, Pi-hole DNS filtering, and no exposure to the internet.
- **Least privilege:** separate non-admin accounts, iOS app permissions, and a device locked to my account.
- **Confidentiality in transit:** avoiding plain HTTP on untrusted networks.
- **Supply-chain risk:** weighing third-party integrations before installing them.
- **Methodical troubleshooting:** testing one layer at a time (network, port, application).
