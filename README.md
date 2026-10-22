# Wammy Homelab

A self-hosted homelab built on Proxmox, used to learn networking, Linux, automation and security alongside a Cybersecurity & Digital Forensics degree.

> **Status:** Phase 1 in progress (moving Pi-hole into its own container and adding Unbound).

## What it is

| Machine | Role |
|---|---|
| **Wammy** (Dell OptiPlex 3080, i5, 2TB NVMe) | Proxmox host running the network services, everyday apps and security tools |
| **Main desktop** (RTX 3060 Ti) | Runs Ollama for local AI |
| **Arch laptop** | Privileged access workstation: the only machine allowed to administer Wammy |

<!-- Add your exported architecture diagram to diagrams/ and link it here -->
![Architecture diagram](diagrams/architecture.png)

## Roadmap

| Phase | Focus | Status | Write-up |
|---|---|---|---|
| 0 | Secure baseline: SSH keys, updates, Proxmox 2FA | 🟡 In progress | [Phase 0](docs/phases/phase-0-baseline.md) |
| 1 | Pi-hole + Unbound in their own LXC | 🟡 In progress | [Phase 1](docs/phases/phase-1-pihole.md) |
| 2 | Backups and a tested restore | ⚪ Planned | |
| 3 | Home Assistant | ⚪ Planned | |
| 4 | Platform: Docker, Caddy, Tailscale, Homepage, Uptime Kuma, Forgejo | ⚪ Planned | |
| 5 | Everyday services: Vaultwarden, Paperless-ngx, Mealie, Syncthing, Jellyfin, fitness | ⚪ Planned | |
| 6 | My own Python services: network watcher, budget app | ⚪ Planned | |
| 7 | AI layer: Ollama, voice, RAG "life admin brain" | ⚪ Planned | |
| 8 | Security and hardening: PAW, Wazuh, CrowdSec, Grafana, isolated attack lab | ⚪ Planned | |

Status key: ⚪ Planned · 🟡 In progress · 🟢 Done

## Repo layout

```
docs/
  phases/          one write-up per phase: what I did, what broke, what I learned
  decisions.md     design decisions and why I made them
  inventory.md     services, where they run, ports (sanitised)
  posting-checklist.md  what to check before anything goes public
services/
  _template/       copy this to start a new service
  <service>/       compose.yml, .env.example, README.md per service
diagrams/          architecture and network diagrams
```

## Security practices

- No services are exposed to the internet; remote access goes through Tailscale.
- Secrets live in `.env` files, which are git-ignored. Only `.env.example` files with placeholder values are committed.
- Every commit is scanned for secrets with [gitleaks](https://github.com/gitleaks/gitleaks) via a pre-commit hook.
- Server administration is restricted to a dedicated, locked-down workstation (PAW) using key-based SSH.
- Public IPs, domains, Wi-Fi names and device lists are never published.

## Licence

MIT. See [LICENSE](LICENSE).
