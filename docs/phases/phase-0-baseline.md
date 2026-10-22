# Phase 0: Secure baseline

**Status:** 🟡 In progress
**Dates:** 2026-10 →

## Goal

Get the basics right before building anything: keep Proxmox updated, replace password logins with SSH keys, and make sure nothing is exposed to the internet.

## Checklist

- [ ] Update Proxmox: `apt update && apt full-upgrade`
- [ ] Create an SSH key on the desktop: `ssh-keygen -t ed25519` (with a passphrase)
- [ ] Copy the public key to Wammy and confirm key login works
- [ ] Set `PasswordAuthentication no` and `PermitRootLogin prohibit-password` in `/etc/ssh/sshd_config`, then `systemctl restart ssh`
- [ ] Test login from a new terminal while keeping the old session open
- [ ] Turn on two-factor authentication for the Proxmox web UI
- [ ] Check the BIOS "power on after power loss" setting
- [ ] Confirm nothing on Wammy is reachable from the internet

## What broke and how I fixed it

| Problem | How I found it | Fix |
|---|---|---|
| | | |

## What I learned

-

## Next

Phase 1: Pi-hole + Unbound.
