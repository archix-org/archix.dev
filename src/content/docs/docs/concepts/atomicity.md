---
title: Atomic deployments
description: How archix makes every switch and rollback all or nothing, by building each generation as a deployment beside the running one and booting into it, and what that costs.
sidebar:
  order: 8
---

Every `archix switch` builds a complete new system next to the one you're running, a **deployment**, and commits it in one step. The running system doesn't change. At the next boot, the new deployment starts instead of the old one. A power cut or a kill at any moment leaves the old deployment or the new one bootable, never a mix of the two, and there's nothing to repair afterwards.

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
2. **Stage** `/archix/deploy/.staging-<n>` from the current deployment. On btrfs that's a snapshot. On ext4, `usr`, `opt` and `boot` are overlays with the current deployment as the read-only lower layer, and `etc`, the database and DKMS state are copies. `/var` is an overlay over the live `/var`.
3. **Build it.** pacman installs and removes packages inside the staging tree, in a private mount and PID namespace that sees nothing of the running system but what it needs. Install scriptlets and pacman hooks run there too. archix writes its managed `/etc` files, users, groups and enabled services into the staged `/etc`, then mkinitcpio and ukify build the new initramfs and boot image from the staged kernel.
4. **Flatten**, on ext4: unchanged files become hardlinks to the previous deployment's, and the overlay's changes are moved in, so the result is a plain directory tree.
5. **Commit**, in this order: flush the root filesystem; rename `.staging-<n>` to `<n>`; copy the boot image to the ESP and write the boot entry; point systemd-boot's `loader.conf` at it; repoint `/archix/current`. Every flush is checked, and nothing is renamed or pointed after one fails.

The order is what makes it atomic. A boot entry or pointer that survives a crash always names a complete deployment. The ESP is FAT, where a rename isn't crash-safe, so a torn `loader.conf` or a missing entry can only make systemd-boot start another complete deployment. A staging tree that was never renamed is deleted by the next switch.

Then the switch ends: `switched to generation 43 (snapshot 2026-10-15); its deployment boots at the next restart`.

## Booting a deployment

archix's mkinitcpio hook, in systemd and busybox initramfs images alike, mounts the deployment before the real root starts. It reads which boot entry systemd-boot started from the `LoaderEntrySelected` EFI variable, and boots that entry's deployment if it's committed, complete, and the entry is one archix recorded writing. Otherwise it boots `current`. It binds the deployment's `usr`, `opt` and `boot` read-only over the root, and its `etc`, pacman database and DKMS state writable.

So picking an older entry in the boot menu boots that deployment exactly as it was. Nothing is reinstalled, and pacman doesn't run at boot. `archix-boot.service` then makes it current and the boot default. With systemd-boot 260 or newer, [boot counting](/docs/reference/options/#boottries) does the same by itself when a new deployment doesn't reach `boot-complete.target`.

A deployment's first boot applies what its switch staged for `/var`, before services start:
- **Accounts.** Package users and groups got their IDs while staging, against the staged `/etc`. A check for duplicate IDs runs first, then `systemd-sysusers`.
- **`/var`.** Files the transaction wrote under `/var` are copied in. A path that's already there live is replaced only when a package in the new deployment owns it, so your data wins. Deletions are never applied.
- **Hooks.** [`on_change` scripts and hooks](/docs/guides/hooks/) run, before the network is up.

## What takes effect when

Everything in the deployment takes effect at the next boot: packages, `/etc` files, users and groups, enabled and disabled services, sysctl, the firewall, the kernel and its modules. Nothing is restarted or reloaded on the running system, so it keeps running the old generation, consistently, until you reboot.

Some things live outside the deployments, and a switch writes them once it has committed: [`home.files`](/docs/reference/options/#homefiles) and other managed files outside `/etc`, `/usr` and `/opt`, the home directories of new users, and [Flatpak apps](/docs/guides/flatpak-and-appimages/).

Changes you make to `/etc` after a switch and before the reboot don't carry over to the new deployment, since it has its own copy of `/etc`: a password you change with `passwd`, a Wi-Fi network you save, a file you edit. archix's own writes to its config, `archix.lock` and `installed.lua`, do reach it. `switch` and `archix status` say when a deployment is waiting: `deployment 43 is committed and boots at the next restart; changes to /etc made before then don't carry over to it, except archix's own config`. Reboot before you change `/etc`. Carrying such changes over continuously is planned.

## Rollback

- **To a generation whose deployment is kept**, rolling back points the boot default and `current` at it. It's instant, and the next boot starts it. Picking its entry in the boot menu does the same.
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

Each kept deployment also has its boot image on the ESP, up to about 40 MB; deployments whose kernel and initramfs are the same share one. The package cache archix keeps for rebuilding dropped generations comes on top, as before.

With `keep = 1`, the previous deployment stays only until the new one has booted, and goes at the next switch or `archix gc`. Older generations are then rebuilt from the cache.

## Requirements

The first switch lays the running system out as deployment 1, and refuses before changing anything when it can't:

- archix needs to manage booting, with [`boot.loader = "systemd-boot"`](/docs/reference/options/#bootloader). GRUB and Limine boot a kernel from `/boot` under a name every deployment shares, which can't be switched together with the rest.
- The ESP must not be mounted in `/boot`, since each deployment has its own `/boot`. Mount it at `/efi`, the default for [`boot.esp`](/docs/reference/options/#bootesp).
- `/boot`, `/usr`, `/opt`, `/var`, `/var/lib`, `/var/lib/pacman` and `/archix` must be on the root filesystem. Caches and logs on their own btrfs subvolumes, such as CachyOS's `@cache` and `@log`, are fine.
- The static archix at `/usr/lib/archix/archix-static`, which the AUR package installs, goes into the initramfs.

`/usr`, `/opt` and `/boot` are read-only from then on, so plain pacman can't change the running system; see [pacman and yay](/docs/concepts/pacman-and-yay/).

## How it's tested

- **Killed switches.** Container tests kill archix at every step of a switch and of its commit, on loop-mounted ext4 and btrfs, and check that only complete deployments are named and the next switch cleans up.
- **Power cuts.** The design was tested with `dm-log-writes`: every write and flush of a switch was logged, and the disk replayed to every flush point, and to 60 evenly spaced points through the four-month update on ext4. Each replayed state was checked with `fsck`, booted, and had to start a complete old or new deployment with `pacman -Qkk` clean. That ran on the prototype whose commit order archix follows. archix's own switches are reset mid-build and between a rollback's two writes in the VM suites.
- **VM suites**, on ext4 and on btrfs laid out as CachyOS does: booting an older entry, rollback by repointing and by rebuilding, Secure Boot, boot counting, a full disk, a broken glibc in a new deployment, NVIDIA's drivers per deployment and kernel, and CachyOS's own packages with its mirror unreachable. After every step they check every older deployment's files against its package database.

See [Feature status](/docs/concepts/status/).

## Why not A/B `/usr`

Before deployments, archix switched in place: pacman replaced files package by package, a journal repaired a switch that was cut off, and rollback reinstalled older packages. An A/B `/usr`, cloned, upgraded and swapped in with one rename, was measured against that on the same four-month update. It rolled back in milliseconds instead of 25 seconds. But pacman writes more than `/usr`: a rollback lost a file the upgrade had removed from `/etc`, left nine files outside `/usr` that broke the next upgrade, and a kill left `/etc` half-upgraded. The unit that has to switch atomically is everything a package transaction writes, which is what a deployment holds.

## snapper and transactional updates

openSUSE's transactional-update, and arkdep and frzr on Arch, also update into a new tree that the next boot starts from. They snapshot the whole root on btrfs, so a rollback also restores `/var/lib` data on the root subvolume. archix's deployments leave `/var` data out, and work the same on ext4.

archix no longer takes btrfs snapshots of its own; the deployments replace them. snapper can still run beside it, with its snapshots costing space on top of the deployments, but that combination isn't covered by archix's test suites.
