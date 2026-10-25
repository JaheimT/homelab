# Phase 4 (Part 2): Domain, HTTPS Reverse Proxy and Monitoring

Oct 24, 2026 · @J

**Status:** 🟢 Done

## What I built

Every service now has a proper name and trusted HTTPS, and is monitored with alerts to my phone. None of it is exposed to the internet.

- A domain on Cloudflare, used **only** to prove ownership for certificates. No internal records are published.
- A **wildcard Let's Encrypt certificate** that renews itself
- **Nginx** in its own hardened LXC as the single front door for every service
- **Pi-hole local DNS** so the names only resolve inside my network (and over my Tailscale exit node)
- **Uptime Kuma** checking nine things every minute, with push alerts to my phone

```
Browser / phone ── DNS (Pi-hole) ──▶ *.<domain> = Nginx
               ── HTTPS ──▶ Nginx (wildcard cert) ──▶ Home Assistant / Proxmox / Pi-hole / Uptime Kuma
```

## Choosing the reverse proxy

|  | Caddy | Nginx | Nginx Proxy Manager |
| --- | --- | --- | --- |
| HTTPS | Automatic | Manual (Certbot) | Automatic via UI |
| Config | Short text file | Detailed text files | Web forms, stored in a database |
| Industry use | Growing | Everywhere | Mostly homelabs |
| Learning value | Medium | High | Low |

I chose **plain Nginx** because it's what I'll meet in real infrastructure, and writing the TLS and proxy config myself teaches how it works. To keep it low-maintenance, everything recurring is automated: certificate renewal, security updates and monitoring.

## Domain and DNS design

- Bought a short domain through **Cloudflare Registrar** (at-cost pricing, WHOIS privacy, auto-renew).
- Kept it **separate from my clothing brand's domain**, so a lab mistake can't affect the brand.
- Cloudflare account secured with **2FA**, plus a separate password so there are two ways in, both behind 2FA. Signing in only with Google would make my Google account a single point of failure.
- **No internal IPs in public DNS.** Both Pi-holes answer every `*.<domain>` name with Nginx's address using one dnsmasq rule:

```
pihole-FTL --config misc.dnsmasq_lines '["address=/<domain>/<nginx-ip>"]'
```

This setting can't be changed through Pi-hole's web UI by design, because raw dnsmasq lines are powerful. On the Docker-based backup Pi-hole it's set with `docker exec`.

## Wildcard certificate

Certbot proves ownership with the **DNS-01 challenge**: it temporarily adds a TXT record through Cloudflare's API, so nothing needs to be reachable from the internet.

- **Least-privilege API token:** DNS edit permission on this one domain only, 1-year expiry
- Stored in `/root/.secrets/cloudflare.ini` with `chmod 600`
- One certificate covers the domain and every subdomain, so new services need no new certificate, and individual service names don't appear in public certificate transparency logs

```
certbot certonly --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 30 \
  -d <domain> -d '*.<domain>'
```

**Renewal is automatic:** Certbot's systemd timer renews, and a deploy hook reloads Nginx. Tested with `certbot renew --dry-run`.

## Nginx setup

Nginx runs in an unprivileged Debian 13 LXC, hardened like every other host (key-only SSH, unattended security updates).

**Shared TLS snippet**, used by every site:

```
ssl_protocols TLSv1.2 TLSv1.3;
add_header Strict-Transport-Security "max-age=31536000" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "same-origin" always;
```

**Default server:** plain HTTP is redirected to HTTPS, and any request for a name I haven't configured is refused with `ssl_reject_handshake on`. Only names I deliberately set up get through.

**Adding a service** is one small file: copy the template, change the name and backend address, reload. About two minutes.

| Site | Backend | Notes |
| --- | --- | --- |
| Home Assistant | HTTP, port 80 | Needs websockets (in the shared proxy snippet) |
| Proxmox | HTTPS, port 8006 | Self-signed upstream, so `proxy_ssl_verify off` for the internal hop; large uploads allowed; buffering off for the VM console |
| Pi-hole | HTTP, port 80 | Root path redirected to `/admin/` |
| Uptime Kuma | HTTP, port 3001 | On the Docker VM |

## Monitoring with Uptime Kuma

Runs in Docker with its data in a mounted folder, pinned to major version 2 so an update can't surprise me.

| Monitor | Type | What a failure means |
| --- | --- | --- |
| Home Assistant, Proxmox, Pi-hole web | HTTPS | That service, Nginx or the certificate is broken. Certificate expiry alerts on |
| DNS – main and backup Pi-hole | DNS query through each Pi-hole | The house can't resolve names |
| Nginx | Ping | The front door is down (explains several HTTPS failures at once) |
| Tailscale exit node | Ping | My phone's internet via home will fail |
| Router, Internet | Ping | Home network or broadband is down |

- 60-second checks with 2 retries, so a single blip doesn't trigger an alert
- **Alerts via ntfy** to my phone. The topic name is long and random, because on the public server anyone who knows a topic name can read it
- **Tested end to end:** stopping the Nginx container turned exactly the four web monitors red and sent alerts; starting it sent recovery alerts
- **Known gap:** Uptime Kuma runs on the same host it monitors, so if the whole server dies it can't alert. An external check is a future improvement.

## Troubleshooting: Home Assistant "400: Bad Request"

DNS, HTTPS and Nginx all worked, but Home Assistant rejected every proxied request.

1. The log said the HTTP integration "is not set-up for reverse proxies".
2. I added `use_x_forwarded_for` and `trusted_proxies` to `configuration.yaml`. The first attempt failed validation because of a missing space after the list dash, and Home Assistant refused to restart with an invalid config, which was a useful safety net.
3. After fixing it, the config check passed and Home Assistant restarted, but the error continued.
4. I verified from the HAOS shell that the saved file was correct, ruled out hidden non-breaking spaces with `grep`, and confirmed with `docker inspect` that the restart happened after the save.
5. The clue was the port: Home Assistant served on port 80 even though no config set it. That meant HTTP settings were coming from somewhere else.
6. **In this Home Assistant version, HTTP and reverse-proxy settings have moved into the UI** (Settings → System → Network → HTTP server). Setting the trusted proxy there, as a single-address network (`/32`), fixed it immediately. I removed the unused YAML block.

Lesson: **software changes faster than documentation.** Confirming each layer with evidence (the file on disk, the restart time, the logs) narrowed it down to "the setting lives somewhere else".

## Security choices

- No ports opened; HTTPS names only resolve on my network or through my VPN
- Internal network layout never published in public DNS
- Least-privilege API token, stored with restrictive file permissions
- Modern TLS only, HSTS, and unknown names rejected at the proxy
- Home Assistant trusts **only** the Nginx address as a proxy, so other devices can't fake their source address
- The Home Assistant app is back on "Most secure" mode now that it uses HTTPS everywhere

## Gotchas and lessons

- **Containers don't always inherit the host's DNS.** Uptime Kuma couldn't resolve anything until I pinned its DNS servers in `compose.yml`.
- **Proxmox login realm:** root uses "Linux PAM"; the wrong realm gives a 401.
- **Similar hostnames cause mistakes.** Commands landed on the Docker VM instead of the Nginx container because both names start the same. Check the prompt.
- **Screenshots can leak secrets.** One showed the start of several passwords, which are being rotated and split into separate password-manager items.
- **Save credentials before entering them.** Generating the password in the password manager first prevents lockouts.
