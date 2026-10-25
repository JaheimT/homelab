# Phase 4 (Part 1): Remote Access with Tailscale

Oct 23, 2026 · @J

**Status:** ✅ Done

## What I built

Private remote access to my homelab from anywhere, with **no ports open** on my router.

- A **Tailscale subnet router** in its own unprivileged Debian 13 LXC, giving my tailnet access to my home network
- The same container as an **exit node**, so my phone's traffic runs through home when I'm out
- Phone set to connect automatically, with the Home Assistant app working on mobile data
- Hardened like every other host: key-only SSH, updates, automatic security patches

```
Phone (anywhere) ── WireGuard tunnel ──▶ Tailscale LXC ──▶ Home Assistant, Pi-hole, Proxmox
                                             └──▶ internet (via Pi-hole DNS)
```

## Why Tailscale

Tailscale builds an encrypted **WireGuard** mesh between my devices (a tailnet). Devices connect **outwards** to Tailscale's coordination servers, which introduce them to each other, so nothing at home accepts incoming connections. My external exposure check stays all-stealth.

I chose it over hosting a plain WireGuard server because that would need an open port on my router and manual key management on every device.

A **subnet router** means I don't need Tailscale on every service: one container forwards tailnet traffic into my home network.

## The container

```
pct create $ID local:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst \
  --hostname tailscale \
  --unprivileged 1 --features nesting=1 \
  --cores 1 --memory 512 --swap 512 \
  --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr0,ip=$IP/24,gw=<router-ip> \
  --nameserver <pihole-ip> \
  --dev0 /dev/net/tun \
  --ssh-public-keys /root/.ssh/authorized_keys \
  --onboot 1 --start 1
```

| Option | Why |
| --- | --- |
| `--dev0 /dev/net/tun` | Gives only this container the TUN device a VPN needs. Unprivileged containers don't get it by default |
| `--nameserver <pihole-ip>` | The container's own DNS goes through Pi-hole |
| `$IP` variable | The IP is set once and reused, which avoids editing placeholders inside a long command |

Tailscale was installed from its **signed APT repository**, not a `curl | bash` script: apt checks every package against Tailscale's signing key.

## Subnet router and exit node

```
printf 'net.ipv4.ip_forward = 1\nnet.ipv6.conf.all.forwarding = 1\n' > /etc/sysctl.d/99-tailscale.conf
sysctl -p /etc/sysctl.d/99-tailscale.conf
tailscale up --advertise-routes=<home-subnet>/24 --accept-dns=false
tailscale set --advertise-exit-node
```

- **IP forwarding** enabled so the container can pass traffic on, like a router.
- Advertised routes and exit nodes must be **approved in the admin console** before they work, so a compromised device can't quietly expose a network.
- **Key expiry disabled** for the always-on server, so remote access doesn't break after 180 days. It stays on for the phone, which can be lost.

## Hardening all hosts

Before building, I brought the Docker VM up to the same standard as Wammy and Pi-hole:

- Key-only SSH, with `PermitRootLogin no` (admin is a normal user plus `sudo`)
- The drop-in file is named `00-hardening.conf` so it's read **first**. Ubuntu can ship a `50-cloud-init.conf` with `PasswordAuthentication yes`, and sshd uses the first value it reads.
- `unattended-upgrades` on the Tailscale container, since all my phone's traffic now passes through it

## Phone setup

- **Exit node:** the Tailscale container, so all traffic goes through home
- **Allow Local Network Access:** on, so at home the phone reaches local devices directly
- **VPN On Demand:** Always on Wi-Fi and mobile data, so it never relies on me remembering
- **Home Assistant app:** External URL set to Home Assistant's local address, reached through the tunnel

## Troubleshooting: no access on EE mobile data

Everything worked on Wi-Fi but nothing loaded on EE mobile data. I worked through it one layer at a time.

| Test | Result | What it ruled out |
| --- | --- | --- |
| `tailscale status`, admin console | All devices connected, route approved | Registration and route approval |
| `sysctl net.ipv4.ip_forward` | `1` | Forwarding on the router |
| `curl` from the container to Home Assistant | `405` response | The container's own LAN access |
| `tailscale ping` to the phone | Pongs via the London relay | Basic connectivity |
| Docker VM → container over the tailnet | Direct, 0% loss | The tailnet itself |
| Access controls | Allow-all policy | ACLs |
| `tailscale ping --icmp` to the phone | Works | The WireGuard data path towards the phone |
| `tcpdump` on `tailscale0` while the phone loads a page | **0 packets** | Showed the phone's apps never sent traffic into the tunnel |
| Safari and Brave, 4G and 5G, Private Relay off, full re-registration | All failed | Browser, radio mode and stale state |
| NetBird (a second WireGuard VPN) with the same setup | Failed the same way | Showed it wasn't specific to Tailscale |
| Home IPv6 to allow a direct path | ISP gives no IPv6 (0/10 on test-ipv6.com) | Not an option on this line |

**Conclusion:** EE's mobile network is IPv6-only. On this iPhone, **split-tunnel routes** ("send only home traffic into the VPN") never received app traffic on EE, while the tunnel itself worked. Two independent VPN clients failing identically pointed to the phone and carrier, not my homelab.

**Fix:** use the container as an **exit node** (a full tunnel). iOS sends all traffic into the VPN, and it works on mobile data. I've reported the split-tunnel behaviour to Tailscale.

## Security trade-offs

**Gains**

- No open ports; remote access is outbound-only
- My carrier can't see my browsing; everything is encrypted to home
- Protection on public Wi-Fi against snooping and evil-twin hotspots
- Ad and tracker blocking and private DNS (Pi-hole + Unbound) everywhere

**Costs**

- Home becomes a single point of failure for my phone's internet while the exit node is on
- Speed is limited by my home upload
- The Tailscale container becomes high-value: it carries all my phone's traffic, so it's hardened and auto-patched
- The Home Assistant app used "Less secure" connection mode with a plain HTTP URL. Accepted because the VPN is always on, so the traffic stays inside WireGuard. Part 2 adds HTTPS through Nginx, which removes the need for it.
- Access controls are currently allow-all; tightening them to least privilege is planned for Phase 8

## Gotchas and lessons

- **Placeholders in commands:** `<x>` in a command is read by bash as input redirection. I use variables instead.
- **Check the prompt before every command.** Several steps landed on the wrong machine (an SSH key copied to the VM itself, SSH config written on Wammy). File timestamps (`ls -la`) showed where commands had actually run.
- **Test from a new terminal before closing the old one.** Locking the Docker VM to key-only before its key was installed would have locked me out; the open session let me fix it.
- **One-off setup keys** (NetBird) become invalid after first use, so a leaked key that's already been used is harmless.
- **A ping isn't proof of a data path.** Tailscale's own ping uses control messages; `--icmp` and `tcpdump` showed what real traffic was doing.
- **Change one variable at a time.** Testing with Wi-Fi on gave a false positive, because the phone reached Home Assistant directly.
- **Test a reboot.** After restarting Wammy, my phone's exit node was deselected and internal names stopped resolving. I re-selected it and added **split DNS** in Tailscale, so my domain always resolves through Pi-hole.
- Sometimes it's good to step away and come back with fresh eyes.
