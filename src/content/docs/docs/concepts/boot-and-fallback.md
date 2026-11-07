---
title: Boot and fallback
description: What falls back by itself on which boot loader and filesystem, the boot entries archix writes, boot counting with the health check, how long a fallback takes, and try mode.
sidebar:
  order: 9
---

With systemd-boot 260 or newer and [rollbacks](/docs/concepts/snapshots/) on btrfs, a broken update doesn't need you at the machine. archix gives the current system's boot entry two tries after every change; a boot that doesn't reach `boot-complete.target` and pass the health check uses one up, and once they're gone systemd-boot starts the snapshot of how the system last booted fine, read-only. You keep it with `archix rollback N`, or reboot into the current system once it's fixed.

## What falls back by itself

| | btrfs + systemd-boot 260 or newer, rollbacks on | btrfs + GRUB | btrfs + Limine | ext4, or rollbacks off |
| --- | --- | --- | --- | --- |
| A power cut during a package update never leaves the machine unbootable (1) | yes | best effort | best effort | best effort |
| A broken update falls back by itself (2) | yes | no: you pick an entry | no: you pick an entry | no |
| A broken update never needs a live USB | yes | if you pick an entry | only through limine-snapper-sync's snapper entries, which archix doesn't control | no |
| Reproduces the declared state (3) | yes | yes | yes | yes |

1. A package update is a libalpm transaction: one `archix switch`, `install`, `remove` or `upgrade` runs, or one a front end runs on its own, `pacman`, `yay`, `paru` or `pamac`. Either way the current entry gets its tries and the slot its entry before anything changes, and the update is marked unfinished until it's on disk, so a boot it cut off falls back to the slot. Before snapshots start, at the boot after the first switch, no package update runs at all: that switch lays the root out and [stops for the reboot](/docs/concepts/limitations/#power-cuts). A bad config can still need a live USB, as on NixOS. The ESP is FAT, which has a few spots no ordering covers; see [Power cuts and the ESP](#power-cuts-and-the-esp).
2. Broken means a boot fails boot counting's checks: it doesn't reach `boot-complete.target`, or fails the [health check](#boot-counting-and-the-health-check). This covers updates through `archix switch` and through plain pacman.
3. Not covered: packages you keep untracked with [`drift = "keep"`](/docs/concepts/pacman-and-yay/#drift), `/var/lib` and home, `npm -g` and installers in `/opt`, CachyOS builds older than archix's package cache, AUR builds, which aren't bit-for-bit, and AUR packages deleted since.

`init` says what booting looks like on the machine: on Limine, that a broken update doesn't fall back by itself; with no systemd-boot, that the entries and automatic fallback need it.

## Boot entries

Set [`boot.loader`](/docs/reference/options/#bootloader) to `"systemd-boot"`, and every switch and every plain pacman transaction that changes the kernel or initramfs writes these entries to the ESP's `loader/entries`:

| Entry | Title | Boots |
| --- | --- | --- |
| `archix-current.conf` | Arch Linux (archix) | the current system, `@` |
| `archix-snapshot-<n>.conf` | archix snapshot 6, read-only: 2026-10-30, generation 42, before pacman transaction | snapshot 6 in [try mode](#try-mode) |
| `archix-current-<kernel>.conf`, then `~1`, `~2` | one per other installed kernel, and the [protected entry](/docs/guides/secure-boot/#the-protected-entry) | the current system |
| `archix-rescue.conf` | archix rescue (kernel 7.2.9-arch1-1) | the [rescue prompt](/docs/guides/recovery/#without-a-live-usb) |

Snapshots taken by switches get an entry, and so do snapshots before pacman transactions that changed the kernel or initramfs, as those boot their own kernel. The menu holds the newest [`rollback.entries`](/docs/reference/options/#rollbackentries), 5 by default, plus the [fallback slot](#the-fallback-slot), and an ESP space check refuses to write more than fits. Snapshots with an entry are never pruned.

Every entry starts a shared boot image in `EFI/archix/` on the ESP, a unified kernel image built with `mkinitcpio` and `ukify`: one per distinct kernel and initramfs, so entries whose snapshots have the same kernel share one. Each is signed under [Secure Boot](/docs/guides/secure-boot/). The image's initramfs reads which entry systemd-boot started, maps it to its snapshot through archix's records, and mounts that snapshot. Records are versioned, and an older snapshot's initramfs reads ones a newer archix wrote.

### Entry file names

The names above are entry ids. On the ESP a counted entry's file name also carries systemd-boot's counter, `archix-current+2.conf` for two tries left. archix never rewrites an entry file under the name it has: a changed entry is written under the other spelling of its counter, so `archix-snapshot-3+1.conf` becomes `archix-snapshot-3+1-0.conf` and back again the next time it changes. Both mean one try left and none used, and systemd-boot and `systemd-bless-boot` read them the same way. An uncounted entry, as with boot counting off and for the entries of other kernels and the protected entry, gets a version instead: `archix-current.conf` becomes `archix-current~1.conf`, then `archix-current~2.conf`. Other kernels' entries are uncounted even with boot counting on, so they're written first as `archix-current-linux-lts.conf` and get a version, `archix-current-linux-lts~1.conf`, the first time they change, a higher one each time after. To pin one with `bootctl set-default` or `bootctl set-oneshot`, use the name it has on the ESP now (`bootctl list` shows it), or a glob that matches every version, `archix-current-linux-lts[.~]*`; the plain name `archix-current-linux-lts.conf` stops matching once the entry has changed. That's deliberate: a power cut can then leave the old file or the new one, never a half-written file under the only name. The id, what `loader.conf`'s `default` and the `LoaderEntrySelected` variable name, stays `archix-snapshot-3.conf`. A blessed entry, one that's lost its counter, gets a counted name again when it changes.

If a cut leaves two files for one entry, archix keeps the whole one, with a title and an image that's there, preferring the booted or the counted one; when neither is whole it keeps both, and the next refresh writes the entry again. `archix status` warns about a snapshot whose entry is missing or torn: `warning: snapshots 5, 6 should have boot entries, but the ESP has none or a torn one; the next switch or pacman transaction writes them again`.

### loader.conf

`loader.conf` gets `preferred archix-*` and `default archix-current[.~]*`, which matches the current entry under any of those names but not `archix-current-protected.conf`. archix reads whether the systemd-boot that started the machine can count boots, 260 or newer, from its `LoaderFeatures` variable, and writes uncounted entries otherwise: ``boot counting is off: the systemd-boot that started this boot is older than 260, so a generation that doesn't boot isn't passed over; `sudo bootctl update` installs the one your systemd package has``.

Your `kernel_params` must contain `root=`. archix declares what it needs to build images, the `boot.kernel` package, `mkinitcpio`, `systemd-ukify` and `dosfstools`, so you don't list them.

## The fallback slot

The slot is the first snapshot of any kind taken since the last boot that counted as good. It moves on only when a boot counts as good again. So after `archix switch`, `pacman -Syu` and `yay -Sua` in a row, a boot that fails falls back to the system as it last booted fine, from before the switch, not to the snapshot just before `yay`.

Entries carry sort keys, so when the current entry runs out of tries systemd-boot's next pick is the slot, then the other snapshots, newest first. A snapshot's entry has one try, so a fallback that fails too passes on to the next.

archix never writes a menu without its slot. If the slot's boot image is missing, the menu isn't written, and the step before a switch or a transaction warns that a boot it breaks may not fall back. The one exception is an ESP with no current entry left at all, as after it was wiped before a keep from a live USB: the slot is then dropped, with a warning, so the current entry gets written.

## Boot counting and the health check

After any change, the current entry gets [`boot.tries`](/docs/reference/options/#boottries) tries, 2 by default, written explicitly into its file name, as `archix-current+2.conf`. A counted boot counts as good, and systemd-boot's counter is cleared, once it reaches `boot-complete.target` and archix's health check passes. By default the check needs:

- the default target reached;
- no unit failed that hadn't already failed at the last boot that counted as good;
- the display manager active, if one is enabled.

A unit that's failing on a good boot before a change is noted by the switch or the transaction, so it doesn't make the next boot fall back. More checks, such as a port that must listen or a host that must answer, are in [`boot.health_check`](/docs/reference/options/#boothealth_check). The check retries for `boot.health_check.timeout`, 120 seconds. A failed check marks the entry bad, so the next boot falls back; the login note and `archix status` say which check failed. `init` runs the check once and shows what it would conclude on that machine now.

A counted boot of an archix entry whose text isn't what archix wrote, as a cut while writing it or an edit leaves it, fails its check too, `archix-current.conf on the ESP isn't what archix wrote, as a cut while writing it or an edit leaves it, so this boot can't count as good`, so it falls back; the login note about an edited entry stays. So does a counted boot whose entry isn't on a mounted ESP, as after a cut mid-kernel-update that left the old kernel without its modules. So does a boot that finds a switch or a [plain pacman transaction](/docs/concepts/snapshots/#snapshots-before-plain-pacman) marked unfinished, because a power cut or a crash stopped it. The check is then named after what was cut off, `switch` or `pacman-transaction`, and its note says what to do:

```
archix: archix-current.conf failed its health check switch (the switch to generation 43 started at 2026-11-04 09:12:40 UTC was cut off before it finished, so the packages and files it was changing may be torn; the fallback slot, snapshot 12, is the system from before it: `archix rollback 12` keeps it, or boot this entry again and run `archix switch` again); its entry is marked bad, and the machine reboots into the fallback now
```

### Nobody at the machine

A boot can hang before `boot-complete.target`, or stop in an emergency shell, and nothing would ever reboot it. archix pulls in a timer early in every counted boot, emergency and rescue mode included: once the boot has gone [`boot.unblessed_timeout`](/docs/reference/options/#bootunblessed_timeout) minutes without counting as good, 10 by default, or has sat in emergency or rescue mode for 2 minutes (or `boot.unblessed_timeout`, if that's shorter), with nobody logged in, it reboots, so the counter runs down and systemd-boot falls back. The emergency shell's message says so:

```
archix: this boot hasn't counted as good within 10 min, with nobody logged in; rebooting so the boot counter can fall back
```

A boot of a switch or transaction a power cut stopped part way, which the unfinished mark fails, reboots into the fallback at once: its root is torn, so there's nothing on it worth waiting for. Any other failed health check waits for the same timeout, so you can look at the boot while you're at it; set [`boot.health_check.reboot_on_failure = true`](/docs/reference/options/#boothealth_check) to reboot into the fallback at once instead.

A broken glibc stops init and panics the kernel. archix's counted images carry `panic=10`, unless your command line sets `panic=` itself, so the machine reboots instead of hanging there.

### How long a fallback takes

From the bad boot to the fallback booting, with nobody at the machine, measured in the VM suites:

| What broke | What reboots it | Measured with | Measured |
| --- | --- | --- | --- |
| A broken glibc, init can't start | the kernel's `panic=10`, at each of the 2 tries | `unblessed_timeout = 10`, the default, which a panic doesn't wait for | about 35 s |
| A unit that hangs the boot | `boot.unblessed_timeout`, at each of the 2 tries | `unblessed_timeout = 1` | about 2 min; at the default 10, expect about 20 min (not measured) |
| A missing required mount that stops it in emergency mode | 2 minutes in emergency mode with nobody logged in, at each of the 2 tries | `unblessed_timeout = 10`, the default | 263 s for both tries |
| A unit that fails, so the health check fails | the check marks the entry bad, then one `boot.unblessed_timeout` | `unblessed_timeout = 10`, the default | about 10 min |
| A switch or plain pacman update cut off part way by a power cut | the unfinished mark fails the check, which marks the entry bad and reboots at once | `unblessed_timeout = 10`, the default, measured on a plain pacman kernel update cut after its new image was built | 16 s after the cut |

Add your firmware's own boot time. A lower `boot.unblessed_timeout` falls back sooner, and gives you less time at a stuck boot before it reboots under you.

## Try mode

Picking a snapshot's entry boots it read-only, with a throwaway writable layer over the root and `/var/lib`, so it comes up as it was without changing the snapshot. An automatic fallback is a try boot too. The layer is a preallocated file in `/.archix/try`, 4 GiB, outside the room kept for a keep; on a nearly full disk it falls back to 1 GiB, then 256 MiB, then memory, so a fallback after a failed update still comes up. A size is used only with that much free plus room for the filesystem's own writes, a sixteenth of it or at least 64 MiB, and the formatted file is flushed and read back before it's mounted; the console says why a size was passed over, such as `archix: no 4096 MiB layer file: only 2210 MiB free, under the 4352 MiB it needs with room for the filesystem's own writes`. Past the layer's size, writes fail with "no space left on device".

The layer is gone at the next boot, and so is everything written to the root or `/var/lib` in it. Your home isn't part of it: changes there stay. A try boot shows a notice at login, in the motd, in `archix status`, and as a desktop notification:

```
archix: this boot is snapshot 6, read-only with a throwaway layer (try mode): changes are lost at the next boot, even if you keep it. `sudo archix rollback 6` keeps it from the next boot on; a plain reboot goes back to the current system's entry. The layer holds 4096 MiB of changes; past that, writes fail with "no space left on device".
```

After an automatic fallback, the login notes and `archix status` also say why the entry it fell back from failed, a health check or a required mount that was missing:

```
archix: the boot of archix-current.conf failed the health check "no unit failed since the last good boot" at 2026-11-04 09:20:13 UTC (foo.service failed); switch again once it passes
archix: archix-current.conf failed its boot tries because /home (UUID=0badc0de-0000-4000-8000-000000000000) was missing, not because of the system, so systemd-boot fell back; reattach it and reboot
```

pacman refuses in a try boot, ``booted snapshot 6 read-only; `archix rollback 6` to keep, or reboot``, and so does `archix switch`. To keep the snapshot, run [`archix rollback 6`](/docs/concepts/snapshots/#keeping-a-snapshot): it becomes the root at the next boot. To go back to the current system, reboot.

### A missing snapshot

If the snapshot an entry names is gone, deleted by hand say, the initramfs boots the newest surviving snapshot whose entry hasn't failed, in try mode, and with none left the rescue entry. It never boots an entry that failed.

## Power cuts and the ESP

The ESP is FAT, which has no atomic rename. archix writes it so a cut leaves the old file or the new one: each file's data is on the disk before any name points at it, then it's renamed, then the old name is removed, with the whole filesystem flushed between steps. A few files keep one name, and a cut in the moment they're written is covered only by best effort:

- **A snapshot entry whose image an older archix built.** That image's initramfs doesn't map a versioned name to its snapshot and would boot the current root instead, so with boot counting off such an entry keeps its one name, and a changed one's old file is removed before the new one is named. A cut then leaves that entry missing until the next switch or pacman transaction writes it. Every other uncounted entry gets a versioned name.
- **`loader.conf` and the rescue entry** are rewritten in place. A cut while one is written can leave it empty, and systemd-boot then starts its own first entry, uncounted, with no fallback. The next boot's check writes them again, a try boot's too, `loader.conf` once the current entry is there and the rescue entry from its record while its image is on the ESP, and so does a switch with nothing else to do. archix keeps a copy of `loader.conf` beside the menu's state, `/.archix/boot/loader.conf`, refreshed by every write and by each boot that finds it whole, so an emptied one comes back with your own lines, such as `timeout` and `console-mode`, not only archix's `default` and `preferred`; `archix status` warns until then.
- **A boot loader archix signs in place.** Under Secure Boot, archix signs systemd-boot's own file in `/usr`, `/usr/lib/systemd/boot/efi/systemd-bootx64.efi`, into `systemd-bootx64.efi.signed` beside it, whenever that copy is missing, older or not signed by archix's key; `bootctl install`, `bootctl update` and `systemd-boot-update.service` copy the signed file to the ESP, so a systemd update reaches the ESP signed without archix rewriting the loader there. A loader on the ESP that archix's key hasn't signed is still signed in place, as a step of its own: only when Secure Boot is first set up, and for Limine's loader, which has no such copy in `/usr`. A cut during that step can leave the machine with no loader until you reinstall it from a live USB, with `bootctl install` and `sbctl sign`.
- **The removable-media loader**, `EFI/BOOT/BOOTX64.EFI`, which the [move from Limine](#moving-from-limine-to-systemd-boot) replaces, keeps its one name too. The previous one is kept as `BOOTX64.EFI.before-archix`, on the disk before the replacement starts.

## GRUB and Limine

archix fully supports systemd-boot. On GRUB and Limine a broken update has to be picked out by hand:

- **GRUB.** `boot.loader = "grub"` writes the same entries, chainloading the same images, into `/boot/grub/archix.cfg`, which a script in `/etc/grub.d/41_archix` makes `grub.cfg` read; archix runs `grub-mkconfig` once for it. GRUB counts no boots, so nothing falls back by itself: pick a snapshot in the menu, and it boots in try mode.
- **Limine.** `boot.loader = "limine"` writes no Limine entries and leaves `limine.conf` to limine-snapper-sync. Fallback is a manual pick of a snapper snapshot, which archix doesn't control, and every such restore replaces `@`, so archix asks you to [`archix resync`](/docs/concepts/snapshots/#a-root-archix-didnt-produce) after it. A keep refuses when the snapshot's kernel or initramfs differs from the current one, since Limine boots the ones on the ESP. `init` never offers to mask snap-pac on Limine.

### Moving from Limine to systemd-boot

On a Limine machine, `init` says automatic fallback needs systemd-boot, recommends it, and offers to move the machine. Setting `boot.loader = "systemd-boot"` there and switching does the same. The move is careful not to leave a boot it can't come back from:

1. The switch puts systemd-boot on the ESP, signed under Secure Boot, adds a firmware entry for it after Limine's, and has the firmware try it once at the next boot: `systemd-boot is on the ESP with a firmware entry (0003) after Limine's, and the next boot tries it once; Limine stays first until a boot through systemd-boot counts as good, then systemd-boot goes first`. A boot that fails goes back to Limine.
2. After the first boot through systemd-boot that counts as good, systemd-boot goes first in the boot order and takes the removable-media path, `EFI/BOOT/BOOTX64.EFI`, with the old file kept beside it as `BOOTX64.EFI.before-archix`.
3. Then limine-snapper-sync's and CachyOS's Limine pacman hooks and units are masked, so they stop copying kernels to the ESP. Limine stays installed.

`archix status` shows the move while it's in progress. It needs `efibootmgr`.

## How it's tested

The VM boot suites run each of these with nobody touching the machine, on btrfs: boot counting with a kernel that panics; a broken glibc from plain pacman, days after the last kernel update; a unit that hangs; a missing required mount; a plain pacman transaction whose unit fails the health check; the health check without a display manager and with a unit that was already failing; the fallback order after a switch, a kernel update and a broken transaction; try mode by hand, filling its layer and on a nearly full disk; a deleted snapshot under the guard; a masked hook; and keep. An ext4 lane checks that switches and reverts work with rollbacks off. GRUB and real Limine firmware aren't in the VM suites yet; see [Feature status](/docs/concepts/status/).
