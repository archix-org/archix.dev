---
title: Feature status
description: What archix ships today, what's partial and what isn't built yet, with the tests behind each.
sidebar:
  order: 7
---

This page labels each feature by what archix's code and tests do today, as of archix commit `184a721`:

- **Shipped**: built, and covered by the test suites named next to it.
- **Partial**: built, with a gap the line names.
- **Planned**: not built yet.

Two test suites stand behind the labels. **VM** is a QEMU virtual machine with OVMF firmware that boots archix-managed generations: one run walks through boot-menu rollback, Secure Boot and rescue entries, and a rollback torture run, on ext4 and on a btrfs root laid out as CachyOS installs it, updates a real Arch install from the 2026-06-01 snapshot to 2026-10-10, cutting the power, killing runs and filling the disk along the way. The same torture run also goes through a CachyOS install with CachyOS's real repos and x86-64-v3 packages, where its later rollbacks have to reinstall CachyOS packages with CachyOS's mirror unreachable. Others cover NVIDIA's DKMS drivers, archix upgrading itself from its previous release, and `archix eject`. **Arch suite** is archix's full test suite, run as root inside a systemd-booted Arch container, with real pacman transactions, AUR builds and network access.

archix hasn't yet been run for weeks on real hardware. [Limitations and failure modes](/docs/concepts/limitations/) has the details behind every gap.

## Shipped

| Feature | Status | What it does | Tested in |
| --- | --- | --- | --- |
| [Lock and switch](/docs/concepts/lockfile/) | **Shipped** | Official packages pinned to a day of the Arch Linux Archive by version and sha256; a switch prepares everything, then applies from a journal. | VM, Arch suite |
| [Generations](/docs/concepts/generations/) | **Shipped** | Every switch is a numbered generation with its packages, files, users and services; `gc` keeps them by count and age. | VM, Arch suite |
| [Rollback from the boot menu](/docs/concepts/generations/#rolling-back-from-the-boot-menu) | **Shipped** | Booting an older entry rolls back to it early in boot, and it stays the default entry. | VM |
| [Boot counting](/docs/reference/options/#boottries) | **Shipped** | With systemd-boot 260 or newer, a generation that doesn't reach `boot-complete.target` in three boots is passed over for the last one that booted, which archix rolls back to. | VM |
| [Rollback at runtime](/docs/concepts/generations/#rolling-back-from-a-running-system) | **Shipped** | `archix rollback` goes back, or forward, to any kept generation, also after `gc`. | VM, Arch suite |
| [Repair after a power cut](/docs/concepts/safety/#a-power-cut-inside-pacman) | **Shipped** | The initramfs restores what a switch cut off inside pacman was changing, so the machine boots. | VM |
| [`archix recover`](/docs/concepts/safety/#archix-recover) | **Shipped** | Continues, rolls back or abandons an unfinished switch, repairing torn pacman database entries first, and pacman's own libraries when it can't start. | VM, Arch suite |
| [DKMS during a boot-time rollback](/docs/concepts/generations/#older-kernels-keep-booting) | **Shipped** | Booting an older entry rolls the kernel, its headers and the DKMS package back, and DKMS rebuilds and owns the module for that kernel; going back to a generation without the driver removes it. Tested with v4l2loopback and NVIDIA's open and closed drivers. | VM |
| [Upgrading archix itself](/docs/concepts/safety/#upgrading-archix-itself) | **Shipped** | Switching to a new archix, and rolling back across that upgrade at runtime, from the boot menu and after a kill; each journal records the oldest archix that can finish it. | VM, Arch suite |
| [`archix eject`](/docs/reference/cli/#eject) | **Shipped** | Hands booting back to plain systemd-boot entries or mkinitcpio UKIs, Secure Boot included, and takes archix's hooks and boot service off. | VM, Arch suite |
| [Config written only when a switch commits](/docs/concepts/safety/#declined-and-killed-switches) | **Shipped** | `archix.lock` and `installed.lua` are evaluated from memory and written by the switch's journal as its generation becomes current; a kill before that leaves them untouched, and `recover` writes them only when it finishes the switch. | Arch suite |
| [Rescue entries](/docs/concepts/safety/#rescue-mode) | **Shipped** | Restores a generation's core packages without pacman, from the initramfs or a live USB. | VM, Arch suite |
| [Checks before a switch](/docs/concepts/safety/#signatures-and-disk-space) | **Shipped** | Signatures, free space on every mount a file lands on, and file conflicts are checked before anything changes. | VM (signatures, full disk), Arch suite (file conflicts) |
| [Keyrings](/docs/concepts/safety/#keyrings) | **Shipped** | Incoming keyring packages are populated into the signature check and installed first; a switch or rollback never takes a keyring back. | Arch suite |
| [AUR builds with review](/docs/guides/aur/) | **Shipped** | Recipes are locked by commit, shown for review, and built in a clean chroot. | Arch suite |
| [Soname rebuilds](/docs/guides/local-pkgbuilds-and-patches/#rebuilds-at-the-same-version) | **Shipped** | A package archix built is rebuilt when a library it links disappears in a switch; a rollback reinstalls the generation's own build. | VM |
| [Secure Boot signing](/docs/guides/secure-boot/) | **Shipped** | Boot entries and systemd-boot are signed with your keys, and a switch that wouldn't be signed is refused. | VM, Arch suite |
| [Managed `/etc` files, users and services](/docs/concepts/safety/#managed-files-backups-and-hand-edits) | **Shipped** | Files are backed up and hand edits saved; users, groups and services follow the config. | VM (files, services), Arch suite (users) |
| [Restarting what uses replaced files](/docs/concepts/safety/#restarting-what-uses-replaced-files) | **Shipped** | Units still running replaced binaries, libraries or interpreted code are restarted. | Arch suite |
| [Local PKGBUILDs and patches](/docs/guides/local-pkgbuilds-and-patches/) | **Shipped** | Recipes in the config and patched official packages are locked by content and built in the chroot. | VM, Arch suite |
| [Third-party repos](/docs/guides/third-party-repos/) | **Shipped** | Declared repos, CPU-level checks, a copy of every package a generation takes from one, and a fallback chain when a locked package is gone. | VM (CachyOS), Arch suite |
| [Package caches](/docs/concepts/lockfile/#package-caches) | **Shipped** | `archix cache push`, or every switch with `package_cache_push`, fills a directory, shared as a mount or over HTTP, that other machines fetch package files and AUR builds from first, checked against the lock. | Arch suite |
| [Archive mirrors](/docs/reference/options/#archive_mirrors) | **Shipped** | `archive_url` points locking, switches, rollbacks and recovery at a mirror of the archive, and `archive_mirrors` lists more, tried when it's down. | VM, Arch suite |
| [Signed TPM PCR policies](/docs/guides/secure-boot/#tpm-bound-secrets) | **Shipped** | With systemd's PCR signing key in place, every UKI carries a PCR 11 policy signed with it, for secrets bound with `systemd-cryptenroll --tpm2-public-key`. | Arch suite (the signed policy; no TPM unlock yet) |
| [Flatpak apps](/docs/guides/flatpak-and-appimages/) | **Shipped** | System-wide apps are locked to commits and pinned back on rollback. | Arch suite |
| [Dotfiles and home](/docs/guides/dotfiles-and-apps/) | **Shipped** | Home files, live links, merged keys and `programs.*` modules. | Arch suite |
| [`archix init`](/docs/quick-start/) | **Shipped** | Captures an existing machine into a config and lock that match it exactly, rebuilding packages no repo or the AUR has as recipes from their installed files. | Arch suite |

## Partial

| Feature | Status | What's missing | Covered in |
| --- | --- | --- | --- |
| [Third-party repo exactness](/docs/concepts/limitations/#locking) | **Partial** | Locked versions are exact only while archix or a package cache has the file; otherwise a fresh machine gets what the repo serves today. | [Third-party repos](/docs/guides/third-party-repos/#why-third-party-versions-can-go-missing) |
| [AUR reproducibility](/docs/concepts/limitations/#locking) | **Partial** | Recipe commits, git sources and `SKIP`-checksum downloads are pinned after the first build; Mercurial, Subversion, Bazaar and Fossil sources aren't, and builds aren't bit-for-bit reproducible. | [The AUR](/docs/guides/aur/#vcs-packages) |

## Planned

| Feature | Status | Today | Covered in |
| --- | --- | --- | --- |
| [Atomic `/usr` tree](/docs/concepts/limitations/#rollback) | **Planned** | Not built: switches and rollbacks are journaled pacman runs that replace files in place. | [Safety](/docs/concepts/safety/) |
| [Secure Boot revocation](/docs/concepts/limitations/#secure-boot-and-the-tpm) | **Planned** | Not built: no SBAT or `dbx` handling, so old signed entries stay bootable. | [Secure Boot](/docs/guides/secure-boot/) |
