---
name: project-public-ip-security-checks
description: "User periodically asks for a vulnerability/security check on their public IP; documents the home network's exposed setup and the scanning method used"
metadata: 
  node_type: memory
  type: project
  originSessionId: b1a1a73d-5242-4d8e-aaa4-cb4baf0cc867
  modified: 2026-09-22T10:15:22.729Z
---

User recurringly asks (in Dutch, "doe een vulnerability en security check op mijn publieke ip") for a port/vulnerability scan of their home network's public IP.

**Setup discovered (2026-08-16):** Only ports 80 and 443 are open on the public IP. Port 80 serves the default "Nginx Proxy Manager" landing page (confirms NPM is the reverse proxy in front of all self-hosted services). Port 443 enforces strict SNI matching — connecting with the bare IP as SNI (no real hostname) gets a TLS "unrecognized name" alert rather than a default cert, so no host/vhost enumeration is possible via the raw IP. Home Assistant (8123) is not directly exposed. The user also runs a QNAP NAS (see `qnap/qnap-containers.yaml`) — its admin ports (5000/5001) and other NAS/self-hosted ports (rsync, NFS, Webmin, torrent, Portainer, Grafana) were all closed too.

The actual external domain/hostname is stored via `!secret config_external_url` in `secrets.yaml` (not in the repo, not read) — so deeper vhost-specific checks (security headers, TLS grade via SSL Labs, actual HA login page) need the user to supply the hostname or run tools like Qualys SSL Labs / Mozilla Observatory themselves against their real domain.

**Hostnames confirmed by user (2026-08-17):** `homeassistant.kakkenberg.nl` and `homeassistant.kakkenberg.com` — both serve the same Home Assistant instance behind NPM (openresty). Notable asymmetry: `.nl` resolves directly to the home ISP IP (149.143.123.164, DELTA Fiber Nederland/Kabelfoon range) — origin IP exposed to anyone who resolves it — while `.com` resolves to Cloudflare anycast IPs, hiding the real origin behind Cloudflare's proxy/WAF. Both have solid baseline hardening: TLS 1.3 (TLS_AES_256_GCM_SHA384), HSTS `max-age=63072000; preload`, `X-Frame-Options: SAMEORIGIN`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`, HTTP→HTTPS 301 redirect, `/api/` returns 401 (auth required), no `.git` exposure, `/local/` forbidden (no directory listing), `robots.txt` disallows all. Only 80/443 open on the `.nl` origin IP (consistent with the general public-IP scan).

**Why:** Recurring request — the user wants to keep tabs on their home network's external attack surface (HA + QNAP NAS behind NPM).

**How to apply:** On the old Windows 11 laptop, no `nmap` was installed, so the check was done via a PowerShell `TcpClient`-based connect scan plus `curl`/`openssl s_client` for HTTP headers and TLS behavior on any open web ports. Since 2026-09-22 we work from a Fedora laptop instead, where `nmap` IS installed (`/usr/bin/nmap`) — prefer a real `nmap` scan (e.g. `nmap -Pn -p <risky-ports> <public-ip>`) over re-deriving the PowerShell approach. Public IP changes (dynamic/home connection) — always re-resolve it (e.g. `curl ifconfig.me`) rather than reusing a cached value. This is treated as a "regular" read-only action on the user's own infrastructure, no confirmation needed before running.
