---
title: Recovering a broken system
description: Bring back a system no deployment boots, from archix's rescue entry or, when even that's gone, a live USB.
sidebar:
  order: 9
---

Most failures need neither of these. A bad update is undone by picking an older deployment in the boot menu, or by boot counting on its own; a lost `loader/` is covered by the fallback boot image in `EFI/Linux`. This guide is for when no deployment boots: every one broken the same way, a root filesystem that needs repair, or a password nobody knows.

## Without a live USB: the rescue entry

Every switch keeps an entry called **archix rescue** in the boot menu, after the deployments. It's never the default and boot counting never picks it; choose it in the menu, or from a running system with `sudo bootctl set-oneshot archix-rescue.conf`.

It boots the kernel of the newest deployment that booted fine, with an initramfs of its own: archix's static binary, a shell, and the tools to check and repair filesystems (`e2fsck`, `btrfs`, `fsck.fat`, and `cryptsetup` and `lvm` when installed). It mounts nothing on its own, so a damaged root can be checked unmounted, and opens the `archix rescue` prompt:

| Command | What it does |
| --- | --- |
| `list` | The deployments, which is current, and their state. |
| `use N` | Makes deployment N current and its entry the default, repairing it from the package cache first if files differ; a deployment whose entry is gone gets it rebuilt. |
| `check` | Checks and repairs the root (`e2fsck`; `btrfs check`, read-only unless `--repair-btrfs`) and the ESP (`fsck.fat`). |
| `verify [--repair]` | Compares every kept deployment with its packages, as `archix verify` does. |
| `password USER [--all]` | Sets a new password for USER in current's `/etc`, or in every deployment's with `--all`. |
| `restore-etc PATH` | Puts a packaged file such as a PAM or sudoers file back from the package cache. |
| `eject` | Ejects to plain Arch, as `archix eject` does. |
| `open DEVICE [NAME]` | Opens a LUKS volume by its passphrase, never a TPM or other token. |
| `shell`, `reboot`, `poweroff` | |

Before its prompt, the rescue entry asks for a login: the password of root or of a member of `wheel` (or `sudo`), checked against the current deployment's `/etc/shadow`. Arch locks root by default, so on most machines that's your own user's password. It allows three tries per boot, two and then four seconds apart, then restarts the machine; nothing in it falls back to a shell. If root is locked and no wheel user has a password, it opens nothing and says so; boot a deployment and set one with `passwd`, or use a live USB as below.

On an encrypted root it asks for the LUKS passphrase first, then still asks for the login, because on a locked-down machine knowing the disk passphrase shouldn't make someone root. It never unlocks with a TPM, FIDO2 or PKCS#11 token: the rescue image has no token plugins and its boot image no PCR policy, so a volume a TPM opens, even one bound only to PCR 7, asks for its passphrase there. `open DEVICE [NAME]` opens other volumes the same way.

Whoever logs in can do anything, including resetting passwords in every deployment. With Secure Boot on, the rescue entry's kernel command line is part of its signed image and can't be changed from the boot menu. Without Secure Boot anyone at the console can edit it, as they could boot a live USB. If root and wheel passwords are shared more widely than you'd like, turn it off with [`boot.rescue_entry = false`](/docs/reference/options/#bootrescue_entry).

It's tested by the `rescue` VM suite on every release: every deployment's systemd broken, then root's password lost and an entry deleted, each repaired through the rescue alone; then, under Secure Boot with root locked, a wrong and an empty password get no prompt or shell, a wheel user's password gets in, and a volume the TPM opens asks the rescue for its passphrase.

## With a live USB

You need a live USB only when the rescue entry can't start either: the ESP wiped or reformatted, systemd-boot broken, or Secure Boot refusing everything. The Arch ISO isn't signed, so turn Secure Boot off to boot it and back on afterwards; the rollback below signs the new boot image with your sbctl keys, as a switch does.

1. **Mount the root and the ESP.** On btrfs, mount the root's subvolume. archix records where the ESP goes in `/var/lib/archix/boot/esp`.
   ```console
   # mount /dev/nvme0n1p2 /mnt          # btrfs: mount -o subvol=@ /dev/nvme0n1p2 /mnt
   # cat /mnt/var/lib/archix/boot/esp
   /efi
   # mount /dev/nvme0n1p1 /mnt/efi
   # ls /mnt/archix/deploy
   ```
   The root's own `/usr` is empty, since each deployment keeps its trees under `/archix/deploy/<n>`, so `arch-chroot /mnt` alone has nothing to run.
2. **Bind a deployment over the root** with the static archix every deployment carries. Without a number it binds current's; it refuses a deployment that's missing a tree and lists the complete ones.
   ```console
   # /mnt/archix/deploy/2/usr/lib/archix/archix-static mount-deployment /mnt 2
   bound deployment 2's trees over /mnt
   ```
3. **Find what's damaged.**
   ```console
   # arch-chroot /mnt
   [root@archiso /]# archix generations
   [root@archiso /]# archix verify
   deployment 2: matches its packages
   deployment 3: 1 files differ from its packages:
     /usr/lib/systemd/systemd (systemd): mode 644, not 755
   ```
4. **Put systemd-boot back if it's gone, then roll back** to a deployment that matches, or to a damaged one whose packages are still cached, since a rollback repairs the deployment it goes to.
   ```console
   [root@archiso /]# bootctl --esp-path=/efi --boot-path=/efi install
   [root@archiso /]# archix rollback 2 --noconfirm
   ```
   `--boot-path` is needed because in a chroot bootctl would take the deployment's read-only `/boot` for a separate boot partition. The rollback makes 2 current and writes its entry, default and fallback boot image, rebuilding the image from the deployment if the ESP lost it. If only the ESP was lost, `archix rollback <current>` rewrites current's entry and changes nothing else.
5. **Reboot**: `exit`, `umount -R /mnt`, `reboot`. Other kept deployments get their entries back as you switch or roll back to them.

The `recovery` VM suite runs exactly these steps on every release, after wiping the ESP and damaging the current deployment.

## Secrets and recovery

Nothing in recovery decrypts a [secret](/docs/guides/secrets/). The rescue entry never runs the decrypt step and has no command that does: repairing a boot never needs one, and every place a value is decrypted is one more place it could leak. A logged-in rescue user can read the machine's identity from the root they mounted, as they could anything else on it. `password USER` still works for a user whose password comes from a `crypt_hash` secret: it records that the rescue set it, and boots keep it, with a login note, until a switch changes the secret's value. The rescue's `eject` drops the secrets' links, as nothing is decrypted there; the identity and ciphertexts stay in `/var/lib/archix/secrets`, so `age -d -i /var/lib/archix/secrets/identity` decrypts one by hand.

From a live USB, a deployment's links into `/run/archix/secrets` dangle, which `rollback` and `verify` don't mind: rolling back, even to a dropped generation it rebuilds, needs no identity, as each deployment carries its own ciphertexts and its boot decrypts them. A `switch` in a chroot decrypts nothing either and says so: `can't check secrets here: this runs in a chroot, so nothing is decrypted; the deployment's boot decrypts them and names any that fail`.

A lost machine identity is recovered with an admin key, not from here: see [Losing the machine's key](/docs/guides/secrets/#losing-the-machines-key).

## What still needs more than this

An ESP the firmware can't read, or firmware that refuses every boot image, can't be fixed from anything archix puts on that disk; a live USB and the steps above, or the firmware's own setup, are the way back. A machine that hasn't completed a switch since the rescue entry shipped has none yet.
