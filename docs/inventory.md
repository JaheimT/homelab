# Inventory (public version)

Services, where they run and which ports they use. This public copy uses placeholders instead of real IPs. Keep the full version with real addresses in the private build plan, not in this repo.

| Service | Runs on | Phase | Address | Ports | Notes |
|---|---|---|---|---|---|
| Proxmox | Wammy (bare metal) | 0 | `<wammy-ip>` | 22, 8006 | Admin only from the PAW |
| Pi-hole + Unbound | LXC | 1 | `<pihole-ip>` | 53, 80 | Pi-hole DHCP off; router hands it out as DNS |
| Home Assistant OS | VM | 3 | `<ha-ip>` | 8123 | |
| Docker host | VM | 4 | `<docker-ip>` | | Runs most services below |
| Caddy | Docker | 4 | | 80, 443 | Reverse proxy and HTTPS |
| Tailscale | Docker | 4 | | | No router ports opened |
| Homepage | Docker | 4 | | | |
| Uptime Kuma | Docker | 4 | | | |
| Forgejo | Docker | 4 | | | |
| Vaultwarden | LXC | 5 | | | Tailscale only |
| Paperless-ngx | Docker | 5 | | | |
| Mealie | Docker | 5 | | | |
| Syncthing | Docker | 5 | | | |
| Jellyfin + Audiobookshelf | Docker | 5 | | | |
| wger + Endurain | Docker | 5 | | | |
| Network watcher | Docker | 6 | | | |
| Budget app | Docker | 6 | | | |
| RAG brain | Docker | 7 | | | |
| Ollama | Main desktop | 7 | | 11434 | Firewalled to Home Assistant and the RAG brain |
| Wazuh | VM | 8 | | | |
| CrowdSec | Alongside Caddy | 8 | | | |
| Grafana | Docker | 8 | | | |
| Attack lab | Isolated VLAN | 8 | | | Cannot reach the rest of the network |
