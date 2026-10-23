# Phase 1: Pi-hole + Unbound on Wammy

Oct 23, 2026 · @J

## What I built

Pi-hole already worked in Docker on the Docker VM, but every Docker restart took DNS down for the whole house. This phase moved it into its own lightweight LXC and added Unbound, so lookups go straight to the authoritative servers instead of a third-party resolver.

A network-wide ad and tracker blocker with my own recursive DNS resolver, running in its own hardened Debian LXC on Proxmox.

```
Devices → Pi-hole (blocks ads/trackers) → Unbound (resolves from the root servers) → Internet
```

- **Pi-hole** filters DNS for every device on the network.
- **Unbound** is a validating recursive resolver. Instead of forwarding every lookup to Google or Cloudflare, it asks the root servers and authoritative servers directly, so no single third party sees all of my household's DNS queries.
- The old Docker Pi-hole stays running as **backup DNS**.

## IP plan

I split my `/24` home network into two zones: a **static zone** at the bottom of the range for servers and infrastructure, and the router's **DHCP pool** above it for everyday devices. Infrastructure never gets its address from DHCP.

| Device | Addressing |
| --- | --- |
| Router | Gateway |
| Wammy (Proxmox host) | Static IP, static zone |
| Pi-hole + Unbound (CT 101) | Static IP, static zone |
| Old Docker Pi-hole | Backup DNS |

I first gave the container an address inside the DHCP pool, which meant the router could have handed the same address to another device. I moved it into the static zone.

## Step 1: The container

Debian 13 (`debian-13-standard_13.6-1`) LXC, CT 101:

| Setting | Value | Why |
| --- | --- | --- |
| Unprivileged | Yes | Root inside the container is not root on the host |
| Cores / RAM / swap | 1 / 512 MB / 512 MB | Pi-hole is tiny; it uses about 10% of this |
| Disk | 8 GB on local-lvm |  |
| Network | Static IP in the static zone, router as gateway | A DNS server must never change address |
| Start at boot | Yes | Otherwise the whole house loses DNS after a reboot |

I chose a **static IP on the container** rather than a DHCP reservation, so Pi-hole doesn't depend on the router to get its own address.

## Step 2: Container hardening (Phase 0 habits)

- `apt update && apt full-upgrade -y`, because templates are already out of date when downloaded.
- My `wammys-house` SSH key copied in with `pct push`, plus a `pihole` entry in my `~/.ssh/config`.
- Key-only SSH via `/etc/ssh/sshd_config.d/hardening.conf`:

```
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
```

Verified with `sshd -T`: `passwordauthentication no`, and a password-only login attempt is refused with `Permission denied (publickey)`.

## Step 3: Unbound

`/etc/unbound/unbound.conf.d/pi-hole.conf`:

```
server:
    verbosity: 0
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-ip6: no
    prefer-ip6: no
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no
    edns-buffer-size: 1232
    prefetch: yes
    num-threads: 1
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
```

- **`interface: 127.0.0.1` + `port: 5335`**: only reachable from inside the container, so only Pi-hole can use it.
- **`harden-dnssec-stripped`**: rejects answers that should be DNSSEC-signed but aren't, protecting against tampered responses.
- **`private-address`**: public domains can't resolve to private IPs, which defends against **DNS rebinding** attacks on devices on my LAN.
- **`edns-buffer-size: 1232`**: avoids fragmented DNS packets, a common cause of flaky lookups.

I also disabled `unbound-resolvconf.service`, which can rewrite the container's `/etc/resolv.conf` to point at a port Unbound isn't listening on.

## Step 4: Testing Unbound

| Test | Result | Meaning |
| --- | --- | --- |
| `dig pi-hole.net @127.0.0.1 -p 5335` | `NOERROR` | Unbound resolves on its own |
| `dig fail01.dnssec.works @127.0.0.1 -p 5335` | `SERVFAIL` | A deliberately broken signature is rejected |
| `dig dnssec.works @127.0.0.1 -p 5335` | `NOERROR` + `ad` flag | The DNSSEC signature was verified |

## Step 5: Pi-hole

- Downloaded the installer and read it before running it, instead of piping `curl` straight into `bash`. Running unseen code from the internet is a supply-chain risk.
- Set the only upstream to Unbound: `pihole-FTL --config dns.upstreams '["127.0.0.1#5335"]'`
- Imported blocklists, groups and clients from the old Pi-hole with **Teleporter** (71,493 domains).
- Tested from my PC: `nslookup pi-hole.net <pihole-ip>` returns an address; `doubleclick.net` returns `0.0.0.0`.

## Step 6: Cutover

- Router DHCP settings → **DNS Server** set to `<pihole-ip>,<old-pihole-ip>` (new Pi-hole first, old one as backup).
- I did not use a public resolver like `8.8.8.8` as the backup. Devices don't only use the secondary when the first fails, so ads and trackers would get through some of the time.
- IPv6 is off on the router, so devices can't receive a router IPv6 DNS server that bypasses Pi-hole.
- Devices moved over as their DHCP leases renewed overnight.

## Results (first night)

- **21,965 queries** from **15 active clients**
- **18.9% blocked** (4,147 queries)
- Container load 0.07, memory 9.9%
- Example block: `browser-intake-datadoghq.com`, browser telemetry sent to Datadog
- Cached answers return in about 3 µs, compared with about 57 ms for a fresh lookup through Unbound

## Gotchas and lessons

- **Teleporter overwrites settings, not just lists.** The import replaced my upstream with Cloudflare and Google (bypassing Unbound) and replaced the admin password. After any import, re-check upstream DNS, DHCP (must stay off) and the password.
- **Debian 13 LXCs use socket-activated SSH.** `systemctl reload ssh` fails because no service is running; `ssh.socket` starts a fresh `sshd` for each connection, so new config applies automatically. `sshd -t` needs `/run/sshd`, so create it first.
- **SSH only tries default key names.** A key named `wammys-house` has to be set per host in `~/.ssh/config` or passed with `-i`.
- **Manual network settings ignore the router.** My PC had a static IP inside the DHCP pool and hardcoded DNS, so the router change never reached it. I switched it to automatic DHCP.
- **DNS over HTTPS bypasses Pi-hole.** Windows' and browsers' DoH settings send DNS straight to a provider, so they stay off on my devices.
- **Pi-hole's NTP client fails in an unprivileged LXC** (`Insufficient permissions`). Containers share the host's clock and aren't allowed to change it. I disabled it with `pihole-FTL --config ntp.sync.active false`; the host keeps time.
- **Always check the prompt** (`lawliet@P1`, `root@Wammy`, `root@pihole`) before running a command. I ran several commands in the wrong terminal.

## Security concepts

- **Defence in depth**: DNS filtering blocks trackers and malicious domains for every device, including ones I can't install software on.
- **Privacy**: recursive resolution means no single upstream provider sees all of my DNS traffic.
- **Integrity**: DNSSEC validation rejects forged DNS answers.
- **Least privilege and isolation**: unprivileged container, key-only SSH, and Unbound bound to localhost.
- **Availability**: static IP, start at boot, and a backup DNS server.
