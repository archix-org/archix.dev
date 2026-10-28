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

archix adds `nofail` and a 30-second wait for the device. If the disk is unplugged, dies or gets another UUID, the boot goes on without it, rather than stopping in an emergency shell that a headless machine can't show anyone, and that boot counting can't fix, since the previous deployment has the same line. Its mount point stays an empty directory on the root filesystem, so a program that writes there fills the root instead. When the system can't do without a filesystem, say so:

```lua
["/home"] = { device = "UUID=5d2a…", fs_type = "ext4", check = true, required = true },
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

## Coming from an existing fstab

`archix init` reads `/etc/fstab` into `file_systems` and `swap`, leaving the root's and the ESP's lines to archix, and keeps what each line does: a line without `nofail` becomes `required = true`, and one with an `fs_passno` gets `check = true`. Mounts of `/dev/mapper` names get `encrypted` from `/etc/crypttab` when every crypttab line belongs to one of them, and zram-generator's config becomes `zram`. A line archix can't declare, such as a separate `/var`, stays out, and init says why.

The first switch writes the fstab from the config. Any line nothing declares is dropped, and the switch lists it before you confirm: `fstab      drop /mnt/old (UUID=4c1f…) at the next boot, as neither file_systems nor swap declares it`.

## Rollbacks

Each deployment keeps the fstab its switch wrote. Rolling back, or picking an older entry in the boot menu, boots with that fstab: a disk you declared since isn't mounted, and one you removed is mounted again. The disks themselves and swap files are outside every deployment and stay as they are.
