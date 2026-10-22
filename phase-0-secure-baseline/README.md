# Phase 0: Hardening SSH on Wammy

Oct 22, 2026 · @J

## The big picture

There are two pieces of SSH software:

- **`ssh`** is the **client**. It runs on your Linux machine and makes the connection.
- **`sshd`** is the **server daemon**. It runs on Wammy and accepts connections. The "d" stands for daemon, a background service.

Everything before this step, like making the key and the `~/.ssh/config` shortcut, was client-side. This step changes **`sshd`'s rules**, which decide who Wammy lets in and how. Those rules live in `/etc/ssh/sshd_config`.

## Why use a separate file (`sshd_config.d/`)

Modern Linux supports **drop-in config folders**. The main config has an `Include` line that pulls in any extra `.conf` files from that folder. Putting your changes in their own file (`hardening.conf`):

- keeps them **in one place**, so you or a marker can see exactly what you changed
- means **system updates won't overwrite them**, because package updates sometimes replace the main `sshd_config`
- makes them **easy to undo**: delete the file and reload

If the folder doesn't exist, check for the `Include` line with:

```
hostname && grep -i include /etc/ssh/sshd_config
```

- **If the `Include` line is there**, sshd already reads the folder, so just create it with `mkdir -p /etc/ssh/sshd_config.d`.
- **If it isn't there**, sshd would ignore the folder completely, so edit the main `/etc/ssh/sshd_config` instead.
- The `hostname` part makes sure you're actually on Wammy and not your own machine.

## The three settings

```
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
```

**`PasswordAuthentication no`** This turns off password login. Otherwise anyone on your network can try guessing the root password. Bots on the internet do this constantly, which is called **brute forcing**. With this set to `no`, Wammy only accepts keys, and guessing a 256-bit key isn't practical.

**`KbdInteractiveAuthentication no`** This one is a common gotcha. "Keyboard-interactive" is a separate login method that can **also ask for a password**, through a different system called PAM. If you only disable `PasswordAuthentication`, some systems will still show a password prompt through this route. Disabling both closes the gap. On older versions this setting is called `ChallengeResponseAuthentication`.

**`PermitRootLogin prohibit-password`** Logging in as root is risky because root controls everything. There are three options:

- `yes` lets root log in with a password or a key
- `no` blocks root completely
- `prohibit-password` lets root log in **with a key only**

You'd normally block root and log in as a regular user with `sudo`. Proxmox, though, is built around root, and its tools expect root SSH. `prohibit-password` is the sensible middle ground. Logging in as a non-root user could be a later Phase 8 improvement.

## The safety steps

```
sshd -t && systemctl reload ssh
```

**`sshd -t`** (test mode) This checks the config for mistakes **without applying it**. If there's a typo, it tells you, and SSH keeps running on the old working config. Skipping this step is how people break SSH on a server they can't physically reach.

**`&&`** This runs the second command only if the first one succeeds. If the test fails, the reload never happens.

**`systemctl reload ssh`** This tells sshd to re-read its config. A **reload** applies the new rules to *new* connections without dropping *existing* ones. A **restart** fully stops and starts the service.

**Testing in a new terminal while the old one stays open** Because existing connections survive a reload, your current session is a **lifeline**. If key login fails in the new terminal, you can still use the old one to fix the mistake. If you closed it first and the config was wrong, you'd be locked out and need a monitor and keyboard on the OptiPlex.

## External exposure check

Nothing on Wammy is reachable from the internet. I checked from the inside (what Wammy listens on) and the outside (what the router lets through).

**Inside view: `ss -tlnp` on Wammy**

| Port | Process | Bound to | Verdict |
| --- | --- | --- | --- |
| 22 | sshd | All interfaces | OK, key-only login |
| 8006 | pveproxy (web UI) | All interfaces | OK, 2FA on |
| 3128 | spiceproxy (VM console) | All interfaces | OK, normal for Proxmox |
| 85 | pvedaemon | Localhost only | OK |
| 25 | Postfix mail relay | Localhost only | OK |
| 111 | rpcbind | All interfaces | Disabled, not needed without NFS |

rpcbind was disabled with `systemctl disable --now rpcbind.service rpcbind.socket`. Both units are needed because systemd holds the socket open and would restart the service on demand.

**Outside view: router and scan**

- Router (`192.168.1.254`): no port forwards, UPnP off, DMZ off.
- GRC ShieldsUP! common ports scan: every port Stealth.
- ShieldsUP! custom probe of 8006 and 3128: Stealth.
- ShieldsUP! only tests IPv4, so the router settings cover the IPv6 side.

Remote access will come later through Tailscale (Phase 4), which needs no open ports.

## Security concepts to note

- **Attack surface reduction**: removing ways in that you don't need.
- **Least privilege**: root only with strong authentication.

Both are good things to name in the Phase 0 GitHub write-up.
