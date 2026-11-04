---
title: Snapshots and keep
description: archix's own btrfs snapshots of the root, before every switch and every plain pacman run; the layout they need, how long they're kept, and how `archix rollback N` makes one the root again.
sidebar:
  order: 8
---

`archix switch` applies your config to the running system, in place. On btrfs, with rollbacks on, archix first takes a read-only snapshot of the root, and it does the same before every plain `pacman`, `yay`, `paru` or `pamac` transaction. A snapshot is the whole root as it was just before the change: packages, `/etc`, pacman's database, the kernel and its modules.

You can boot a snapshot from the menu to look at it, which is [try mode](/docs/concepts/boot-and-fallback/#try-mode), and the boot menu falls back to one by itself when an update breaks the boot; see [Boot and fallback](/docs/concepts/boot-and-fallback/). `archix rollback N` keeps one: it becomes the root from the next boot. This page covers the snapshots themselves and keeping one.

archix doesn't use snapper. Its snapshots live in `/.archix/snapshots/<n>`, with a record each, and archix deletes a snapshot and its boot entry together.

## The layout

A snapshot of the root can only replace the root if it doesn't take your data with it. So rollbacks need the root in a btrfs subvolume of its own, `@`, with the things that must not roll back in other subvolumes beside it, none of them nested inside one that gets snapshotted:

| Subvolume | Mounted at | Rolls back with the root |
| --- | --- | --- |
| `@` | `/` | yes |
| `@archix` | `/.archix` | no: the snapshots, their records, root's copy of your config, archix's package cache and try mode's layers |
| `@varlib` | `/var/lib` | no: services' data, such as databases, stays as it is |
| `@home`, `@root`, `@srv`, `@cache`, `@log`, `@tmp` | where CachyOS and the Arch wiki put them | no |

Three directories under `/var/lib` belong with the root, as they describe what's installed on it: pacman's database, DKMS's state and archix's own. They live on `@` under `/var/rollback/pacman`, `/var/rollback/dkms` and `/var/rollback/archix`, and are bind-mounted over `/var/lib/pacman`, `/var/lib/dkms` and `/var/lib/archix`. They're binds rather than links because reinstalling pacman would replace a link with an empty directory. A bind that fails stops the boot: pacman with an empty `/var/lib/pacman` would think nothing is installed. The directories underneath each hold a permanent `db.lck` and a README, so pacman run without the binds, as from a live USB that mounted only `@`, refuses loudly instead. [`archix chroot`](/docs/guides/recovery/#from-a-live-usb) mounts the whole layout.

`archix init` asks whether to turn rollbacks on, on by default where the root allows it, and the first switch lays the root out. `/.archix` comes at once; `/var/lib` moves into `@varlib` at the next boot, before any service starts, so the first switch takes no snapshot and says rollbacks start after the reboot. To do it by hand:

```console
$ sudo archix snapshot setup
the root is laid out for rollbacks once you reboot: the next boot moves /var/lib into its own subvolume before any service starts, so services' data stays put when you roll back
```

The move is resumable: after a power cut partway, the next boot finishes it. In the [power-cut tests](#how-its-tested), every cut booted with the checksums of `/var/lib`'s files unchanged. Subvolumes nested in `/var/lib`, such as Docker's or machinectl's, move by a snapshot into their new place, then the old one is deleted.

### A root at the top level

Some installers put the root on btrfs's top level, subvolume ID 5, rather than in a subvolume. That can't be swapped for a snapshot, so rollbacks stay off there, and `archix status` says why: ``/ is btrfs's top level, not a subvolume, so a snapshot of it can't become the root; `archix init` or `archix snapshot setup --move-root` moves it into an `@` subvolume at the next boot``.

`init` offers the move, and `archix snapshot setup --move-root` schedules it. It needs archix to manage booting first, as the move happens in archix's initramfs:

```console
$ sudo archix snapshot setup --move-root
the root moves into an @ subvolume at the next boot, in the initramfs: don't power off while it says so. Other subvolumes at the top level stay as they are, and get fstab lines to stay mounted where they are now, which the next switch asks you to declare in file_systems; after that boot, the next switch lays the root out for rollbacks
```

At that boot, the initramfs snapshots the top level into `@`, rewrites `@`'s `fstab` so `/` is `subvol=/@`, moves subvolumes nested in the root's directories into `@`, and deletes the old root's files from the top level. It never deletes a subvolume. Like the `/var/lib` move, it's resumable, and power cuts at each of its steps are tested.

### ext4 and rollbacks off

On ext4, or with [`rollback.enable`](/docs/reference/options/#rollbackenable) off, archix is configuration management only: switches, [reverts](/docs/concepts/generations/#revert-and-keep) and [drift tracking](/docs/concepts/pacman-and-yay/#drift) work as usual, and there are no snapshots, no keep and no fallback to a snapshot. Every generation is still recorded, and `archix revert` applies an older one in place. `init` says rollback isn't available on ext4. See [What falls back by itself](/docs/concepts/boot-and-fallback/#what-falls-back-by-itself) for what that costs.

## When snapshots are taken

Every snapshot is taken before its change, so it holds the system as it was before:

| Kind | Taken | Its label |
| --- | --- | --- |
| `switch` | by `archix switch` and `archix revert`, after every check and before anything is applied; not when nothing changes | `before switch to tree 3f2a9c1e07b4` |
| `pacman` | by archix's pacman hook, before a plain pacman transaction | `before pacman transaction` |
| `manual` | by `archix snapshot create`; never pruned | `taken by hand` |
| `replaced` | by `archix rollback N`: the root it replaces | `replaced by keeping snapshot 7` |

A snapshot's record says what it contains: the config tree, the generation and the lock of the system it holds. archix lists snapshots by that state, `system as of tree 5d7b21c4e9a1 (generation 42)`, not by what happened after, so keeping a snapshot makes its tree current:

```console
$ archix snapshot list
   5  2026-10-28 09:12 UTC  switch    system as of tree 3f2a9c1e07b4 (generation 41), before switch to tree 5d7b21c4e9a1
   6  2026-10-30 18:40 UTC  pacman    system as of tree 5d7b21c4e9a1 (generation 42), before pacman transaction
   7  2026-11-02 08:55 UTC  manual    system as of tree 5d7b21c4e9a1 (generation 42), taken by hand  (before the GPU driver)
```

`archix generations` shows the same list after the generations. `archix snapshot create --note TEXT` takes one by hand, and `archix snapshot delete N` deletes one with its boot entry.

### Snapshots before plain pacman

archix doesn't wrap pacman. It ships pacman hooks in `/usr/share/libalpm/hooks` instead, so every front end on libalpm is covered: `01-archix-snapshot.hook` snapshots the root before each plain transaction, sorted after [strict mode's](/docs/concepts/pacman-and-yay/#strict-mode) hook and before every other pre-transaction hook, mkinitcpio's included, so the snapshot holds the system untouched. It skips archix's own transactions, which the switch's snapshot covers. Before the transaction changes anything, the hook also gives the current system's boot entry its tries and the [fallback slot](/docs/concepts/boot-and-fallback/#the-fallback-slot) its entry, and marks the transaction unfinished; `95-archix-post.hook` rebuilds the boot image when the kernel or initramfs changed, resets boot counting and drops the mark:

```console
archix: the kernel or initramfs changed; building the boot images
archix: the boot menu is up to date, and the current system's entry has 2 tries again
```

A boot that finds the mark, because a power cut stopped the transaction, fails its health check, so the boot menu falls back to the system from before it instead of counting a half-upgraded root as good.

Every switch writes the hooks again. `archix status`, and every boot, warn when one is masked by a link to `/dev/null` or shadowed by a file of the same name in `/etc/pacman.d/hooks`: ``archix's pacman hook 01-archix-snapshot.hook is masked by /etc/pacman.d/hooks/01-archix-snapshot.hook, so plain pacman runs take no snapshot; remove that file``. See [pacman and yay](/docs/concepts/pacman-and-yay/).

### Short of space

A snapshot pins every file the transaction replaces, so the hook estimates the transaction's whole install size first, without subtracting what it removes, plus a margin of a tenth of it or at least 64 MiB. pacman's own check subtracts the removed files, so it would pass a transaction that the snapshot then leaves short. When the root would end below [`rollback.min_free`](/docs/reference/options/#rollbackmin_free), the hook warns and asks on the terminal:

```console
archix: warning: this transaction needs about 2310 MiB with the snapshot that keeps what it replaces, and the root has 9120 MiB free; after it 8% would be left, below rollback.min_free = 10%
archix: [a]bort, [p]rune old snapshots, or [s]till try (snapshot and install anyway)? [a]
```

"still try" is offered only while the root keeps 1 GiB for a keep. With no terminal, with `--noconfirm`, or from pamac, [`rollback.low_space`](/docs/reference/options/#rollbacklow_space) decides: `"prune"`, the default, deletes the oldest snapshots it may, but only while a safe recovery point stays, a snapshot of a system that booted fine with its boot entry, and aborts otherwise; `"abort"` aborts the transaction; `"skip"` runs it without a snapshot. Each decision goes to the journal and to `archix status`.

## Retention

After each switch and transaction, archix prunes:

| What | Kept |
| --- | --- |
| `switch` snapshots | the newest [`rollback.keep_switch`](/docs/reference/options/#rollbackkeep_switch), 10 |
| `pacman` snapshots | the newest [`rollback.keep_pacman`](/docs/reference/options/#rollbackkeep_pacman), 10 |
| `manual` snapshots | all, until you delete them |
| Snapshots with a boot entry | all; the menu holds the newest [`rollback.entries`](/docs/reference/options/#rollbackentries), 5, plus the fallback slot |

Below `rollback.min_free`, 10% free by default, or 1 GiB, which is room for one keep, archix drops the oldest snapshots it may, never manual ones, ones with a boot entry or the newest. It waits for btrfs to free their space and measures with `btrfs filesystem usage -b`, since `df` lags behind a deleted subvolume. When no snapshot may go and the root is still short, it drops cached packages it can fetch again from the Arch Linux Archive, then warns, naming the package cache's size.

## Keeping a snapshot

`archix rollback N` keeps snapshot N: it becomes the root, `@`, from the next boot, and the running system is kept as a snapshot of kind `replaced`, with a boot entry of its own:

```console
$ sudo archix rollback 5
keeping snapshot 5: 2026-10-28 09:12 UTC  switch    system as of tree 3f2a9c1e07b4 (generation 41), before switch to tree 5d7b21c4e9a1
from the next boot it is the root; the running system is kept as a snapshot of kind replaced. Package changes and /etc edits made before that reboot are lost
/var/lib stays: package-owned files there go back to the snapshot's versions where unchanged, and files of packages installed since move to /var/lib/.archix-moved/5
keep snapshot 5? [Y/n]
snapshot 5 is now @, from the next boot; the running system is snapshot 8
archix: rollback to snapshot 5 is pending: it becomes the root at the next boot. Package changes and /etc edits made before that reboot are lost; `sudo archix rollback --cancel` keeps the running system instead
```

archix makes a writable clone of N, carries accounts into it, takes out pacman's lock, copies back package-owned files under `/var/lib`, then swaps the clone with `@` in one rename, so the root keeps the name `@` and nothing in `fstab` or on the kernel command line changes. It applies N's config from root's own verified copy of N's generation, never from your config directory. The current boot entry boots N's image from then on, with fresh tries.

Until the reboot, pacman and `archix switch` refuse: ``rollback to snapshot 5 is pending; reboot, or `archix rollback --cancel`; nothing was changed``. The pending notice shows at login and in `archix status`. `archix rollback --cancel` swaps the running system back, deletes the `replaced` snapshot and undoes the `/var/lib` changes:

```console
$ sudo archix rollback --cancel
the rollback to snapshot 5 is cancelled: the running system stays the root, and snapshot 8 is gone again
```

Each step of a keep and of `--cancel` is power-cut tested: the next boot finishes a keep the cut stopped after its swap, and drops one it stopped before.

You can keep a snapshot from a [try boot](/docs/concepts/boot-and-fallback/#try-mode), as after an automatic fallback: the accounts then come from the `@` being replaced, not from the try boot's throwaway `/etc`. Picking a snapshot in the boot menu only tries it; `archix rollback N` is the only way to keep one, and the rescue entry's `rollback N` runs it.

A keep is different from a revert, which applies an older generation's declared state live and leaves the rest as it is; see [Revert and keep](/docs/concepts/generations/#revert-and-keep).

On Limine, the kernel and initramfs live on the ESP, shared by every snapshot, so a keep refuses when N's kernel or initramfs differs from the current one: ``Limine boots the kernel and initramfs on the ESP, and the snapshot's kernel differs from the current one's, so it would boot with the wrong one: install the snapshot's kernel first, or switch to systemd-boot; nothing was changed``.

### Accounts after a keep

A snapshot holds `/etc/passwd`, `/etc/shadow` and `/etc/group` as they were, so a keep carries accounts forward by name from the root it replaces:

- Users your config declares follow N's config.
- Users, groups and shadow entries created or changed since N are carried over, so a password you changed since stays changed, and a password you revoked stays revoked.
- Every person's account deleted since N, and every declared user deleted since, including one N's config declares, comes back locked: no password, expired, with a `nologin` shell, so its UID stays reserved and nobody can log in as it. Package accounts only N has stay as N has them.
- An account whose ID belongs to another name in N is left out, with a warning.

```console
alice was deleted since snapshot 5, so it comes back locked (no password, expired, nologin shell); `archix user unlock alice` enables a declared one
```

`archix user unlock NAME` enables a declared account a keep brought back locked, with its declared shell, and its declared password if it has one:

```console
$ sudo archix user unlock alice
alice is unlocked with shell /bin/zsh; it has no password yet: set one with `passwd alice`
```

### /var/lib after a keep

`/var/lib` is on `@varlib`, so a keep leaves it as it is, and a snapshot of `@` holds it empty. Package-owned files there would then belong to newer packages than the root has. So the keep puts each file a package of N owns back to N's version, from archix's cache, pacman's cache or the Arch Linux Archive, but only if the live file is unchanged from the current package's version; a file you or a service changed stays, with a warning, as pacman does with backup files. Files owned only by packages installed since N move aside to `/var/lib/.archix-moved/<N>/`, with a `manifest.json`, so installing those packages again doesn't stop at "exists in filesystem". That directory is kept as long as snapshot N, and its size shows in `archix status`.

Data that isn't a package's file, such as a database's, stays as it is: a database a newer version migrated stays migrated. See [Downgrades that may break data](/docs/concepts/generations/#downgrades-that-may-break-data).

Your home is never part of a snapshot, a keep or a try boot.

## A root archix didn't produce

archix records `@`'s subvolume UUID after setup and after every keep. If `@` is replaced by another tool, such as `snapper rollback`, `limine-snapper-restore`, or a restore of a backup from before `archix init`, archix's records describe another root. Every boot and `archix status` say so, and switches and keeps refuse until you adopt it:

```console
$ sudo archix resync
the root subvolume (8e1c…) isn't the one archix last produced (2b7f…): another tool (a snapper or limine-snapper-sync rollback, a restore) replaced it, so archix's records describe another root; on Limine every fallback goes through limine-snapper-restore, so this is expected there. Switches and keeps refuse until `sudo archix resync` adopts it
carried users, groups and passwords over from the root it replaced (@snapshots/41/snapshot)
recorded the root; switches and keeps work again
```

`resync` lays the root out for rollbacks again if it needs to, carries accounts over from the root it replaced if it can still find it by its recorded UUID, and records the new root. If it can't, it says so and carries nothing, rather than guess. archix's pacman hook never blocks pacman on such a root: it skips the snapshot, with a warning in pacman's output and in the boot notice.

## Other snapshot tools

archix leaves snapper, snap-pac, Timeshift and limine-snapper-sync alone, their configs and their snapshots included, and ignores their snapshots.

- **snap-pac** also snapshots every pacman run. `init` offers to mask its pacman hooks, so only archix does, or to keep both; `archix status` notes once that snap-pac takes them too. On Limine it stays on, as its snapshots are the only bootable ones there.
- **Timeshift** in btrfs mode makes two tools that manage snapshots of the root, and `init` warns. Restoring a Timeshift snapshot replaces the root behind archix's back; `archix resync` adopts it.
- **limine-snapper-sync** keeps writing Limine's menu; see [GRUB and Limine](/docs/concepts/boot-and-fallback/#grub-and-limine).

## How it's tested

VM suites boot archix-managed machines in QEMU with OVMF on btrfs, and on ext4 with rollbacks off:

- **Power cuts** at chosen steps, through a test hook in a debug build and a hard reset: during a real `pacman -Syu`'s package extraction and at each step after it, at each step of `archix rollback N` and `--cancel`, of the `/var/lib` move and of the move of a top-level root into `@`. Every cut booted unattended. After each boot the test checks that the ESP is mounted, that `archix status` runs and doesn't call the root foreign, that every entry's image is on the ESP, that no snapshot lacks its root or record and no half-done keep's clone is left, and that `pacman -Dk` and `pacman -Qkk` find nothing outside `/etc` and `/var/log`, ignoring time, permission and owner mismatches. The keep cuts also check that the root holds either the system before the keep or the kept snapshot, and that an account made before the keep is still there; the `/var/lib` cuts check that its files' checksums are unchanged.
- **What the power cuts don't test.** They reset the VM through QEMU, which keeps every write the guest had already sent to its disk. So they don't test writes a real power cut loses or reorders before they're flushed. Write-replay tests for that are in progress.
- **Keep** on a layout that mounts `@` by name in `fstab` and on the kernel command line, from a try boot, with users created, changed and deleted since, and `--cancel`; pacman refused while it's pending.
- **Records** written by a newer archix, read by an older snapshot's initramfs.

Container tests cover the hooks' order, the space estimate, retention and `/var/lib` after a keep. See [Feature status](/docs/concepts/status/).
