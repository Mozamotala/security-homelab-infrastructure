# Incidents

Real failures on this NAS, how they were diagnosed, and how they were fixed. Kept as a log rather than cleaned up after the fact, because the diagnosis process is the point.

## Drive failure and live RAID1 replacement

**The failure.** One drive (a Seagate ST2000DL001 "Barracuda Green," an era-notorious model) began failing: 29,744 reallocated sectors, 344 pending/uncorrectable sectors, and over 3,300 logged read errors. Caught via a custom SMART-monitoring script (`drive-monitor.sh`) emailing nightly health reports at 2am — the first alert actually pointed at the wrong device, and diagnosing which drive was really failing (`smartctl -a`, checking reallocated/pending/uncorrectable counts) was the first real step.

**The fix.** Bought a Seagate IronWolf 2TB (NAS-rated for 24/7 use) as the replacement. Ran the full btrfs swap live:
1. `btrfs device delete` to migrate data off the failing drive onto the healthy one while the pool stayed online
2. Physically swapped the drive
3. `btrfs device add` to bring the new drive into the pool
4. `btrfs balance start -dconvert=raid1 -mconvert=raid1` to rebuild the mirror onto the new drive

**The residual.** After the swap, 18 sectors on the new drive came back as uncorrectable and couldn't be fixed despite repeated attempts. Rather than treat this as unresolved, it became the new baseline: the monitoring alert threshold was updated so a stable count of 18 is normal, and the alert only fires if that number ever increases — meaning any real degradation gets caught immediately rather than buried in noise from a known, stable quirk.

## Write-time btrfs corruption + RAM fault (2026-09-24)

The btrfs data pool (`nextcloud` label, spanning `sdb` + `sdc`, hosting the `killa` share including Immich's Postgres data) was forced read-only by write-time tree corruption, taking Immich down. On reboot, the NAS then failed to come back up.

**Diagnosis.** Traced to a faulty RAM stick — pulling it let the system boot again.

**Fix and verification.** With the bad RAM removed, the pool came back read-write, `btrfs device stats` showed no I/O errors, and all containers returned to healthy. Frigate didn't auto-restart with the rest and had to be started manually. A full scrub (1.65TiB) was kicked off to verify the pool was clean afterward.

**Lesson.** This is the concrete case for the storage.md note that RAID1 isn't a backup: the corruption happened at write time, before RAID1's mirroring could help — both copies would have gotten the bad write. The fix here was hardware (RAM), not RAID.

*Follow-up still open: confirm the scrub finished clean, memtest or replace the pulled RAM stick, and back up Immich photos off the NAS separately from the RAID mirror.*

## Data loss: lost Nextcloud encryption key

Early on, lost the Nextcloud encryption key before backups were in place, losing that dataset entirely. This is the direct reason the current automated monthly backup strategy (`config-backup.sh`, see [scripts.md](./scripts.md)) exists — though note that script backs up configuration, not data; the gap this incident exposed (no data backup) isn't fully closed by it even now.

## Compound outage: power cut

A single power cut caused two simultaneous failures:
- The TP-Link Deco mesh's bridging between wired and WiFi clients corrupted — fixed with a full node power-cycle
- Rockstor's immutability flag tripped on `/mnt2/home`, blocking Docker from starting entirely

**Fix.** Cleared the immutable flag with `chattr -i`, then fixed it permanently with a custom systemd drop-in that auto-clears the flag on every future boot, so a repeat power cut doesn't require manual intervention.

## RAM fault (earlier occurrence)

System reported 7.6GB instead of the expected 16GB installed. Diagnosed via `free -h` and `/proc/meminfo` before physically opening the case and finding a loose RAM stick.

## DNS hijack via Tailscale MagicDNS

Tailscale's MagicDNS silently overrode local name resolution. Diagnosed and disabled once the symptom was traced back to it.

## *arr stack integration bugs

A rotating set of smaller issues while integrating the *arr stack (and later Bookshelf + Kavita on top of it): a nonexistent Docker image tag, a port collision between two services defaulting to the same port, file permission errors blocking writes to an app's own data folder, a missing volume mount that left one app blind to files another app had already downloaded, and — hardest to pin down — a qBittorrent authentication failure that turned out to be a confirmed compatibility bug in a specific qBittorrent version, fixed by pinning to an older release.

## Jellyfin issues (recurring, over time)

Several separate problems surfaced with Jellyfin over time, not all at once:
- Images silently failing to load in the web UI
- Transcoding and hardware acceleration settings needing real tuning to work correctly
- A persistent annoyance where already-watched shows kept reappearing in "Recently Added" — traced to Jellyfin sorting by file date instead of scan date, fixed by changing the library's date-added behavior in settings

## RAM fault — the numbers

(Expands on the RAM fault entry above.) The server was showing only 7.6GB of RAM instead of the expected 16GB. Diagnosed methodically — `free -h`, `/proc/meminfo`, load tests — before physically opening the case and finding a loose RAM stick, probably from earlier hardware work. After reseating it, containers that had been RAM-starved recovered immediately: qBittorrent alone dropped from 1.35GB to ~99MB memory usage once it had headroom again instead of thrashing.
