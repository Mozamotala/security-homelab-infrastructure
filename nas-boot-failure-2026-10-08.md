# Incident: NAS failed to boot after a reboot (2026-10-08)

**System:** Rockstor (openSUSE Leap 15.6) NAS, Ryzen 5500, OS on a single SATA disk, data on a two-drive btrfs RAID1 pool.
**Impact:** NAS offline for several hours; no data loss found.
**Status:** Resolved. The NAS now boots by itself from the default GRUB entry. The exact root cause of the earlier hang is not confirmed (see below).

## Summary

I reset my shell passwords and rebooted. The NAS never came back on the network (Tailscale showed it offline). Getting it back needed a screen, a rescue USB and several rounds of diagnosis, because the failure had more than one layer.

## Timeline and diagnosis

| Symptom | What it meant | Action |
|---|---|---|
| Rockstor flagged device errors on both pool drives | Btrfs error counters, not necessarily failing disks | Planned SMART + scrub checks |
| After reboot: no ping, Tailscale offline, NIC light amber and not blinking | Link up but OS never brought networking up | Needed a display |
| Ping sweep + port 443 scan found only the router and a printer | NAS not on the network at any address | Connected a GPU to see the console |
| GRUB: `shim_lock protocol not found` | Secure Boot on, but GRUB not started through shim | Disabled Secure Boot in BIOS |
| Kernel panic: `Unable to mount root fs on unknown-block(0,0)`, empty partition list | Kernel started but saw no disks; initrd/driver problem | Checked disks from the GRUB prompt: all three visible, files present |
| Same panic on the older kernel | Not a kernel-version problem | Booted an openSUSE Leap 15.6 rescue USB |
| Rescue shell `lsblk -f` | OS disk and both pool drives all healthy and visible | Mounted root + EFI, chrooted in |
| `grub2-mkconfig` worked; `dracut` failed with `/var/tmp: No such file or directory` | initrd rebuild needs a temp dir that did not exist in the chroot | Created `/var/tmp`, retried (result not confirmed) |
| Default entry hung after "Loading initial ramdisk", screen dead | Display/GPU path or initrd | Recovery entry from the GRUB menu booted fully |
| Later reboots | Default entry boots by itself, pools mount, Docker starts after a few minutes | Most likely fixed by the rebuilt initrd and regenerated GRUB config, but the earlier hang could also have been the temporary GPU/riser display path. Not confirmed. |

## Useful techniques

- **Reading ping output properly:** `Destination host unreachable` replies came from my own PC, not the NAS. "Received = 2" in the summary was a false positive. Always check the source address of the reply.
- **Finding a device with no known IP:** ping sweep to fill the ARP table, then a TCP connect scan for port 443 (Rockstor's web UI). The one hit turned out to be a printer, so a hit still needs verifying.
- **GRUB prompt triage:** `ls` to list disks and partitions, `ls (hd0,gptN)/` to find the root filesystem, `ls (hd0,gptN)/boot/` to confirm kernels and initrds exist. GRUB's `ls` does not support `-l`.
- **Rescue chroot:**
  1. Mount the root partition on `/mnt` and the EFI partition on `/mnt/boot/efi`.
  2. Bind-mount `/dev`, `/proc`, `/sys`, `/run` into `/mnt`.
  3. `chroot /mnt`, then rebuild with `dracut -f --regenerate-all` and `grub2-mkconfig -o /boot/grub2/grub.cfg`.
  4. On openSUSE, `mkinitrd` may not exist; `dracut` is the real tool.
- **No sudo on Rockstor:** work as root with `su -`. A plain user does not have `/sbin` in its PATH, so `btrfs` shows "command not found".

## Drive and filesystem checks

- `smartctl -a` on both pool drives: overall health PASSED, 0 reallocated sectors, 0 pending sectors, 0 offline-uncorrectable, 0 CRC errors, empty error logs. The huge raw values on Raw_Read_Error_Rate and Seek_Error_Rate are normal for Seagate drives; judge them by the normalised value.
- `btrfs device stats`: 0 read, write, flush and generation errors. Corruption counters were non-zero (37 and 32), consistent with an unclean shutdown on top of an older known baseline.
- `btrfs scrub` (full pool, 1.66 TiB, finished in 1h49m at about 265 MiB/s): `csum=18`, 0 corrected, 18 uncorrectable, 0 unverified. The errors sit in two files (a log file and a database file). 18 is my known baseline from the earlier RAID5 to RAID1 conversion, so this is **not new damage**. Only a result above 18 would have indicated new problems.
- OS disk (1 TB WD Blue, single-copy btrfs): SMART PASSED with 0 reallocated, pending, offline-uncorrectable and CRC errors. `btrfs device stats /` showed 0 I/O errors and 1 corruption error. A full scrub of the 64 GiB root filesystem (about 10 minutes) finished with **no errors found**, so that single counter was a one-off, most likely from the unclean shutdown. Because the root filesystem has only one copy of its data, a bad block there could not be repaired automatically, which is why this scrub mattered.
- Rockstor's "DEV ERRORS DETECTED" flag stays on while any btrfs error counter is above zero, even for old errors. Once the scrubs were clean and the counters had stopped rising for hours, I reset them with `btrfs device stats -z` (it prints the old values, then zeroes them) and confirmed 0 on a second read, so any new error will stand out.
- After the second reboot, Docker took a few minutes to start while the pools mounted. All containers came back without intervention.
- RAM: `free -h` shows the full 15 GiB again after reseating a stick earlier. The btrfs counters stayed flat for hours afterwards, but a MemTest86 pass is still on the to-do list.
- Lesson: a scrub on a pool this size took about 1h50m. Run it in the background (`btrfs scrub start`, then `btrfs scrub status`) so a stray Ctrl+C cannot cancel it. I initially misjudged the pool size from a partial run.

## What I would do differently

1. Keep a recorded root password and a break-glass access method before touching credentials.
2. Keep a spare working GPU or a known-good display path. A headless NAS with a CPU that has no iGPU is hard to debug when it fails to boot.
3. Do not use a riser cable and a stack of adapters in the recovery path.
4. Confirm that the `dracut` rebuild completes and check the output before rebooting.
5. Back up the data that matters off the pool. RAID1 protects against a disk failure, not against corruption or a bad write.

## Open items

- Check whether the app's database setting makes the damaged database file a live database or a leftover, then delete or restore it so the baseline can go back to 0.
- Network: the link had negotiated only 100 Mbps. After plugging into the switch it reports 1000 Mbps, so the cable or port was the cause. The NAS address is reserved in the router.
- Replace the temporary GPU setup with a permanent, reliable display card.
- Memtest or replace the pulled RAM stick, and back up the photo library off the NAS.
