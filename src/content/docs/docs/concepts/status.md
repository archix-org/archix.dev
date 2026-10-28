---
title: Feature status
description: What archix ships today, what's partial and what isn't built yet, with the tests behind each.
sidebar:
  order: 7
---

This page labels each feature by what archix's code and tests do today, as of archix commit `5deb0e3`:

- **Shipped**: built, and covered by the test suites named next to it.
- **Partial**: built, with a gap the line names.
- **Experimental**: built and tested in a VM or container, but not in the way it's meant to be used: real firmware, a real TPM.
- **Planned**: not built yet.
- **Not planned**: measured or weighed, and turned down for a reason the line gives.

Three kinds of test stand behind the labels. **VM** is a set of QEMU virtual machines with OVMF firmware that boot archix's deployments, each run on ext4 and on a btrfs root laid out as CachyOS installs it. The boot suite lays out a booted install and covers boot entries, booting older deployments, boot counting, Secure Boot, the `/etc` merge and `eject`. The rollback suite updates a real Arch install from the 2026-06-01 snapshot to 2026-10-10, resetting the machine, killing archix and filling the disk along the way. The CachyOS suite does the same with CachyOS's real repos and x86-64-v3 packages, and rebuilds dropped deployments with CachyOS's mirror unreachable. The NVIDIA and dkms2 suites cover DKMS drivers per deployment and per kernel, the pcr suite TPM-bound LUKS with swtpm, the firewall suite ssh through the firewall, the health suite the health check, the selfupgrade suite archix upgrading itself from an older build, and the churn suite 52 weekly switches with a reboot each. After every step, these check every older deployment's files against its package database. **Arch suite** is archix's full test suite, run as root inside a systemd-booted Arch container, with real pacman transactions, AUR builds, network access, and kill tests of the staging and commit logic and of `eject` at every step on loop-mounted ext4 and btrfs, with archix's real ESP step on a FAT ESP. **Power-cut replay** logs every write with `dm-log-writes` and checks the disk replayed to every flush point, booting every point after a write to the ESP and every 20th of the others; it runs on archix's own switch, kept rollback, rebuild of a dropped generation and first boots, after first running on the prototype whose commit order archix follows. It passes on ext4 and btrfs roots on a desktop machine, with no boot retried ([published evidence](https://github.com/archix-org/archix/releases/tag/powercut-evidence-8648816): logs, seed, summary and checksums); on the release gate's nested GitHub VMs it hasn't yet passed, as their boots run 10 to 50 times slower and shards run out of time. On the ESP it also tries reorderings: at each flush, up to 16 seeded random subsets of the writes since the last one, as a power cut may keep any of them; the root's writes are replayed in the order they were logged. Each real boot is checked with `pacman -Qkk`, must have no failed unit, and fails if it takes over three times a clean boot.

All of this is VM-only: QEMU with OVMF firmware, and swtpm for the TPM. archix hasn't yet been run for weeks on real hardware. [Limitations and failure modes](/docs/concepts/limitations/) has the details behind every gap.

## Shipped

| Feature | Status | What it does | Tested in |
| --- | --- | --- | --- |
| [Atomic deployments](/docs/concepts/atomicity/) | **Shipped** | Every switch builds a new deployment beside the running one, by btrfs snapshot or by overlays flattened to hardlinks on ext4, and commits it in one step, designed so a kill or power cut at any point leaves the old or the new one bootable. Kill tests of archix's staging and commit logic and VM power cuts during staging pass, and write-level power-cut replay runs on archix's own switch, rollbacks and first boots; the flaws it found are fixed. | VM, Arch suite, power-cut replay |
| [Lock and switch](/docs/concepts/lockfile/) | **Shipped** | Official packages pinned to a day of the Arch Linux Archive by version and sha256; a switch prepares everything, then builds the next deployment, which boots at the next restart. | VM, Arch suite |
| [Generations](/docs/concepts/generations/) | **Shipped** | Every switch is a numbered generation with its packages, files, users and services; `gc` keeps deployments by `deployments.keep`, and records by count and age. | VM, Arch suite |
| [Rollback from the boot menu](/docs/concepts/generations/#rolling-back-from-the-boot-menu) | **Shipped** | Booting an older entry mounts its deployment exactly, with nothing reinstalled, and it stays current and the default entry. | VM |
| [Boot counting](/docs/reference/options/#boottries) | **Shipped** | With systemd-boot 260 or newer, a deployment that doesn't reach `boot-complete.target` in three boots is passed over for the last one that booted, which becomes current; a counted try whose deployment can't mount or activate reboots by itself. | VM |
| [Health check](/docs/reference/options/#boothealth_check) | **Shipped** | `boot.health_check` holds a new deployment's boot to units, listening ports, reachable hosts and logins, and falls back to the previous deployment when they fail, saying which and when. | VM, Arch suite |
| [Fallback boot image](/docs/concepts/atomicity/#the-fallback-boot-image) | **Shipped** | A copy of the current deployment's UKI in `EFI/Linux` boots it when `loader/entries` is gone. | VM, power-cut replay |
| [Rescue entry](/docs/guides/recovery/#without-a-live-usb-the-rescue-entry) | **Shipped** | An 'archix rescue' boot entry, independent of every deployment, opens `archix rescue` to repair filesystems, repoint current, reset passwords or eject when no deployment boots. | VM, Arch suite |
| [Live-USB recovery](/docs/guides/recovery/#with-a-live-usb) | **Shipped** | `archix mount-deployment` binds a deployment over the mounted root, so `arch-chroot` and `archix rollback` bring back a system whose ESP was wiped. | VM, Arch suite |
| [Rollback at runtime](/docs/concepts/generations/#rolling-back-from-a-running-system) | **Shipped** | `archix rollback` points the boot default at a kept deployment, after checking its files against its packages, or rebuilds a dropped generation from the nearest kept deployment and the package cache, atomically. Users and passwords carry over by name. | VM, Arch suite |
| [`archix verify`](/docs/reference/cli/#verify) | **Shipped** | Checks every kept deployment's `/usr`, `/opt` and `/boot` against its packages, and repairs them from the package cache. | Arch suite |
| [Changes to `/etc` before the reboot](/docs/concepts/safety/#changes-to-etc-before-the-reboot) | **Shipped** | Changes to the running `/etc` after a switch are merged into the waiting deployment within seconds, both-sides changes as `.pacnew`, accounts by name; the boot after a power cut carries what was left. | VM, Arch suite |
| [`/var` and accounts at first boot](/docs/concepts/atomicity/#booting-a-deployment) | **Shipped** | A deployment's first boot applies the `/var` files its switch staged, without overwriting data, after checking account IDs and running sysusers. | VM, Arch suite |
| [DKMS per deployment](/docs/concepts/atomicity/#what-a-deployment-is) | **Shipped** | Each deployment carries its own kernel, modules and DKMS state, so an older entry boots with the driver it was built with. NVIDIA's open and closed drivers are built and installed per deployment in the VM suite, but never loaded: the VMs have no NVIDIA GPU. | VM |
| [An entry per kernel](/docs/concepts/generations/#boot-entries) | **Shipped** | Every installed kernel gets its own UKI and entry, and DKMS builds for each; an older deployment's entries roll every kernel back. | VM |
| [Config written only when a switch commits](/docs/concepts/safety/#declined-and-killed-switches) | **Shipped** | `archix.lock` and `installed.lua` are evaluated from memory and written into the new deployment and the running system as it commits; a kill before that leaves them untouched. | Arch suite |
| [Checks before a switch](/docs/concepts/safety/#signatures-and-disk-space) | **Shipped** | Signatures, expired keys included, free space and inodes, the ESP's filesystem and space, file conflicts and Secure Boot signing are checked before anything is built. | VM (signatures, full disk), Arch suite (file conflicts, inodes, ESP, expired keys) |
| [Keyrings](/docs/concepts/safety/#keyrings) | **Shipped** | Incoming keyring packages are populated into the signature check and installed first; a switch or rollback never takes a keyring back. | VM, Arch suite |
| [AUR builds with review](/docs/guides/aur/) | **Shipped** | Recipes are locked by commit, shown for review, and built in a clean chroot. | Arch suite |
| [Soname rebuilds](/docs/guides/local-pkgbuilds-and-patches/#rebuilds-at-the-same-version) | **Shipped** | A package archix built is rebuilt when a library it links disappears in a switch; a rollback reinstalls the generation's own build. | VM |
| [Secure Boot signing](/docs/guides/secure-boot/) | **Shipped** | Boot entries and systemd-boot are signed with your keys, and a switch that wouldn't be signed is refused, leaving nothing behind. | VM, Arch suite |
| [Managed `/etc` files, users and services](/docs/concepts/safety/#managed-files-backups-and-hand-edits) | **Shipped** | Files are backed up and hand edits saved; users, groups and services follow the config, in the new deployment. | VM (files, services), Arch suite (users) |
| [Disks, mounts and swap](/docs/guides/file-systems/) | **Shipped** | `file_systems`, `swap` and `zram` make each deployment's `/etc/fstab` after archix's own root and ESP lines, `nofail` unless required, with crypttab lines for LUKS volumes, swap files made at boot (on btrfs in a subvolume of their own) and zram through zram-generator; `archix init` captures an existing fstab. | VM, Arch suite |
| [Activation hooks](/docs/guides/hooks/) | **Shipped** | Hooks and `on_change` scripts run once at a deployment's first boot, and again at the next two boots until all succeed, then wait for `archix hooks retry` or `skip`; `on_change` of files outside the deployment runs right after the switch writes them. | Arch suite |
| [Local PKGBUILDs and patches](/docs/guides/local-pkgbuilds-and-patches/) | **Shipped** | Recipes in the config and patched official packages are locked by content and built in the chroot. | VM, Arch suite |
| [Third-party repos](/docs/guides/third-party-repos/) | **Shipped** | Declared repos, CPU-level checks, a copy of every package a generation takes from one, and a fallback chain when a locked package is gone. | VM (CachyOS), Arch suite |
| [Package caches](/docs/concepts/lockfile/#package-caches) | **Shipped** | `archix cache push`, or every switch with `package_cache_push`, fills a directory, shared as a mount or over HTTP, that other machines fetch package files and AUR builds from first, checked against the lock. | Arch suite |
| [Archive mirrors](/docs/reference/options/#archive_mirrors) | **Shipped** | `archive_url` points locking, switches and rollbacks at a mirror of the archive, and `archive_mirrors` lists more, tried when it's down. | VM, Arch suite |
| [Flatpak apps](/docs/guides/flatpak-and-appimages/) | **Shipped** | System-wide apps are locked to commits and pinned back on rollback; apps that come with flatpak itself install at the deployment's first boot. | Arch suite |
| [Dotfiles and home](/docs/guides/dotfiles-and-apps/) | **Shipped** | Home files, written as their user, live links, merged keys and `programs.*` modules. | Arch suite |
| [`archix eject`](/docs/reference/cli/#eject) | **Shipped** | Turns the running deployment into a plain Arch root with a UKI from mkinitcpio's preset, and finishes when run again after a cut. | VM, Arch suite |
| [`archix init`](/docs/quick-start/) | **Shipped** | Captures an existing machine into a config and lock that match it exactly; packages no repo or the AUR has are rebuilt as recipes from their installed files when you pick them. | Arch suite |

## Partial

| Feature | Status | What's missing | Covered in |
| --- | --- | --- | --- |
| [Third-party repo exactness](/docs/concepts/limitations/#locking) | **Partial** | Locked versions are exact only while archix or a package cache has the file; otherwise a fresh machine gets what the repo serves today. | [Third-party repos](/docs/guides/third-party-repos/#why-third-party-versions-can-go-missing) |
| [AUR reproducibility](/docs/concepts/limitations/#locking) | **Partial** | Recipe commits, git sources and `SKIP`-checksum downloads are pinned after the first build; Mercurial, Subversion, Bazaar and Fossil sources aren't, and builds aren't bit-for-bit reproducible. | [The AUR](/docs/guides/aur/#vcs-packages) |
| [Upgrading archix itself](/docs/concepts/safety/#upgrading-archix-itself) | **Partial** | A VM suite upgrades archix from an older build to a newer one and boots both; from one release to the next isn't tested yet, as there's no second release. A new deployment's own archix writes its initramfs hook and boot units. | [Limitations](/docs/concepts/limitations/#not-yet-known) |

## Experimental

| Feature | Status | What it does | Tested in |
| --- | --- | --- | --- |
| [Secure Boot revocation](/docs/guides/secure-boot/#revoking-old-boot-images) | **Experimental** | `archix secure-boot revoke`, or `boot.revoke_unused` after every switch, appends the hashes of UKIs archix signed and no longer uses to `dbx`, signed with your KEK, leaving every other entry. | VM (OVMF only) |
| [Signed TPM PCR policies](/docs/guides/secure-boot/#tpm-bound-secrets) | **Experimental** | With systemd's PCR signing key in place, every UKI carries a PCR 11 policy signed with it; `archix secure-boot rotate-pcr-key` replaces the key and re-binds LUKS volumes. | VM (swtpm, not a real TPM), Arch suite |

## Not planned

| Feature | Status | Why | Covered in |
| --- | --- | --- | --- |
| [GRUB and Limine](/docs/concepts/limitations/#switching) | **Not planned** | archix boots deployments from systemd-boot only. GRUB and Limine boot a kernel from `/boot` under one name every deployment would share, so it couldn't switch together with the rest without archix managing their config. | [Atomic deployments](/docs/concepts/atomicity/#requirements) |
| [Atomic A/B `/usr`](/docs/concepts/atomicity/#why-not-ab-usr) | **Not planned** | Measured: rolls back in milliseconds, but pacman writes `/etc` and `/var` too, so a rollback left missing and orphaned files there and the next upgrade failed. Deployments hold everything a transaction writes instead. | [Atomic deployments](/docs/concepts/atomicity/#why-not-ab-usr) |
