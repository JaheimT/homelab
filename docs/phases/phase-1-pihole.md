# Phase 1: Pi-hole + Unbound

**Status:** 🟡 In progress
**Dates:** 2026-10 →

## Goal

Network-wide ad blocking and private DNS. Pi-hole already worked in Docker on the Docker VM, but that meant every Docker restart took DNS down for the whole house. This phase moves it into its own lightweight LXC container and adds Unbound, so DNS lookups go straight to the authoritative servers instead of through a third-party resolver.

## Starting point

- The router does DHCP and hands out the Docker VM's Pi-hole as the DNS server.
- Pi-hole's own DHCP is off and stays off.

## Checklist

- [ ] Snapshot the Docker VM in Proxmox before starting
- [ ] Create a Debian LXC (1 core, 512MB RAM) with a static IP outside the router's DHCP range
- [ ] Install Pi-hole and set a strong admin password
- [ ] Export settings from the old Pi-hole with Teleporter and import them into the new one
- [ ] Install Unbound and set it as Pi-hole's upstream (`127.0.0.1#5335`), using `services/pihole/unbound-pi-hole.conf`
- [ ] Test on one device by setting its DNS manually to the new Pi-hole
- [ ] Change the router's DHCP **DNS Server** field to the new Pi-hole
- [ ] Reconnect devices and watch them appear in the new Pi-hole's client list
- [ ] Keep the old Pi-hole for a few days, then remove it or keep it as a backup

## Rollback

Set the router's DNS Server field back to the old Pi-hole. Everything returns to how it was.

## What broke and how I fixed it

| Problem | How I found it | Fix |
|---|---|---|
| | | |

## Security notes

- The Pi-hole admin page is only reachable on the local network.
- Unbound removes the need to trust a third-party DNS provider with every lookup.
- DNS now lives outside the Docker VM, so experimenting there can't take the network down.

## What I learned

-

## Next

Phase 2: backups, before building anything else.
