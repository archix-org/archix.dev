---
title: Installation
description: Install archix from the AUR on Arch Linux or CachyOS, and check that your machine meets the requirements.
sidebar:
  order: 2
---

archix ships as one AUR package, `archix-git`, which builds the newest commit on `main` from source. It installs the tool, plus the static `archix` binary that [rescue mode](/docs/concepts/safety/#rescue-mode) puts in the initramfs. There's no release package or prebuilt binary yet.

## Requirements

- **An Arch-based system on x86_64.** Arch Linux and CachyOS are tested, each in its own VM suite. Other Arch-based systems such as EndeavourOS should work but aren't tested. Boot entries build their own initramfs with mkinitcpio, which archix installs, even where the system's own boot uses dracut. CachyOS is the primary target, so its repos get special care (see [Third-party repos](/docs/guides/third-party-repos/)).
- **systemd-boot**, if you want a boot entry per generation, rollback from the boot menu and rescue entries. Entries are unified kernel images (UKIs) built with `mkinitcpio` and `ukify`. You don't install those yourself: with `boot.loader` set, archix declares the kernel, `mkinitcpio` and `systemd-ukify`, plus `sbctl` with `boot.secure_boot = true`, and the next switch installs whatever is missing. GRUB and Limine are not supported: on those bootloaders archix still manages your system and `archix rollback` still works from a running system, but you don't get generation boot entries.
- **sbctl keys**, optional. If `sbctl` has keys, or you set `boot.secure_boot = true`, archix signs every UKI and systemd-boot itself, so Secure Boot stays on. `sudo archix secure-boot setup` creates the keys and enrolls them; see [Secure Boot](/docs/guides/secure-boot/).
- **Any filesystem.** Rollback reinstalls a generation's exact packages and files from the cache, which works the same on ext4, btrfs or anything else. On a btrfs root, archix also takes a read-only snapshot of `/` before every switch, as an extra recovery point you can use by hand. See [Generations and rollback](/docs/concepts/generations/#btrfs-snapshots).
- **base-devel and git** for the bootstrap below, and **devtools** for AUR builds later, which run in a clean chroot with `makechrootpkg`.

## Bootstrap from the AUR

You don't have an AUR helper yet, or you're about to replace the one you have. Build archix the way you'd build yay for the first time:

```console
$ sudo pacman -S --needed base-devel git devtools
$ git clone https://aur.archlinux.org/archix-git.git && cd archix-git && makepkg -si
```

`archix-git` builds the newest commit on `main`. If you already have yay or paru, `yay -S archix-git` works too. Once archix has made its first switch, it replaces them.

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

archix is an AUR package on your system, so `archix init` declares it in `installed.lua` like your other explicitly installed packages, and locks it by AUR commit. From then on it updates like everything else: `archix update` and `archix switch`, or `archix -Syu`, build the new version in a clean chroot and install it in the same transaction as the rest. If an archix update turns out bad, rolling back the generation rolls back archix too.

Don't update it with `makepkg` again.

Each switch or rollback keeps a copy of the archix that started it next to its journal, so if it's cut off after installing an older archix, that older one can still hand the unfinished switch back; see [Upgrading archix itself](/docs/concepts/safety/#upgrading-archix-itself).

## Removing archix

`sudo archix eject` hands the machine back to plain pacman, mkinitcpio and systemd-boot: it writes ordinary boot entries for your kernels, makes one the default, and takes archix's own entries, hooks and boot service off. Your packages, files and services stay as they are. Then remove the package:

```console
$ sudo archix eject
$ sudo pacman -Rns archix-git
```

See [`eject`](/docs/reference/cli/#eject) for what it changes, and its flags for putting original files back and deleting archix's state.

## The static rescue binary

All three AUR packages also install a statically linked archix at `/usr/lib/archix/archix-static`. When archix manages booting, it adds an mkinitcpio hook that copies that binary into the initramfs, and gives the newest two generations an `archix rescue` boot entry. That entry repairs a system pacman can't repair, without a live USB.

Rescue entries are only built when `/usr/lib/archix/archix-static` exists. All three AUR packages, `archix-git` included, install it. If you built archix by hand, build the static binary from the source tree yourself. With Arch's `rust` package, the musl target comes from `rust-musl`:

```console
$ cargo build --release --target x86_64-unknown-linux-musl
$ sudo install -Dm755 target/x86_64-unknown-linux-musl/release/archix /usr/lib/archix/archix-static
```

The same binary also works from a live USB against the mounted system, since it has no dependencies:

```console
# mount /dev/nvme0n1p2 /mnt
# /mnt/usr/lib/archix/archix-static rescue --root /mnt
```

You should rarely need that. See [Safety](/docs/concepts/safety/#rescue-mode).
