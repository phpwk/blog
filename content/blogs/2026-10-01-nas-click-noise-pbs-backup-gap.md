---
title: "Day 2 with my custom build NAS server for the homelab"
date: 2026-10-01T20:00:00+07:00
tags: ["homelab", "truenas", "proxmox", "zfs", "backup", "incus"]
draft: true
---

## Symptom

A scheduled Proxmox Backup Server job had been running against one of my LXC containers every night since 2026-09-27. Every run reported `TASK OK`. The run on 2026-10-01 was incremental and reused 98.9% of the previous backup, which read as healthy.

Line 7 of that same task log said something else:

```
INFO: excluding bind mount point mp0 ('/mnt/immich-data') from backup (not a volume)
```

I didn't catch that line myself. Claude, who I was working through this with, is the one who flagged it. I wasn't reading the log line by line looking for it. I'm OK with that: Claude catching the backup gap doesn't bother me.

`/mnt/immich-data` is the Immich photo library, confirmed from the container's own Resources tab in the Proxmox UI. Immich itself is still in a testing phase here, with only a handful of photos uploaded so far, not the full library yet. So nothing was actually at risk when this was found, but the backup mechanism had been silently incomplete since the day it was set up, and that's worth knowing before it does matter.

Getting to that line took two earlier incidents on the same NAS, in the same week, so the order matters here:

1. A regular clicking noise from the NAS with no disk I/O to explain it
2. Rebuilding Proxmox Backup Server (PBS) onto a dedicated SSD, a separate architectural fix unrelated to what actually stopped the clicking (more on that below)
3. Re-triggering the backup job after the rebuild, which produced the log above

## Environment

- NAS: ASRock B660M Steel Legend, Intel i3-12100, Jonsbo N4 case, TrueNAS Community Edition 25.10.7 ("Goldeye")
- Pool `hdd1`: 2x Seagate IronWolf Pro 4TB (`ST4000NT001-3M2101`), ZFS mirror, 7200rpm, SATA 3.3 / 6.0 Gb/s
- PBS 4.2.6, running as a privileged Incus system container (Debian Trixie, the official `proxmox-backup-server` apt package; there's no Docker image for PBS)
- PBS's own chunk store lives on a separate dataset, `hdd1/apps/pbs`, bind-mounted into the container at `/data`, kept apart from the container's own root filesystem on purpose
- The Proxmox VE host runs the container being backed up: CT 201, Immich (still in testing), tagged `community-script` and `photos` in the PVE UI. The bind-mount shape of the data path is believed to be a default of the Proxmox Community Scripts Immich installer ("I think," not fully confirmed), but the specific host path (`/mnt/pve/ssd-storage/immich-data`) was changed by hand afterward, not left at whatever the installer originally set.
- CT 201's actual config, from its Resources tab:
  - Root Disk: `local-lvm:vm-201-disk-0,size=24G`, a real Proxmox-managed volume (storage name + volume id + size)
  - Mount Point (mp0): `/mnt/pve/ssd-storage/immich-data,mp=/mnt/immich-data`, a bare host path, no storage:volume id, no size. That's the tell, visible right in the PVE UI, for a bind mount versus a managed volume.
  - iGPU passthrough for hardware transcode: `/dev/dri/renderD128`, `/dev/dri/card0`, `/dev/kfd`

## What I checked, in order

### The clicking noise

1. Heard a regular click from the NAS. Ran `iostat -x` on both drives:

   ```
   Device            r/s     rkB/s   w/s     wkB/s   w_await wareq-sz   %util
   sda              0.01      0.48    8.38    214.85    2.52    25.62    1.85
   sdb              0.02      0.56    8.28    214.82    2.58    25.95    1.87
   ```

2. And `zpool iostat -v hdd1 5`, sampling every 5 seconds. Every single sample showed write activity, split almost exactly in half between the two mirror drives:

   ```
   pool                                      alloc   free   read  write   read  write
   ----------------------------------------  -----  -----  -----  -----  -----  -----
   hdd1                                      4.50G  3.62T      0     15    186   431K
     mirror-0                                4.50G  3.62T      0     15    186   431K
       <vdev-a-guid>                            -      -      0      7    118   215K
       <vdev-b-guid>                            -      -      0      7     67   215K
   ----------------------------------------  -----  -----  -----  -----  -----  -----
   hdd1                                      4.50G  3.62T      0     21      0   198K
   hdd1                                      4.50G  3.62T      0     17      0   205K
   hdd1                                      4.50G  3.62T      0     19      0   184K
   hdd1                                      4.50G  3.62T      0     22      0   267K
   hdd1                                      4.50G  3.62T      0     19      0   198K
   hdd1                                      4.50G  3.62T      0     19      0   195K
   hdd1                                      4.50G  3.62T      0     19      0   198K
   hdd1                                      4.50G  3.62T      0     18      0   210K
   hdd1                                      4.50G  3.62T      0     15      0   184K
   ```

   No sample came back idle. The difference between the two commands: `zpool iostat` targets the pool, `iostat` targets the actual disk device. Two different layers, not two different sampling methods.

3. Ruled out drive failure directly:

   ```
   $ sudo zpool status -v hdd1
     pool: hdd1
    state: ONLINE
   config:
           NAME                                      STATE     READ WRITE CKSUM
           hdd1                                      ONLINE       0     0     0
             mirror-0                                ONLINE       0     0     0
   errors: No known data errors
   ```

   `smartctl -a` on both drives: overall-health self-assessment **PASSED**, `Reallocated_Sector_Ct`, `Current_Pending_Sector`, `Offline_Uncorrectable`, and `UDMA_CRC_Error_Count` all at 0, `Power_On_Hours` at 27. `dmesg -T | grep -iE 'ata|reset|error'` turned up nothing but normal boot-time AHCI link-up lines and one unrelated false match: `Error: Driver 'pcspkr' is already registered, aborting...` (the PC speaker driver, not storage).

4. Claude's read of the pattern at the time pointed at two possible writers to `hdd1`: the TrueNAS System Dataset (telemetry/RRD data), and the Incus "Instances" storage pool backing the PBS container's own root filesystem. I don't agree with crediting the second one. The PVE backup job runs once a day on a cron schedule. That doesn't explain a click recurring every few minutes. The System Dataset, which writes continuously in the background regardless of any backup schedule, fits the pattern much better.

### Moving storage off the pool that was clicking

1. Moved the System Dataset to boot-pool (System Settings → Advanced → Storage). The clicking stopped. The Instances storage pool was still sitting on `hdd1` at this point and stayed there for several more days; the SSD it eventually moved to wasn't even purchased yet. So this was the System Dataset alone, not a simultaneous fix.
2. Separately (and unrelated to the clicking, which was already gone), TrueNAS's own documentation advises against putting Instances/Apps storage on boot-pool, and Instances storage can't be live-migrated between pools anyway. That's a reason to get it off `hdd1` on its own architectural merits, not something done to silence the noise.
3. Decided to add a dedicated SSD instead of using boot-pool: to improve performance, and to prevent HDD noise from app/instance activity in the future. It made sense because the apps themselves can be recreated quickly, while the actual data stays on the separate `hdd1` pool. Checked the B660M Steel Legend's SATA layout against its manual: 6 ports total (4 right-angle, 2 vertical), no lane-sharing conflicts for this build.
4. [Correction along the way] Assumed the Jonsbo N4's 2.5" bays had a backplane like the 3.5" bays. They don't. Only 4 of the case's 6 3.5" bays run through the backplane; the 2.5" slots are bare cavities needing a manually run SATA data cable and a power cable from the PSU. The 2.5" mounting kit needed for the SSD came included with the case itself: no separate bracket purchase needed.

### The rebuild, a repeat bug, and the log that gave it away

The backup job was only ever disabled for the duration of this maintenance window, not left off indefinitely. Leaving backups off mid-fix is fine as long as the fix is quick, so there isn't much delta between the last known-good backup and the actual state of things if something went wrong in between. Exact sequence:

1. Added the new SSD as its own pool, `ssd1`.
2. Imported and started the PBS container on `ssd1`.
3. Hit the PBS proxy issue: port 8007 came back empty/reset. Root cause: `/run/proxmox-backup` (tmpfs, rebuilt on every container start) was owned by `root` instead of `backup` inside the container. Fixed with `chown -R backup:backup /run/proxmox-backup/` and a service restart.
4. Both times this permission bug showed up (the first time during the original PBS deployment, this time during the rebuild), it coincided with a container identity change: first switching the container from unprivileged to privileged, this time reimporting PBS fresh onto `ssd1`. Not something that showed up during ordinary operation in between. I don't know if a regular/standard PBS install would trigger the same bug: open question.
5. The PBS datastore reattached cleanly once the container was back up. Its chunk data on `hdd1/apps/pbs` was never touched by any of this, since it lives on its own dataset separate from the container's root filesystem. That split pays off doubly: backup data sits on `hdd1`, the cheaper, larger pool, while the application itself runs on `ssd1`, the faster one, each workload on the storage tier that actually fits it.
6. Re-enabled the `truenas` PBS storage target and the backup job in PVE.
7. Triggered a manual run.
8. The run reported `TASK OK`. Reviewing the log with Claude afterward, line 7 showed the bind mount excluded, "not a volume."

## Root cause

Two separate root causes, chained:

1. **The clicking noise** wasn't drive failure. It was the TrueNAS System Dataset writing continuously to `hdd1` in the background. Claude's original diagnosis also pinned some of the blame on the Incus Instances storage pool backing the PBS container. I don't think that holds up: the PVE backup job fires once a day on a cron schedule, which can't explain a noise recurring every few minutes. Moving the System Dataset to boot-pool alone stopped the clicking, days before the Instances-storage pool was ever moved off `hdd1`.
2. **The backup gap**: Proxmox's vzdump/PBS backup mechanism structurally cannot back up an LXC bind mount point, no matter what the `backup=1` flag on it says. Only the container's rootfs and real volume-backed mount points are ever included. Not a one-off bug, and not new as of this rebuild: every run of this job, since 2026-09-27, excluded the same bind mount. Confirmed two ways: the official vzdump documentation states plainly, "Device and bind mounts are never backed up as their content is managed outside the Proxmox VE storage library," and a Proxmox staff member on the forum gave the same answer to someone hitting this exact issue: "you mounted the zfs volume with a bind mount, those can not be included in the backups," recommending the ZFS storage backend (a real volume through the container's resource settings) instead. The tell was sitting right in the PVE UI the whole time: the rootfs is `local-lvm:vm-201-disk-0,size=24G` (storage:volume,size: a managed volume), while mp0 is `/mnt/pve/ssd-storage/immich-data,mp=/mnt/immich-data` (a bare host path: a bind mount).

The bind mount itself is believed to be a default of that installer; the specific path was customized afterward. Either way, the backup exclusion applies the same regardless of how the mount point got there. It is purely a function of it being a bind mount, not a volume.

## Fix

- Clicking noise: System Dataset moved to boot-pool. This alone fixed it. Instances storage was separately moved to a dedicated SSD (`ssd1`) days later, for architectural reasons unrelated to the noise.
- Repeat permission bug: `chown -R backup:backup /run/proxmox-backup/` plus a service restart: identical fix to the first time this bug class showed up.
- Backup gap: not fixed yet. The planned next step, already on the list before this was even found, is moving Immich's data off the bind mount onto a proper NFS-mounted ZFS dataset instead, which would close this gap as a side effect of a move made for other reasons. No time to do it yet. Tracked as a to-do.

## How I know it is fixed

- Clicking noise: confirmed gone, from moving the System Dataset off `hdd1` to boot-pool. The PVE backup cron's once-a-day schedule rules out the Instances-storage pool as an explanation for noise recurring every few minutes, and the clicking stopped days before that pool was even moved.
- Permission bug: port 8007 reachable again, backup job completing.
- Backup gap: **not fixed yet.** `TASK OK` turned out not to mean what it looked like it meant, and the fix (moving Immich's data to the NFS-mounted dataset) hasn't happened. The real "it's fixed" will be when Immich, running on the PVE host, uses a storage dataset provided by TrueNAS with mirror redundancy applied, instead of relying on local storage on the PVE host for personal data like photos. Not just backed up, but not sitting on a single unprotected disk in the first place. I also don't know yet whether there's other configuration in the bind-mounted path worth backing up beyond the photos themselves. That's still open too.

## Why it took me a while

This is my first time building a NAS server and installing TrueNAS SCALE. It's also been a long time since I last worked with physical servers day to day. I'm still learning and familiarizing myself with this stack, which is why the clicking noise took several steps instead of being obvious right away.

## What I'd do differently

The best practice I'd follow from the start now: two separate pools, HDD and SSD, one for data and one for app/instance storage. TrueNAS also runs containers and VMs directly, so this split matters beyond just PBS. The actual recommendation for the System Dataset is to put it on a separate pool with real redundancy, not on boot-pool. An SSD or NVMe pool can wear faster under that write-heavy load, but it's much quieter than HDD, which is the trade I'd make again.

## References

- [Proxmox vzdump manual page](https://pve.proxmox.com/pve-docs/vzdump.1.html): "Device and bind mounts are never backed up as their content is managed outside the Proxmox VE storage library."
- [Mount Point excluded from backup although backup=1-Flag is set](https://forum.proxmox.com/threads/mount-point-excluded-from-backup-although-backup-1-flag-is-set.86087/), Proxmox forum thread, answered by Proxmox staff (Oguz)
