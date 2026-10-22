# Decisions log

Every significant design choice, with the reason. Newest at the bottom.

| Date | Decision | Why |
|---|---|---|
| 2026-10-22 | Secure baseline first, build, then harden | Avoids lockouts while experimenting |
| 2026-10-22 | Arch laptop becomes the PAW; attack tools run as isolated VMs | Keeps admin keys away from untrusted tools |
| 2026-10-22 | Move Pi-hole out of the Docker VM into its own LXC; the router keeps doing DHCP | DNS shouldn't go down every time the Docker VM restarts |
| 2026-10-22 | Vaultwarden runs in its own LXC, reachable only through Tailscale | Most sensitive service, so it's isolated from the rest |
| 2026-10-22 | wger for the gym, Endurain for runs | One tool for each side of hybrid training |
| 2026-10-22 | Build the RAG "life admin brain" myself, fed by Paperless-ngx | Better learning and portfolio value than an off-the-shelf tool |
| 2026-10-22 | Upgrade Wammy to 32GB (used DDR4-2666) before Phase 8 | Wazuh is the heaviest service; the OptiPlex runs RAM at ~2666MHz anyway |
| 2026-10-22 | Defer Immich and Navidrome | Waiting for more storage |
