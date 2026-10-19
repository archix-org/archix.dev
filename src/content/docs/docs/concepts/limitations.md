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

**Boot counting needs systemd-boot 260 or newer, and stops at `boot-complete.target`.** A new generation's entry gets [`boot.tries`](/docs/reference/options/#boottries) boots to reach `boot-complete.target`, after which systemd-boot starts the last generation that booted and archix rolls back to it. With an older systemd-boot on the ESP, archix writes plain entries, and a new generation that hangs or panics stays the default: pick the previous generation in the boot menu yourself. A generation that gets that far but is broken past it, such as a desktop that doesn't start, counts as good.

**Package rollback doesn't roll back data.** A rollback reinstalls older packages and puts back managed files, but leaves everything else on disk as it is, databases included. A newer PostgreSQL or MariaDB may have migrated its data, and the older version can't read it. archix asks before going back across such versions, from a list it ships plus your [`unsafe_downgrades`](/docs/reference/options/#unsafe_downgrades), but a rollback from the boot menu goes ahead anyway; see [Downgrades that may break data](/docs/concepts/generations/#downgrades-that-may-break-data). Back up data you can't lose.

## Locking

**Third-party repo versions are exact only while a copy is cached.** Most third-party repos keep only the current build of each package. archix keeps a copy of everything it installs from them, but a fresh machine, or one whose cache was cleared, can only get what the repo serves today. Then the [fallback chain](/docs/guides/third-party-repos/#the-fallback-chain) picks a replacement and rewrites `archix.lock` with it, so the lock no longer says what the first machine had. A [package cache](/docs/concepts/lockfile/#package-caches) that the first machine fills, with `archix cache push` or after every switch with [`package_cache_push`](/docs/reference/options/#package_cache_push), closes the gap for every machine that can reach it, for as long as it keeps the files.

**Only git sources and downloads are pinned for AUR rebuilds.** An AUR package is locked to the commit of its build recipe, and its first build records in `archix.lock` the commit each git source was cloned at and the sha256 of each download whose checksums are `SKIP`; a later build of the same lock checks out those commits and checks those downloads. Sources from Mercurial, Subversion, Bazaar or Fossil aren't pinned: a rebuild takes whatever they serve that day, and `archix status` names such packages. Builds aren't bit-for-bit reproducible either, so another machine's build of the same pinned sources is the same code, not the same file, unless a [package cache](/docs/concepts/lockfile/#package-caches) it lists holds the first machine's build.

**init can only rebuild foreign packages whose files are untouched.** A package you built by hand, or one that left the AUR, is rebuilt by [`archix init`](/docs/reference/cli/#init) as a recipe in [`pkgbuilds/`](/docs/guides/local-pkgbuilds-and-patches/) from the files pacman installed, so it's locked and another machine can build it. That needs every file it owns, except the config files it lets you edit, to be as pacman installed it, and so do the foreign packages it depends on. One that changed stays in `ignored_packages`, where archix never installs, removes or rolls it back, and init says why: `! nvidia-dkms-custom is in no repo or the AUR, and archix leaves it alone: usr/lib/modules-load.d/nvidia.conf changed since pacman installed it`. A rebuilt package carries its files in the config, `files.tar.zst` next to the recipe, so a big one makes the config repository big.

## Trust and signatures

**Install scriptlets run as root.** Once you approve an AUR recipe, its package is installed by pacman like any other, and its install scriptlet runs as root, outside the build chroot. Review is the safeguard: a new package's review shows every file it ships, its `.install` file included, so read what that does before you trust a recipe.

**Rollbacks trust the pinned sha256, not today's keyring.** In prepare, archix checks every package file against the sha256 in the lock, and has pacman check every incoming package's signature, so a bad signature or an unknown key stops a switch before anything changes. A rollback skips the signature: it installs packages that were verified when they were first installed, by the sha256 the generation pinned, so a packager key revoked since doesn't block going back. That also means a key revoked because it was compromised doesn't block it either. Packages archix built itself, from the AUR or your recipes, aren't signed; the lock and the build are what you trust.

**An AUR build the lock has no sha256 for is trusted as the cache says.** Packages from a [package cache](/docs/concepts/lockfile/#package-caches) are checked against the sha256 in the lock, like any download, AUR builds included: the lock records the first build of each recipe commit, and a cache's build of the same commit that differs is passed over. Until some machine has built it, as right after `archix update` locks a new commit, the lock has no sha256 for it, and archix takes the build the cache's record names, checked only against the record's own hashes. Whoever can write to the cache can hand your machines that package. List only caches you control.

## Secure Boot and the TPM

**Old signed boot entries stay bootable until you revoke them.** Every UKI archix signs is signed with your key, so a copy of one from an older generation, with an older kernel, boots wherever it's put, until [`archix secure-boot revoke`](/docs/guides/secure-boot/#revoking-old-boot-images), or [`boot.revoke_unused`](/docs/reference/options/#bootrevoke_unused) after every switch, puts its hash in `dbx`. That takes sbctl's KEK as a file, and archix adds at most 128 entries, so a machine that kept revoking for years would have to stop. SBAT isn't used: without shim, nothing enforces it. Old signed copies of systemd-boot itself aren't revoked.

**PCR 11 bindings need a signed policy.** A switch that changes the kernel, initramfs or command line writes a new UKI, which measures differently into PCR 11, so a secret sealed to PCR 11's values has to be re-sealed after it. Once you create systemd's PCR signing key, archix signs a PCR policy into every UKI, and a secret bound to the key with `systemd-cryptenroll --tpm2-public-key` keeps unlocking; see [TPM-bound secrets](/docs/guides/secure-boot/#tpm-bound-secrets). archix's tests check the signed policy in the UKI, but don't yet unlock a secret with a TPM across a switch.

**Which generation booted isn't signed.** The UKIs and their command lines are signed. Which generation a boot belongs to comes from the boot menu entry, `loader/entries/archix-<n>.conf`, an unsigned file on the FAT ESP, which systemd-boot reports in the `LoaderEntrySelected` EFI variable. archix keeps a copy of every entry it writes on the root filesystem, and rolls nothing back when the entry that started doesn't match its copy, so a renamed or edited entry doesn't move the system. Someone who can write to the ESP can still change `loader.conf` or the boot menu's EFI variables so that an older entry archix wrote starts, which rolls the system back to that kept generation, a set of packages archix once installed; the boot menu offers it anyway. archix also rolls nothing back when the variable is missing, as when the firmware starts a UKI or `BOOTX64.EFI` directly, when it names an entry that isn't archix's, or when it names a generation archix doesn't have.

**The signing keys sit on the root filesystem.** sbctl's Secure Boot keys, in `/var/lib/sbctl/keys`, and systemd's PCR signing key, `/etc/systemd/tpm2-pcr-private-key.pem`, are plain files that root can read, and so can anyone who mounts an unencrypted root from another system. With the db key they can sign a boot image your firmware accepts, with the KEK they can change `db` and `dbx`, and with the PCR key they can sign a policy that unlocks a secret bound with `systemd-cryptenroll --tpm2-public-key` for any UKI they build. Every older UKI archix signed also carries a valid signed policy, so it unlocks such a secret too until it's [revoked](/docs/guides/secure-boot/#revoking-old-boot-images). Keep the root filesystem encrypted, and treat root on the machine as owning these keys; sbctl can keep its keys in a TPM or a YubiKey instead, though archix can't then sign `dbx` updates.

See [Secure Boot](/docs/guides/secure-boot/) for what archix does sign and check.

## Disk space

**The free-space check is an estimate.** Before a switch or rollback changes anything, archix checks every mount the incoming files land on, the way pacman does, from the file sizes in each package's `.MTREE` rounded up to whole blocks, less what the replaced versions free there, plus a cushion; see [Signatures and disk space](/docs/concepts/safety/#signatures-and-disk-space). It doesn't count directories, inodes or a filesystem's own metadata, and a boot-menu rollback skips it. Official packages to download are counted ahead from the snapshot's repo databases; third-party packages and AUR builds from a package cache aren't, and a full cache disk stops the switch while those download, before anything changes. If a filesystem runs out anyway while pacman installs, the switch is left unfinished, and `archix recover` continues it once you've made room.

## The boot-time rollback and initramfs hooks

When a rollback from the boot menu reinstalls a kernel, pacman runs mkinitcpio's own hook, which rebuilds the images your mkinitcpio presets name, `/boot/initramfs-*.img` on a stock install. archix's UKIs, in `EFI/archix/`, aren't named by any preset; archix builds them itself, with its own `mkinitcpio` and `ukify` run, and the boot-time rollback doesn't rebuild or replace them.

With `boot.loader` set, declaring `dkms` or any `*-dkms` package also declares `boot.kernel`'s headers, `linux-cachyos-headers` for `linux-cachyos`, so every generation can rebuild its modules for its own kernel when a boot-time rollback reinstalls it. A DKMS module that only arrives as a dependency, as `v4l2loopback-dkms` does for `v4l2loopback-utils`, brings the headers too: when the resolved lock has `dkms` but not the kernel's headers, archix resolves again with them.

## Not yet known

- **Weeks of use on real hardware.** The failure paths are exercised in VMs and containers: power cuts mid-update, killed runs, a full disk, a destroyed glibc, a revoked key. archix hasn't yet run for weeks on real machines, which is where the failures nobody thought to test turn up.

[Feature status](/docs/concepts/status/) lists what's shipped, partial and planned, and what tests each.
