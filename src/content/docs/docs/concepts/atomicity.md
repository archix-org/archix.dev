---
title: Atomic deployments
description: How archix makes every switch and rollback all or nothing, by building each generation as a deployment beside the running one and booting into it, and what that costs.
sidebar:
  order: 8
---

Every `archix switch` builds a complete new system next to the one you're running, a **deployment**, and commits it in one step. The running system doesn't change. At the next boot, the new deployment starts instead of the old one. It's designed so a power cut or a kill at any moment leaves the old deployment or the new one bootable, never a mix of the two, with nothing to repair afterwards; [How it's tested](#how-its-tested) says how far that's proven.

This page explains how that works, what it costs in disk space, and how it's tested.

## What a deployment is

A deployment is the part of the system a package transaction writes, kept per generation under `/archix/deploy/<n>`:

| Tree | What it holds | Shared with other deployments |
| --- | --- | --- |
| `usr`, `opt` | Programs, libraries, kernel modules | Unchanged files are shared: by copy-on-write on btrfs, as hardlinks on ext4 |
| `boot` | The kernel, microcode and what packages put in `/boot` | The same way |
| `etc` | System configuration | A real copy on ext4, copy-on-write on btrfs |
| `pacman` | pacman's local database | A real copy on ext4, copy-on-write on btrfs |
| DKMS state | `/var/lib/dkms` | A real copy on ext4, copy-on-write on btrfs |

`/archix/current` points at the deployment the next boot starts. Everything else stays on the root filesystem as one copy shared by every deployment: `/home`, `/root`, `/srv`, and `/var` with its databases, logs and caches. That's the state you keep across a rollback.

`/etc` and pacman's database are real copies on ext4, a few MB each, because a hardlink would share edits: an editor that writes in place, or `chmod`, would change the file in every deployment at once, including the one you'd fall back to.

## A switch

1. **Prepare.** As before: evaluate the config, lock, download and check every package, build AUR packages, check signatures, conflicts and disk space. Nothing on the system changes. See [Safety](/docs/concepts/safety/#prepare-then-build).
2. **Stage** `/archix/deploy/.staging-<n>` from the current deployment. On btrfs that's a snapshot. On ext4, `usr`, `opt` and `boot` are overlays with the current deployment as the read-only lower layer, and `etc`, the database and DKMS state are copies. `/var` is an overlay over a private copy of what the current deployment's packages own in `/var`, never the live `/var`. Before staging, the switch also takes a copy of the running `/etc` as the base for the [`/etc` merge](#what-takes-effect-when).
3. **Build it.** pacman installs and removes packages inside the staging tree, in a private mount and PID namespace that sees nothing of the running system but what it needs; sbctl's Secure Boot keys and systemd's PCR signing key are hidden from it, and only `ukify` sees them, to sign. Install scriptlets and pacman hooks run there too. archix writes its managed `/etc` files, users, groups and enabled services into the staged `/etc`, then mkinitcpio and ukify build the new initramfs and boot image from the staged kernel.
4. **Flatten**, on ext4: unchanged files become hardlinks to the previous deployment's, and the overlay's changes are moved in, so the result is a plain directory tree.
5. **Commit**, in this order: flush the root filesystem; rename `.staging-<n>` to `<n>`; copy the boot images to the ESP and write the boot entries; point systemd-boot's `loader.conf` at the new one; copy its boot image to the [fallback image](#the-fallback-boot-image); repoint `/archix/current`. Every flush is checked, and nothing is renamed or pointed after one fails.

The order is what makes it atomic. A boot entry or pointer that survives a crash always names a complete deployment. The ESP is FAT, where a rename isn't crash-safe: a torn `loader.conf` or a missing entry makes systemd-boot start another complete deployment, and a torn `loader/entries` directory, which can lose every entry at once, leaves the fallback image. A staging tree that was never renamed is deleted by the next switch.

Once it has committed, the switch starts the [`/etc` merge](/docs/concepts/safety/#changes-to-etc-before-the-reboot), and turns hibernation off until the reboot: resuming the running system's memory image under the new deployment's kernel would mix the two, so `/run/systemd/sleep.conf.d/zz-archix-pending-deployment.conf` sets `AllowHibernation`, `AllowSuspendThenHibernate` and `AllowHybridSleep` to `no`. Suspend still works, and the file goes with the reboot.

### The fallback boot image

Besides each deployment's own entries, the ESP holds one more copy of the current deployment's boot image, `EFI/Linux/archix-fallback-<n>.efi`. systemd-boot lists a UKI in `EFI/Linux` by itself, without an entry file, as "archix (fallback)", and sorts it so it starts when `loader.conf` names no entry it has, as when `loader/entries` is gone. It boots the current deployment. Every switch and rollback, and a boot that makes an older deployment current, point it at the new `current`; the older copy is removed once the new one is on the ESP. The copy is the main kernel's signed image, so it costs one more image's space on the ESP; see [Storage](#storage).

Then the switch ends: `switched to generation 43 (snapshot 2026-10-15); its deployment boots at the next restart`.

## Booting a deployment

archix's mkinitcpio hook, in systemd and busybox initramfs images alike, mounts the deployment before the real root starts. It reads which boot entry systemd-boot started from the `LoaderEntrySelected` EFI variable, and boots that entry's deployment if it's committed, complete, and the entry is one archix recorded writing. Otherwise it boots `current`. It binds the deployment's `usr`, `opt` and `boot` read-only over the root, and its `etc`, pacman database and DKMS state writable. `/archix` itself is bound read-only, so no deployment, the ones you'd roll back to included, can be changed through it; only archix, in a mount namespace of its own, writes there.

So picking an older entry in the boot menu boots that deployment with the same packages, kernel and `/etc` it had; `/var` and your home are shared and stay as they are now. Nothing is reinstalled, and pacman doesn't run at boot. `archix-boot.service` then makes it current and the boot default, and carries the users and passwords of the deployment that was current into it by name; see [Users and passwords across deployments](/docs/concepts/safety/#users-and-passwords-across-deployments). With systemd-boot 260 or newer, [boot counting](/docs/reference/options/#boottries) does the same by itself when a new deployment doesn't reach `boot-complete.target`.

When the initramfs can't mount the deployment, or its first-boot steps fail, on one of a new deployment's counted tries, archix waits 10 seconds and reboots, so systemd-boot tries again or falls back without anyone at the machine: `archix: mounting this boot's deployment failed (...) on one of a new deployment's counted boot tries; rebooting in 10 seconds so systemd-boot tries it again or falls back to the previous deployment`. On a boot that isn't counted, a reboot would loop, so it stops at a shell or the emergency target instead.

A deployment's first boot applies what its switch staged for `/var`, before services start:
- **`/etc`.** If the previous deployment's last [`/etc` merge](/docs/concepts/safety/#changes-to-etc-before-the-reboot) pass didn't finish, after a power cut or a hang, `archix-etc-merge-boot.service` carries what its `/etc` held on disk first.
- **Accounts.** Package users and groups got their IDs while staging, against the staged `/etc`. A check for duplicate IDs runs first, then `systemd-sysusers`. If `systemd-sysusers` fails, the `/var` changes aren't applied and services don't start; the next boot tries again, and after three boots the deployment boots without the users it couldn't create, with a note at your next login.
- **`/var`.** Files the transaction wrote under `/var` are copied in. A path that's already there live is replaced only when a package in the new deployment owns it, so your data wins. A file a package lists as `backup` follows pacman's rule: if you changed it since the version the deployment was staged over, it stays, with the package's version beside it as `.pacnew`: `archix: /var/lib/foo/foo.conf was changed on this machine, so it stays; the package's version is beside it as .pacnew`. Deletions are never applied. A power cut partway leaves a marker, and the next boot applies them again from the start.
- **Hooks.** [`on_change` scripts and hooks](/docs/guides/hooks/) run, before the network is up.

A second switch before the reboot builds on the waiting deployment and carries its `/var` changes along, so they're applied at the first boot of the newer one.

## What takes effect when

Everything in the deployment takes effect at the next boot: packages, `/etc` files, users and groups, enabled and disabled services, sysctl, the firewall, the kernel and its modules. Nothing is restarted or reloaded on the running system, so it keeps running the old generation, consistently, until you reboot.

Some things live outside the deployments, and a switch writes them once it has committed: [`home.files`](/docs/reference/options/#homefiles) and other managed files outside `/etc`, `/usr` and `/opt`, with their `on_change` scripts, the home directories of new users, and [Flatpak apps](/docs/guides/flatpak-and-appimages/).

Changes you make to `/etc` after a switch and before the reboot are carried into the new deployment's own copy of `/etc` as you make them: a password you change with `passwd`, a Wi-Fi network you save, a file you edit. `archix-etc-merge.service` watches the running `/etc` from the commit until shutdown and merges each change into the waiting deployment, with the deployment's own version of a file both changed kept beside it as `.pacnew`. `switch` and `archix status` say when a deployment is waiting: `deployment 43 is committed and boots at the next restart; changes to /etc made before then are carried over to it`. See [Changes to `/etc` before the reboot](/docs/concepts/safety/#changes-to-etc-before-the-reboot).

`archix status`, and a note at every login, list the updates waiting for a reboot, and name the ones that matter for security: `4 updates are waiting for a reboot (deployment 2): bc removed, openssl 3.5.0-1 -> 3.5.1-1, tree 2.1-1 -> 2.2-1, linux 6.17-1; security-relevant, so reboot soon: openssl, linux`. Kernels, `glibc`, `openssl`, `openssh`, `systemd` and `sudo` count.

## Rollback

- **To a generation whose deployment is kept**, rolling back first checks the deployment's files against its packages, as [`archix verify`](/docs/reference/cli/#verify) does, and repairs what differs from the package cache, then points the boot default and `current` at it. The next boot starts it. Picking its entry in the boot menu does the same, without the check.
- **To a generation whose deployment was dropped**, archix builds it again from the nearest kept deployment and the generation's package files in the cache, then commits it like a switch. That takes as long as an update, and is just as atomic.

[`deployments.keep`](/docs/reference/options/#deploymentskeep) decides how many deployments stay, 2 by default; [`generations.keep`](/docs/reference/options/#generationskeep) how many generations stay rebuildable. A rollback leaves data in `/var` and your home as it is: a database a newer version migrated stays migrated. See [Generations and rollback](/docs/concepts/generations/).

## Storage

A kept deployment costs the files that differ from the next one, plus a few MB for `/etc` and pacman's database. Measured on a four-month update of 214 packages over a 3 GB `/usr`, and a one-week update of 36 packages:

| | Four months | One week |
| --- | --- | --- |
| `deployments.keep = 1` | the same as updating in place | the same as updating in place |
| `deployments.keep = 2`, ext4 | +2.9 GB | +0.9 GB |
| `deployments.keep = 2`, btrfs | +2.9 GB | +0.86 GB |
| `deployments.keep = 2`, btrfs with zstd | +1.7 GB | +0.59 GB |

Each kept deployment also has its boot image on the ESP, up to about 40 MB, one per installed kernel, and a second with [`boot.protected_entry`](/docs/reference/options/#bootprotected_entry); deployments whose kernel and initramfs are the same share one. The [fallback image](#the-fallback-boot-image) is one more copy of the current deployment's. The package cache archix keeps for rebuilding dropped generations comes on top, as before.

The ESP checks count all of them, and a switch also needs room for the new deployment's images while the booted and current ones stay. A 1 GiB ESP, as the Arch installation guide suggests, holds that; on a smaller one, such as 512 MiB with protected entries, even a `deployments.keep = 1` switch can be refused for space: ``the ESP at /efi has 152 MiB free (0 MiB more after dropping the deployments it may), but this switch's boot entries need about 210 MiB; nothing was changed. Free up space on it, or set deployments.keep lower``.

With `keep = 1`, the previous deployment stays only until the new one has booted, and goes at the next switch or `archix gc`. Older generations are then rebuilt from the cache.

## Requirements

The first switch lays the running system out as deployment 1, and refuses before changing anything when it can't:

- archix needs to manage booting, with [`boot.loader = "systemd-boot"`](/docs/reference/options/#bootloader). GRUB and Limine boot a kernel from `/boot` under a name every deployment shares, which can't be switched together with the rest.
- The ESP must not be mounted in `/boot`, since each deployment has its own `/boot`. Mount it at `/efi`, the default for [`boot.esp`](/docs/reference/options/#bootesp). Every switch, rollback and `gc` checks that a FAT filesystem is mounted there and, when systemd-boot reports it, that it's the partition systemd-boot started from. New deployments' `fstab` mounts the ESP on access and unmounts it after a minute idle, so a forced reboot rarely catches it mounted; when the config manages `/etc/fstab`, the switch warns instead.
- `/boot`, `/usr`, `/opt`, `/var`, `/var/lib`, `/var/lib/pacman` and `/archix` must be on the root filesystem. Caches and logs on their own btrfs subvolumes, such as CachyOS's `@cache` and `@log`, are fine.
- The static archix at `/usr/lib/archix/archix-static`, which the AUR package installs, goes into the initramfs.

`/usr`, `/opt` and `/boot` are read-only from then on, so plain pacman can't change the running system; see [pacman and yay](/docs/concepts/pacman-and-yay/).

## How it's tested

- **Killed switches.** Container tests kill archix's staging and commit logic at every step, on loop-mounted ext4 and btrfs, with archix's real ESP step on a FAT ESP, and check that only complete deployments are named and the next switch cleans up. `eject` is killed at every step the same way.
- **Power cuts.** The design was first tested with `dm-log-writes` on the prototype whose commit order archix follows: every write and flush of a switch was logged, and the disk replayed to every flush point, and to 60 evenly spaced points through the four-month update on ext4. Each replayed state was checked with `fsck`, booted, and had to start a complete old or new deployment with `pacman -Qkk` clean. The same replay now runs on archix's own switch, kept rollback, rebuild of a dropped generation and first boots, on a FAT ESP and an ext4 or btrfs root: every entry has to name a committed deployment whose boot image matches, and every ESP point and a sample of the others boot for real. The flaws it found are fixed: a boot image or entry left empty by FAT's rename, records of entries that never landed, a hook's output lost during a first boot. In the VM suites, archix's own switches are also reset during staging and inside the `/etc` merge's window.
- **VM suites**, on ext4 and on btrfs laid out as CachyOS does: booting an older entry, rollback by repointing and by rebuilding, Secure Boot, boot counting, the health check, a full disk, a broken glibc in a new deployment, NVIDIA's drivers per deployment and kernel, two kernels with a DKMS module, a TPM-bound LUKS volume, the firewall over ssh, `/etc` changes carried before a reboot, `eject`, archix upgrading itself from an older build, 52 weekly switches with a reboot each, and CachyOS's own packages with its mirror unreachable. After every step they check every older deployment's files against its package database.

See [Feature status](/docs/concepts/status/).

## Why not A/B `/usr`

Before deployments, archix switched in place: pacman replaced files package by package, a journal repaired a switch that was cut off, and rollback reinstalled older packages. An A/B `/usr`, cloned, upgraded and swapped in with one rename, was measured against that on the same four-month update. It rolled back in milliseconds instead of 25 seconds. But pacman writes more than `/usr`: a rollback lost a file the upgrade had removed from `/etc`, left nine files outside `/usr` that broke the next upgrade, and a kill left `/etc` half-upgraded. The unit that has to switch atomically is everything a package transaction writes, which is what a deployment holds.

## snapper and transactional updates

openSUSE's transactional-update, and arkdep and frzr on Arch, also update into a new tree that the next boot starts from. They snapshot the whole root on btrfs, so a rollback also restores `/var/lib` data on the root subvolume. archix's deployments leave `/var` data out, and work the same on ext4.

archix no longer takes btrfs snapshots of its own; the deployments replace them. snapper can still run beside it, with its snapshots costing space on top of the deployments, but that combination isn't covered by archix's test suites.
