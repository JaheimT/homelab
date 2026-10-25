# Phase 2: Backups (3-2-1, Encrypted Offsite, Tested Restores)

Oct 25, 2026 · @J

**Status:** ✅ Done (follow-ups listed at the end)

## What I built

A 3-2-1 backup setup for the whole homelab, with every copy restore-tested and monitored. Three copies, on different machines, with one encrypted offsite.

| Copy | Where | What | Schedule | Tested |
| --- | --- | --- | --- | --- |
| 1 | Proxmox host, local disk | Full backups of all VMs and containers (\~8 GB per run) | Nightly 03:00, keep 3 | ✅ Restored a container to a test ID |
| 2 | My main PC, separate drive | Copies of the local backups, pulled over SSH | Weekly | — |
| 3 | Dropbox, encrypted with restic | Nightly: host + container configs (\~4 MB). Weekly: full Docker and Home Assistant VMs | Nightly 04:00, weekly Sunday 05:00 | ✅ Restored and compared byte-for-byte |

Media is deliberately **not** backed up offsite. It's replaceable, and terabytes wouldn't fit through a home upload. Backups protect what can't be recreated.

## Before backing up: updates and credential rotation

I updated and secured everything first, so the first backup captured a clean state:

- Took temporary snapshots of every VM and container as an undo point, then updated all guests, the Docker containers (`docker compose pull && up -d`), Home Assistant and the Proxmox host
- Rotated every credential. Each now has its own unique password-manager item: passphrases for anything I might type at a console, long random strings for anything I paste
- Rotated the Home Assistant backup encryption key and deleted backups made with the old key
- Set password-manager URL matching to **Host**, because the default (base domain) treated every `*.<domain>` service as one site

## Copy 1: Proxmox nightly backups

- **Snapshot mode** (no downtime), **ZSTD** compression, **keep last 3**
- Each run is \~8 GB. Only used space is backed up, and it compresses well: the containers are 200–300 MB each, the two VMs \~3.5 GB each
- Proxmox backups are full copies every run, so retention was sized to fit the free space on the host's system disk
- Gotcha: the `local` storage wasn't allowed to hold backups until I added **Backup** to its content types

**Restore test:** restored the Nginx container's backup to a new ID with its network link **down** (an exact clone would clash with the live container's IP), checked its sites, certificates and config (`nginx -t`), then destroyed it.

**Snapshots aren't backups:** I deleted old VM snapshots (some with saved RAM state). They live on the same disk as the VM, so they die with it, and they grow over time.

## Copy 2: pulled to my PC

```
rsync -av wammy:/var/lib/vz/dump/ /mnt/<drive>/wammy-backups/
find /mnt/<drive>/wammy-backups -name "vzdump-*" -mtime +28 -delete
```

- **The PC pulls from the server, not the other way round.** The server has no credentials for the PC, so a compromised server or ransomware can't reach or delete these copies.
- No `--delete` on rsync, so the PC keeps more history than the server's three.
- Stored on a dedicated, otherwise unused drive.

## Copy 3: encrypted offsite to Dropbox

**Least-privilege access:** a dedicated Dropbox app with **App folder** access, so the server can only see one folder, never the rest of my Dropbox. Permissions limited to file read/write. The Dropbox account itself was hardened first (2FA, unique password, reviewed linked devices).

**restic over rclone:** restic encrypts on the server before upload (Dropbox only stores ciphertext), deduplicates, and keeps snapshots. The repository password is in my password manager and on a paper copy, because without it the backups can never be opened.

**Nightly config backup** (\~4 MB): Proxmox's `/etc/pve` (every VM and container definition, jobs, users), network and SSH config, plus Pi-hole, Unbound, Tailscale and Nginx configs and certificates, collected from each container with `pct exec ... tar`. The script uses `set -euo pipefail`, so any failure stops it before it reports success. Retention: 7 daily, 4 weekly, 6 monthly.

**Weekly full VM backup:** each VM is streamed straight into restic, without temporary files:

```
vzdump <id> --mode snapshot --compress 0 --stdout | restic backup --stdin --stdin-filename vzdump-qemu-<id>.vma
```

Uncompressed on purpose: restic compresses the data itself, and uncompressed input deduplicates far better from week to week, so after the first upload only changes are sent.

**Restore test:** restored the latest snapshot from Dropbox, compared the network config byte-for-byte with `diff`, and checked the Nginx archive contained every site.

**Scheduling:** systemd timers with `Persistent=true`, so a backup missed because the server was off runs as soon as it's back.

## Monitoring: alerts when a backup doesn't run

Uptime Kuma **Push monitors** work as a dead man's switch: each backup checks in when it finishes successfully, and if Kuma doesn't hear from it within 25 hours (8 days for the weekly one), my phone gets an alert. This catches every failure mode: a crash, a script error, an expired token, or the server being off.

- The offsite script checks in on its last line, which `set -e` only reaches after success
- The Proxmox backup checks in from a **vzdump hook script** at `job-end`, filtered to the real nightly job so the weekly offsite stream can't trigger it
- Check-ins use Kuma's direct address, so they don't depend on DNS or the reverse proxy

## Gotchas and lessons

- **Repository locks:** the nightly backup's prune step failed while the weekly VM upload held the restic lock. Fixed with `--retry-lock 2h`. The "use unlock" hint was wrong here: the lock belonged to a running job, not a stale one.
- **OAuth setup:** Dropbox rejected the login until rclone's redirect URI was registered on the app, and the first token lacked file permissions because they weren't submitted before authorising. A token only gets the scopes the app had when you clicked Allow.
- **A written password is only good once tested.** I changed my password-manager master password to a passphrase but copied one word wrong onto paper and locked myself out. I recovered it by checking each word against the generator's public word list. Now: type it from the paper when setting it, and test in a private window before anything logs out.
- **Never create plaintext exports.** An accidental unencrypted vault export had to be cleaned up. On an SSD with TRIM, deleted data is wiped by the drive; on an HDD it stays recoverable until overwritten, which is where forensic recovery comes in.
- **Test failure on purpose.** A deliberate reboot showed my phone's VPN exit node can drop and that internal names relied on luck. Fixed with **split DNS** in Tailscale, so `*.<domain>` always resolves through Pi-hole.
- **Timezones:** some tools report UTC while the system shows local time. Mixing them up would break a timeline.
- **Enabled isn't running.** The weekly timer was enabled but never started, so `list-timers` didn't show it and it would never have fired. `systemctl enable --now` starts it too; always confirm with `systemctl list-timers`.
- **No `$HOME` under systemd.** Services run without `$HOME`, so restic ran without its cache (slower, more requests to Dropbox). Fixed with `export HOME=/root` at the top of both scripts.

## Still to do

- [x] Confirm the first weekly VM upload finished (two snapshots in the repository)
- [x] Add a weekly Push monitor (8-day interval) and `--retry-lock` to the VM script
- [x] Confirm the local and offsite Push monitors turn green after the nightly runs
- [ ] Move local backups to a dedicated drive (or NAS) when the new hard drive arrives
- [ ] Automate the weekly pull to the PC
