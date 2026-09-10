# security-homelab-infrastructure

A self-hosted home NAS and Docker homelab, built and maintained solo over ~2 years — started on a dead Core 2 Duo box and grown into a mature ~18-service stack. Total project timeline, counting earlier dead builds and downtime, is closer to 3 years.

## Hardware and OS

- Ryzen 5500, 16GB RAM, running Rockstor (openSUSE-based)
- Bulk storage on a dedicated disk (sdc), with service data and the Nextcloud webroot under `/mnt2/killa` and `/mnt2/nextcloud`
- Predecessor build: a Core 2 Duo with 8GB RAM, which failed within months — the current Ryzen box is the second full rebuild

## Services

Runs roughly 18 Docker services:

- **Media & storage:** Jellyfin, Nextcloud, Immich, Kavita, Bookshelf
- **Media automation:** Sonarr, Radarr, Prowlarr, qBittorrent, FlareSolverr
- **Security & networking:** ClamAV (on-access antivirus, verified with an EICAR test pipeline, wired to ntfy/email alerting), Pi-hole, Cloudflared
- **Home monitoring:** Frigate (NVR and camera detection)
- **AI/local tooling:** Ollama
- **Dashboard & monitoring:** Homepage, Glances, Dozzle
- **Alerting:** ntfy, Gotify

## Networking and access

- Remote access via Tailscale, no ports forwarded or opened on the router
- ntfy and cron used for notifications, including a small script for DTM race alerts

## Storage: Btrfs RAID1 and a full drive replacement

Storage runs on Btrfs RAID1 — every write is mirrored live across two drives, so a single drive failure doesn't cost any data.

**The failure.** One drive (a Seagate ST2000DL001 "Barracuda Green," an era-notorious model) began failing: 29,744 reallocated sectors, 344 pending/uncorrectable sectors, and over 3,300 logged read errors. Caught via custom SMART-monitoring scripts (`drive-monitor.sh`) emailing nightly health reports at 2am — the first alert actually pointed at the wrong device, and diagnosing which drive was really failing (`smartctl -a`, checking reallocated/pending/uncorrectable counts) was the first real step.

**The fix.** Bought a Seagate IronWolf 2TB (NAS-rated for 24/7 use) as the replacement. Ran the full Btrfs swap live: `btrfs device delete` to migrate data off the failing drive onto the healthy one, physically swapped the drive, then `btrfs device add` and `btrfs balance start -dconvert=raid1 -mconvert=raid1` to rebuild the mirror onto the new drive.

**The residual.** After the swap, 18 sectors on the new drive came back as uncorrectable and couldn't be fixed despite repeated attempts. Rather than treat this as unresolved, it became the new baseline: the monitoring alerts were updated so a stable count of 18 is normal, and the alert only fires if that number ever increases — meaning any real degradation gets caught immediately rather than buried in noise from a known, stable quirk.

## Other incidents & troubleshooting

- **Data loss, early on:** lost the Nextcloud encryption key before backups were in place, losing the dataset. This is what led to the current automated monthly backup strategy.
- **Compound outage:** a power cut caused two simultaneous failures — the TP-Link Deco mesh's bridging between wired and WiFi clients corrupted (fixed with a full node power-cycle), while Rockstor's immutable flag tripped on `/mnt2/home` and blocked Docker from starting entirely. Cleared with `chattr -i`, then fixed permanently with a custom systemd drop-in that auto-clears the flag on every future boot.
- **RAM fault:** system reporting 7.6GB instead of the expected 16GB. Diagnosed via `free -h` and `/proc/meminfo` before physically opening the case and finding a loose RAM stick.
- **DNS hijack:** Tailscale's MagicDNS silently overrode local name resolution — diagnosed and disabled.
- ***arr stack integration bugs:** a nonexistent Docker image tag, a port collision between two services, file permission errors, and a confirmed qBittorrent version-compatibility bug, fixed by pinning an older release.

## Monitoring & alerting

- Custom SMART drive-health and RAID-status monitoring script, emailing a consolidated nightly report (drive health, marginal attributes, temperatures, RAID status) rather than spamming per-issue alerts
- ntfy/email alerting on ClamAV detections, service health, and server restarts
- ntfy + cron used for personal notifications too, including a small script for DTM race alerts

## Tools and research

This project was built through heavy self-directed research, with AI assistance (Claude, ChatGPT, Gemini, DeepSeek) used throughout for troubleshooting, explaining concepts, and thinking through fixes — not as a substitute for understanding, but as a research and learning aid, the same way documentation or Stack Overflow would be used.

No command was copy-pasted without understanding it. Every command was typed out personally, and when something didn't make sense, the question came before the keystroke — asking why a fix worked, not just applying it. The lab was treated as a learning platform throughout, not something AI set up on its own.

That said, real gaps remained — hands-on infrastructure work and structured security fundamentals aren't the same thing. That's the direct reason for the OverTheWire Bandit work below, and the planned move to Hack The Box afterward: closing those gaps deliberately, not just accumulating more infrastructure.

## Security learning, alongside this project

As of September 2026, working through OverTheWire's Bandit wargame (currently level 10 of 34), self-directed and ungraded — separate from formal coursework. Plan is to move on to Hack The Box's beginner tracks once Bandit is complete.

## Why this exists

Started as a way to stop paying for subscriptions (streaming, cloud storage, photo backup); turned into an ongoing way to build hands-on Linux, Docker, and infrastructure experience alongside formal cybersecurity studies.

## Status

This repo currently holds project documentation only. Docker Compose files and service configs will be added incrementally once secrets (passwords, API keys, Tailscale auth keys, etc.) are stripped out and replaced with placeholder values via `.env` files (not committed).
