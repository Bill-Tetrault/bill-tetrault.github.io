---
layout: post
title: "How To: Partition Rocky Linux 9 or Ubuntu LTS VMs for Container Workloads"
author: "Bill Tetrault"
date: 2026-05-29
description: "Standardize new VM builds for containerized workloads with clean OS/data separation, LVM growth headroom, and Docker data isolated from the root filesystem.  "
tags: [Linux, Rocky, Docker, Tutorial, Guide, DevOps]
categories: [guides, tutorials, linux]
---
<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# How To: Partition Rocky Linux 9 or Ubuntu LTS VMs for Container Workloads

**Purpose:** Standardize new VM builds for containerized workloads with clean OS/data separation, LVM growth headroom, and Docker data isolated from the root filesystem.  
**Applies To:** Rocky Linux 9, Ubuntu LTS, UEFI-booted VMs, single system disk starting at 128GB, optional secondary disk or LVM-backed data volume.  
**Last Updated:** 2026-05-30  
**Difficulty:** Intermediate to advanced

## Overview

This layout assumes a UEFI VM with one primary system disk and a separate storage target for container data, either as a second virtual disk or as an LVM logical volume. The main design goal is to keep the OS stable and small while giving Docker a dedicated filesystem under `/data/docker` so container growth does not fill `/` or `/var`.[^12][^13]

The partitioning pattern below favors practical separation over maximal fragmentation. `/var` gets isolated because logs, package caches, and container-related state can grow quickly; `/tmp` gets hardening flags because it is a common abuse target; `/home` is separated mainly for user isolation and simpler rebuilds; and `/data` becomes the standardized landing zone for Docker and any adjacent app state. Mount hardening options like `nodev`, `nosuid`, and `noexec` are widely recommended for scratch-like filesystems such as `/tmp`, while `noatime` is a common performance-oriented choice for data volumes.[^16][^25]

## Prerequisites

Before you start, assume the VM will have 8GB RAM initially and may later grow to 16GB or more. Use LVM so logical volumes can be expanded without redesigning the disk layout, and leave unallocated space in the volume group for future growth.[^4]

Use XFS on Rocky Linux where that is the typical default, and use either ext4 or XFS on Ubuntu depending on site standard and operational preference. XFS remains a common default on RHEL-family systems and is a solid fit for VM and container-host filesystems.[^29]

Have the following ready:

- UEFI boot enabled.
- One 128GB primary disk.
- One secondary disk, or one LVM logical volume dedicated to data.
- Docker installed from distro packages or the standard repo.
- `lvm2`, `xfsprogs` or `e2fsprogs`, and `rsync` available.

## Step-by-Step Instructions

### 1) Recommended disk layout

On the 128GB primary disk, keep boot partitions outside LVM and place everything else into one VG, such as `vg0`. A practical layout is:

- `/boot/efi` — 512MB, FAT32.
- `/boot` — 1GB to 2GB, ext4 or XFS.
- One LVM PV consuming the remaining space, with `vg0` holding:
- `lv_root` mounted at `/`.
- `lv_var` mounted at `/var`.
- `lv_tmp` mounted at `/tmp`.
- `lv_home` mounted at `/home`.
- `lv_swap` used for swap.
- Optionally `lv_data` mounted at `/data` if there is no secondary disk.

A practical 128GB example for 8–16GB RAM is:

| Mount | Size | Notes |
|---------|------:|-------|
| `/boot/efi` | 512MB | UEFI system partition. |
| `/boot` | 1GB | Use 2GB if more kernel headroom is preferred. |
| `lv_root` | 20GB | Enough for a lean OS, agents, and base tooling. |
| `lv_var` | 15GB | Logs, package state, caches, and service runtime growth. |
| `lv_tmp` | 4GB | Scratch space with restrictive mount options. |
| `lv_home` | 5GB | Small because these VMs are not user workstations. |
| `lv_swap` | 8GB | Reasonable default for 8–16GB RAM VMs. |
| VG free space | ~40–50GB | Reserved for future growth. |

If a separate data disk or dedicated `lv_data` is available, do not consume all remaining VG space for the initial build. Keeping a meaningful amount of VG free space makes it straightforward to extend `/`, `/var`, or `/home` later without touching partition tables.

### 2) Why these splits matter

Separating `/var` protects the root filesystem from log storms, package cache growth, and service state churn. Separating `/tmp` allows restrictive mount options and keeps temporary-file abuse away from `/`, while separating `/home` simplifies rebuilds, user isolation, and policy control.

This is not about blindly following a benchmark. It is about limiting blast radius on small-to-medium container hosts where operational failure is usually caused by uncontrolled growth in exactly these paths.

### 3) Data volume for Docker

Use either a second disk such as `/dev/sdb1` or a dedicated LVM logical volume such as `/dev/vg0/lv_data` as the single storage backend for container data. Standardize on mounting this filesystem at `/data`, then place Docker data under `/data/docker` so the container graph, writable layers, and local volumes do not compete with the OS filesystem.[^12][^13]

Filesystem choice:

- **XFS**: preferred on Rocky Linux and a strong default for dedicated container data volumes.[^29]
- **ext4**: also valid, especially on Ubuntu if that aligns with local standards.

Recommended mount options:

- For `/data`: `defaults,noatime`.[^25]
- For `/tmp`: `defaults,nodev,nosuid,noexec`.[^16][^17]

> ⚠️ WARNING: Do not let Docker, image caches, and persistent volumes live on the root filesystem unless the VM is intentionally disposable.

### 4) Example `/etc/fstab`

Below is a concrete example using LVM paths. Use UUIDs for the EFI partition, `/boot`, and standalone partitions or disks; using `/dev/vg0/lv_name` for LVM logical volumes is common, readable, and operationally acceptable.

```fstab
# <fs>                <mountpoint>  <type>  <options>                         <dump> <pass>
UUID=EFI-UUID          /boot/efi     vfat    umask=0077,shortname=winnt        0      2
UUID=BOOT-UUID         /boot         xfs     defaults                          0      2
/dev/vg0/lv_root       /             xfs     defaults                          0      1
/dev/vg0/lv_var        /var          xfs     defaults                          0      2
/dev/vg0/lv_tmp        /tmp          xfs     defaults,nodev,nosuid,noexec      0      2
/dev/vg0/lv_home       /home         xfs     defaults                          0      2
/dev/vg0/lv_swap       none          swap    sw                                0      0
/dev/vg0/lv_data       /data         xfs     defaults,noatime                  0      2
```

On Ubuntu, ext4 is equally reasonable for `/boot`, `/`, `/var`, `/tmp`, `/home`, and `/data` if that is the local standard. On Rocky Linux, XFS remains the conservative default for the main filesystems and for `/data`.[^29]

## Verification

Verify the filesystem layout after provisioning or first boot:

```bash
lsblk -f
findmnt / /var /tmp /home /data /boot /boot/efi
swapon --show
```

Confirm mount options and capacity where it matters:

```bash
findmnt /tmp
findmnt /data
df -h / /var /data
vgs
lvs -a -o +devices
```

Expected characteristics:

- `/tmp` shows `nodev,nosuid,noexec`.
- `/data` shows `noatime`.
- `vg0` still has free extents available for later growth.

## Troubleshooting

If the VM boots into emergency mode after editing `/etc/fstab`, the usual cause is an incorrect UUID, LV path, or filesystem type. Validate fstab entries with `mount -a` before rebooting, and confirm devices and filesystem signatures with `lsblk -f` and `blkid`.

On Rocky Linux, if SELinux is enforcing and a new mount under `/data` will host service-managed content, restore or define the appropriate context after creating directories. A conservative first step is:

```bash
restorecon -RFv /data
```

If `/tmp` is mounted with `noexec`, expect badly behaved installers or ad hoc scripts to fail when they try to execute from `/tmp`. That behavior is intentional; redirect temporary build or installer work to another path if necessary.

## Security Considerations

Keep `nodev`, `nosuid`, and `noexec` on `/tmp` because temporary storage should not behave like a general execution surface.[^16][^17] Do not apply `noexec` to `/home` by default unless there is a specific policy reason and a clear understanding of the operational impact.

The most useful security control in this layout is isolation by function. Separating `/var` and container data from `/` limits the blast radius of log floods, runaway cache growth, and storage abuse from application workloads.

## Notes/Tips

Use swap conservatively on these VMs; 8GB is a good default for 8–16GB RAM unless workload-specific behavior suggests otherwise.[^4] Leave free space in the VG on day one instead of allocating everything immediately, because extending an LV later is easier than reclaiming badly sized filesystems.

For a repeatable homelab pattern, keep the system disk focused on the OS and use `/data` as the only approved landing zone for container persistence. That simplifies monitoring, backup targeting, growth planning, and migration across hypervisors.


<span style="display:none">[^10][^11][^12][^13][^14][^15][^16][^17][^18][^19][^20][^21][^22][^23][^24][^25][^26][^27][^28][^29][^30][^8][^9]</span>

<div align="center">⁂</div>

[^1]: https://docs.docker.com/engine/daemon/

[^2]: https://gist.github.com/plembo/0070059bde27bb8fb37735a899b16e41

[^3]: https://oneuptime.com/blog/post/2026-03-04-set-mount-options-security-nosuid-noexec-nodev-rhel-9/view

[^4]: https://oneuptime.com/blog/post/2026-03-04-noatime-nodiratime-mount-options-performance-rhel-9/view

[^5]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/6/html/installation_guide/s2-diskpartrecommend-x86

[^6]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/storage_administration_guide/ch-xfs

[^7]: https://access.redhat.com/solutions/223643

[^8]: https://forums.rockylinux.org/t/need-to-migrate-all-logical-volumes-from-one-mbr-rocky-9-host-ssd-to-a-new-rocky-9-install-on-another-host-ssd-going-from-mbr-to-uefi-w-xfs/14142

[^9]: https://forums.rockylinux.org/t/recommended-partition-scheme-for-rocky-linux-9-1/9616

[^10]: https://forums.rockylinux.org/t/default-root-partition-size-in-cloud-image-lvm/14823

[^11]: https://support-desk.iij.us/hc/en-us/articles/24252170440987-Extend-RockyLinux9-LVM-with-xfs-partition

[^12]: https://diegocarrasco.com/change-docker-data-directory-vps-optimization/

[^13]: https://www.reddit.com/r/docker/comments/16xg1j1/moving_root_directory_docker/

[^14]: https://www.youtube.com/watch?v=LBrjy4QiHuQ

[^15]: https://community.opendronemap.org/t/how-to-specify-different-root-directory-for-docker-on-linux/12061

[^16]: https://www.ibm.com/docs/en/z-logdata-analytics/5.1.0?topic=software-relocating-docker-root-directory

[^17]: https://forums.rockylinux.org/t/why-is-most-of-my-ssd-lvm/13135

[^18]: https://forums.rockylinux.org/t/cant-boot-after-moving-all-partitions/16052

[^19]: https://stackoverflow.com/questions/43689271/wheres-dockers-daemon-json-missing

[^20]: https://forum.endeavouros.com/t/setting-noexec-nodev-nosuid-mount-parameters-for-home-partition/7618

[^21]: https://forum.openmediavault.org/index.php?thread%2F5959-mounting-tmp-with-nodev-nosuid-noexec%2F

[^22]: https://www.baeldung.com/linux/etc-fstab-mount-options

[^23]: https://www.tenable.com/audits/items/CIS_Ubuntu_18.04_LTS_Server_v2.1.0_L1.audit:e5449fe2177664ccb3fec4c4bd645f75

[^24]: https://github.com/cockroachdb/docs/issues/10630

[^25]: https://forums.whonix.org/t/re-mount-home-and-other-with-noexec-and-nosuid-among-other-useful-mount-options-for-better-security/7707

[^26]: https://www.reddit.com/r/linuxadmin/comments/ekzbi0/should_i_set_security_mount_options_for/

[^27]: https://www.tenable.com/audits/items/CIS_Amazon_Linux_2_STIG_v2.0.0_L1_Server.audit:c63f57da161368c2192076e103a9d820

[^28]: https://bbs.archlinux.org/viewtopic.php?id=79283

[^29]: https://www.facebook.com/groups/linuxh/posts/6901273379934054/

[^30]: https://www.reddit.com/r/sysadmin/comments/1fjamf0/recommended_mount_flags_for_storage_devices_on/
