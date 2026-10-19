---
title: Why archix isn't atomic
description: archix switches and rolls back by running pacman, not by swapping in a prebuilt tree. Here is why, with the measurements behind it, and how that relates to snapper and transactional updates.
sidebar:
  order: 8
---

NixOS switches by pointing the system at a complete, prebuilt closure, and rolling back is pointing at the previous one. archix doesn't do that. A switch and a rollback both run pacman, which replaces files in place, package by package. A crash halfway leaves a mix of old and new files until the boot-time repair and [`archix recover`](/docs/concepts/safety/#archix-recover) sort it out.

That's a choice, made after measuring the obvious alternative. This page shows the numbers.

## What was measured

An A/B `/usr` works like this:
1. Clone `/usr`.
2. Have pacman upgrade the clone, against a copy of its database, in a private mount namespace where the clone is `/usr`.
3. Swap the clone and the database copy in with one atomic rename.
4. To roll back, swap them back.

On btrfs the clone was made three ways: with hardlinks, with reflinks, and as a snapshot of `/usr` as a subvolume of its own.

Both that and today's in-place switch ran the real Arch update from the 2026-06-01 snapshot to 2026-10-10:
- **Where:** in the same VM, on ext4 and on btrfs, with cold caches before every step.
- **The update:** 214 packages over a 3 GB `/usr` of base, base-devel, firefox, rust and python.
- **The script:** `tests/vm/atomic` in archix's repository.

| | ext4 | btrfs |
| --- | --- | --- |
| Upgrade in place | 23 s, writes 2.9 GB | 26 s, writes 2.9 GB |
| A/B upgrade | 21 s + a 1 s clone | 26–29 s + a clone of 2.5 s (reflinks) or 16 ms (snapshot) |
| Rollback in place | 23 s, writes 2.8 GB | 25 s, writes 2.8 GB |
| A/B rollback | 3 ms | 4 ms |
| Space taken by the upgrade, in place | 0.3 GB | 2.7 GB (see below) |
| Space taken by the upgrade, A/B | 2.8 GB, until the old tree is dropped | 2.7–2.8 GB |

Timings varied by up to 10 s between runs, so the upgrade times are the same. The rollback is where A/B wins: milliseconds instead of half a minute, and almost nothing written.

## Why archix doesn't do it anyway

pacman doesn't only write to `/usr`. Packages put files in `/etc`, `/var/lib` and `/boot`, and their install scriptlets and hooks act on the running system. Swapping `/usr` doesn't make any of that atomic, and the measurement showed what goes wrong. Every A/B variant, on both filesystems, failed the same way:

- **A rollback can't put back what the upgrade removed outside `/usr`.** The new `shadow` package dropped `/etc/pam.d/groupmems`. pacman deleted it from the live `/etc` while upgrading the clone, and swapping `/usr` back left the old `shadow` without it.
- **A rollback leaves the new packages' files behind.** Nine files outside `/usr` (vapoursynth's profile script and eight tpm2-tss profiles) stayed after the packages that owned them were rolled back, and pacman then refused the next upgrade over them.
- **A kill partway leaves `/etc` half-upgraded,** with new config files next to the old binaries.

Making that correct means staging everything pacman writes outside `/usr` too, and merging `/etc` the way rpm-ostree does. That's a different system, not an option on top of this one. archix instead makes the in-place path recoverable:
- **A journal,** written before anything changes, that `archix recover` finishes or reverses.
- **A boot-time repair** for a switch cut off inside pacman.
- **A rescue entry** that restores pacman's own libraries without pacman.
- **Rollbacks** that reinstall a generation's exact package files.

Those paths are tested by cutting the power, killing runs and filling the disk in a VM; see [Feature status](/docs/concepts/status/).

## btrfs snapshots, and what they cost

On a btrfs root, archix already takes a read-only snapshot of `/` before every switch ([`boot.snapshots`](/docs/reference/options/#bootsnapshots)). They're manual recovery points: rollback never restores one.

They aren't free. A snapshot keeps every file the following switches replace, so the in-place upgrade above took 2.7 GB on btrfs rather than 0.3 GB on ext4. Snapshots go with their generation, so retention follows [`generations.keep`](/docs/reference/options/#generationskeep) and [`generations.keep_days`](/docs/reference/options/#generationskeep_days). With the default of 10 generations, up to ten snapshots each hold what was replaced after them. On a small SSD, lower `generations.keep`, or turn snapshots off.

## snapper, limine-snapper-sync, transactional-update

CachyOS can set up snapper, snap-pac and Limine with `limine-snapper-sync`, which lists snapper's snapshots in the boot menu. openSUSE's transactional-update, and arkdep and frzr on Arch, go further and update into a new snapshot that the next boot starts from. Those give whole-root atomicity on btrfs. archix doesn't, and doesn't try to replace them. It works on a different level:

- **A snapshot rollback** restores everything on the root subvolume, including anything in `/var/lib` that isn't on a subvolume of its own, such as databases.
- **An archix rollback** reinstalls packages and puts back the files archix manages, and leaves your data alone. That works the same on ext4.
- **Which state you get:** snapper's snapshots are whatever the system was at the time. archix's generations are the declared config and the locked package set, on any machine that shares the config.

They can run side by side. snap-pac's pacman hooks fire on the pacman runs a switch makes, so each switch also gets snapper's pre and post snapshots, and both sets of snapshots cost disk space. archix's own snapshots can be turned off with `boot.snapshots = false` if snapper already covers that. This combination isn't covered by archix's test suites yet.

Whole-root transactional updates for archix itself, running pacman inside a snapshot and booting from it, may come after the beta. They start with a design note that has to say why that beats integrating with snapper and Limine. It also has to settle where pacman's database lives: it's in `/var/lib/pacman`, so moving `/var/lib` onto its own subvolume to protect data would take the database out of the snapshot it describes.
