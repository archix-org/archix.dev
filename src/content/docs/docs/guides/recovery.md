---
title: Recovering a broken system
description: Get back to a working system after a bad update or a bad config, from the boot menu, archix's rescue entry or, when even that's gone, a live USB.
sidebar:
  order: 9
---

Most failures need nothing from you. On systemd-boot, a package update that leaves the system unable to boot, or booting with failed units, falls back by itself to a snapshot of how the system last booted fine; see [What falls back by itself](/docs/concepts/boot-and-fallback/#what-falls-back-by-itself). This guide is for everything else: picking a snapshot by hand, keeping one, a config you regret, a root that needs repair, a password nobody knows.

A bad config is different from a bad package update. A switch applies it live and its snapshot holds the system from before, so most config mistakes are one revert or one keep away. But a config that locks you out, breaks the boot or the disk layout, can need the rescue entry or a live USB, as on NixOS.

## Without a live USB

### A snapshot from the boot menu

Every switch snapshot, every pacman snapshot that changed the kernel or initramfs, and the fallback slot have an entry in the boot menu:

```
  Arch Linux (archix)
› archix snapshot 5, read-only: 2026-10-15, generation 41, before switch to tree 5d7b21c4e9a0
  archix snapshot 3, read-only: 2026-10-09, generation 40, before switch to tree 3f2a9c1e07b4
  archix rescue
```

Picking one boots it in **try mode**: read-only, with a throwaway layer over the root and `/var/lib`, so you can look around and check that it works. Nothing changes on disk, the next boot goes back to the current system's entry, and pacman and `archix switch` refuse. A notice at login, in the motd, in `archix status` and on the desktop says so:

```
archix: this boot is snapshot 5, read-only with a throwaway layer (try mode): changes are lost at the next boot, even if you keep it. `sudo archix rollback 5` keeps it from the next boot on; a plain reboot goes back to the current system's entry.
```

An automatic fallback boots the slot in try mode the same way. See [Try mode](/docs/concepts/boot-and-fallback/#try-mode).

### Keeping a snapshot

To go back for good, keep the snapshot, from a try boot of it or from the running system:

```console
$ sudo archix rollback 5
keeping snapshot 5: 5  2026-10-15 08:54 UTC  switch    system as of tree 3f2a9c1e07b4 (generation 41), before switch to tree 5d7b21c4e9a0
from the next boot it is the root; the running system is kept as a snapshot of kind replaced. Package changes and /etc edits made before that reboot are lost
keep snapshot 5? [Y/n]
snapshot 5 is now @, from the next boot; the running system is snapshot 6
archix: rollback to snapshot 5 is pending: it becomes the root at the next boot. Package changes and /etc edits made before that reboot are lost; `sudo archix rollback --cancel` keeps the running system instead
```

Reboot to finish it. Until then pacman and `archix switch` refuse, and `sudo archix rollback --cancel` undoes it: `the rollback to snapshot 5 is cancelled: the running system stays the root, and snapshot 6 is gone again`. After the reboot, the system you left is snapshot 6, with a boot entry of its own, so the keep can be undone by keeping 6.

Accounts made since the snapshot come along. One deleted since comes back locked, with no password, expired and a `nologin` shell, so its UID stays reserved and nobody can log in as it:

```console
alice was deleted since snapshot 5, so it comes back locked (no password, expired, nologin shell); `archix user unlock alice` enables a declared one
$ sudo archix user unlock alice
alice is unlocked with its declared password and shell /bin/zsh
```

A declared user without a declared password is unlocked without one: `alice is unlocked with shell /bin/zsh; it has no password yet: set one with `passwd alice``. See [Keeping a snapshot](/docs/concepts/snapshots/#keeping-a-snapshot) for what a keep carries over and what it puts back in `/var/lib`.

### Reverting a config

For a config change you regret, `sudo archix revert` applies the previous generation's record live, in place, with no reboot, or `archix revert G` the one you name. It puts back what the generation declares and leaves everything else as it is. See [Reverting](/docs/concepts/generations/#reverting), and [Revert and keep](/docs/concepts/generations/#revert-and-keep) for which to use.

### The rescue entry

When nothing boots, not the current system and no snapshot, or the root filesystem needs repair, there's an entry called **archix rescue** in the boot menu, after every other archix entry. It's never the default; choose it in the menu, or from a running system with `sudo bootctl set-oneshot archix-rescue.conf`. systemd-boot also starts it on its own once every counted entry has failed, and archix's initramfs boots it when no snapshot whose entry hasn't failed is left.

It boots the current kernel with an initramfs of its own: archix's static binary, a shell, and the tools to check and repair filesystems (`e2fsck`, `btrfs`, `fsck.fat`, and `cryptsetup` and `lvm` when installed). It mounts nothing on its own, not the root's subvolumes, `@varlib` or the binds, so a damaged root can be checked unmounted, and opens the `archix rescue` prompt:

| Command | What it does |
| --- | --- |
| `check [--repair-btrfs]` | Checks and repairs the root (`e2fsck`; `btrfs check`, read-only unless `--repair-btrfs`, which can make some damage worse) and the ESP (`fsck.fat`). |
| `password USER` | Sets USER's password in the root's `/etc`, read twice from the console. |
| `open DEVICE [NAME]` | Opens a LUKS volume with its passphrase as `/dev/mapper/NAME`, never with a TPM, FIDO2 or PKCS#11 token. |
| `rollback N` | Keeps snapshot N from the next boot on: `archix rollback N` on the root with its layout mounted. Only listed with rollbacks on btrfs. |
| `shell` | A shell; the root is under `/run/archix-rescue/root`, and `exit` comes back to the prompt. |
| `reboot`, `poweroff` | Unmount everything and restart, or power off. |

Before a command, `--root DEVICE [--rootflags OPTIONS]` points it at another root, such as `/dev/mapper/NAME` after `open`.

Before its prompt, the rescue entry asks for a login: the password of root or of a member of `wheel` (or `sudo`), checked against the root's `/etc/shadow`. Arch locks root by default, so on most machines that's your own user's password. It allows three tries per boot, two and then four seconds apart, then restarts the machine; nothing in it falls back to a shell. If root is locked and no wheel user has a password, it opens nothing and says so; boot the system and set one with `passwd`, or use a live USB as below.

On an encrypted root it asks for the LUKS passphrase first, then still asks for the login, because on a locked-down machine knowing the disk passphrase shouldn't make someone root. It never unlocks with a TPM, FIDO2 or PKCS#11 token: the rescue image has no token plugins and its boot image no PCR policy, so a volume a TPM opens, even one bound only to PCR 7, asks for its passphrase there. `open DEVICE [NAME]` opens other volumes the same way.

Whoever logs in can do anything, including resetting passwords and keeping a snapshot. With Secure Boot on, the rescue entry's kernel command line is part of its signed image and can't be changed from the boot menu. Without Secure Boot anyone at the console can edit it, as they could boot a live USB. If root and wheel passwords are shared more widely than you'd like, turn it off with [`boot.rescue_entry = false`](/docs/reference/options/#bootrescue_entry).

The rescue entry is kept on systemd-boot and GRUB, refreshed after each switch and each pacman transaction that changed the kernel or initramfs; on Limine it's taken off.

### GRUB and Limine

- **GRUB** lists the same entries as systemd-boot, so picking a snapshot boots it in try mode and `archix rollback N` keeps it. GRUB counts no boots, so nothing falls back by itself: pick the entry by hand.
- **Limine** gets no archix entries. Its bootable snapshots are limine-snapper-sync's snapper snapshots, restored with `limine-snapper-restore`. That replaces the root behind archix's back, so archix asks you to adopt it afterwards, [below](#a-root-another-tool-replaced). `archix init` offers to [move a Limine machine to systemd-boot](/docs/concepts/boot-and-fallback/#moving-from-limine-to-systemd-boot) for automatic fallback.

## From a live USB

You need a live USB only when the rescue entry can't start either: the ESP wiped or reformatted, systemd-boot broken, Secure Boot refusing everything, or a system without rollbacks. The Arch ISO isn't signed, so turn Secure Boot off to boot it and back on afterwards; a keep signs the new boot image with your sbctl keys, as a switch does.

`archix chroot` mounts the root as it boots, with its layout: `@` at the root, `@archix` at `/.archix`, `@varlib` at `/var/lib`, and pacman's database, DKMS's state and archix's own state bound from `/var/rollback` over `/var/lib`, then runs `arch-chroot` there. Without the binds, pacman in a chroot would run against an empty database, so the directories under them hold a permanent `db.lck` and a README: after a plain `arch-chroot` into `@`, pacman refuses instead.

1. **Open an encrypted root** first, if it is one: `cryptsetup open /dev/nvme0n1p2 root`.
2. **Take archix from the root.** The Arch ISO has no archix, but the root carries a static one:
   ```console
   # mount -o subvol=@,ro /dev/mapper/root /mnt
   # cp /mnt/usr/lib/archix/archix-static /tmp/archix
   # umount /mnt
   ```
3. **Chroot into the root with its layout**, with a shell or a single command:
   ```console
   # /tmp/archix chroot /dev/mapper/root
   [root@archiso /]# mount /efi
   [root@archiso /]# archix snapshot list
   [root@archiso /]# archix rollback 5 --noconfirm
   ```
   `archix chroot` doesn't mount the ESP; `mount /efi` (or wherever [`boot.esp`](/docs/reference/options/#bootesp) says) mounts it from the root's `fstab`. Put systemd-boot back first if it's gone, with `bootctl install`. Inside, `archix rollback N` keeps a snapshot, and pacman works on the root's own database, to reinstall a package or roll a single one back.
4. **Reboot**: `exit`, then `reboot`. `archix chroot` unmounts what it mounted when the shell or command ends.

On a root that isn't laid out for rollbacks, `archix chroot` stops: `/dev/mapper/root has no /.archix, so it isn't laid out for rollbacks; mount it by hand`. Mount it and `arch-chroot` as you would on any Arch system.

## A root another tool replaced

A snapper rollback, `limine-snapper-restore`, Timeshift or a restore from a backup puts another subvolume in `@`'s place. archix records `@`'s subvolume UUID after its setup and each keep, and notices at boot and in `archix status` when the root isn't the one it produced. Its records then describe another root, so switches and keeps refuse until you adopt it:

```console
$ sudo archix resync
the root subvolume (…) isn't the one archix last produced (…): another tool (a snapper or limine-snapper-sync rollback, a restore) replaced it, so archix's records describe another root; on Limine every fallback goes through limine-snapper-restore, so this is expected there. Switches and keeps refuse until `sudo archix resync` adopts it
carried users, groups and passwords over from the root it replaced (@snapshots/12/snapshot)
recorded the root; switches and keeps work again
```

`resync` lays the root out for rollbacks again if it needs to, and carries accounts over from the root it replaced when it can find it by its recorded UUID. When it can't, as when the other tool moved or deleted it, it says so and carries nothing, rather than guessing; check users and passwords then. The pacman snapshot hook never blocks pacman meanwhile: it skips, with a warning in pacman's output. See [A root archix didn't produce](/docs/concepts/snapshots/#a-root-archix-didnt-produce).

## What still needs more than this

An ESP the firmware can't read, or firmware that refuses every boot image, can't be fixed from anything archix puts on that disk; a live USB and the steps above, or the firmware's own setup, are the way back. A machine that hasn't completed a switch with archix managing booting has no rescue entry yet. On ext4, or with rollbacks off, there are no snapshots to boot or keep: the rescue entry can check the disk and reset a password, and `archix revert` goes back to a generation's declared state, but a system a package update broke needs a live USB.
