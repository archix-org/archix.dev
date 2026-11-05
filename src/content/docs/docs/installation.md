---
title: Installation
description: Install archix from the AUR on Arch Linux or CachyOS, and check that your machine meets the requirements.
sidebar:
  order: 2
---

archix ships as one AUR package, `archix-git`, which builds the newest commit on `main` from source. It installs the tool, plus the static `archix` binary that the initramfs uses to boot [snapshots](/docs/concepts/boot-and-fallback/#try-mode) and the rescue entry. There's no release package or prebuilt binary yet.

## Requirements

- **An Arch-based system on x86_64.** Arch Linux and CachyOS are tested, each in its own VM suite. Other Arch-based systems such as EndeavourOS should work but aren't tested. Boot images build their own initramfs with mkinitcpio, which archix installs, even where the system's own boot uses dracut. CachyOS is the primary target, so its repos get special care (see [Third-party repos](/docs/guides/third-party-repos/)).
- **A filesystem.** On btrfs, archix takes [snapshots](/docs/concepts/snapshots/) of the root before every switch and every plain pacman transaction, and lays the root out for them at the first switch: `@` for the root, `@archix` at `/.archix` and `@varlib` at `/var/lib`, beside the subvolumes you have, such as CachyOS's `@home`, `@cache` and `@log`. A root at btrfs's top level, not in a subvolume, has to move into `@` first, which `archix init` offers; see [A root at the top level](/docs/concepts/snapshots/#a-root-at-the-top-level). On ext4, archix manages the config, locks and reverts it, with no snapshots and no rollbacks; see [ext4 and rollbacks off](/docs/concepts/snapshots/#ext4-and-rollbacks-off).
- **systemd-boot 260 or newer, for automatic fallback.** archix writes an entry for the current system and for recent snapshots, with boot counting, so an update that doesn't boot falls back by itself. Boot images are unified kernel images (UKIs) built with `mkinitcpio` and `ukify`. You don't install those yourself: with `boot.loader` set, archix declares the kernel, `mkinitcpio`, `systemd-ukify` and `dosfstools` (so the ESP is checked at boot after a power cut), plus `sbctl` with `boot.secure_boot = true`. GRUB gets the same entries, picked by hand. On Limine, archix writes no entries and leaves `limine.conf` to limine-snapper-sync, and `archix init` offers to move the machine to systemd-boot. Without a managed boot loader, switches and snapshots still work, with no boot entries. See [What falls back by itself](/docs/concepts/boot-and-fallback/#what-falls-back-by-itself).
- **The ESP at `/efi`** by default, where archix keeps the boot images. On a machine that mounts it at `/boot`, `archix init` writes the config for `/efi`. See [`boot.esp`](/docs/reference/options/#bootesp).
- **sbctl keys**, optional. If `sbctl` has keys, or you set `boot.secure_boot = true`, archix signs every boot image and systemd-boot itself, so Secure Boot stays on. `sudo archix secure-boot setup` creates the keys and enrolls them; see [Secure Boot](/docs/guides/secure-boot/).
- **base-devel and git** for the bootstrap below, and **devtools** for AUR builds later, which run in a clean chroot with `makechrootpkg`.

## Bootstrap from the AUR

You don't have an AUR helper yet, or you're about to replace the one you have. Build archix the way you'd build yay for the first time:

```console
$ sudo pacman -S --needed base-devel git devtools
$ git clone https://aur.archlinux.org/archix-git.git && cd archix-git && makepkg -si
```

`archix-git` builds the newest commit on `main`. If you already have yay or paru, `yay -S archix-git` works too. They keep working after archix's first switch: archix snapshots the root before each of their transactions, and the next switch asks whether to declare what they installed.

Check that it runs:

```console
$ archix --version
archix 0.1.0
```

Then run `sudo archix init` to capture your system into a config and a lock that match it exactly, so the first `archix diff` is empty. Until there's a config, commands that need one say so: ``there's no archix config at /etc/archix yet; run `sudo archix init` to write one that describes this machine, or point --config at yours``. The [quick start](/docs/quick-start/) walks through it.

## `ax`, the short name

archix-git installs two commands: `archix`, and `ax`, a symlink to it at `/usr/bin/ax`. `ax` is the same program, with every command and flag, so `sudo ax switch` is `sudo archix switch`. Help and usage errors name the command you typed, `Usage: ax [OPTIONS] [COMMAND]`; everything else archix prints still says `archix`. This documentation uses `archix` throughout.

A few unrelated AUR packages also install `/usr/bin/ax`: `ax`, `ax-bin`, `ax-git` and `ax-cli`. archix-git conflicts with `ax`, which all of those provide, so pacman asks whether to remove the other one instead of stopping with a file conflict:

```console
:: archix-git-0.1.0.r35.g76a7920-1 and ax-bin-0.4.2-1 are in conflict (ax). Remove ax-bin? [y/N]
```

Answer no and pacman installs nothing. The two can't be installed side by side.

## About archix-git

`archix-git` builds the newest commit on `main`, not a release. It installs `/usr/bin/archix`, the `/usr/bin/ax` link and `/usr/lib/archix/archix-static`, owned by pacman, with a version such as `0.1.0.r35.g76a7920`. `archix init` declares it like any AUR package. As with every `-git` package, a switch only rebuilds it when its recipe changes, not on every upstream commit; run `makepkg -si` in the clone again to pick up new commits sooner.

The same PKGBUILD is in the archix repository, so you can also build from a checkout, or build another branch with `ARCHIX_BRANCH`:

```console
$ git clone https://github.com/archix-org/archix && cd archix/packaging/aur/archix-git
$ ARCHIX_BRANCH=<branch> makepkg -si
```

## Updates to archix itself

archix is an AUR package on your system, so `archix init` declares it in `installed.lua` like your other explicitly installed packages, and locks it by AUR commit. From then on it updates like everything else: `archix update` and `archix switch`, or `archix -Syu`, build the new version in a clean chroot and install it with the rest. Each snapshot holds the archix it had, so booting or keeping an older snapshot runs that archix, which reads the records newer ones wrote.

Don't update it with `makepkg` again. See [Upgrading archix itself](/docs/concepts/safety/#upgrading-archix-itself).

### From archix 0.2.0

archix 0.2.0 built each switch as a deployment under `/archix/deploy` and booted into it. This archix applies in place and has no migration from that model. On a system 0.2.0 set up, every command but `--help` and `--version` stops, names what it found, and leaves the system as it was:

```console
$ sudo archix switch
error: this system uses archix's deployments model, which this version no longer supports; it was left unchanged (/archix/deploy holds deployments). This archix applies configs in place and has no migration from deployments. Keep the archix that set this system up: boot an entry from before archix was upgraded and keep archix at v0.2.0 (build archix-git from the v0.2.0 tag); or reinstall Arch and set it up with this archix
```

 A 0.2.0 `archix switch` that would upgrade archix to this version fails while it builds the new initramfs, with the same sentence, before it touches the ESP or the deployments. A plain `pacman -U` of the new package is refused by 0.2.0's own pacman hook.

## Removing archix

archix applies everything in place, so there's nothing to convert back: the root is a plain Arch root, and your packages, files and services stay as they are. With [strict mode](/docs/concepts/pacman-and-yay/#strict-mode) on, removing archix is the one transaction it lets through anyway. Remove the package:

```console
$ sudo pacman -Rns archix-git
```

archix's pacman hooks in `/usr/share/libalpm/hooks` aren't owned by the package; the step after that transaction finds archix gone and deletes them, so plain pacman runs as before. archix's mkinitcpio hook stays in `/etc`, and with the static binary gone, mkinitcpio can't build it: delete `/etc/mkinitcpio.conf.d/archix.conf` and `/etc/initcpio/install/archix`, `/etc/initcpio/hooks/archix` and `/etc/initcpio/archix/`, and set up your own boot entries before the next kernel update. The btrfs layout stays: `@archix` and `@varlib` keep their `fstab` lines, and pacman's database stays bound from `/var/rollback`. The snapshots in `/.archix/snapshots` stay until you delete them with `btrfs subvolume delete`, and archix's boot entries on the ESP until you delete them, under `loader/entries/archix-*.conf` and `EFI/archix`.

## The static binary

All three AUR packages also install a statically linked archix at `/usr/lib/archix/archix-static`. archix's mkinitcpio hook copies it into every initramfs it builds, where it maps the boot entry that started to the snapshot it boots and mounts that snapshot read-only with its throwaway layer, and the rescue entry runs it before the root's own libraries are usable. If you built archix by hand, build the static binary from the source tree yourself. With Arch's `rust` package, the musl target comes from `rust-musl`:

```console
$ cargo build --release --target x86_64-unknown-linux-musl
$ sudo install -Dm755 target/x86_64-unknown-linux-musl/release/archix /usr/lib/archix/archix-static
```
