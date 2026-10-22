# Before anything goes public

Run through this before every `git push` to a public repo and every LinkedIn or TikTok post.

## Code and configs

- [ ] `pre-commit run --all-files` passes (gitleaks finds no secrets)
- [ ] No real `.env` files committed; only `.env.example` with placeholders
- [ ] No passwords, API keys, Tailscale auth keys or tokens anywhere, including comments
- [ ] No real IPs in public docs; use placeholders like `<pihole-ip>`
- [ ] No public IP address, domain name or Tailscale network name

## Screenshots and images

- [ ] Public IP not visible (router pages, speed tests, dashboards)
- [ ] Wi-Fi network names hidden
- [ ] MAC addresses and device names blurred
- [ ] Browser tabs, bookmarks and notifications cropped out
- [ ] No family members' devices or client lists visible

## If a secret leaks

1. **Change the secret immediately.** Assume it's already been copied; bots scan GitHub within minutes.
2. Then remove it from the repo history (deleting the file isn't enough, it stays in Git history).
3. Note what happened in the phase write-up. Handling it properly is a good story, not an embarrassment.

---

# LinkedIn post template

Post once per phase or milestone, not every step.

```
🛠️ Homelab update: <phase name>

What I built: <one line, plain English>

Why it matters: <one line, e.g. "DNS no longer goes down when I restart my Docker host">

What broke: <the problem and how you diagnosed it>

What I learned: <one takeaway, ideally linked to your degree>

Next up: <next phase>

Write-up and configs: <link to the phase doc on GitHub>

#homelab #cybersecurity #selfhosted #proxmox
```

Attach one image: the architecture diagram, roadmap or a (checked) dashboard screenshot.
