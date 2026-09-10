# security-homelab-infrastructure

A self-hosted home NAS and Docker homelab, built and maintained solo over ~2 years, started on an old Core 2 Duo box and grown into a mature ~18-service stack.

## Hardware and OS

- Ryzen 5500, running Rockstor (openSUSE-based)
- Bulk storage on a dedicated disk (sdc), with service data and the Nextcloud webroot under /mnt2/killa and /mnt2/nextcloud

## Services

Runs roughly 18 Docker services, including:

- Jellyfin - media server
- Nextcloud - file sync and storage
- Immich - photo and video backup
- Frigate - NVR and camera detection
- ClamAV - on-access antivirus, verified with an EICAR test pipeline and wired to ntfy/email alerting
- Homepage - dashboard, with Glances and Dozzle for monitoring

## Networking and access

- Remote access via Tailscale, no ports forwarded or opened on the router
- ntfy and cron used for notifications, including a small script for DTM race alerts

## Reliability

- Recovered from a real outage caused by a Docker immutability-flag issue compounding with a power cut, which informed the current backup and monitoring setup
- Alerting on service health via ntfy/email, plus a server-restart listener

## Status

This repo currently holds project documentation only. Docker Compose files and service configs will be added incrementally once secrets (passwords, API keys, Tailscale auth keys, etc.) are stripped out and replaced with placeholder values.

## Why this exists

Started as a way to stop paying for subscriptions (streaming, cloud storage, photo backup); turned into an ongoing way to build hands-on Linux, Docker, and infrastructure experience alongside formal cybersecurity studies.
