---
title: Limitations and failure modes
description: What archix doesn't guarantee, where its safety checks stop, and the failure modes worth knowing before you rely on it.
sidebar:
  order: 6
---

archix makes a rolling-release system declared, locked and reversible, on top of pacman and a normal filesystem. Some guarantees that come for free with an immutable store don't come with that. This page lists them, so you know where the edges are.

## Rollback

**A rollback is a repair, not an atomic switch.** There's no store to point at an older closure, so rolling back runs pacman to reinstall the older generation's packages on the system as it is. From the boot menu, it happens early in boot with the userspace of the generation you're leaving, before the rest of the system starts. It's journaled and can be [recovered](/docs/concepts/safety/#archix-recover) like a switch, but if pacman itself is broken, rollback can't run; [rescue mode](/docs/concepts/safety/#rescue-mode) is the way out then.

**`/usr` isn't switched atomically.** A switch, like a rollback, has pacman replace files in place, package by package. A crash halfway leaves a mix of old and new files until the boot-time repair and `archix recover` sort it out; see [A power cut inside pacman](/docs/concepts/safety/#a-power-cut-inside-pacman). Nothing points the system at a complete, prebuilt `/usr` tree.

**A killed switch leaves its config edits on disk until the next run.** `archix install` writes `installed.lua`, and a switch writes `archix.lock`, before anything is applied, because the config is evaluated from disk. A declined or failed switch puts both back at once; a killed one leaves them until the next archix run or boot puts them back; see [Declined and killed switches](/docs/concepts/safety/#declined-and-killed-switches). Evaluating the edited config in memory, so nothing is written until the switch succeeds, is planned.

**Package rollback doesn't roll back data.** A rollback reinstalls older packages and puts back managed files, but leaves everything else on disk as it is, databases included. A newer PostgreSQL or MariaDB may have migrated its data, and the older version can't read it. archix asks before going back across such versions, from a list it ships plus your [`unsafe_downgrades`](/docs/reference/options/#unsafe_downgrades), but a rollback from the boot menu goes ahead anyway; see [Downgrades that may break data](/docs/concepts/generations/#downgrades-that-may-break-data). Back up data you can't lose.

## Locking

**Third-party repo versions are exact only while a copy is cached.** Most third-party repos keep only the current build of each package. archix keeps a copy of everything it installs from them, but a fresh machine, or one whose cache was cleared, can only get what the repo serves today. Then the [fallback chain](/docs/guides/third-party-repos/#the-fallback-chain) picks a replacement and rewrites `archix.lock` with it, so the lock no longer says what the first machine had. A [package cache](/docs/concepts/lockfile/#package-caches) that the first machine fills with `archix cache push` closes the gap for every machine that can reach it, for as long as it keeps the files.

**The AUR lock pins the recipe, not the sources.** An AUR package is locked to the commit of its build recipe, and the package file it built is recorded only by the generation that installed it, not by the lock. The sources the recipe downloads are pinned only by the checksums in it. VCS packages, such as `-git` ones, build from the newest upstream commit, and a source whose checksum is `SKIP` isn't checked at all, so rebuilding the same locked recipe can give different code; archix records the version the build gave itself, but can't bring an older one back except from its cache. Another machine building from your lock builds again, from whatever those sources serve that day, unless a [package cache](/docs/concepts/lockfile/#package-caches) it lists holds the first machine's build of the same commit at the same snapshot.

**The Arch Linux Archive is the one source of old official packages.** Official packages are locked to a day of the archive, and fetched from it when they aren't cached. [`archive_url`](/docs/reference/options/#archive_url) points archix at a mirror with the same layout instead, and `switch` records it so rollback and `recover` use it too. It's still one URL with nothing behind it: while it's down, packages that neither archix's cache nor a [package cache](/docs/concepts/lockfile/#package-caches) holds can't be fetched, and `update` can't move to a new snapshot.

**init leaves foreign packages that aren't on the AUR alone.** A package you built by hand, or one that left the AUR, can't be locked, so [`archix init`](/docs/reference/cli/#init) puts it in `ignored_packages` instead of declaring it. archix never installs, removes or rolls it back. To manage it, put its recipe in [`pkgbuilds/`](/docs/guides/local-pkgbuilds-and-patches/).

## Trust and signatures

**Install scriptlets run as root.** Once you approve an AUR recipe, its package is installed by pacman like any other, and its install scriptlet runs as root, outside the build chroot. Review is the safeguard: a new package's review shows every file it ships, its `.install` file included, so read what that does before you trust a recipe.

**Rollbacks trust the pinned sha256, not today's keyring.** In prepare, archix checks every package file against the sha256 in the lock, and has pacman check every incoming package's signature, so a bad signature or an unknown key stops a switch before anything changes. A rollback skips the signature: it installs packages that were verified when they were first installed, by the sha256 the generation pinned, so a packager key revoked since doesn't block going back. That also means a key revoked because it was compromised doesn't block it either. Packages archix built itself, from the AUR or your recipes, aren't signed; the lock and the build are what you trust.

**AUR builds from a package cache are trusted as the cache says.** Repo packages from a [package cache](/docs/concepts/lockfile/#package-caches) are checked against the sha256 in the lock, like any download. An AUR build has no sha256 in the lock, so archix takes the build the cache's record names for that recipe commit and snapshot, checked only against the record's own hashes. Whoever can write to the cache can hand your machines any package. List only caches you control.

## Secure Boot and the TPM

**Old signed boot entries stay bootable.** Every entry archix signs is signed with your key, and archix doesn't use SBAT or the firmware's `dbx` to revoke older ones. An entry `gc` deleted from the ESP is gone, but a copy of a signed entry from an older generation, with an older kernel, still boots wherever it's put.

**PCR 11 bindings break when the UKI changes.** archix doesn't sign PCR policies for its UKIs. A switch that changes the kernel, initramfs or command line writes a new UKI, which measures differently into PCR 11, so a disk key or other secret sealed to PCR 11 has to be re-sealed after it. Bind to PCRs that don't change per switch, such as PCR 7, or use a passphrase as well.

**Which generation booted isn't signed.** The UKIs and their command lines are signed. Which generation a boot belongs to comes from the boot menu entry, `loader/entries/archix-<n>.conf`, an unsigned file on the FAT ESP, which systemd-boot reports in the `LoaderEntrySelected` EFI variable. Someone who can write to the ESP can rename or edit entries so that a boot rolls back to another generation archix still keeps: a package downgrade done at boot, to a set of packages archix once installed. They can't add a kernel command-line option or boot an unsigned UKI with Secure Boot on, and an older kept generation is one the boot menu offers anyway. archix rolls nothing back when the variable is missing, as when the firmware starts a UKI or `BOOTX64.EFI` directly, when it names an entry that isn't archix's, or when it names a generation archix doesn't have.

See [Secure Boot](/docs/guides/secure-boot/) for what archix does sign and check.

## Disk space

**The free-space check is an estimate.** Before a switch or rollback changes anything, archix checks every mount the incoming files land on, the way pacman does, from the file sizes in each package's `.MTREE`, less what the replaced versions free there, plus a cushion; see [Signatures and disk space](/docs/concepts/safety/#signatures-and-disk-space). It doesn't count filesystem overhead, such as blocks a small file rounds up to, and a boot-menu rollback skips it. Downloads into the package cache aren't counted ahead either; a full cache disk stops the switch while downloading, before anything changes. If a filesystem runs out anyway while pacman installs, the switch is left unfinished, and `archix recover` continues it once you've made room.

## The boot-time rollback and initramfs hooks

When a rollback from the boot menu reinstalls a kernel, pacman runs mkinitcpio's own hook, which rebuilds the images your mkinitcpio presets name, `/boot/initramfs-*.img` on a stock install. archix's UKIs, in `EFI/archix/`, aren't named by any preset; archix builds them itself, with its own `mkinitcpio` and `ukify` run, and the boot-time rollback doesn't rebuild or replace them.

DKMS rebuilds a module for the rolled-back kernel only if the generation has that kernel's headers. If it doesn't, you get the module DKMS last built for that kernel, from archix's copy of its modules, which can be a different version from the one the generation locks.

## Not yet known

- **Weeks of use on real hardware.** The failure paths are exercised in VMs and containers: power cuts mid-update, killed runs, a full disk, a destroyed glibc, a revoked key. archix hasn't yet run for weeks on real machines, which is where the failures nobody thought to test turn up.

[Feature status](/docs/concepts/status/) lists what's shipped, partial and planned, and what tests each.
