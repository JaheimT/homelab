# Phase 3: Home Assistant on Wammy

Oct 23, 2026 · @J

**Status:** ✅ Done

## What I built

Home Assistant OS (HAOS) running as a VM on Proxmox: a self-hosted smart home hub that connects devices, runs automations and keeps control on my own network instead of each brand's cloud.

- HAOS VM with a static IP in the static zone, using Pi-hole for DNS
- 2FA, updates and a first backup before adding anything
- Companion app on my phone in "Most secure" connection mode
- Network printer integrated, with a low-ink notification automation

## Why a VM, not an LXC

HAOS is a complete appliance operating system. It manages its own add-ons, updates and container runtime, so it expects a whole machine to itself. A VM is the officially supported way to run it on Proxmox and keeps it isolated from my other services.

## The VM

I downloaded the current stable `haos_ova` image and built the VM from the command line:

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
- **Backup before changes:** a local backup before adding any devices. Off-machine backups were added in Phase 2.
- **Companion app in "Most secure" mode:** Home Assistant used plain `http://` on my LAN, so the app only connects when it can confirm I'm on my home Wi-Fi. It won't send my session over unencrypted HTTP on a network that just looks similar. Remote access was added in Phase 4 through
