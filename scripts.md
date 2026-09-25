# Automation Scripts

All scripts run as root via cron. Notifications go out via `mail`/`mailx` and/or [ntfy](https://ntfy.sh) to a private topic (redacted below — treat ntfy topic names as secrets, since anyone who knows the topic can read or post to it).

**Redacted from all scripts below:** alert email addresses and the ntfy topic name. Replace with your own before reuse.

## Schedule

| Time | Script | Purpose |
|---|---|---|
| Daily 02:00 | `drive-monitor.sh` | Drive health + RAID status check |
| Monthly, 1st @ 02:00 | `config-backup.sh` | Config backup |
| Weekly, Mon @ 02:00 + on boot | `autocompress.sh` | Media re-encoding to save space |
| Weekly, Sun @ 02:55 | `pre-reboot-check.sh` | Pre-flight check before scheduled reboot |
| Weekly, Sun @ 03:00 | `reboot` | Scheduled full reboot |
| Hourly | `container-health.sh` | Container uptime monitor + auto-restart |
| Daily 08:00 | `dtm_notify.py` | DTM race calendar reminders (not NAS-related) |

## drive-monitor.sh

Runs `smartctl` health/temp/reallocated/pending/uncorrectable-sector checks on each drive, plus `btrfs device stats` and `btrfs scrub status` on the data pool. Also checks that `config-backup.sh`'s log is less than 40 days old — flags it if the backup hasn't run recently or is missing.

Sends an immediate alert email if any check fails (failed/marginal health, high temp, reallocated/pending/uncorrectable sectors, RAID errors, stale/missing backup). Separately sends a full monthly summary report on the 1st of the month regardless of status.

## config-backup.sh

The monthly backup job. **Backs up configuration, not data** — it does not copy photos, media, or Nextcloud files. Saves:
- Current crontab
- Docker configs (`docker inspect` for frigate, nextcloud, ollama, qbittorrent, radarr, sonarr, prowlarr)
- Samba config (`/etc/samba/smb.conf`)
- Frigate config (`config.yml`)

Output goes to `/mnt2/killa/backups/system/`. Sends a status notification via both ntfy and email on completion, reporting OK or which parts failed.

*Because this only covers config, not data, restoring from it after a disk loss would rebuild the setup but not the actual files — those rely on the RAID1 mirror plus a separate backup for irreplaceable data (see storage.md).*

## autocompress.sh

Long-running loop (re-triggered weekly and on boot, then sleeps 7 days between scans) that finds H264 movies/shows over 500MB and re-encodes them to H265 (libx265, CRF 24) via a disposable ffmpeg Docker container, to reduce storage footprint.

Safety checks: uses a lock file to prevent overlapping runs, cleans up any `*_compressed.mkv` left behind by an interrupted run on startup, skips files still actively downloading (checked by comparing file size 60 seconds apart), and discards the compressed output if it isn't meaningfully smaller (or isn't smaller at all) than the original. Emails a summary of files compressed and space saved after each full scan.

## pre-reboot-check.sh

Runs 5 minutes before the weekly scheduled reboot. Checks that the expected containers (sonarr, radarr, prowlarr, qbittorrent, jellyfin, frigate, ollama, nextcloud) are running, logs CPU temperature, and flags if a compression job is mid-run (since the reboot will kill it). Emails a "rebooting now" notice with this status snapshot, then reboots.

## container-health.sh

Runs hourly. Checks 15 containers are running and auto-restarts any that are down (sonarr, radarr, prowlarr, qbittorrent, jellyfin, the four Immich containers, frigate, ollama, nextcloud, pihole, flaresolverr, ntfy). Also watches CPU temperature (alerts above 85°C) and CPU usage (alerts above 85%, reporting top processes and swap usage).

Alerts by email only when something's actually wrong (a restart happened or a check failed), with a 1-hour cooldown so a persistent issue doesn't spam — the cooldown clears automatically once everything's healthy again.

## dtm_notify.py

Not part of NAS operations — a personal notifier for the DTM (German Touring Car) racing calendar, hardcoded with the 2026 season schedule. Pings via ntfy and email 7, 3, and 1 days before each race weekend. Included here only because it runs on the same cron; has a `--setup` mode that installs its own cron entry and sends a test notification.
