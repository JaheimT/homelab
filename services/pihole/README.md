# Pi-hole + Unbound

**What it does:** blocks ads and trackers for the whole network, and resolves DNS privately with Unbound.
**Runs on:** its own Debian LXC on Proxmox (1 core, 512MB RAM)
**Phase:** 1
**Access:** local network only

## How devices use it

The router does DHCP and hands out this Pi-hole's IP as the DNS server. Pi-hole's own DHCP is off.

## Setup

```bash
# Inside the LXC
apt update && apt full-upgrade -y
curl -sSL https://install.pi-hole.net | bash     # read the script first if you prefer

apt install -y unbound
cp unbound-pi-hole.conf /etc/unbound/unbound.conf.d/pi-hole.conf
systemctl restart unbound

# Test Unbound (should return an IP address)
dig pi-hole.net @127.0.0.1 -p 5335
```

Then in Pi-hole: **Settings → DNS**, untick all upstream servers and add a custom upstream of `127.0.0.1#5335`.

## Migrating from the old Pi-hole

1. Old Pi-hole: **Settings → Teleporter → Export**.
2. New Pi-hole: **Settings → Teleporter → Import**.
3. Change the router's DHCP **DNS Server** field to the new Pi-hole's IP.

## Backups

Included in the Proxmox scheduled backup job. Teleporter exports are a handy extra copy of settings.

## Security notes

- Admin page only reachable on the local network.
- Unbound avoids sending every lookup to a third-party resolver.
- Running outside the Docker VM means Docker experiments can't take DNS down.
