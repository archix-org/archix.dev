---
title: Feature status
description: What archix ships today, what's partial and what isn't built yet, with the tests behind each.
sidebar:
  order: 7
---

This page labels each feature by what archix's code and tests do today, as of archix commit `a53af2e`:

- **Shipped**: built, and covered by the test suites named next to it.
- **Partial**: built, with a gap the line names.
- **Experimental**: built and tested in a VM or container, but not in the way it's meant to be used: real firmware, a real TPM.
- **Planned**: not built yet.
- **Not planned**: measured or weighed, and turned down for a reason the line gives.

Three kinds of test stand behind the labels. **VM** is a set of QEMU virtual machines with OVMF firmware and systemd-boot. The boot suite installs Arch on a btrfs root, lets archix lay it out, and covers boot counting, the fallback slot and the order entries fall back in, try mode and its layer on a full disk, the missing-snapshot guard, keeping a snapshot from the running system and from a try boot, the health check, a hanging unit and an emergency shell falling back with nobody at the machine, a broken glibc installed by plain pacman, a masked pacman hook, old records read by a newer archix, and strict mode; an ext4 lane runs switches and reverts with rollbacks off. **Arch suite** is archix's full test suite, run as root inside a systemd-booted Arch container, with real pacman transactions, AUR builds, network access, and roots on loop-mounted btrfs and ext4: switches, snapshots, keeps, reverts, the pacman hooks, the Lua sandbox and the config capture. **Power cuts** reset a VM through QEMU at chosen steps, marked by a test hook in archix's debug build, initramfs included, then boot it and check what came up: each step of `archix switch` and of systemd-bless-boot's rename of the entry, a real `pacman -Syu` cut during extraction and at each step of the boot image rebuild and the boot counter reset after it, each step of `archix rollback N` and `--cancel`, the move of `/var/lib` into `@varlib`, and the move of a top-level root into `@`. Every cut booted unattended, on btrfs. What each boot was checked for is listed under [How it's tested](/docs/concepts/snapshots/#how-its-tested): `pacman -Dk` and `pacman -Qkk`, unfiltered, against the same commands on the uncut system, ignoring only modification times, plus the ESP, `archix status`, the entries' images, the snapshots, pacman's lock and the unfinished mark. Every power-cut suite has passed with those checks. A QEMU reset keeps every write the guest had already sent to its disk, so it can't catch a missing flush. **Write replay** covers that: it logs every write of one real run through QEMU's `blklogwrites` and boots the disk at every flush and mark, with up to 16 subsets of each flush's ESP writes, against the same unfiltered baseline. Full replays of an install through plain pacman, `archix switch`, keep and the root move pass it, and a sampled replay of the `/var/lib` move; a sampled replay of a `pacman -Syu` with its kernel found the rescue-entry bug, a cut while the rescue entry is rewritten leaving it empty, now fixed; its rerun on the fix is pending; mutations that drop a root flush or an ESP sync fail it.

The **feature suites** each boot a VM image of their own on btrfs, on the current model: NVIDIA's open driver and an AUR one across a kernel and driver update; `v4l2loopback-dkms` built for `linux` and `linux-lts`, tried and kept; CachyOS's repos and v3 builds booted by Limine, with the move to systemd-boot; 52 weekly switches with the ESP, menu, snapshots and records bounded; signed PCR policies; live firewall changes and revert; the rescue entry's rollback and password reset; a live-USB recovery through `archix chroot`; secrets on try boots and a keep from a chroot; and a system set up by archix 0.2.0 refusing this one. Every suite has passed on both filesystems where it applies, every power-cut suite, and the replays above but the pacman one. These passes were recorded on commits before the last fixes; the release gate on the final commit is the confirmation.

All of this is VM-only: QEMU with OVMF firmware, and swtpm for the TPM. archix hasn't yet been run for weeks on real hardware. [Limitations and failure modes](/docs/concepts/limitations/) has the details behind every gap.

## Shipped

| Feature | Status | What it does | Tested in |
| --- | --- | --- | --- |
| [Lock and switch](/docs/concepts/lockfile/) | **Shipped** | Official packages pinned to a day of the Arch Linux Archive by version and sha256; a switch prepares everything, shows the config's and the plan's diff since the last switch, then applies in place, live. | VM, Arch suite |
| [Generations and revert](/docs/concepts/generations/) | **Shipped** | Every switch is a numbered generation with its packages, files, users, services and evaluated plan, kept in a root-owned copy; `archix revert` applies one live, from archix's cache and the archive, and refuses a downgrade that would break untracked packages unless `--allow-partial`. | VM (ext4), Arch suite |
| [Snapshots](/docs/concepts/snapshots/) | **Shipped** | A read-only btrfs snapshot of the root before every switch and every plain pacman transaction, with a space estimate before the transaction, retention by kind, and a free-space floor that keeps room for one keep. | VM, Arch suite |
| [Boot entries and boot counting](/docs/concepts/boot-and-fallback/#boot-counting-and-the-health-check) | **Shipped** | With systemd-boot 260 or newer, entries for the current system, recent snapshots and the fallback slot, sharing deduplicated signed images; a boot that doesn't count as good in its tries falls back to how the system last booted fine. | VM, Arch suite |
| [Health check](/docs/concepts/boot-and-fallback/#boot-counting-and-the-health-check) | **Shipped** | A boot counts as good once it reaches its default target with no newly failed unit and the display manager up, plus the units, ports, hosts and logins `boot.health_check` adds. | VM, Arch suite |
| [Unattended fallback](/docs/concepts/boot-and-fallback/#nobody-at-the-machine) | **Shipped** | A counted boot that hangs, sits in an emergency shell, or fails its health check reboots itself after `boot.unblessed_timeout`, with nobody logged in, so the fallback starts. | VM |
| [Try mode](/docs/concepts/boot-and-fallback/#try-mode) | **Shipped** | An older snapshot boots read-only with a throwaway layer, smaller or in memory on a full disk, with a notice and pacman refused. | VM |
| [Keep](/docs/concepts/snapshots/#keeping-a-snapshot) | **Shipped** | `archix rollback N` makes snapshot N the root from the next boot, carrying accounts made since and putting package files in `/var/lib` back; `--cancel` undoes it before the reboot. A cut at any step finishes or undoes it at the next boot. | VM, power cuts, Arch suite |
| [Power cuts during package updates](/docs/concepts/boot-and-fallback/#what-falls-back-by-itself) | **Shipped** | A package update cut off at any point boots afterwards, whether `archix switch` or plain pacman ran it: the current entry gets its tries and the slot its entry before anything changes, and an update marked unfinished fails its health check, so the boot falls back to the snapshot from before it. A cut before a switch's mark boots the system as it was, and one after it the new generation. [Until the boot after the first switch](/docs/concepts/limitations/#power-cuts) there's no snapshot, so a cut has no fallback. | Power cuts, VM, Arch suite |
| [The missing-snapshot guard](/docs/concepts/boot-and-fallback/#a-missing-snapshot) | **Shipped** | An entry whose snapshot was deleted by hand boots the newest surviving snapshot, else the rescue entry, never an entry that failed. | VM |
| [The layout](/docs/concepts/snapshots/#the-layout) | **Shipped** | `@archix` and `@varlib` beside the root, pacman's, DKMS's and archix's state bound from the root, and `/var/lib` moved at the next boot, resumably. | VM, power cuts, Arch suite |
| [A root at btrfs's top level](/docs/concepts/snapshots/#a-root-at-the-top-level) | **Shipped** | Moved into `@` at the next boot, in the initramfs, resumably, leaving other subvolumes where they are. | Power cuts, Arch suite |
| [Foreign roots and `archix resync`](/docs/concepts/snapshots/#a-root-archix-didnt-produce) | **Shipped** | A root another tool put in place is detected, switches and keeps refuse, and `resync` adopts it. | Arch suite |
| [pacman hooks](/docs/concepts/pacman-and-yay/) | **Shipped** | archix's hooks snapshot plain transactions, rebuild the boot image after a kernel or initramfs change and reset boot counting; `archix status` and every boot warn when one is masked or shadowed. | VM, Arch suite |
| [Drift](/docs/concepts/pacman-and-yay/#drift) | **Shipped** | Packages installed with pacman or an AUR helper are added to `installed.lua`, kept untracked, or asked about at the next switch. | Arch suite |
| [Strict mode](/docs/concepts/pacman-and-yay/#strict-mode) | **Shipped** | A guardrail: plain pacman and libalpm front ends refuse every transaction archix doesn't run, and never wedge pacman once archix, its state or the config is gone. | VM, Arch suite |
| [Config capture and the evaluator](/docs/concepts/safety/) | **Shipped** | Root never evaluates or reads the live config: it captures the tree as the owner, copies it into a root-owned repository, and evaluates it as a transient user in the Lua sandbox with resource limits. | Arch suite |
| [Keyrings](/docs/concepts/safety/) | **Shipped** | Incoming keyring packages are populated into the signature check and installed first; a switch or revert never takes a keyring back. | Arch suite |
| [AUR builds with review](/docs/guides/aur/) | **Shipped** | Recipes are locked by commit, shown for review, and built in a clean chroot. | Arch suite |
| [Soname rebuilds](/docs/guides/local-pkgbuilds-and-patches/#rebuilds-at-the-same-version) | **Shipped** | A package archix built is rebuilt when a library it links disappears in a switch; a revert reinstalls the generation's own build. | Arch suite |
| [Secure Boot signing](/docs/guides/secure-boot/) | **Shipped** | Boot images, the rescue entry and systemd-boot are signed with your keys, and the boot menu never points at an unsigned image. | Arch suite |
| [Managed files, users and services](/docs/concepts/safety/) | **Shipped** | Files are backed up and hand edits saved; users, groups and services follow the config, applied live, with changed units restarted. | VM, Arch suite |
| [Activation hooks](/docs/guides/hooks/) | **Shipped** | Hooks and `on_change` scripts run on the running system at the end of every switch and revert. | Arch suite |
| [Local PKGBUILDs and patches](/docs/guides/local-pkgbuilds-and-patches/) | **Shipped** | Recipes in the config and patched official packages are locked by content and built in the chroot. | Arch suite |
| [Third-party repos](/docs/guides/third-party-repos/) | **Shipped** | Declared repos, CPU-level checks, a copy of every package a generation takes from one, and a fallback chain when a locked package is gone. | VM, Arch suite |
| [Package caches](/docs/concepts/lockfile/#package-caches) | **Shipped** | `archix cache push`, or every switch with `package_cache_push`, fills a directory, shared as a mount or over HTTP, that other machines fetch package files and AUR builds from first, checked against the lock. | Arch suite |
| [Archive mirrors](/docs/reference/options/#archive_mirrors) | **Shipped** | `archive_url` points locking, switches and reverts at a mirror of the archive, and `archive_mirrors` lists more, tried when it's down. | Arch suite |
| [Flatpak apps](/docs/guides/flatpak-and-appimages/) | **Shipped** | System-wide apps are locked to commits, installed in the same switch as `flatpak` itself, and pinned back on revert. | Arch suite |
| [Dotfiles and home](/docs/guides/dotfiles-and-apps/) | **Shipped** | Home files, written as their user, live links, merged keys and `programs.*` modules. | Arch suite |
| [`archix init`](/docs/quick-start/) | **Shipped** | Captures an existing machine into a config and lock that match it exactly, asks about rollbacks, drift, snap-pac and moving the root, and shows what the health check concludes on this machine. | Arch suite |

## Partial

| Feature | Status | What's missing | Covered in |
| --- | --- | --- | --- |
| [Rescue entry](/docs/guides/recovery/#without-a-live-usb) | **Partial** | Built, refreshed after each switch, and its `rollback N` runs `archix rollback N`; the rescue suite boots it and runs its rollback and password reset, on btrfs only. | [Recovering a broken system](/docs/guides/recovery/) |
| [Live-USB recovery](/docs/guides/recovery/#from-a-live-usb) | **Partial** | `archix chroot` mounts the layout with its binds and the ESP for `arch-chroot`; the recovery suite runs a keep through it, on btrfs only. | [Recovering a broken system](/docs/guides/recovery/#from-a-live-usb) |
| [GRUB](/docs/concepts/boot-and-fallback/#grub-and-limine) | **Partial** | archix writes the same entries into GRUB's menu, but GRUB counts no boots, so a fallback is a pick by hand. Its menu is unit-tested, not VM-tested. | [Boot and fallback](/docs/concepts/boot-and-fallback/#grub-and-limine) |
| [Limine](/docs/concepts/boot-and-fallback/#grub-and-limine) | **Partial** | archix writes no Limine entries and leaves `limine.conf` to limine-snapper-sync; a fallback is a pick by hand, and a keep across a kernel change is refused. The move to systemd-boot is tested in containers, not yet on real firmware. | [Moving from Limine to systemd-boot](/docs/concepts/boot-and-fallback/#moving-from-limine-to-systemd-boot) |
| [DKMS and NVIDIA](/docs/concepts/snapshots/#the-layout) | **Partial** | DKMS's state lives on the root, so each snapshot boots with the modules it was built with; the NVIDIA and dkms2 suites pass in VMs, which have no NVIDIA GPU, so the driver is built and installed but never drives one. | [Limitations](/docs/concepts/limitations/#not-yet-known) |
| [Third-party repo exactness](/docs/concepts/limitations/#locking) | **Partial** | Locked versions are exact only while archix or a package cache has the file; otherwise a fresh machine gets what the repo serves today, and a revert on CachyOS reaches back only as far as archix has been installed. | [Third-party repos](/docs/guides/third-party-repos/#why-third-party-versions-can-go-missing) |
| [AUR reproducibility](/docs/concepts/limitations/#locking) | **Partial** | Recipe commits, git sources and `SKIP`-checksum downloads are pinned after the first build; Mercurial, Subversion, Bazaar and Fossil sources aren't, and builds aren't bit-for-bit reproducible. | [The AUR](/docs/guides/aur/#vcs-packages) |
| [Upgrading archix itself](/docs/concepts/safety/) | **Partial** | archix updates like any AUR package, and newer archix reads every older record and plan format; the selfupgrade suite checks that a 0.2.0 system refuses this archix and still boots its own; from one release of this model to the next isn't tested yet, as there's no second release. | [Limitations](/docs/concepts/limitations/#not-yet-known) |
| Secrets | **Partial** | Age-encrypted values decrypted into `/run/archix/secrets`, at boot and by each switch. The machine's identity lives in the root's own `/var/lib/archix`, so it rolls back with the root; if it's missing at boot, no secret is decrypted, and a counted boot fails its health check and falls back to the slot. The secrets suite runs them on try boots and a keep from a chroot. | [Options reference](/docs/reference/options/) |

## Experimental

| Feature | Status | What it does | Tested in |
| --- | --- | --- | --- |
| [Secure Boot revocation](/docs/guides/secure-boot/#revoking-old-boot-images) | **Experimental** | `archix secure-boot revoke`, or `boot.revoke_unused` after every switch, appends the hashes of images archix signed and no longer uses to `dbx`, signed with your KEK, leaving every other entry. | Arch suite; VM (OVMF only) on the earlier boot model |
| [Signed TPM PCR policies](/docs/guides/secure-boot/#tpm-bound-secrets) | **Experimental** | With systemd's PCR signing key in place, every image carries a PCR 11 policy signed with it; `archix secure-boot rotate-pcr-key` replaces the key and re-binds LUKS volumes. | Arch suite; VM (swtpm, not a real TPM) on the earlier boot model |

## Planned

| Feature | Status | What it will do | Covered in |
| --- | --- | --- | --- |
| Converting ext4 to btrfs | **Planned** | `archix convert-to-btrfs`, converting in the initramfs at the next boot and then moving the root into `@`, behind a destructive-operation warning, shipping only with power-cut tests at every step. | [ext4 and rollbacks off](/docs/concepts/snapshots/#ext4-and-rollbacks-off) |

## Not planned

| Feature | Status | Why | Covered in |
| --- | --- | --- | --- |
| Atomic deployments with a read-only `/usr` | **Not planned** | Staging each switch as a complete new system beside the running one and booting into it means blocking plain pacman, yay and `npm -g`, which expect to change the running system, and making every change wait for a reboot. archix applies in place instead, with snapshots before each change and boot counting to fall back. | [Snapshots](/docs/concepts/snapshots/) |
| Writing Limine entries | **Not planned** | limine-snapper-sync owns `limine.conf` on the machines that use Limine, and archix leaves it alone; automatic fallback needs systemd-boot, which `archix init` offers to move to. | [GRUB and Limine](/docs/concepts/boot-and-fallback/#grub-and-limine) |
