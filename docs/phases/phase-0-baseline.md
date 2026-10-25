cd ~/homelab && cat > docs/phases/phase-0-baseline.md <<'EOF'
# Phase 0: Secure Baseline (SSH, 2FA, Exposure Check)

Oct 22, 2026 · @J

**Status:** ✅ Done

## Goal

Lock down how Wammy is accessed before any services run on it: key-only SSH, 2FA on the web UI, and nothing reachable from the internet.

## What I did

- Generated an **Ed25519 SSH key with a passphrase** on my PC. The passphrase lives in my password manager, and `keychain` caches it so I only enter it once per session.
- Added `~/.ssh/config` aliases for each host, so I connect by name rather than by address.
- Turned off every form of password login over SSH, using a drop-in config file.
- Enabled **2FA** on the Proxmox web UI and brought the host fully up to date.
- Checked what Wammy listens on and what's reachable from outside, and disabled the one service I didn't need.

## Decisions

| Decision | Why |
| --- | --- |
| Keys only, no passwords | Passwords can be brute-forced; a 256-bit key can't be guessed in practice. |
| Passphrase on the key | If the key file is ever stolen, it's useless without the passphrase. |
| Changes in a drop-in file (`sshd_config.d/`) instead of `sshd_config` | All my changes are in one place, package updates won't overwrite them, and rolling back is deleting one file. |
| Disable `KbdInteractiveAuthentication` as well | It's a separate login method that can still prompt for a password through PAM. Disabling only `PasswordAuthentication` leaves that route open. |
| `PermitRootLogin prohibit-password`, not `no` | Proxmox is built around root and its tools expect root SSH. Key-only root is the compromise. A non-root admin account is planned for Phase 8. |
| Disable rpcbind | It's only needed for NFS, which I don't use. One less listening service. |
| No port forwards | Remote access comes through Tailscale (Phase 4), which needs no open ports. |

## Config

`/etc/ssh/sshd_config.d/hardening.conf`:

```
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
```

Applied with a test first, so a typo can't take SSH down:

```
sshd -t && systemctl reload ssh
```

A reload keeps existing sessions open, so I kept my current session as a lifeline and tested key login from a new terminal before closing it.

## Exposure check

**Inside: what Wammy listens on (`ss -tlnp`)**

| Port | Service | Bound to | Verdict |
| --- | --- | --- | --- |
| 22 | sshd | All interfaces | OK, key-only |
| 8006 | Proxmox web UI | All interfaces | OK, 2FA on |
| 3128 | spiceproxy (VM console) | All interfaces | OK, normal for Proxmox |
| 85 | pvedaemon | Localhost | OK |
| 25 | Postfix | Localhost | OK |
| 111 | rpcbind | All interfaces | Disabled |

**Outside: what the internet can reach**

- Router: no port forwards, UPnP off, DMZ off.
- GRC ShieldsUP! common-ports scan: every port Stealth, including a custom probe of 8006 and 3128.

## Lessons

- **Disabling a service isn't always enough.** rpcbind is socket-activated: systemd holds the port open and starts the service on demand. I had to disable both `rpcbind.service` and `rpcbind.socket`.
- **`PasswordAuthentication no` alone doesn't stop password prompts.** Keyboard-interactive auth has to go too.
- **External scans have blind spots.** ShieldsUP! only tests IPv4, so the router settings are what cover IPv6.
- **Never change SSH without a test and a spare session.** `sshd -t` plus an open session means a mistake costs a minute, not a trip to the server with a monitor and keyboard.
EOF
git diff --stat
