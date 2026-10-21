---
title: Feature status
description: What archix ships today, what's partial and what isn't built yet, with the tests behind each.
sidebar:
  order: 7
---

This page labels each feature by what archix's code and tests do today, as of archix commit `8b5e81f`:

- **Shipped**: built, and covered by the test suites named next to it.
- **Partial**: built, with a gap the line names.
- **Experimental**: built and tested in a VM or container, but not in the way it's meant to be used: real firmware, a real TPM.
- **Planned**: not built yet.
- **Not planned**: measured or weighed, and turned down for a reason the line gives.

Three kinds of test stand behind the labels. **VM** is a set of QEMU virtual machines with OVMF firmware that boot archix's deployments, each run on ext4 and on a btrfs root laid out as CachyOS installs it. The boot suite, which runs on every push, lays out a booted install and covers boot entries, booting older deployments, boot counting and Secure Boot. The rollback suite updates a real Arch install from the 2026-06-01 snapshot to 2026-10-10, resetting the machine, killing archix and filling the disk along the way. The CachyOS suite does the same with CachyOS's real repos and x86-64-v3 packages, and rebuilds dropped deployments with CachyOS's mirror unreachable, and the NVIDIA suite covers DKMS drivers per deployment. After every step, these check every older deployment's files against its package database. **Arch suite** is archix's full test suite, run as root inside a systemd-booted Arch container, with real pacman transactions, AUR builds, network access, and switches killed at every step on loop-mounted ext4 and btrfs. **Power-cut replay** logs every write of a switch with `dm-log-writes` and boots the disk replayed to every flush point; it ran on the prototype whose commit order archix follows.

archix hasn't yet been run for weeks on real hardware. [Limitations and failure modes](/docs/concepts/limitations/) has the details behind every gap.

## Shipped

| Feature | Status | What it does | Tested in |
| --- | --- | --- | --- |
| [Atomic deployments](/docs/concepts/atomicity/) | **Shipped** | Every switch builds a new deployment beside the running one, by btrfs snapshot or by overlays flattened to hardlinks on ext4, and commits it in one step; a kill or power cut at any point leaves the old or the new one bootable, with nothing to repair. | VM, Arch suite, power-cut replay |
| [Lock and switch](/docs/concepts/lockfile/) | **Shipped** | Official packages pinned to a day of the Arch Linux Archive by version and sha256; a switch prepares everything, then builds the next deployment, which boots at the next restart. | VM, Arch suite |
| [Generations](/docs/concepts/generations/) | **Shipped** | Every switch is a numbered generation with its packages, files, users and services; `gc` keeps deployments by `deployments.keep`, and records by count and age. | VM, Arch suite |
| [Rollback from the boot menu](/docs/concepts/generations/#rolling-back-from-the-boot-menu) | **Shipped** | Booting an older entry mounts its deployment exactly, with nothing reinstalled, and it stays current and the default entry. | VM |
| [Boot counting](/docs/reference/options/#boottries) | **Shipped** | With systemd-boot 260 or newer, a deployment that doesn't reach `boot-complete.target` in three boots is passed over for the last one that booted, which becomes current. | VM |
| [Rollback at runtime](/docs/concepts/generations/#rolling-back-from-a-running-system) | **Shipped** | `archix rollback` points the boot default at a kept deployment, or rebuilds a dropped generation from the nearest kept deployment and the package cache, atomically. | VM, Arch suite |
| [`/var` and accounts at first boot](/docs/concepts/atomicity/#booting-a-deployment) | **Shipped** | A deployment's first boot applies the `/var` files its switch staged, without overwriting data, after checking account IDs and running sysusers. | VM, Arch suite |
| [DKMS per deployment](/docs/concepts/atomicity/#what-a-deployment-is) | **Shipped** | Each deployment carries its own kernel, modules and DKMS state, so an older entry boots with the driver it was built with. Tested with NVIDIA's open and closed drivers. | VM |
| [Config written only when a switch commits](/docs/concepts/safety/#declined-and-killed-switches) | **Shipped** | `archix.lock` and `installed.lua` are evaluated from memory and written into the new deployment and the running system as it commits; a kill before that leaves them untouched. | Arch suite |
| [Checks before a switch](/docs/concepts/safety/#signatures-and-disk-space) | **Shipped** | Signatures, free space, file conflicts and Secure Boot signing are checked before anything is built. | VM (signatures, full disk), Arch suite (file conflicts) |
| [Keyrings](/docs/concepts/safety/#keyrings) | **Shipped** | Incoming keyring packages are populated into the signature check and installed first; a switch or rollback never takes a keyring back. | VM, Arch suite |
| [AUR builds with review](/docs/guides/aur/) | **Shipped** | Recipes are locked by commit, shown for review, and built in a clean chroot. | Arch suite |
| [Soname rebuilds](/docs/guides/local-pkgbuilds-and-patches/#rebuilds-at-the-same-version) | **Shipped** | A package archix built is rebuilt when a library it links disappears in a switch; a rollback reinstalls the generation's own build. | VM |
| [Secure Boot signing](/docs/guides/secure-boot/) | **Shipped** | Boot entries and systemd-boot are signed with your keys, and a switch that wouldn't be signed is refused, leaving nothing behind. | VM, Arch suite |
| [Managed `/etc` files, users and services](/docs/concepts/safety/#managed-files-backups-and-hand-edits) | **Shipped** | Files are backed up and hand edits saved; users, groups and services follow the config, in the new deployment. | VM (files, services), Arch suite (users) |
| [Activation hooks](/docs/guides/hooks/) | **Shipped** | Hooks and `on_change` scripts run once at a deployment's first boot, and again at the next boot until all succeed. | Arch suite |
| [Local PKGBUILDs and patches](/docs/guides/local-pkgbuilds-and-patches/) | **Shipped** | Recipes in the config and patched official packages are locked by content and built in the chroot. | VM, Arch suite |
| [Third-party repos](/docs/guides/third-party-repos/) | **Shipped** | Declared repos, CPU-level checks, a copy of every package a generation takes from one, and a fallback chain when a locked package is gone. | VM (CachyOS), Arch suite |
| [Package caches](/docs/concepts/lockfile/#package-caches) | **Shipped** | `archix cache push`, or every switch with `package_cache_push`, fills a directory, shared as a mount or over HTTP, that other machines fetch package files and AUR builds from first, checked against the lock. | Arch suite |
| [Archive mirrors](/docs/reference/options/#archive_mirrors) | **Shipped** | `archive_url` points locking, switches and rollbacks at a mirror of the archive, and `archive_mirrors` lists more, tried when it's down. | VM, Arch suite |
| [Flatpak apps](/docs/guides/flatpak-and-appimages/) | **Shipped** | System-wide apps are locked to commits and pinned back on rollback. | Arch suite |
| [Dotfiles and home](/docs/guides/dotfiles-and-apps/) | **Shipped** | Home files, live links, merged keys and `programs.*` modules. | Arch suite |
| [`archix init`](/docs/quick-start/) | **Shipped** | Captures an existing machine into a config and lock that match it exactly; packages no repo or the AUR has are rebuilt as recipes from their installed files when you pick them. | Arch suite |

## Partial

| Feature | Status | What's missing | Covered in |
| --- | --- | --- | --- |
| [Changes to `/etc` before the reboot](/docs/concepts/safety/#changes-to-etc-before-the-reboot) | **Partial** | archix's own config writes reach a waiting deployment; other changes to `/etc` made between a switch and the reboot don't carry over. | [Limitations](/docs/concepts/limitations/#switching) |
| [Third-party repo exactness](/docs/concepts/limitations/#locking) | **Partial** | Locked versions are exact only while archix or a package cache has the file; otherwise a fresh machine gets what the repo serves today. | [Third-party repos](/docs/guides/third-party-repos/#why-third-party-versions-can-go-missing) |
| [AUR reproducibility](/docs/concepts/limitations/#locking) | **Partial** | Recipe commits, git sources and `SKIP`-checksum downloads are pinned after the first build; Mercurial, Subversion, Bazaar and Fossil sources aren't, and builds aren't bit-for-bit reproducible. | [The AUR](/docs/guides/aur/#vcs-packages) |
| [Upgrading archix itself](/docs/concepts/safety/#upgrading-archix-itself) | **Partial** | Upgrading from one release with deployments to the next isn't tested yet: there's no second release. A new deployment's own archix writes its initramfs hook and boot units. | [Limitations](/docs/concepts/limitations/#not-yet-known) |
| [`archix eject`](/docs/reference/cli/#eject) | **Partial** | Hands booting back to plain systemd-boot, but only before the first switch; on a system laid out as deployments it refuses. | [Limitations](/docs/concepts/limitations/#switching) |

## Experimental

| Feature | Status | What it does | Tested in |
| --- | --- | --- | --- |
| [Secure Boot revocation](/docs/guides/secure-boot/#revoking-old-boot-images) | **Experimental** | `archix secure-boot revoke`, or `boot.revoke_unused` after every switch, appends the hashes of UKIs archix signed and no longer uses to `dbx`, signed with your KEK, leaving every other entry. | VM (OVMF only) |
| [Signed TPM PCR policies](/docs/guides/secure-boot/#tpm-bound-secrets) | **Experimental** | With systemd's PCR signing key in place, every UKI carries a PCR 11 policy signed with it. | Arch suite (the policy is in the UKI; nothing unlocks a secret with a TPM) |

## Planned

| Feature | Status | What it would do | Covered in |
| --- | --- | --- | --- |
| [Merging `/etc` into a waiting deployment](/docs/concepts/safety/#changes-to-etc-before-the-reboot) | **Planned** | Carry changes made to the running `/etc` after a switch into the deployment that boots next, as they happen. | [Limitations](/docs/concepts/limitations/#switching) |

## Not planned

| Feature | Status | Why | Covered in |
| --- | --- | --- | --- |
| [GRUB and Limine](/docs/concepts/limitations/#switching) | **Not planned** | archix boots deployments from systemd-boot only. GRUB and Limine boot a kernel from `/boot` under one name every deployment would share, so it couldn't switch together with the rest without archix managing their config. | [Atomic deployments](/docs/concepts/atomicity/#requirements) |
| [Atomic A/B `/usr`](/docs/concepts/atomicity/#why-not-ab-usr) | **Not planned** | Measured: rolls back in milliseconds, but pacman writes `/etc` and `/var` too, so a rollback left missing and orphaned files there and the next upgrade failed. Deployments hold everything a transaction writes instead. | [Atomic deployments](/docs/concepts/atomicity/#why-not-ab-usr) |
