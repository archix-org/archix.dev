---
title: Feature status
description: What archix ships today, what's partial and what isn't built yet, with the tests behind each.
sidebar:
  order: 7
---

This page labels each feature by what archix's code and tests do today, as of archix commit `2a5781d`:

- **Shipped**: built, and covered by the test suites named next to it.
- **Partial**: built, with a gap the line names.
- **Planned**: not built yet.

Two test suites stand behind the labels. **VM** is a QEMU virtual machine with OVMF firmware that boots archix-managed generations: one run walks through boot-menu rollback, Secure Boot and rescue entries, and a rollback torture run updates a real Arch install from the 2026-06-01 snapshot to 2026-10-10, cutting the power, killing runs and filling the disk along the way. **Arch suite** is archix's full test suite, run as root inside a systemd-booted Arch container, with real pacman transactions, AUR builds and network access.

archix hasn't yet been run for weeks on real hardware. [Limitations and failure modes](/docs/concepts/limitations/) has the details behind every gap.

## Shipped

| Feature | Status | What it does | Tested in |
| --- | --- | --- | --- |
| [Lock and switch](/docs/concepts/lockfile/) | **Shipped** | Official packages pinned to a day of the Arch Linux Archive by version and sha256; a switch prepares everything, then applies from a journal. | VM, Arch suite |
| [Generations](/docs/concepts/generations/) | **Shipped** | Every switch is a numbered generation with its packages, files, users and services; `gc` keeps them by count and age. | VM, Arch suite |
| [Rollback from the boot menu](/docs/concepts/generations/#rolling-back-from-the-boot-menu) | **Shipped** | Booting an older entry rolls back to it early in boot, and it stays the default entry. | VM |
| [Rollback at runtime](/docs/concepts/generations/#rolling-back-from-a-running-system) | **Shipped** | `archix rollback` goes back, or forward, to any kept generation, also after `gc`. | VM, Arch suite |
| [Repair after a power cut](/docs/concepts/safety/#a-power-cut-inside-pacman) | **Shipped** | The initramfs restores what a switch cut off inside pacman was changing, so the machine boots. | VM |
| [`archix recover`](/docs/concepts/safety/#archix-recover) | **Shipped** | Continues, rolls back or abandons an unfinished switch, repairing torn pacman database entries first. | VM, Arch suite |
| [Rescue entries](/docs/concepts/safety/#rescue-mode) | **Shipped** | Restores a generation's core packages without pacman, from the initramfs or a live USB. | VM, Arch suite |
| [Checks before a switch](/docs/concepts/safety/#signatures-and-disk-space) | **Shipped** | Signatures, free space on every mount a file lands on, and file conflicts are checked before anything changes. | VM (signatures, full disk), Arch suite (file conflicts) |
| [AUR builds with review](/docs/guides/aur/) | **Shipped** | Recipes are locked by commit, shown for review, and built in a clean chroot. | Arch suite |
| [Soname rebuilds](/docs/guides/local-pkgbuilds-and-patches/#rebuilds-at-the-same-version) | **Shipped** | A package archix built is rebuilt when a library it links disappears in a switch; a rollback reinstalls the generation's own build. | VM |
| [Secure Boot signing](/docs/guides/secure-boot/) | **Shipped** | Boot entries and systemd-boot are signed with your keys, and a switch that wouldn't be signed is refused. | VM, Arch suite |
| [Managed `/etc` files, users and services](/docs/concepts/safety/#managed-files-backups-and-hand-edits) | **Shipped** | Files are backed up and hand edits saved; users, groups and services follow the config. | VM (files, services), Arch suite (users) |
| [Restarting what uses replaced files](/docs/concepts/safety/#restarting-what-uses-replaced-files) | **Shipped** | Units still running replaced binaries, libraries or interpreted code are restarted. | Arch suite |
| [Local PKGBUILDs and patches](/docs/guides/local-pkgbuilds-and-patches/) | **Shipped** | Recipes in the config and patched official packages are locked by content and built in the chroot. | VM, Arch suite |
| [Third-party repos](/docs/guides/third-party-repos/) | **Shipped** | Declared repos, CPU-level checks, and a fallback chain when a locked package is gone. | Arch suite |
| [Package caches](/docs/concepts/lockfile/#package-caches) | **Shipped** | `archix cache push` fills a directory, shared as a mount or over HTTP, that other machines fetch package files and AUR builds from first. | Arch suite |
| [Archive mirrors](/docs/reference/options/#archive_url) | **Shipped** | `archive_url` points locking, switches, rollbacks and recovery at a mirror of the archive. | VM, Arch suite |
| [Flatpak apps](/docs/guides/flatpak-and-appimages/) | **Shipped** | System-wide apps are locked to commits and pinned back on rollback. | Arch suite |
| [Dotfiles and home](/docs/guides/dotfiles-and-apps/) | **Shipped** | Home files, live links, merged keys and `programs.*` modules. | Arch suite |
| [`archix init`](/docs/quick-start/) | **Shipped** | Captures an existing machine into a config and lock that match it exactly. | Arch suite |

## Partial

| Feature | Status | What's missing | Covered in |
| --- | --- | --- | --- |
| [Third-party repo exactness](/docs/concepts/limitations/#locking) | **Partial** | Locked versions are exact only while archix or a package cache has the file; otherwise a fresh machine gets what the repo serves today. | [Third-party repos](/docs/guides/third-party-repos/#why-third-party-versions-can-go-missing) |
| [AUR reproducibility](/docs/concepts/limitations/#locking) | **Partial** | Recipe commits are pinned, the sources they download aren't: `-git` and `SKIP`-checksum sources float unless a package cache holds the build. | [The AUR](/docs/guides/aur/#how-aur-packages-are-locked) |
| [DKMS during a boot-time rollback](/docs/concepts/limitations/#not-yet-known) | **Partial** | Rolling a kernel back at boot works; whether DKMS modules build for it that early isn't verified. | [Limitations](/docs/concepts/limitations/) |

## Planned

| Feature | Status | Today | Covered in |
| --- | --- | --- | --- |
| [Atomic `/usr` tree](/docs/concepts/limitations/#rollback) | **Planned** | Not built: switches and rollbacks are journaled pacman runs that replace files in place. | [Safety](/docs/concepts/safety/) |
| [Secure Boot revocation](/docs/concepts/limitations/#secure-boot-and-the-tpm) | **Planned** | Not built: no SBAT or `dbx` handling, so old signed entries stay bootable. | [Secure Boot](/docs/guides/secure-boot/) |
| [Signed TPM PCR policies](/docs/concepts/limitations/#secure-boot-and-the-tpm) | **Planned** | Not built: secrets sealed to PCR 11 need re-sealing after every switch. | [Secure Boot](/docs/guides/secure-boot/) |
