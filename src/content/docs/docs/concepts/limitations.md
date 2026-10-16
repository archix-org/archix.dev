---
title: Limitations and failure modes
description: What archix doesn't guarantee, where its safety checks stop, and the failure modes worth knowing before you rely on it.
sidebar:
  order: 6
---

archix makes a rolling-release system declared, locked and reversible, on top of pacman and a normal filesystem. Some guarantees that come for free with an immutable store don't come with that. This page lists them, so you know where the edges are.

## Rollback

**A rollback is a repair, not an atomic switch.** There's no store to point at an older closure, so rolling back runs pacman to reinstall the older generation's packages on the system as it is. From the boot menu, it happens early in boot with the userspace of the generation you're leaving, before the rest of the system starts. It's journaled and can be [recovered](/docs/concepts/safety/#archix-recover) like a switch, but if pacman itself is broken, rollback can't run; [rescue mode](/docs/concepts/safety/#rescue-mode) is the way out then.

**Package rollback doesn't roll back data.** A rollback reinstalls older packages and puts back managed files, but leaves everything else on disk as it is, databases included. A newer PostgreSQL or MariaDB may have migrated its data, and the older version can't read it. archix asks before going back across such versions, from a list it ships plus your [`unsafe_downgrades`](/docs/reference/options/#unsafe_downgrades), but a rollback from the boot menu goes ahead anyway; see [Downgrades that may break data](/docs/concepts/generations/#downgrades-that-may-break-data). Back up data you can't lose.

## Locking

**Third-party repo versions are exact only while archix has them cached.** Most third-party repos keep only the current build of each package. archix keeps a copy of everything it installs from them, but a fresh machine, or one whose cache was cleared, can only get what the repo serves today. Then the [fallback chain](/docs/guides/third-party-repos/#the-fallback-chain) picks a replacement and rewrites `archix.lock` with it, so the lock no longer says what the first machine had.

**The AUR lock pins the recipe, not the sources.** An AUR package is locked to the commit of its build recipe. The sources the recipe downloads are pinned only by the checksums in it. VCS packages, such as `-git` ones, build from the newest upstream commit with no checksum, so rebuilding the same locked recipe can give different code; archix records the version the build gave itself, but can't bring an older one back except from its cache.

**The Arch Linux Archive is a single source.** Official packages are locked to a day of `archive.archlinux.org`, and fetched from it when they aren't cached. There's no mirror of it to fall back to: while the archive is down, packages that aren't cached can't be fetched, and `update` can't move to a new snapshot.

**init leaves foreign packages that aren't on the AUR alone.** A package you built by hand, or one that left the AUR, can't be locked, so [`archix init`](/docs/reference/cli/#init) puts it in `ignored_packages` instead of declaring it. archix never installs, removes or rolls it back. To manage it, put its recipe in [`pkgbuilds/`](/docs/guides/local-pkgbuilds-and-patches/).

## Trust and signatures

**Install scriptlets run as root.** Once you approve an AUR recipe, its package is installed by pacman like any other, and its install scriptlet runs as root, outside the build chroot. Review is the safeguard: read what an `.install` file does before you trust a recipe.

**pacman checks package signatures, at apply time.** In prepare, archix checks every package file against the sha256 in the lock, so a file that changed since locking never gets further. The packager's signature is checked by pacman when it installs the file, during apply. Packages archix built itself, from the AUR or your recipes, aren't signed; the lock and the build are what you trust.

## Secure Boot and the TPM

**Old signed boot entries stay bootable.** Every entry archix signs is signed with your key, and archix doesn't use SBAT or the firmware's `dbx` to revoke older ones. An entry `gc` deleted from the ESP is gone, but a copy of a signed entry from an older generation, with an older kernel, still boots wherever it's put.

**PCR 11 bindings break on every switch.** archix doesn't sign PCR policies for its UKIs. Each switch writes a new UKI, which measures differently into PCR 11, so a disk key or other secret sealed to PCR 11 has to be re-sealed after each switch. Bind to PCRs that don't change per switch, such as PCR 7, or use a passphrase as well.

See [Secure Boot](/docs/guides/secure-boot/) for what archix does sign and check.

## Disk space

**Free space is checked on the ESP only.** Before a switch, archix makes sure the ESP can hold the new boot entries. It doesn't check the root filesystem or the cache. A download that fills the disk stops in prepare, with nothing changed, but if the root filesystem runs out while pacman installs, the switch is left unfinished, and `archix recover` continues it once you've made room.

## The boot-time rollback and initramfs hooks

When a rollback from the boot menu reinstalls a kernel, pacman runs mkinitcpio's own hook, which rebuilds the images your mkinitcpio presets name, `/boot/initramfs-*.img` on a stock install. archix's per-generation UKIs, `EFI/Linux/archix-<n>.efi`, aren't named by any preset; archix builds them itself, with its own `mkinitcpio` and `ukify` run, and the boot-time rollback doesn't rebuild or replace them.

## Not yet known

- **DKMS during a boot-time rollback.** Reinstalling an older kernel at boot makes DKMS build its modules for that kernel, which needs that kernel's headers. They come from the cache only if the generation you roll back to had them installed; whether the build then succeeds that early in boot isn't established yet.
