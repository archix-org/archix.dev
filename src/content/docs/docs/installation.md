---
title: Installation
description: Install archix from the AUR on Arch Linux or CachyOS, and check that your machine meets the requirements.
sidebar:
  order: 2
---

archix ships as one AUR package, `archix-git`, which builds the newest commit on `main` from source. It installs the tool, plus the static `archix` binary that the initramfs uses to mount [deployments](/docs/concepts/atomicity/). There's no release package or prebuilt binary yet.

## Requirements

- **An Arch-based system on x86_64.** Arch Linux and CachyOS are tested, each in its own VM suite. Other Arch-based systems such as EndeavourOS should work but aren't tested. Boot entries build their own initramfs with mkinitcpio, which archix installs, even where the system's own boot uses dracut. CachyOS is the primary target, so its repos get special care (see [Third-party repos](/docs/guides/third-party-repos/)).
- **systemd-boot**, installed with `bootctl install`. archix boots every generation as its own deployment from its own systemd-boot entry, and refuses to switch without it. Entries are unified kernel images (UKIs) built with `mkinitcpio` and `ukify`. You don't install those yourself: with `boot.loader` set, archix declares the kernel, `mkinitcpio` and `systemd-ukify`, plus `sbctl` with `boot.secure_boot = true`. GRUB and Limine are not supported.
- **The ESP outside `/boot`**, at `/efi` by default, since each deployment has its own `/boot`. Many installs mount it at `/boot`; move it before the first switch, which refuses otherwise. See [`boot.esp`](/docs/reference/options/#bootesp).
- **sbctl keys**, optional. If `sbctl` has keys, or you set `boot.secure_boot = true`, archix signs every UKI and systemd-boot itself, so Secure Boot stays on. `sudo archix secure-boot setup` creates the keys and enrolls them; see [Secure Boot](/docs/guides/secure-boot/).
- **ext4 or btrfs**, with `/boot`, `/usr`, `/opt`, `/var`, `/var/lib` and `/var/lib/pacman` on the root filesystem. On btrfs, deployments are snapshots; on ext4, unchanged files are shared as hardlinks. Caches and logs on their own btrfs subvolumes, as CachyOS lays them out, are fine. See [Requirements](/docs/concepts/atomicity/#requirements).
- **base-devel and git** for the bootstrap below, and **devtools** for AUR builds later, which run in a clean chroot with `makechrootpkg`.

## Bootstrap from the AUR

You don't have an AUR helper yet, or you're about to replace the one you have. Build archix the way you'd build yay for the first time:

```console
$ sudo pacman -S --needed base-devel git devtools
$ git clone https://aur.archlinux.org/archix-git.git && cd archix-git && makepkg -si
```

`archix-git` builds the newest commit on `main`. If you already have yay or paru, `yay -S archix-git` works too. Once archix has made its first switch, it replaces them: plain pacman and yay can't change a system laid out as deployments.

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

archix is an AUR package on your system, so `archix init` declares it in `installed.lua` like your other explicitly installed packages, and locks it by AUR commit. From then on it updates like everything else: `archix update` and `archix switch`, or `archix -Syu`, build the new version in a clean chroot and install it into the next deployment with the rest. Each deployment has its own archix, so booting an older one boots the archix it had.

Don't update it with `makepkg` again. See [Upgrading archix itself](/docs/concepts/safety/#upgrading-archix-itself).

## Removing archix

`sudo archix eject` leaves plain Arch behind: the deployment you're running becomes the root's own `/usr`, `/etc`, `/opt` and `/boot`, a UKI that mkinitcpio's preset writes becomes the boot default, and archix's entries, hooks, services and `/archix` go. Your packages, files and services stay as they are. Then remove the package:

```console
$ sudo archix eject
$ sudo pacman -Rns archix-git
```

The deployment you run has to be the current one, so reboot into a waiting switch first. An eject that's cut off finishes when you run it again. See [`eject`](/docs/reference/cli/#eject).

## The static binary

All three AUR packages also install a statically linked archix at `/usr/lib/archix/archix-static`. archix's mkinitcpio hook copies it into every initramfs it builds, where it mounts the deployment the boot entry belongs to before the system starts. The first switch refuses without it. If you built archix by hand, build the static binary from the source tree yourself, before the first switch makes `/usr` read-only. With Arch's `rust` package, the musl target comes from `rust-musl`:

```console
$ cargo build --release --target x86_64-unknown-linux-musl
$ sudo install -Dm755 target/x86_64-unknown-linux-musl/release/archix /usr/lib/archix/archix-static
```
