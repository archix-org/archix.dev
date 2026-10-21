---
title: Limitations and failure modes
description: What archix doesn't guarantee, where its safety checks stop, and the failure modes worth knowing before you rely on it.
sidebar:
  order: 6
---

archix makes a rolling-release system declared, locked and reversible, on top of pacman and a normal filesystem, by building every switch as a [deployment](/docs/concepts/atomicity/) that the next boot starts. Some of that has a price, and some guarantees stop short. This page lists them, so you know where the edges are.

## Switching

**Changes take effect at the next boot, and only then.** A switch never changes the running system: new packages, `/etc` files, services, sysctl and firewall rules all arrive when you reboot into the new deployment. That includes `archix install htop`; until the reboot, `htop` isn't there. Applying changes live is not planned for the first release. Dotfiles, other files outside `/etc`, `/usr` and `/opt`, and Flatpak apps are the exception: a switch writes them right away. See [What takes effect when](/docs/concepts/safety/#what-takes-effect-when).

**Changes to `/etc` between a switch and the reboot are lost.** The new deployment has its own copy of `/etc`, made by the switch. A password you change, a Wi-Fi network you save or a file you edit before rebooting stays behind in the old deployment. archix's own writes to `archix.lock` and `installed.lua` are carried over, but if your config lives in `/etc/archix`, edits you make to it by hand are not; keep it in git, or in your home with `/etc/archix` linking to it, as `archix init` can set up. `switch` and `archix status` say when a deployment is waiting. Merging those changes into the waiting deployment continuously is planned.

**Plain pacman and yay can't change the system.** `/usr`, `/opt` and `/boot` are mounted read-only, so `pacman -S`, `pacman -Syu` and `yay` fail, and so does anything else that writes there at runtime, such as `sudo npm install -g`, an installer script that writes to `/opt`, or a file copied into `/usr/local/bin`. Install with [`archix install`](/docs/reference/cli/#install--s), or `archix install --temp` to try something without declaring it, and declare files under `/usr/local` in [`files`](/docs/reference/options/#files); both take effect at the next boot. Reading works as before: `pacman -Q`, `-Ss`, `-Si` and `-F`. See [pacman and yay](/docs/concepts/pacman-and-yay/).

**systemd-boot only, with the ESP outside `/boot`.** archix boots each deployment from its own systemd-boot entry, and refuses to switch without [`boot.loader = "systemd-boot"`](/docs/reference/options/#bootloader). GRUB and Limine aren't supported: they boot a kernel from `/boot` under one name that every deployment would share. The ESP has to be mounted somewhere other than `/boot`, `/efi` by default, since each deployment has its own `/boot`.

**Some disk layouts are refused.** The first switch lays the system out as deployments, and refuses before changing anything when `/boot`, `/usr`, `/opt`, `/var`, `/var/lib`, `/var/lib/pacman` or `/archix` is a separate filesystem, or the root is overlayfs, as in a container. Caches and logs on their own btrfs subvolumes, such as CachyOS's `@cache` and `@log`, are fine. See [Requirements](/docs/concepts/atomicity/#requirements).

**Hooks run before the network.** [Hooks and `on_change` scripts](/docs/guides/hooks/) run at the new deployment's first boot, before networking and before anyone logs in. A hook that needs a running service, a network connection or a user's session fails there, and runs again at every boot until it succeeds.

**There's no way back from deployments yet.** [`archix eject`](/docs/reference/cli/#eject) works only before the first switch; on a system laid out as deployments it refuses, since they boot only through archix's initramfs hook. A way back to plain Arch from deployments is being built, and isn't available yet.

**A torn entries directory can leave nothing to boot.** The ESP is FAT, and a crash while archix writes boot entries can lose the whole `loader/entries` directory, not just one entry. A fallback boot image that starts `current` without an entry is being added.

**A rollback brings back that deployment's passwords.** archix never manages passwords, but `/etc/shadow` is part of each deployment's `/etc`, so booting or rolling back to an older deployment brings back the passwords it had, and a password changed since is undone. Carrying accounts over between deployments is planned.

**Some failures still need a live USB.** Rolling back to a kept deployment needs only the boot menu. A damaged ESP, a broken systemd-boot, or a bug in archix's initramfs hook that keeps every deployment from mounting still needs a live USB to repair.

## Rollback

**A rollback keeps what's in `/var`.** Deployments hold `/usr`, `/opt`, `/boot`, `/etc`, pacman's database and DKMS state. Data under `/var`, such as databases, and your home are shared by every deployment and stay as they are. A newer PostgreSQL or MariaDB may have migrated its data, and the older version can't read it. `archix rollback` asks before going back across such versions, from a list it ships plus your [`unsafe_downgrades`](/docs/reference/options/#unsafe_downgrades), but picking an older entry in the boot menu boots it without asking; see [Downgrades that may break data](/docs/concepts/generations/#downgrades-that-may-break-data). Back up data you can't lose.

**Only kept deployments roll back instantly.** With the default [`deployments.keep = 2`](/docs/reference/options/#deploymentskeep), the previous deployment is kept. Older generations are rebuilt from the nearest kept deployment and their package files, which takes as long as an update and needs those files: in archix's cache, a [package cache](/docs/concepts/lockfile/#package-caches), the Arch Linux Archive, or, for third-party packages, archix's own copies.

**Boot counting needs systemd-boot 260 or newer, and stops at `boot-complete.target`.** A new deployment's entry gets [`boot.tries`](/docs/reference/options/#boottries) boots to reach `boot-complete.target`, after which systemd-boot starts the last deployment that booted. With an older systemd-boot on the ESP, archix writes plain entries, and a new deployment that hangs or panics stays the default: pick the previous one in the boot menu yourself. A deployment that gets that far but is broken past it, such as a desktop that doesn't start, counts as good.

## Locking

**Third-party repo versions are exact only while a copy is cached.** Most third-party repos keep only the current build of each package. archix keeps a copy of everything it installs from them, but a fresh machine, or one whose cache was cleared, can only get what the repo serves today. Then the [fallback chain](/docs/guides/third-party-repos/#the-fallback-chain) picks a replacement and rewrites `archix.lock` with it, so the lock no longer says what the first machine had. A [package cache](/docs/concepts/lockfile/#package-caches) that the first machine fills, with `archix cache push` or after every switch with [`package_cache_push`](/docs/reference/options/#package_cache_push), closes the gap for every machine that can reach it, for as long as it keeps the files.

**Only git sources and downloads are pinned for AUR rebuilds.** An AUR package is locked to the commit of its build recipe, and its first build records in `archix.lock` the commit each git source was cloned at and the sha256 of each download whose checksums are `SKIP`; a later build of the same lock checks out those commits and checks those downloads. Sources from Mercurial, Subversion, Bazaar or Fossil aren't pinned: a rebuild takes whatever they serve that day, and `archix status` names such packages. Builds aren't bit-for-bit reproducible either, so another machine's build of the same pinned sources is the same code, not the same file, unless a [package cache](/docs/concepts/lockfile/#package-caches) it lists holds the first machine's build.

**Foreign packages init rebuilds stay on the machine they came from.** A package you built by hand, or one that left the AUR, is left in `ignored_packages` unless you pick it in [`archix init`](/docs/reference/cli/#init)'s wizard or with `--repack NAME`. A picked one becomes a recipe in [`pkgbuilds/`](/docs/guides/local-pkgbuilds-and-patches/) that names its installed files by sha256, while the files themselves stay in `/var/cache/archix/sources`, out of the config, and its builds stay out of package caches, since they may hold what isn't yours to share. So another machine sharing the config can't build it. A config file you edited goes into the recipe as the package shipped it, from the package file in pacman's, yay's or paru's cache, or not at all. Rebuilding needs every other file the package owns to be as pacman installed it, and the same of the foreign packages it depends on; init says why when one can't: `! nvidia-dkms-custom is in no repo or the AUR, and archix leaves it alone: usr/lib/modules-load.d/nvidia.conf changed since pacman installed it`.

## Trust and signatures

**Install scriptlets run as root.** Once you approve an AUR recipe, its package is installed by pacman like any other, and its install scriptlet runs as root inside the new deployment's staging tree. That tree is cut off from the running system, but the scriptlet can change anything in the deployment you'll boot. Review is the safeguard: a new package's review shows every file it ships, its `.install` file included, so read what that does before you trust a recipe.

**Rebuilding rollbacks trust the pinned sha256, not today's keyring.** In prepare, archix checks every package file against the sha256 in the lock, and has pacman check every incoming package's signature, so a bad signature or an unknown key stops a switch before anything changes. A rollback that rebuilds a dropped generation skips the signature: it installs packages that were verified when they were first installed, by the sha256 the generation pinned, so a packager key revoked since doesn't block going back. That also means a key revoked because it was compromised doesn't block it either. Packages archix built itself, from the AUR or your recipes, aren't signed; the lock and the build are what you trust.

**An AUR build the lock has no sha256 for is trusted as the cache says.** Packages from a [package cache](/docs/concepts/lockfile/#package-caches) are checked against the sha256 in the lock, like any download, AUR builds included: the lock records the first build of each recipe commit, and a cache's build of the same commit that differs is passed over. Until some machine has built it, as right after `archix update` locks a new commit, the lock has no sha256 for it, and archix takes the build the cache's record names, checked only against the record's own hashes. Whoever can write to the cache can hand your machines that package. List only caches you control.

## Secure Boot and the TPM

**Old signed boot entries stay bootable unless you revoke them, and revoking is experimental.** A copy of any UKI archix signed boots wherever it's put. [`archix secure-boot revoke`](/docs/guides/secure-boot/#revoking-old-boot-images) adds the ones archix no longer uses to `dbx`; it's tested on OVMF, not on real firmware, whose variable stores vary. It needs sbctl's KEK as a file, adds at most 128 entries, and leaves old signed copies of systemd-boot alone. SBAT isn't used: without shim, nothing enforces it.

**PCR 11 bindings break when the UKI changes; signed policies are experimental.** A switch that changes the kernel, initramfs or command line writes a new UKI, which measures differently into PCR 11, so a secret sealed to PCR 11's values has to be re-sealed. With systemd's PCR signing key in place, archix signs a policy into every UKI for secrets bound to the key ([TPM-bound secrets](/docs/guides/secure-boot/#tpm-bound-secrets)), but no test unlocks such a secret with a TPM yet, so don't rely on it for a disk you can't open otherwise.

**Which deployment boots isn't signed.** The UKIs and their command lines are signed. Which deployment a boot mounts comes from the boot menu entry, `loader/entries/archix-<n>.conf`, an unsigned file on the FAT ESP, which systemd-boot reports in the `LoaderEntrySelected` EFI variable. archix keeps a copy of every entry it writes on the root filesystem, and boots `current` when the entry that started has a name it didn't record, so a renamed entry doesn't move the system. An edited entry isn't checked against archix's copy when the deployment is mounted yet. Someone who can write to the ESP can still change `loader.conf` or the boot menu's EFI variables so that an older entry archix wrote starts, which boots that kept deployment; the boot menu offers it anyway.

**The signing keys sit on the root filesystem.** sbctl's Secure Boot keys, in `/var/lib/sbctl/keys`, and systemd's PCR signing key, `/etc/systemd/tpm2-pcr-private-key.pem`, are plain files that root can read, and so can anyone who mounts an unencrypted root from another system. With the db key they can sign a boot image your firmware accepts, with the KEK they can change `db` and `dbx`, and with the PCR key they can sign a policy that unlocks a secret bound with `systemd-cryptenroll --tpm2-public-key` for any UKI they build. Every older UKI archix signed also carries a valid signed policy, so it unlocks such a secret too until it's [revoked](/docs/guides/secure-boot/#revoking-old-boot-images). Keep the root filesystem encrypted, and treat root on the machine as owning these keys; sbctl can keep its keys in a TPM or a YubiKey instead, though archix can't then sign `dbx` updates.

See [Secure Boot](/docs/guides/secure-boot/) for what archix does sign and check.

## Disk space

**A kept deployment costs an update's changed files.** With `deployments.keep = 2`, the previous deployment keeps every file the last switch replaced: 2.9 GB for a four-month update of 214 packages, 0.9 GB for a week's 36, on ext4 or btrfs, and 1.7 GB and 0.59 GB on btrfs with zstd. `deployments.keep = 1` costs the same as updating in place, once the new deployment has booted. Each kept deployment's boot image takes up to about 40 MB on the ESP. See [Storage](/docs/concepts/atomicity/#storage).

**The free-space check is an estimate.** Before a switch or rollback builds anything, archix checks every mount the incoming files land on, the way pacman does, from the file sizes in each package's `.MTREE` rounded up to whole blocks, plus a cushion; see [Signatures and disk space](/docs/concepts/safety/#signatures-and-disk-space). It doesn't count directories, inodes or a filesystem's own metadata. Official packages to download are counted ahead from the snapshot's repo databases; third-party packages and AUR builds from a package cache aren't, and a full cache disk stops the switch while those download, before anything changes. If the root fills up anyway while the deployment is built, the build fails and its staging tree is deleted; nothing was changed.

## Not yet known

- **Power-cut replay on archix's own code.** Replaying every write of a switch to each flush point has passed on the prototype archix's commit order follows; on archix itself it's in progress. Kill tests of the staging and commit logic and VM power cuts during staging pass.
- **Weeks of use on real hardware.** The failure paths are exercised in VMs and containers: VM power cuts and kills mid-switch, a full disk, a broken glibc, a revoked key. archix with deployments hasn't yet run for weeks on real machines, which is where the failures nobody thought to test turn up.
- **Upgrading from one release with deployments to the next.** The VM suite for that waits for a second release.

[Feature status](/docs/concepts/status/) lists what's shipped, partial and planned, and what tests each.
