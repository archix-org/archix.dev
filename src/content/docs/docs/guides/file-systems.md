---
title: Disks, mounts and swap
description: Declare data disks, LUKS volumes, bind and tmpfs mounts, swap files and zram, and how they reach each deployment's fstab.
sidebar:
  order: 7
---

archix writes each deployment's `/etc/fstab` itself: the root's line as the deployment had it, the ESP's with its protections, and then the mounts and swap the config declares. You don't edit the fstab, and it can't be in `files`; you declare [`file_systems`](/docs/reference/options/#file_systems), [`swap`](/docs/reference/options/#swap) and [`zram`](/docs/reference/options/#zramenable), and the next boot mounts them.

## A data disk

Find the filesystem's UUID with `lsblk -f` or `blkid`, and declare where it goes:

```lua title="lua/hosts/desktop/init.lua"
file_systems = {
  ["/mnt/data"] = { device = "UUID=2b6f0c7e-…", fs_type = "ext4", options = { "noatime" }, check = true },
},
```

`archix diff` and `archix switch` list it, `fstab      mount /mnt/data (UUID=2b6f0c7e-…) at the next boot`, and the new deployment's fstab gets:

```
UUID=2b6f0c7e-… /mnt/data ext4 noatime,nofail,x-systemd.device-timeout=30s 0 2
```

The switch doesn't mount it; the reboot does, as with everything in a deployment. The running deployment's fstab doesn't have the line, so to use the disk before then, mount it by hand: `sudo mkdir -p /mnt/data && sudo mount UUID=2b6f0c7e-… /mnt/data`.

Outside `/home`, `/var` and `/srv`, archix adds `nofail` and a 30-second wait for the device. If the disk is unplugged, dies or gets another UUID, the boot goes on without it, rather than stopping in an emergency shell that a headless machine can't show anyone, and that boot counting can't fix, since the previous deployment has the same line. Its mount point stays an empty directory on the root filesystem, so a program that writes there fills the root instead. When the system can't do without a filesystem, say so with `required = true`.

Mounts at `/home`, `/var` and `/srv`, and anything under them, are required by default: without the disk, the boot stops in emergency mode. An empty `/home` is worse than a stopped boot, as the desktop would log you in and write fresh settings there, on the root filesystem, where the real `/home` hides them once it's back, and your files would seem gone meanwhile. Set `required = false` for a mount there the system can do without, such as a network share:

```lua
["/home"] = { device = "UUID=5d2a…", fs_type = "ext4", check = true },
["/home/alice/nas"] = { device = "nas:/alice", fs_type = "nfs", required = false },
```

A switch warns about a disk that isn't attached when you switch, or holds another filesystem than you declared.

## Encrypted disks

For a LUKS volume, mount its `/dev/mapper` name and say how to open it:

```lua
file_systems = {
  ["/mnt/vault"] = {
    device = "/dev/mapper/vault",
    fs_type = "ext4",
    encrypted = { device = "UUID=9a2e…", key_file = "/var/lib/keys/vault.key", options = { "discard" } },
  },
},
```

archix writes `/etc/crypttab` with `vault UUID=9a2e… /var/lib/keys/vault.key luks,discard,nofail`, and systemd opens it at boot. Without `key_file`, systemd asks for the passphrase at boot. Keep key files out of `/etc`, which is per deployment, and out of the config repo. Once any mount is `encrypted`, archix writes the whole crypttab, so it can't be in `files` too.

A `/dev/mapper` mount without `encrypted` is fine when something else opens it: a crypttab line from `files`, an `rd.luks.name=` kernel parameter, `/etc/crypttab.initramfs`, or LVM. Otherwise the switch stops before changing anything, as the boot would wait for a device nothing creates.

## Bind mounts and tmpfs

```lua
file_systems = {
  ["/srv/www"] = { device = "/mnt/data/www", options = { "bind" } },
  ["/tmp"] = { device = "tmpfs", fs_type = "tmpfs", options = { "size=4G", "mode=1777" } },
},
```

A bind mount's `device` is the directory it shows; its type is written as `none`.

## Lines archix won't drop

A line of the running `/etc/fstab` that nothing declares is left out of the next deployment's, but not when that could make data seem to vanish. The switch refuses, changing nothing, when such a line's mount point is mounted now or is `/home`, `/var`, `/srv` or under one of them, and when it would drop a crypttab line whose volume is open now or backs such a mount. That's the line added to `/etc/fstab` by hand after `archix init`, or one init couldn't capture. The refusal names each line, with the `file_systems` entry that keeps it:

```
the next deployment would leave out lines that nothing in the config declares:
  UUID=5d2a… /home ext4 rw,relatime 0 2    (/home is mounted now; /home is protected)
Booting without a mount leaves its directory empty on the root filesystem, where programs write fresh files that the mount hides once it's back. Declare each in file_systems, as `archix init` would:
  ["/home"] = { device = "UUID=5d2a…", fs_type = "ext4", options = { "rw", "relatime" }, check = true },
To drop them instead, set drop_mounts = { "/home" }; nothing was changed
```

Declare it, or list it in [`drop_mounts`](/docs/reference/options/#drop_mounts) to drop it: its mount point, or `/dev/mapper/<name>` for a crypttab volume; a volume backing a listed mount point goes with it. `drop_mounts` is in the config rather than a switch flag so the decision is written down beside the mounts, and holds for a second switch before the reboot, when the line is still in the running fstab and still mounted. Take the entry out once the deployment without the line has booted; until then the switch warns about entries neither file has, as a line added for that mount point later would be dropped without asking. A rollback can't be refused, so it warns instead when the deployment it goes back to lacks such a line.

## Editing /etc/fstab by hand

archix writes the fstab, so a hand edit lasts until the next switch. `archix status` and `archix diff` say so:

```
warning: /etc/fstab was edited since archix wrote it; the next switch writes it whole again from the config:
  + UUID=9c3e… /mnt/backup ext4 rw,nofail 0 0    (dropped then, unless file_systems declares it)
```

and warn the same way about an `/etc/crypttab` archix wrote. An edit made after a switch, while its deployment waits for the reboot, still reaches that deployment: the [/etc merge](/docs/concepts/safety/#changes-to-etc-before-the-reboot) carries `fstab` and `crypttab` into the ones archix wrote line by line, by mount point (swap by device, crypttab by name), so archix's new lines and yours both boot. Where both changed the same line, yours goes in and the deployment's whole file is kept as `fstab.pacnew` beside it. The merge says, in the login notes too, that the next switch drops the edit unless you declare it. It carries the edit rather than setting it aside, since the waiting deployment is what boots next: a `/home` line kept in a `.pacnew` would boot an empty `/home`, the very thing the switch refuses to do.

## Swap

A swap partition is a `device`; a swap file is a `file`, and with a `size` archix makes it:

```lua
swap = {
  { file = "/swap/swapfile", size = "8G" },
},
```

At boot, before swap is turned on, `archix-swapfile-swap-swapfile.service` makes the file if it's missing or has another size. On btrfs it creates `/swap` as a subvolume of its own when the directory is missing, and makes the file there with `btrfs filesystem mkswapfile`, NOCOW and preallocated, since btrfs can't swap on a copy-on-write file or snapshot a subvolume that holds an active swap file; a switch refuses a swap file it would make in an existing directory that isn't its own subvolume, such as `/swapfile` in the root's. Deployment snapshots never include it. A swap file can't be in `/archix` or in a tree a deployment keeps, such as `/etc` or `/usr`.

zram needs no disk at all:

```lua
zram = { enable = true, size = "ram / 2", algorithm = "zstd" },
```

archix declares zram-generator and writes its config; `swapon --show` lists `/dev/zram0` after the next boot.

### Hibernation

Hibernating to a swap file works through systemd: `systemctl hibernate` writes the memory image to the file and records where it is in the `HibernateLocation` EFI variable, which the initramfs's `systemd` hook reads to resume, so no `resume=` or `resume_offset=` kernel parameter is needed, on btrfs too. zram is never used for it. archix doesn't set those parameters itself, since `archix-swapfile` makes the file again when its size changes, which would leave every boot entry pointing at the old file's blocks. mkinitcpio's default HOOKS have `systemd`. With the older `udev` hooks, which resume only from `resume=`, a deployment that swaps on a file and has no `resume=` in `kernel_params` turns hibernation off through `/etc/systemd/sleep.conf.d/archix-swap-file.conf` rather than write an image nothing resumes, and the switch says so. Either way, hibernation is off while a switched deployment waits for its reboot.

## Coming from an existing fstab

`archix init` reads `/etc/fstab` into `file_systems` and `swap`, leaving the root's and the ESP's lines to archix, and keeps what each line does: `required = true` for a line without `nofail` outside `/home`, `/var` and `/srv`, `required = false` for one with `nofail` under them, and `check = true` for one with an `fs_passno`. Mounts of `/dev/mapper` names get `encrypted` from `/etc/crypttab` when every crypttab line belongs to one of them, and zram-generator's config becomes `zram`. A line archix can't declare, such as a separate `/var`, stays out, and init says why.

The first switch writes the fstab from the config. Any line nothing declares is dropped, and the switch lists it before you confirm: `fstab      drop /mnt/old (UUID=4c1f…) at the next boot, as neither file_systems nor swap declares it`, unless it's a line [archix won't drop](#lines-archix-wont-drop).

## Rollbacks

Each deployment keeps the fstab its switch wrote. Rolling back, or picking an older entry in the boot menu, boots with that fstab: a disk you declared since isn't mounted, and one you removed is mounted again. The disks themselves and swap files are outside every deployment and stay as they are.
