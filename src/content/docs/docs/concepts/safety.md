---
title: Safety
description: How archix prepares before it builds, builds each switch as a new deployment beside the running system, manages files without losing your edits, and why an interrupted switch needs no recovery.
sidebar:
  order: 5
---

The goal is simple: no failure during a switch, including pulling the power cord halfway through, should ever need a live USB or a repair step. archix gets there by never changing the running system: every switch builds a new [deployment](/docs/concepts/atomicity/) beside it and commits it in one step, and it boots at the next restart. This page explains what a switch checks, what it builds, and what happens when something goes wrong. [Limitations and failure modes](/docs/concepts/limitations/) covers what it doesn't guarantee.

## Prepare, then build

`archix switch` runs in two phases.

**Prepare** does everything that can fail without touching anything:

- evaluates the config and renders every managed file,
- resolves the lock with pacman itself, so dependencies are consistent, and shows new AUR build recipes for review,
- checks that no locked package is built for a newer x86-64 level than this CPU runs, which happens when another machine locked the host, and stops if one is, leaving the lock for `archix lock` on this machine to fix ([CPU levels](/docs/guides/third-party-repos/#cpu-levels)),
- offers to adopt packages that are newer than the lock,
- checks that every managed file can replace what's there now, keeping a backup ([below](#managed-files-backups-and-hand-edits)),
- checks that the ESP can hold the new boot entry, and with Secure Boot on, that it will be signed ([below](#secure-boot)),
- with the [firewall](/docs/guides/environment-and-firewall/#firewall) on and `nft` installed, checks the generated rules with `nft -c -f`,
- lists the changes and asks you to confirm,
- stores every managed file's contents, and downloads files declared with a `url`, checking each against its `sha256`,
- builds every AUR package in a clean chroot,
- downloads every package and checks it against the lock,
- checks that no two packages ship the same file, and that no incoming file is already there without an owner ([below](#packages-that-ship-the-same-file)),
- has pacman check every incoming package's signature, and checks that the root filesystem has room for the new deployment ([below](#signatures-and-disk-space)).

If anything fails here, a typo, a network error, an AUR build that won't compile, a bad signature, a full disk, a declined prompt, archix stops with `nothing was changed`. Most problems surface in this phase.

**Build** then makes the next deployment from the current one, in `/archix/deploy/.staging-<n>`, and commits it:

```console
$ sudo archix switch
upgrade    linux 7.2.8.arch1-1 -> 7.2.9.arch1-1
upgrade    mesa 1:25.3.0-1 -> 1:25.3.1-1
a new deployment with its own boot entry; it boots at the next restart
apply 3 changes? [Y/n]
:: building deployment 42 from deployment 41
...
hibernation is off until you reboot into deployment 42: resuming would start its kernel on the running system's memory image; suspend still works
switched to generation 42 (snapshot 2026-10-15); its deployment boots at the next restart
deployment 42 is committed and boots at the next restart; changes to /etc made before then are carried over to it
```

Inside the staging tree, pacman installs and removes packages, install scriptlets and pacman hooks run, and archix writes users and groups, enables and disables services, and writes its managed `/etc` files. Then the new initramfs and boot image are built from the staged kernel, and the deployment is committed: renamed into place, its boot entry written, made systemd-boot's default and `current`. [Atomic deployments](/docs/concepts/atomicity/#a-switch) has the details and the commit order.

None of that touches the running system. pacman runs in a private mount and PID namespace that sees the staging tree and nothing else of the live system but pacman's log, the package cache and what `/var` needs, so nothing it starts outlives the switch, and a scriptlet can't signal your session.

### Packages that ship the same file

pacman only finds out that two packages own the same file once it's installing them. archix checks first. Once every package is downloaded or built, it reads each incoming package's file list and compares it with the other incoming packages and with the installed packages the switch keeps:

```console
error: pacman would refuse these packages over files that are already taken; nothing was changed. Drop one of each pair from the config, or move the stray files away:
  usr/lib/udev/rules.d/70-wooting.rules is in both wooting-udev and wootility
```

Packages that are leaving don't count, and that includes the ones pacman swaps out. An incoming package's `conflicts` and `replaces` name packages that go when it arrives, and an installed package whose `conflicts` or `replaces` names an incoming one goes too, as when CachyOS's `zlib-ng-compat` and Arch's `zlib` replace each other.

It also looks at the current deployment. An incoming file that already exists there, with no installed package owning it, is what pacman reports as "exists in filesystem", and archix refuses it the same way, as `usr/bin/foo from foo exists on disk, and no package owns it`. Files a package lists as `backup`, its config files, don't count, since pacman installs those as `.pacnew` instead.

A rollback that rebuilds a dropped generation runs the same check, against the deployment it builds from.

### Signatures and disk space

archix asks pacman and the disk before it builds anything, once every package is downloaded or built:

- **Signatures.** pacman loads every incoming package with `pacman -U --print`, which checks signatures and keys exactly as the install will, and installs nothing. A package it refuses, signed by an unknown, expired or revoked key or with a bad signature, stops the switch with `pacman refuses these packages, so nothing was changed:` and pacman's own errors, plus a line from gpg's reading of the signature that names the package and the signer, since pacman only says `required key missing from keyring`: ``linux-7.2.9.arch1-1-x86_64.pkg.tar.zst is signed by Jan Packager <jan@archlinux.org>, whose key expired on 2026-10-01; a newer keyring package from its repo (archlinux-keyring for Arch's) renews it``. pacman checks again in the staging tree, against the current deployment's keyring.
- **Disk space.** Like pacman, archix counts per mount. Every incoming file's size, from the package's `.MTREE` and rounded up to whole blocks, counts on the mount it lands on. The versions it replaces free nothing, since the deployment it's staged from keeps them. The root filesystem always needs a cushion of 5% of its size, at most 128 MiB, for the copies of `/etc` and pacman's database and the new initramfs, even for a switch that only removes packages. Otherwise it names the mount, which for the root filesystem is `/archix`, where deployments are staged: ``/archix has 96 MiB free, but this needs about 412 MiB; nothing was changed. Free some space (`archix gc`, `pacman -Sc`) and try again``. btrfs hands a deleted file's space back only at its next commit, so a mount that looks short gets committed and measured again before archix refuses. btrfs can also show free space and still fail writes once data chunks fill the device, so on a btrfs mount with under 1 GiB unallocated, archix first hands empty data chunks back with `btrfs balance start -dusage=0`, and if metadata still has no room it stops: ``btrfs on /archix has room for data but almost none for metadata (12 MiB of metadata free, 1 MiB unallocated), so writes would fail halfway; nothing was changed. Run `sudo btrfs balance start -dusage=10 /archix` and try again``.
- **Downloads** are counted first: before any starts, the official packages a switch has to download are sized from the snapshot's repo databases, and if `/var/cache/archix/pkg` can't hold them, the switch stops with ``/var/cache/archix/pkg has 12 MiB free, but this needs about 380 MiB; nothing was changed``. Third-party packages and AUR builds from a package cache aren't sized ahead; if the disk fills while they download, the switch stops there with ``/var/cache/archix/pkg ran out of space while downloading packages; nothing was changed``. A download cut short is never kept under the package's real name.
- **Inodes.** On filesystems that report free inodes, which btrfs doesn't, the root has to have enough for staging: the copies of `/etc`, pacman's database and DKMS state, the directories of `/usr`, `/opt` and `/boot`, and every incoming file, plus a margin. Otherwise: ``/archix/deploy/41 has 6120 free inodes, but staging this deployment takes about 48300; nothing was changed. Free some (`archix gc`, or delete many small files), or give the filesystem more inodes``.

`switch`, `rollback`, `install --temp` and the other commands that build a deployment run these checks. If the disk fills anyway while the deployment is built, the build fails, its staging tree is deleted at once, and nothing was changed. The ESP has its own check, for the boot entry; see [Boot entries](/docs/concepts/generations/#boot-entries).

A rollback that rebuilds a dropped generation installs packages that were verified when they were first installed, by the sha256 the generation pinned and without their `.sig` file, so a packager key revoked since doesn't block going back.

### Keyrings

Packages that ship a pacman keyring, the files `pacman-key --populate` reads from `/usr/share/pacman/keyrings`, get two rules: `archlinux-keyring`, and third-party ones such as `cachyos-keyring` and `chaotic-keyring`.

- **They go in first.** A newer snapshot can bring packages signed by a packager that only its newer `archlinux-keyring` knows. Plain pacman refuses those until you update the keyring on its own. archix runs the signature check above against a copy of pacman's keyring with the incoming keyrings already populated into it, and in the staging tree it installs the keyring packages on their own first, so pacman checks the rest against the keys they add. If the copy can't be made, the check warns and uses the installed keyring.
- **They never go back.** An older keyring trusts packager keys that were revoked since, and lacks the ones added since. A switch or a rebuilding rollback that would downgrade one keeps the installed version instead: `keeping archlinux-keyring 20260909-1 rather than going back to 20260602-1: an older keyring trusts keys revoked since`.

### One archix at a time

Only one archix changes the system at a time. `switch`, `install`, `remove`, `upgrade` and bare `archix`, `rollback`, `gc`, `hooks`, `verify`, `eject`, `init`, `secure-boot setup`, `revoke` and `rotate-pcr-key`, and the boot-time checks take an exclusive lock on `/var/lib/archix/running`, and a second one stops at once:

```console
error: another archix is already running (pid 4242, `archix switch`); wait for it to finish
```

Commands that only read or write the config, `status`, `diff`, `search`, `eval`, `options`, `generations`, `lock` and `update`, don't take it.

archix also checks pacman before it starts. If any process holds pacman's lock open, `switch` refuses with `pacman is running; wait for it to finish`. A `db.lck` that's left over with nothing holding it is refused too, and archix tells you why.

Packages are downloaded to `/var/cache/archix/pkg`, and every file is checked against the `sha256` in the lock before it's used. Files already in pacman's own cache are reused. A file that doesn't match its checksum stops the switch here. The same goes for files declared with [`archix.fetchurl`](/docs/reference/lua-api/#archixfetchurl):

```console
error: https://downloads.example.org/hello/1.2.0/hello-x86_64 has sha256 0c1f3a5e7b9d2f4a6c8e0b1d3f5a7c9e2b4d6f8a0c1e3b5d7f9a2c4e6b8d0f1a, but the config says 5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03
```

The tools archix runs and reads the output of, pacman among them, run with `LC_MESSAGES=C`, so they answer in English whatever your locale. Started from a login session, the commands that change the system re-run themselves in a systemd scope of their own (`archix-<pid>.scope`), so logind ending the session, as it does when an ssh connection drops with `KillUserProcesses=yes`, doesn't reach them.

## What takes effect when

The new deployment takes effect at the next boot, all of it at once: packages, `/etc` files, users and groups, enabled and disabled services, sysctl, udev rules, the firewall, the kernel and its modules. archix doesn't restart or reload anything on the running system, which keeps running the old generation, consistently, until you reboot. Services start and stop as the new deployment boots.

A few things live outside the deployments and are written once the switch has committed, on the running system:

- [`home.files`](/docs/reference/options/#homefiles), dotfiles and `programs.*` files, and managed files outside `/etc`, `/usr` and `/opt`, with their `on_change` scripts, which run right after;
- home directories of users the switch created;
- [Flatpak apps](/docs/guides/flatpak-and-appimages/), installed, pinned and removed, unless the switch brings `flatpak` itself, when the new deployment's first boot installs them;
- `archix.lock` and `installed.lua`, when your config isn't in `/etc`.

A failure in those is a [soft failure](#hard-and-soft-failures): the deployment is committed.

[Hooks, and `on_change` scripts of files in the deployment](/docs/guides/hooks/#when-they-run), run at the new deployment's first boot, before the network starts.

### Changes to `/etc` before the reboot

The new deployment has its own copy of `/etc`, made when the switch built it. Changes you make to the running `/etc` after that, until you reboot, are carried into it: a password changed with `passwd`, a Wi-Fi network saved, a file edited by hand. `switch`, `rollback`, `install --temp` and `archix status` say when a deployment is waiting:

```
deployment 42 is committed and boots at the next restart; changes to /etc made before then are carried over to it
```

They say so only while the merge runs. Where it couldn't start, they say ``no /etc merge is running, so changes to /etc made before then don't reach it`` instead, after a warning with the reason.

Before it stages anything, a switch takes a copy of the running `/etc` as the base, and once it has committed, it starts `archix-etc-merge.service`, which watches the running `/etc` with inotify until shutdown. Within about two seconds of a change, it merges it into the waiting deployment's `/etc`, three ways, against that base:

- **Changed only on the running system**: the change goes into the deployment, with its owner, mode, extended attributes, ACLs and times. Files are written through a temporary file and renamed into place. New directories, deletions, symlinks, wherever they point, and a file that became a directory or a link, all follow the running system.
- **Changed on both sides**, such as `/etc/shells` you edited while the switch's packages also changed it: the running system's version wins, and the deployment's goes beside it as `/etc/shells.pacnew`, as pacman would name it.
- **Never carried**: `/etc/.updated`, `/etc/mtab`, `/etc/ld.so.cache`, pacman's keyring in `/etc/pacman.d/gnupg`, and managed files the switch rewrote, which stay as the deployment has them. `/etc/machine-id` follows the running system only when it changed there. Sockets, FIFOs and device nodes aren't carried, and neither are names that aren't UTF-8.

`/etc/passwd`, `group`, `shadow` and `gshadow` merge by name rather than by line, so an account you add and one a package in the deployment adds both arrive, and group members merge as sets. Staging holds back every UID and GID the machine ever handed out, so the deployment's packages never take a number an account on the running system has. If two names still end up with one number, the merge stops carrying new users and groups and says so in `archix status` and at your next login: ``archix: the /etc merge stopped carrying users and groups into deployment 42: UID 1001 would be both bob's, added or changed on the running system since the switch, and builder's in deployment 42's /etc/passwd. Give the running system's account another number (usermod -u, groupmod -g) and the merge goes on; until then deployment 42 boots without the accounts and group members added since the switch, though deleted accounts, removed members and password changes still reach it``. Only additions wait. Taking access away keeps being carried, so it isn't undone at the reboot:
- a deleted account (`userdel`);
- a member or group admin removed from a group (`gpasswd -d alice wheel`);
- a password changed because it leaked;
- a locked or expired account (`usermod -L`, `chage -E`).

New accounts and new group members wait until the clash is resolved.

A second switch, `archix install` or rollback before the reboot builds from the waiting deployment, so it first stops the watcher, which makes its last pass as it stops. The build then copies a whole `/etc`, never one caught between the merge's writes, such as a new `/etc/passwd` before its `/etc/shadow`. Once the new deployment is committed, or the build failed, the watcher starts again for whichever deployment is waiting, and carries what changed during the build.

At shutdown the watcher makes a last pass, bounded to 20 seconds, which never holds up the shutdown. After a power cut, a hang or a pass that ran out of time, the next boot carries what the old `/etc` held on disk before anything starts, and the login note says so; a change the disk hadn't stored yet when the power went is lost. `archix status` lists what was carried and the conflicts:

```
3 /etc changes carried into the pending deployment 42, 1 conflicts (.pacnew)
  /etc/shells: changed on the running system and in deployment 42; the running system's version goes over, deployment 42's is /etc/shells.pacnew there
  last pass: 2026-10-15 09:12:40 UTC
```

Where systemd isn't running, as in a container or a chroot during an install, there's no running system whose `/etc` changes, so the merge isn't started and the switch doesn't fail. With systemd running, a merge that can't start is a [soft failure](#hard-and-soft-failures).

A rollback doesn't take a base first, so it carries the changes made after it. `install --temp` doesn't start the merge.

### Users and passwords across deployments

Each deployment has its own `/etc/shadow`, but a rollback, and booting a deployment other than `current` from the boot menu, carry the running system's people into the target's `/etc` by name: root and the accounts with IDs from 1000 to 65533, with their passwords, and accounts only the running system has. Package accounts keep the target's lines, with the running system's group members added. An account whose number belongs to another name in the target isn't carried, and the rollback lists it as a soft failure: `these accounts weren't carried into /archix/deploy/41/etc, as their number is another account's there: bob (UID 1001, which is builder's there)`.

## Secure Boot

With Secure Boot enforcing, an unsigned boot entry would be refused by the firmware. archix stops that in two places. Before a switch builds anything, it refuses one whose entry wouldn't be signed:

```console
error: Secure Boot is on, but this switch's boot entry wouldn't be signed (sbctl has no keys), so the firmware would refuse it; nothing was changed. Run `sudo archix secure-boot setup`, or set boot.secure_boot = true if sbctl already has keys
```

As a backstop, once the boot image is built and before the commit, it checks the image and the systemd-boot loader on the ESP for a signature. An unsigned one stops the build, its staging tree is deleted, and nothing was committed: ``Secure Boot is on, but 7.2.9-arch1-1-aedff12d49d2.efi isn't signed, so the firmware would refuse it; nothing was committed. Run `sudo archix secure-boot setup` (or set boot.secure_boot = true if sbctl already has keys)``. Inside a chroot, where the firmware is the host's, both checks only warn. See [Secure Boot](/docs/guides/secure-boot/).

## Unsafe downgrades

Some packages migrate their data when a new major version starts, so the older version can't read it again: PostgreSQL, MariaDB, MySQL and a few more. A downgrade across such a version can leave a service that won't start, or worse. `switch` and `rollback` list those downgrades before the prompt and ask, defaulting to no:

```console
unsafe downgrades: the older versions may refuse or damage data the newer ones wrote
  postgresql 17.2-1 -> 16.6-1 goes back from major version 17 to 16, and postgresql migrates its data between them
downgrade anyway? [y/N]
```

Pressing Enter stops with `aborted; nothing was changed`. `--noconfirm` refuses them, ``refusing unsafe downgrades with --noconfirm; to go ahead, run `archix switch` or `archix rollback` with --allow-unsafe-downgrade``, and `--allow-unsafe-downgrade` goes ahead without asking. Add your own boundaries with [`unsafe_downgrades`](/docs/reference/options/#unsafe_downgrades). See [Downgrades that may break data](/docs/concepts/generations/#downgrades-that-may-break-data) for the full list.

Picking an older entry in the boot menu doesn't ask: it boots that deployment as it is.

## Interrupted switches

A switch can be stopped at any point, by Ctrl-C, `kill -9`, a full disk or a power cut, and it's designed so there's nothing to recover. Kill tests of archix's staging and commit logic and VM power cuts during staging pass; write-level power-cut replay runs on archix's own switch, rollbacks and first boots, and the flaws it found are fixed. See [How it's tested](/docs/concepts/atomicity/#how-its-tested).

- **Before the commit**, the running system and its boot entries are as they were. What the switch staged is deleted by the failure itself or, after a kill or power cut, by the next switch.
- **During the commit**, the steps are ordered so that whatever survives names a complete deployment: the next boot starts either the old one or the new one.
- **After the commit**, the switch is done; what was left is the work outside the deployment, which the next switch does again.

`installed.lua` and `archix.lock` are written only once the switch commits; see [Declined and killed switches](#declined-and-killed-switches). There's no journal and nothing to recover: run `archix switch` again.

### Upgrading archix itself

archix is one of the packages a switch installs, and each deployment has its own archix. The archix in a new deployment writes that deployment's integration files, its initramfs hook, boot units and pacman hook, from inside the staging tree, so an upgraded archix boots with its own. Booting an older deployment boots the archix it had. Generation records carry a format version, and an older archix won't rewrite a record in a newer format, or rebuild its dropped deployment, so it never drops what it doesn't know; see [What a generation contains](/docs/concepts/generations/#what-a-generation-contains).

## Atomic writes

Managed files are written to a temporary file and renamed into place, with their mode and owner already set. A rename is atomic, so a file is either the old version or the new one, never half of each. Files in a deployment's `/etc`, `/usr` and `/opt` are written into the staging tree, before the commit.

The temporary file has a random name and is created exclusively, never through a link, and its owner and mode are set on the open file, not by path. That goes for managed files, for `archix.lock` and `installed.lua` in a config directory you own, and for home files, which a process running as their user writes, so a link someone plants in a directory they own can't make root write or chown a file elsewhere.

## Managed files, backups and hand edits

Everything archix manages on disk, from `files` and `units` to `sysctl`, `hostname`, `locales` and dotfiles, becomes a managed file. archix is careful never to lose a file it didn't write:

- **A file archix replaces** is moved to `/var/lib/archix/backups/<path>` first, or for a file in a user's home, to `/var/lib/archix/users/<user>/backups/<path>`, which only that user and root can read. When the file leaves your config, the original comes back.
- **A directory archix replaces** with a file or a link, such as an existing `~/.config/nvim` that a [live link](/docs/guides/dotfiles-and-apps/#live-links-for-app-written-configs) takes over, is moved to the same place, contents and all. When the entry leaves your config, the link goes and the directory comes back.

  Backups and restores are renames. When the backup and the original are on different filesystems, such as a separate `/home` partition or a btrfs `@home` subvolume, archix copies instead, keeping modes, owners and symlinks, and then deletes the source.
- **A file archix takes over unchanged**, such as `/etc/fstab` after `archix init` captured it, is copied to the same place when archix first manages it. Unmanaging it later restores it instead of deleting it.
- **A managed file you edited by hand** is saved to `/var/lib/archix/edited/<time>/<path>`, or `/var/lib/archix/users/<user>/edited/<time>/<path>` for a home file, before archix overwrites it, with a warning. `archix diff` flags it beforehand:

  ```console
  update     /etc/issue (edited outside archix; your version gets saved)
  ```

  On FAT and exFAT, such as the ESP, only the content counts: those filesystems can't store a mode or owner, so every file reads back with the mount's, and archix doesn't take that for an edit.

- **A file archix created** is deleted when it leaves your config, but only if it still has the contents archix wrote.
- **A read-only file**, with [`mutable_files = false`](/docs/reference/options/#mutable_files) or `mutable = false`, is written without write bits, and like every managed file, a switch puts its contents and mode back if they changed.
- **A merged file**, declared with [`merge`](/docs/guides/dotfiles-and-apps/#setting-single-keys), is never backed up, saved as edited, restored or deleted. archix only sets its declared keys, and when the entry leaves the config, the file keeps them. It's swapped in with one atomic exchange, and if the app saved it since archix read it, archix merges the keys into the app's version again, so the app's change isn't lost. For `kconfig` files, archix also holds KDE's own lock, `<file>.lock`, while it does, and waits up to 10 seconds for an app that holds it.

archix only replaces a directory it has never managed, and only once. It refuses when an earlier backup of that path still exists, `/home/me/.config/nvim is a directory and an earlier backup of it exists; move one of them away`, and when archix managed a file at that path in the current generation: `/home/me/.config/nvim exists and is not a file or symlink; archix won't replace it`. Both `switch` and `rollback` check this before they change anything, so a refusal is a plain error. Move the directory or the old backup aside and run the command again.

Users and groups archix created are removed when they leave the config, and their home directories are kept. Passwords are left alone unless the config sets them with [`initial_hashed_password` or `hashed_password`](/docs/reference/options/#users); a new user without one gets a hint instead, ``created user me; set a password with `passwd me` after the reboot``, since the user exists only in the new deployment until then. An enforced `hashed_password` never unlocks an account that's locked or expired, `archix diff` and the switch say when it resets a password, and a switch refuses to leave nobody a password while the running system has one. `/etc/shadow` belongs to each deployment, and a rollback carries the running system's users and passwords into the deployment it goes back to; see [Users and passwords across deployments](#users-and-passwords-across-deployments).

The same goes for the [firewall](/docs/guides/environment-and-firewall/#firewall): it lives in its own `inet archix` table, and turning it off deletes only that table, never the ones Docker, Tailscale or libvirt manage.

archix is strict about what *it* manages, and only that. It doesn't revert changes to `/etc` files it never managed, and it doesn't disable services it never managed. Doing that blindly could, for example, delete users that packages create. `archix init` captures the files and services you'd changed, which makes them managed.

Every switch also stores the contents of every file its generation manages in `/var/lib/archix/blobs`, so a rollback that rebuilds it never finds them missing.

### What init captures and never captures

`archix init` looks for the `/etc` files you changed: package config files edited since they were installed (it compares them with the checksums in pacman's database), and files nobody owns in well-known directories such as `/etc/systemd/system`, `/etc/sudoers.d`, `/etc/sysctl.d`, `/etc/modprobe.d`, `/etc/udev/rules.d`, `/etc/X11/xorg.conf.d`, `/etc/pacman.d/hooks` and `/etc/polkit-1/rules.d`. It sorts what it finds into three groups:

- **Secrets** are never shown or copied, only counted: `· 3 files that hold secrets stay out of the config and aren't listed`. That covers SSH host keys, NetworkManager connections, wpa_supplicant, iwd, WireGuard, OpenVPN and PPP configs, `/etc/ssl/private`, `/etc/cryptsetup-keys.d`, `krb5.keytab`, `smbpasswd`, `ipsec.secrets`, and any path containing `secret`, `password`, `passwd`, `token`, `credential`, `keyfile`, `.key`, `.pem`, `.p12` or `.pfx`. A file also counts as a secret by what's in it: private keys, including PGP private key blocks, GitHub, GitLab, Slack and Anthropic access tokens, AWS secret keys, OAuth refresh tokens, and `psk=` or `password=` lines. The full list is `secret_content` in `src/capture.json`. Files over 1 MiB aren't read, and neither is a directory past its first 200 files.
- **Worth managing** files are pre-selected: `/etc/fstab`, `/etc/crypttab`, `/etc/mkinitcpio.conf`, `/etc/sudoers`, `/etc/pacman.d/mirrorlist`, `/etc/makepkg.conf`, `/etc/hosts`, `sshd_config`, the systemd, NetworkManager, bluetooth and display manager configs, and everything hand-written in the directories above.
- **The rest**, mostly files that package install scripts change, such as `/etc/shells`, are off by default.

init's Config files step shows them all in one checklist, `Config files archix manages`, each tagged with why it was found: `recommended` for the ones worth managing, then `changed from its package's version` or `not from any package`. The recommended ones start ticked. `space` toggles a file, `a` and `n` select all or none, and `r` goes back to the recommended set. `--defaults` takes the recommended ones. Chosen files are copied into `files/<host>/` and declared with their mode, owner and group.

It never captures files that archix or other tools own: `/etc/pacman.conf`, `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow` and their backups, `/etc/machine-id`, `/etc/resolv.conf`, `/etc/mtab`, `/etc/ld.so.cache`, `/etc/kernel/cmdline`, pacman's keyring, the extracted CA certificates, and archix's own files.

Since init only declares what's already there, the first switch after it changes no packages. It lays the system out as deployment 1 and gives it a boot entry, and dotfiles you chose to import are replaced with links into the config; init's Result section lists those under `! the first switch would still change these:`.

## Hard and soft failures

Not every failure means the same thing.

**Hard failures** stop the switch before its commit: a pacman error in the staging tree, a file that can't be written, a user that can't be created, an initramfs that doesn't build. The staging tree is deleted, and the running system, its boot entries and your config are as they were:

```console
:: building deployment 43 from deployment 42
...
error: pacman exited with exit status: 1 in the staging root
```

**Soft failures** come after the commit, or don't stop it: a unit that can't be enabled, a file outside the deployment that can't be written, a Flatpak app that can't be installed, pinned or removed, `gc`. The new generation is committed, archix lists what failed and exits non-zero, like NixOS does:

```console
switched to generation 43 (snapshot 2026-10-15); its deployment boots at the next restart
failed: writing files outside the deployment: ...
error: generation 43 is committed and boots at the next restart, but 1 step failed (listed above)
```

A hook or `on_change` script that fails at boot is listed on the console and at your next login, and runs again at the next boot, up to three boots; see [Activation hooks](/docs/guides/hooks/#failures).

## Declined and killed switches

Nothing a switch changes in your config is written before the switch commits. `install` and `remove` evaluate the edited `installed.lua` from memory, and the new `archix.lock` stays in memory too. When the config is in `/etc`, both go into the new deployment's `/etc` before the commit, and into the running `/etc` right after it. A config elsewhere gets them right after the commit. So declining, a failed check, a failed build, a kill or a power cut before the commit leaves both files byte for byte as they were.

archix runs as root, but files it writes in your config keep their owner and mode, and a new one, like the first `installed.lua`, gets the owner of the directory it's created in, as do new folders, which are 0755, so `git commit` keeps working after `sudo archix install`.
