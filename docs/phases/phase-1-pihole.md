# Phase 1: Pi-hole + Unbound on Wammy

Oct 23, 2026 · @J

**Status:** ✅ Done

## Goal

Pi-hole already ran in Docker, but every Docker restart took DNS down for the whole house. I moved it into its own LXC and added Unbound, so DNS stays up independently of anything else and my lookups don't all go to one third-party resolver.

```
Devices → Pi-hole (blocks ads/trackers) → Unbound (resolves from the root servers) → Internet
```

The old Docker Pi-hole stays running as backup DNS.

## Decisions

| Decision | Why |
| --- | --- |
| Its own LXC, not Docker | DNS shouldn't go down when I restart unrelated containers. An LXC is lighter than a VM and starts in seconds. |
| Unprivileged container | Root inside the container isn't root on the host. |
| 1 core, 512 MB RAM | Pi-hole uses about 10% of it. |
| Static IP on the container, not a DHCP reservation | A DNS server must never change address or depend on the router to get one. |
| Static zone below the DHCP pool | I first put it inside the DHCP range, where the router could hand the same address to another device. Infrastructure now always sits below the pool. |
| Start at boot | Otherwise the whole house loses DNS after a reboot. |
| Unbound as the only upstream | Recursive resolution means no single provider sees all of my household's DNS, and DNSSEC is validated locally. |
| Old Pi-hole as backup, not `8.8.8.8` | Devices don't only use the secondary when the first fails. A public backup would let ads and trackers through some of the time. |
| Read the installer before running it | Piping `curl` straight into `bash` runs unseen code from the internet, a supply-chain risk. |
| IPv6 off on the router | Otherwise devices can receive an IPv6 DNS server that bypasses Pi-hole. |

## Config

Container hardening matches Phase 0: fully updated, key-only SSH via `sshd_config.d/hardening.conf`, verified with `sshd -T` and a refused password-only login.

Unbound (`/etc/unbound/unbound.conf.d/pi-hole.conf`):

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

- **Localhost on port 5335**, so only Pi-hole can use it.
- **`harden-dnssec-stripped`** rejects answers that should be signed but aren't.
- **`private-address`** stops public domains resolving to LAN addresses, which blocks DNS rebinding attacks.
- **`edns-buffer-size: 1232`** avoids fragmented DNS packets, a common cause of flaky lookups.
- Disabled `unbound-resolvconf.service`, which can point the container's own DNS at the wrong place.

Pi-hole's only upstream is `127.0.0.1#5335`. Blocklists, groups and clients were imported from the old Pi-hole with Teleporter (71,493 domains). The router's DHCP now hands out the new Pi-hole first and the old one second.

## Verification

| Test | Result | Meaning |
| --- | --- | --- |
| `dig pi-hole.net @127.0.0.1 -p 5335` | `NOERROR` | Unbound resolves on its own |
| `dig fail01.dnssec.works @127.0.0.1 -p 5335` | `SERVFAIL` | A deliberately broken signature is rejected |
| `dig dnssec.works @127.0.0.1 -p 5335` | `NOERROR` + `ad` flag | DNSSEC validated |
| `nslookup doubleclick.net <pihole-ip>` from my PC | `0.0.0.0` | Blocking works for clients |

## Results (first night)

- **21,965 queries** from **15 clients**, **18.9% blocked**
- Container load 0.07, memory 9.9%
- Cached answers in about 3 µs, compared with about 57 ms for a fresh recursive lookup
- Example block: `browser-intake-datadoghq.com`, browser telemetry sent to Datadog

## Lessons

- **Teleporter overwrites settings, not just lists.** The import replaced my upstream with Cloudflare and Google, bypassing Unbound, and reset the admin password. After any import I now re-check upstreams, DHCP (must stay off) and the password.
- **Debian 13 LXCs use socket-activated SSH.** `systemctl reload ssh` fails because no service is running; `ssh.socket` starts a fresh `sshd` per connection, so config changes apply automatically. `sshd -t` needs `/run/sshd` to exist first.
- **SSH only tries default key names.** A custom-named key has to be set per host in `~/.ssh/config`.
- **Manual network settings ignore the router.** My PC had a hardcoded IP and DNS, so the DHCP change never reached it until I switched it to automatic.
- **DNS over HTTPS bypasses Pi-hole.** Windows and browser DoH settings stay off on my devices.
- **Pi-hole's NTP client fails in an unprivileged LXC.** Containers share the host clock and can't change it, so I disabled it (`ntp.sync.active false`) and let the host keep time.
- **Check the prompt before every command.** I ran several in the wrong terminal before making it a habit.

## Still to do

- [ ] Point the backup Pi-hole's upstream at Unbound instead of Cloudflare/Google
