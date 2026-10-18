---
title: Safety and recovery
description: How archix prepares before it applies, journals every step, manages files without losing your edits, and recovers from failed or interrupted switches without a live USB.
sidebar:
  order: 5
---

The goal is simple: no failure during a switch, including pulling the power cord halfway through, should ever need a live USB. This page explains how archix gets there. [Limitations and failure modes](/docs/concepts/limitations/) covers what it doesn't guarantee.

## Prepare, then apply

`archix switch` runs in two phases.

**Prepare** does everything that can fail without touching the system:

- evaluates the config and renders every managed file,
- resolves the lock with pacman itself, so dependencies are consistent, and shows new AUR build recipes for review,
- checks that no locked package is built for a newer x86-64 level than this CPU runs, which happens when another machine locked the host, and stops if one is, leaving the lock for `archix lock` on this machine to fix ([CPU levels](/docs/guides/third-party-repos/#cpu-levels)),
- offers to adopt packages that are newer than the lock,
- checks that every managed file can replace what's on disk now, keeping a backup ([below](#managed-files-backups-and-hand-edits)),
- when archix manages booting, checks that the ESP can hold the new boot entries, and with Secure Boot on, that they'll be signed ([below](#secure-boot)),
- with the [firewall](/docs/guides/environment-and-firewall/#firewall) on and `nft` installed, checks the generated rules with `nft -c -f`,
- lists the changes and asks you to confirm,
- stores every managed file's contents, and downloads files declared with a `url`, checking each against its `sha256`,
- builds every AUR package in a clean chroot,
- downloads every package and checks it against the lock,
- checks that no two packages ship the same file, and that no incoming file is already on disk without an owner ([below](#packages-that-ship-the-same-file)),
- has pacman check every incoming package's signature, and checks that the root filesystem has room for the change ([below](#signatures-and-disk-space)),
- flushes everything it downloaded, copied and built to disk, so a power cut later can't leave recovery with empty package files.

If anything fails here, a typo, a network error, an AUR build that won't compile, a bad signature, a full disk, a declined prompt, archix stops and the packages, files and services are exactly as they were, with no journal. Most problems surface in this phase.

### Packages that ship the same file

pacman only finds out that two packages own the same file once it's installing them, halfway through a switch. archix checks first. Once every package is downloaded or built, it reads each incoming package's file list and compares it with the other incoming packages and with the installed packages the switch keeps:

```console
error: pacman would refuse these packages over files that are already taken; nothing was changed. Drop one of each pair from the config, or move the stray files away:
  usr/lib/udev/rules.d/70-wooting.rules is in both wooting-udev and wootility
```

Packages that are leaving don't count, and that includes the ones pacman swaps out. An incoming package's `conflicts` and `replaces` name packages that go when it arrives, and an installed package whose `conflicts` or `replaces` names an incoming one goes too, as when CachyOS's `zlib-ng-compat` and Arch's `zlib` replace each other.

It also looks at the disk. An incoming file that already exists there, with no installed package owning it, is what pacman reports as "exists in filesystem", and archix refuses it the same way, as `usr/bin/foo from foo exists on disk, and no package owns it`. Move the stray file away, or drop the package. Files a package lists as `backup`, its config files, don't count, since pacman installs those as `.pacnew` instead, and neither do the copies of kernel modules archix keeps itself.

`rollback` runs the same check. A rollback from the boot menu doesn't: it goes back to a generation that was installed once already, and at boot it's better to let pacman decide than to refuse.

### Signatures and disk space

pacman checks a package's signature only as it installs it, and finds out the disk is full only when a write fails. Both used to stop a switch halfway, after packages were already gone. archix asks first, once every package is downloaded or built:

- **Signatures.** pacman loads every incoming package with `pacman -U --print`, which checks signatures and keys exactly as the install will, and installs nothing. A package it refuses, signed by an unknown or revoked key or with a bad signature, stops the switch with `pacman refuses these packages, so nothing was changed:` and pacman's own errors.
- **Disk space.** Like pacman, archix counts per mount. Every incoming file's size, from the package's `.MTREE`, counts on the mount the file lands on, less what the files of the versions it replaces free on that mount. On btrfs they free nothing, since the snapshot archix takes before applying keeps them. btrfs also hands back a deleted file's space only at its next commit, so a mount that looks short gets committed and measured again before archix refuses, and freeing space then retrying at once works. btrfs can also show free space and still fail writes: once data chunks fill the device, metadata can't grow. So on a btrfs mount with under 1 GiB unallocated, archix first hands completely empty data chunks back with `btrfs balance start -dusage=0`, which moves no data, and if metadata still has no room it stops before anything changes: ``btrfs on / has room for data but almost none for metadata (12 MiB of metadata free, 1 MiB unallocated), so writes would fail halfway; nothing was changed. Run `sudo btrfs balance start -dusage=10 /` and try again``. `archix recover` does the same check before it continues or rolls back. Each mount also keeps a cushion of 5% of its size, at most 128 MiB, since pacman extracts file by file and the boot step writes initramfs images; a small ESP still takes a microcode update. Packages that just leave the config don't count, because they're only removed after the install. Otherwise: ``/ has 96 MiB free, but this needs about 412 MiB; nothing was changed. Free some space (`archix gc`, `pacman -Sc`) and try again``, naming the mount that's short. Downloads aren't counted ahead of time: if `/var/cache/archix/pkg` fills up while packages download, the switch stops there with ``/var/cache/archix/pkg ran out of space while downloading packages; nothing was changed``. A download or copy cut short is never kept under the package's real name.

`switch`, `rollback`, `install --temp` and `recover --rollback` run both checks. A rollback from the boot menu skips them, as it skips the file check. Either failure comes before the journal is written, so the system is untouched. The ESP has its own check, for the boot entries; see [Boot entries](/docs/concepts/generations/#boot-entries).

A rollback reinstalls packages that were verified when they were first installed. It installs them by the sha256 the generation pinned, without their `.sig` file, so a packager key revoked since then doesn't block going back. `recover --rollback` does the same for the packages the current generation pinned, and pacman checks any other file against today's keyring.

### Keyrings

Packages that ship a pacman keyring, the files `pacman-key --populate` reads from `/usr/share/pacman/keyrings`, get two rules: `archlinux-keyring`, and third-party ones such as `cachyos-keyring` and `chaotic-keyring`.

- **They go in first.** A newer snapshot can bring packages signed by a packager that only its newer `archlinux-keyring` knows. Plain pacman refuses those until you update the keyring on its own. archix runs the signature check above against a copy of pacman's keyring with the incoming keyrings already populated into it. The install step then installs the keyring packages by themselves, so their install scripts add the keys, syncs the disk, and installs the rest after. A crash during the rest can't tear the keyring, and `archix recover` rebuilds one that a crash did tear. If the copy can't be made, the check warns and uses the installed keyring.
- **They never go back.** An older keyring trusts packager keys that were revoked since, and lacks the ones added since. A switch or rollback that would downgrade one keeps the installed version instead: `keeping archlinux-keyring 20260909-1 rather than going back to 20260602-1: an older keyring trusts keys revoked since`.

### One archix at a time

Only one archix changes the system at a time. `switch`, `install`, `remove`, `upgrade` and bare `archix`, `rollback`, `recover`, `gc`, `init`, `secure-boot setup` and the boot-time check take an exclusive lock on `/var/lib/archix/running`, and a second one stops at once:

```console
error: another archix is already running (pid 4242, `archix switch`); wait for it to finish
```

Commands that only read or write the config, `status`, `diff`, `search`, `eval`, `options`, `lock` and `update`, don't take it.

Packages are downloaded to `/var/cache/archix/pkg`, and every file is checked against the `sha256` in the lock before it's used. Files already in pacman's own cache are reused. A file that doesn't match its checksum stops the switch here. The same goes for files declared with [`archix.fetchurl`](/docs/reference/lua-api/#archixfetchurl):

```console
error: https://downloads.example.org/hello/1.2.0/hello-x86_64 has sha256 0c1f3a5e7b9d2f4a6c8e0b1d3f5a7c9e2b4d6f8a0c1e3b5d7f9a2c4e6b8d0f1a, but the config says 5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03
```

**Apply** changes the system, in a fixed order of steps:

```console
$ sudo archix switch
upgrade    linux 7.2.8.arch1-1 -> 7.2.9.arch1-1
upgrade    mesa 1:25.3.0-1 -> 1:25.3.1-1
boot entry for the new generation
apply 3 changes? [Y/n]
:: installing packages
:: removing packages
:: setting install reasons
:: updating users and groups
:: stopping and disabling services
:: writing managed files
:: reloading systemd, sysctl, udev and locales
:: enabling and starting services
:: restarting what uses replaced files
:: verifying the result
:: writing the boot entry
:: making the new generation current
switched to generation 42 (snapshot 2026-10-15)
```

Packages are installed before anything is removed. One pacman run installs every incoming package and, in the same transaction, removes the installed packages they conflict with or replace. Packages that are simply leaving the config go afterwards, in `:: removing packages`. So pacman and everything it needs are never missing in between. The one exception is an installed package whose files an incoming one takes over without declaring a conflict: pacman would refuse the install while it still owns them, so it goes first.

Services are stopped and disabled while their unit files still exist, and enabled and started only after `systemctl daemon-reload` has seen the new ones. A service that stays enabled is restarted when its unit file or one of its drop-ins changed.

`:: reloading systemd, sysctl, udev and locales` reloads whatever the changed files feed, so they apply at once: `systemctl daemon-reload`, `sysctl --system`, udev's rules, `locale-gen`, `systemd-tmpfiles --create` for `tmpfiles.d`, `systemd-sysusers` for `sysusers.d`, and `systemd-modules-load` for `modules-load.d`. A change to dbus's or logind's config reloads that service, never restarts it, since a restart would end every session. When user units change, the user manager of every logged-in user reloads too.

Users and groups follow the services. A user that leaves the config is deleted after its services have stopped, since `userdel` refuses an account that's still in use. A user or group archix creates again gets back the uid or gid it had before, so files it owned on disk are still its own.

### The running kernel

Upgrading or removing a kernel package deletes its modules from `/usr/lib/modules`, including the running kernel's. On plain Arch, nothing new loads after that until you reboot: a USB stick doesn't mount, a VPN or filesystem module isn't found.

archix keeps them loadable. When a switch, rollback or recovery is about to replace or remove the package that owns `/usr/lib/modules/$(uname -r)`, which it finds through the `pkgbase` file pacman's kernels leave in that directory, archix first keeps a hardlinked copy in `/var/lib/archix/kernels`. Once pacman is done, it puts the directory back:

```console
:: installing packages
...
:: kept the running kernel's modules, so they still load until you reboot
```

At the end, `switch`, `rollback` and `recover` say that only a reboot finishes the job:

```console
switched to generation 43 (snapshot 2026-10-15)
the running kernel was replaced: reboot to finish (its modules stay loadable until then)
```

They say so after every switch until you reboot, since the running kernel's package is still gone. A rollback from the boot menu keeps the modules the same way, but prints no notice.

archix records every kernel whose modules it kept in `/var/lib/archix/kept-modules`, and cleans up once that kernel is no longer running: at the next switch or rollback, or at the next boot from an archix boot entry. It deletes the directory it put back in `/usr/lib/modules`, unless a package owns it again because you reinstalled the same kernel version, and the copy in `/var/lib/archix/kernels`, unless a [boot entry](/docs/concepts/generations/#older-kernels-keep-booting) was built with that kernel and still needs it.

[Activation hooks](/docs/guides/hooks/) aren't one of these steps. They run after the switch is complete, once the new generation is current and Flatpak apps are installed, under `:: running activation hooks`: first the `on_change` scripts of files the switch changed, then every hook. A hook can't leave a switch unfinished, so the journal doesn't track them. With hooks declared, the preview before the prompt lists them on a `run hooks` line.

archix also checks pacman before it starts. If any process holds pacman's lock open, pacman itself or another front end such as pamac or PackageKit, `switch` refuses with `pacman is running; wait for it to finish`. A `db.lck` that's left over with nothing holding it is refused too, and archix tells you why, rather than fight another package manager.

The tools archix runs and reads the output of, pacman among them, run with `LC_MESSAGES=C`, so they answer in English whatever your locale.

## Restarting what uses replaced files

A running program keeps using the files it started with. When pacman replaces a binary or a shared library, the old one is deleted, but every process that loaded it keeps running the old code until it restarts. After a `glibc` or `openssl` upgrade, a daemon that has been up for a week still runs last week's library.

archix restarts those units itself, in the `:: restarting what uses replaced files` step, after services are started and before the result is verified. It covers system units, and the user units of every user whose user manager is running, logged in or lingering, through `systemctl --user -M <user>@`. A unit is restarted when:

- **its processes use replaced files.** A process that maps or holds open a deleted file under `/usr` is still running code pacman replaced. That catches binaries and shared libraries exactly, glibc included: a glibc upgrade restarts only what loaded the old glibc. It counts every such file, including ones left by an earlier `pacman -Syu`.
- **interpreted code it needs changed.** Python, Perl, Node, Ruby and Lua code isn't mapped, so a daemon written in them doesn't show up that way. When the switch changes a package that ships files under `/usr/lib/python3*`, `/usr/share/perl5`, `/usr/lib/perl5`, `/usr/lib/node_modules`, `/usr/lib/ruby`, `/usr/share/lua` or `/usr/lib/lua`, archix restarts the running units whose packages depend on it, through `depends` and installed optional dependencies. A unit's packages are the ones that own its unit file and the absolute paths in its `ExecStart`. So upgrading `python-urllib3` restarts a Python daemon that uses it through `python-requests`.

Restarts use `systemctl try-restart`, so a unit that isn't running isn't started. If pid 1 itself still maps replaced files, after a systemd or glibc upgrade, archix runs `systemctl daemon-reexec`; a user manager in that position gets `systemctl --user daemon-reexec`.

```console
:: restarting what uses replaced files
:: restarting cups.service php-fpm.service
:: restarting pipewire.service wireplumber.service for alice
:: still using replaced files until you log out or reboot: firefox, plasma-plasmashell.service, sddm.service
```

Some units are never restarted, because restarting them would end your session, cut you off or break what running apps hold open. archix lists them on the last line instead, for you to pick up by logging out or rebooting:

- system units: the display manager, `display-manager.service` and the unit it points to, such as `sddm.service`; `dbus.service`, `dbus-broker.service`, `systemd-logind.service`; the network, `NetworkManager.service`, `systemd-networkd.service`, `systemd-resolved.service`, `iwd.service` and `wpa_supplicant.service`, since the switch may be running over ssh; `sshd.service`, and every `getty@`, `serial-getty@` and `user@` unit;
- user units whose names start with `dbus`, `plasma-kwin`, `plasma-plasmashell`, `plasma-ksmserver`, `plasma-workspace`, `niri`, `gnome-shell`, `org.gnome.Shell`, `hyprland`, `sway` or `wayland-wm`, the desktop session itself, or with `pipewire`, `wireplumber` or `xdg-desktop-portal`, which apps hold audio and screen-sharing streams to;
- programs in session and app scopes, your desktop apps, listed by process name.

Two more kinds of unit are skipped without being listed:

- the unit archix itself runs in, found from its own cgroup, such as a systemd service or timer that runs `archix switch`; restarting it would kill the switch halfway;
- units that started at or after the moment the switch began, judged by their `ActiveEnterTimestamp`; they already run the newly installed files.

[`restart_if_changed`](/docs/reference/options/#restart_if_changed) changes the list per unit: `true` restarts a unit even if it's on the list, `false` never restarts it and lists it instead. It doesn't override the two skips above.

```lua title="lua/hosts/server/init.lua"
return {
  restart_if_changed = { ["sshd.service"] = true, ["postgresql.service"] = false },
}
```

A restart that fails is a [soft failure](#hard-and-soft-failures), listed as `failed: systemctl try-restart: ...` with systemctl's message, and the switch is kept. A rollback from the boot menu skips the step: it runs before the services it would restart have started. `archix recover --continue` runs it if the interrupted switch hadn't got to it.

## Secure Boot

With Secure Boot enforcing, an unsigned boot entry would be refused by the firmware, and a switch that made one the default would leave the machine unable to boot it. archix stops that in two places. Before a switch changes anything, it refuses one whose entry wouldn't be signed:

```console
error: Secure Boot is on, but this switch's boot entry wouldn't be signed (sbctl has no keys), so the firmware would refuse it; nothing was changed. Run `sudo archix secure-boot setup`, or set boot.secure_boot = true if sbctl already has keys
```

As a backstop, in `:: writing the boot entry`, it checks every built entry and the systemd-boot loader for a signature before copying the entry to the ESP. An unsigned one is a hard failure there, ``Secure Boot is on, but archix-43.efi isn't signed, so the firmware would refuse it; nothing was moved. Run `sudo archix secure-boot setup` (or set boot.secure_boot = true if sbctl already has keys)``: the previous generation stays current and the default, and `archix recover` finishes or rolls back. The entry is built from the kernel the switch installs, so this can't be checked in prepare. Inside a chroot, where the firmware is the host's, both checks only warn. See [Secure Boot](/docs/guides/secure-boot/).

## Unsafe downgrades

Some packages migrate their data when a new major version starts, so the older version can't read it again: PostgreSQL, MariaDB, MySQL and a few more. A downgrade across such a version can leave a service that won't start, or worse. `switch` and `rollback` list those downgrades before the apply prompt and ask, defaulting to no:

```console
unsafe downgrades: the older versions may refuse or damage data the newer ones wrote
  postgresql 17.2-1 -> 16.6-1 goes back from major version 17 to 16, and postgresql migrates its data between them
downgrade anyway? [y/N]
```

Pressing Enter stops with `aborted; nothing was changed`. `--noconfirm` refuses them, ``refusing unsafe downgrades with --noconfirm; to go ahead, run `archix switch` or `archix rollback` with --allow-unsafe-downgrade``, and `--allow-unsafe-downgrade` goes ahead without asking. Add your own boundaries with [`unsafe_downgrades`](/docs/reference/options/#unsafe_downgrades). See [Downgrades that may break data](/docs/concepts/generations/#downgrades-that-may-break-data) for the full list.

`archix recover --rollback` asks the same way, since going back to the packages from before an interrupted switch can be a downgrade too; `--allow-unsafe-downgrade` skips the question there as well.

A rollback from the boot menu goes ahead without asking. If it crossed one, every login shows a note, in login shells and as a desktop notification:

```
archix: the boot-menu rollback to generation 41 went back across versions that may not read newer data; check these services:
  postgresql 17.2-1 -> 16.6-1 goes back from major version 17 to 16, and postgresql migrates its data between them
```

The note stays, in `/var/lib/archix/notes`, until the next switch or rollback.

## The journal

Before apply starts, archix writes `/var/lib/archix/journal.json`, fsynced and renamed into place. It records the target generation, the package set from before the switch, the exact transaction, and which of the steps above are done. Each step is marked done as it completes.

Every step is idempotent. A step doesn't mean "upgrade mesa from A to B"; it means "make the system match the target". Running a step twice is harmless, which is what makes it safe to continue an interrupted switch from wherever it stopped.

pacman doesn't fsync what it writes. So archix syncs the filesystem after every pacman step, before it marks the step done, and the journal never claims an install that is still only in memory. Generations, built packages, boot entries and the journal's own deletion are written durably too.

While the journal runs, archix ignores Ctrl-C, and pacman runs in its own process group, so neither can stop pacman halfway through a transaction. A dropped ssh session can't stop a switch either:

- Started from a login session, `switch`, `rollback`, `recover` and the other commands that change the system re-run themselves in a systemd scope of their own (`archix-<pid>.scope`, through `systemd-run --scope`), as NixOS runs its switch. So logind ending the session, which it does to every process in it with `KillUserProcesses=yes`, doesn't reach them. They stay on the same terminal, so prompts work as before.
- When the terminal hangs up mid-switch, archix writes the rest of its output to `/var/lib/archix/after-hangup.log` instead, since writing to a dead terminal fails. Log back in and read it, or check `archix status`.

If the switch is cut off anyway, by a power cut or `kill -9`, `archix recover` finishes it.

The `on_change` scripts a switch owes are recorded before it changes any file, and cleared only once they've run, so a crash in between doesn't lose them.

When every step is done, the journal is deleted. A journal on disk always means a switch is unfinished.

### Upgrading archix itself

archix is one of the packages a switch installs, so a switch can replace archix while it runs, and a rollback can put an older archix back before it finishes. A journal cut off at that point is read by a different archix than the one that wrote it.

So each journal records the oldest archix that can finish it: format 1, or 2 when the switch writes `archix.lock` and `installed.lua` as it commits, which an archix that reads only format 1 would skip. Next to the journal, archix keeps a copy of itself, `/var/lib/archix/journal-archix`, the static build when the package installed one, so the initramfs can run it too. An archix that finds a journal it can't finish hands it to that copy, with the same command line:

```text
/var/lib/archix/journal.json was started by a newer archix; handing over to it
```

Without the copy, it stops without touching anything: `/var/lib/archix/journal.json needs journal version 2, but this archix reads up to version 1; finish it with the archix that started it (/var/lib/archix/journal-archix), or install a newer archix`. The copy goes when the journal does.

A VM test starts from the previous release, 0.1.0-dogfood, installed as a package from a local repo. The previous release switches to the new archix, which reads the generations, lock and state it wrote, and then replaces itself in a switch of its own. The new archix rolls back to the previous release at runtime, and the previous release reads the new archix's state. A rollback to the previous release is killed once pacman has installed it, and the previous release finishes it. Booting the previous release's entry has the new archix roll back to it, and booting the new archix's entry has the previous release roll forward.

## Atomic writes

Managed files are written to a temporary file and renamed into place, with their mode and owner already set. A rename is atomic, so a file is either the old version or the new one, never half of each.

The two things that decide what "current" means only change at the very end, after the verify step has confirmed the result:

- the boot entry for the new generation, written in the second-to-last step;
- the current-generation pointer, replaced atomically in the last step.

Until then, the previous generation is current, and a reboot lands in it.

On btrfs, archix also takes a read-only snapshot of `/` right before apply. It's a manual recovery point, not something rollback uses; see [btrfs snapshots](/docs/concepts/generations/#btrfs-snapshots).

## Managed files, backups and hand edits

Everything archix manages on disk, from `files` and `units` to `sysctl`, `hostname`, `locales` and dotfiles, becomes a managed file. archix is careful never to lose a file it didn't write:

- **A file archix replaces** is moved to `/var/lib/archix/backups/<path>` first. When the file leaves your config, the original comes back.
- **A directory archix replaces** with a file or a link, such as an existing `~/.config/nvim` that a [live link](/docs/guides/dotfiles-and-apps/#live-links-for-app-written-configs) takes over, is moved to the same place, contents and all. When the entry leaves your config, the link goes and the directory comes back.

Backups and restores are renames. When the backup and the original are on different filesystems, such as a separate `/home` partition or a btrfs `@home` subvolume, archix copies instead, keeping modes, owners and symlinks, and then deletes the source.
- **A file archix takes over unchanged**, such as `/etc/fstab` after `archix init` captured it, is copied to the same place when archix first manages it. Unmanaging it later restores it instead of deleting it.
- **A managed file you edited by hand** is saved to `/var/lib/archix/edited/<time>/<path>` before archix overwrites it, with a warning. `archix diff` flags it beforehand:

  ```console
  update     /etc/issue (edited outside archix; your version gets saved)
  ```

  On FAT and exFAT, such as the ESP, only the content counts: those filesystems can't store a mode or owner, so every file reads back with the mount's, and archix doesn't take that for an edit.

- **A file archix created** is deleted when it leaves your config, but only if it still has the contents archix wrote.
- **A managed symlink over a mount point**, such as the `/etc/resolv.conf` Docker and `arch-chroot` bind-mount, can't be replaced, so archix leaves it and warns, `warning: /etc/resolv.conf is mounted over here (a container or arch-chroot does that), so it's left alone; the next switch outside replaces it`. Files with contents are still written through the mount.
- **A read-only file**, with [`mutable_files = false`](/docs/reference/options/#mutable_files) or `mutable = false`, is written without write bits, and like every managed file, a switch puts its contents and mode back if they changed.
- **A merged file**, declared with [`merge`](/docs/guides/dotfiles-and-apps/#setting-single-keys), is never backed up, saved as edited, restored or deleted. archix only sets its declared keys, and when the entry leaves the config, the file keeps them.

archix only replaces a directory it has never managed, and only once. It refuses when an earlier backup of that path still exists, `/home/me/.config/nvim is a directory and an earlier backup of it exists; move one of them away`, and when archix managed a file at that path in the current generation: `/home/me/.config/nvim exists and is not a file or symlink; archix won't replace it`. Both `switch` and `rollback` check this before they change anything, so a refusal is a plain error. Move the directory or the old backup aside and run the command again.

Users and groups archix created are removed when they leave the config, and their home directories are kept. archix never touches passwords: a new user gets a hint instead, ``created user me; set a password with `passwd me` ``.

The same goes for the [firewall](/docs/guides/environment-and-firewall/#firewall): it lives in its own `inet archix` table, and turning it off deletes only that table, never the ones Docker, Tailscale or libvirt manage.

archix is strict about what *it* manages, and only that. It doesn't revert changes to `/etc` files it never managed, and it doesn't disable services it never managed. Doing that blindly could, for example, delete users that packages create. `archix init` captures the files and services you'd changed, which makes them managed.

Files archix takes over unchanged are backed up even when a switch has nothing else to do, as with the first switch after `archix init`, which usually prints only `switched to generation 1 (snapshot 2026-10-09); no package changes were needed`. Every switch that records a generation, that one included, also stores the contents of every file the generation manages in `/var/lib/archix/blobs`, so a later rollback to it never finds them missing.

### What init captures and never captures

`archix init` looks for the `/etc` files you changed: package config files edited since they were installed (it compares them with the checksums in pacman's database), and files nobody owns in well-known directories such as `/etc/systemd/system`, `/etc/sudoers.d`, `/etc/sysctl.d`, `/etc/modprobe.d`, `/etc/udev/rules.d`, `/etc/X11/xorg.conf.d`, `/etc/pacman.d/hooks` and `/etc/polkit-1/rules.d`. It sorts what it finds into three groups:

- **Secrets** are never shown or copied, only counted: `· 3 files that hold secrets stay out of the config and aren't listed`. That covers SSH host keys, NetworkManager connections, wpa_supplicant, iwd, WireGuard, OpenVPN and PPP configs, `/etc/ssl/private`, `/etc/cryptsetup-keys.d`, `krb5.keytab`, `smbpasswd`, `ipsec.secrets`, and any path containing `secret`, `password`, `passwd`, `token`, `credential`, `keyfile`, `.key`, `.pem`, `.p12` or `.pfx`. A file also counts as a secret by what's in it: private keys, including PGP private key blocks, GitHub, GitLab, Slack and Anthropic access tokens, AWS secret keys, OAuth refresh tokens, and `psk=` or `password=` lines. The full list is `secret_content` in `src/capture.json`. Files over 1 MiB aren't read, and neither is a directory past its first 200 files.
- **Worth managing** files are pre-selected: `/etc/fstab`, `/etc/crypttab`, `/etc/mkinitcpio.conf`, `/etc/sudoers`, `/etc/pacman.d/mirrorlist`, `/etc/makepkg.conf`, `/etc/hosts`, `sshd_config`, the systemd, NetworkManager, bluetooth and display manager configs, and everything hand-written in the directories above.
- **The rest**, mostly files that package install scripts change, such as `/etc/shells`, are off by default.

init's Config files step shows them all in one checklist, `Config files archix manages`, each tagged with why it was found: `recommended` for the ones worth managing, then `changed from its package's version` or `not from any package`. The recommended ones start ticked. `space` toggles a file, `a` and `n` select all or none, and `r` goes back to the recommended set. `--defaults` takes the recommended ones. Chosen files are copied into `files/<host>/` and declared with their mode, owner and group.

It never captures files that archix or other tools own: `/etc/pacman.conf`, `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow` and their backups, `/etc/machine-id`, `/etc/resolv.conf`, `/etc/mtab`, `/etc/ld.so.cache`, `/etc/kernel/cmdline`, pacman's keyring, the extracted CA certificates, and archix's own files.

Since init only declares what's already there, the first switch after it changes nothing, with one exception: dotfiles you chose to import. The first switch backs up the originals and replaces them with links into the config, and init's Result section lists them under `! the first switch would still change these:`.

## Hard and soft failures

Not every failure during apply means the same thing.

**Hard failures** stop the switch: a pacman error, a file that can't be written, a user that can't be created, a verify step that finds the system doesn't match the target. The journal stays open, and archix tells you to run `archix recover`:

```console
:: installing packages
error: installing packages failed; the switch is unfinished, run `archix recover`: pacman exited with exit status: 1
```

**Soft failures** don't: a service that fails to start, a reload such as `sysctl --system` that fails, a [restart](#restarting-what-uses-replaced-files) that fails, a [hook or `on_change` script](/docs/guides/hooks/#failures) that exits non-zero, or a Flatpak app that can't be installed, pinned or removed. Flatpak apps are synced after the new generation becomes current, outside the journal. The new generation becomes current, archix lists what failed and exits non-zero, like NixOS does. A broken unit is usually a config problem you fix and switch again, and undoing the whole generation for it would be worse.

```console
:: enabling and starting services
:: restarting what uses replaced files
:: verifying the result
:: writing the boot entry
:: making the new generation current
switched to generation 43 (snapshot 2026-10-15)
failed: systemctl enable --now nginx.service: Job for nginx.service failed because the control process exited with error code.
error: the new generation is current, but 1 step failed (listed above)
```

A failed hook is listed the same way, `failed: hook tailscale: exited with exit status: 1`, and so is an `on_change` script, by its file: `failed: on_change of /etc/foo.conf: exited with exit status: 1`. Hooks also run on a switch that changes nothing, where a failure ends with `error: nothing was changed, but 1 step failed (listed above)`, and on rollbacks, where it's `error: generation 41 (snapshot 2026-10-09) is current, but 1 step failed (listed above)`.

## Declined and killed switches

Nothing a switch changes in your config is written before the switch commits. `install` and `remove` evaluate the edited `installed.lua` from memory, and the new `archix.lock` stays in memory too. Both ride in the switch's journal, and the step that makes the new generation current writes them right after it does. So declining, a failed check or a failed switch leaves both files byte for byte as they were, and so does a kill or power cut before the journal exists.

A kill or power cut after the journal exists leaves the switch unfinished, and both files still as they were. [`archix recover`](#archix-recover) then decides for the config as for the system: `--continue` finishes the switch and writes the new `archix.lock` and `installed.lua` with it, while `--rollback` and `--abandon` leave them untouched. A switch that needs no journal, because no package changes, writes them once its generation is current.

archix runs as root, but files it writes in your config keep their owner and mode, and a new one, like the first `installed.lua`, gets the owner of the directory it's created in, as do new folders, which are 0755, so `git commit` keeps working after `sudo archix install`.

## Unfinished switches

archix never decides how an unfinished switch ends. Guessing wrong after a crash could make things worse, so a human decides. The one thing it does on its own is make the machine bootable again when the switch was cut off inside pacman; see [A power cut inside pacman](#a-power-cut-inside-pacman).

- `archix-boot.service` reports the unfinished switch at boot and changes nothing.
- `switch`, `upgrade`, `install`, `remove`, `rollback`, `lock`, `update` and `gc` refuse to run, and say which switch is unfinished. Read-only commands such as `diff` and `status` keep working, so you can look around first.
- pacman and yay keep working. The pacman hook adds a warning after each transaction.
- You're told at login, by a message from `/etc/profile.d/archix.sh` and a desktop notification. Both also show the note a boot-menu rollback leaves after [unsafe downgrades](#unsafe-downgrades).

```console
$ archix status
the switch to generation 43 (snapshot 2026-10-15) (started at unix time 1792051200) was interrupted while installing packages
run `archix recover` to deal with it
current generation: 42 (snapshot 2026-10-15)
desktop is locked to snapshot 2026-10-15 (214 packages)
12 changes pending; see `archix diff`
```

## `archix recover`

`recover` shows what was interrupted and at which step, then offers three ways out:

- **continue** reruns the steps that hadn't completed and finishes the switch.
- **rollback** returns to the package set from before the switch, and undoes its changes to files, users and services. Packages come from pacman's cache or the Arch Linux Archive, the one the switch used, mirror or not, and anything the transaction touched is reinstalled.
- **abandon** changes nothing. It shows how the installed packages differ from the switch's target, runs `pacman -Qk` on the packages the switch touched, then closes the journal, so you see the real state of the system and fix it by hand.

```console
$ sudo archix recover
the switch to generation 43 (snapshot 2026-10-15) (started at unix time 1792051200) was interrupted while installing packages
[c]ontinue it, [r]oll back, [a]bandon (close the journal without changes), or [q]uit? c
finished the interrupted switch to generation 43 (snapshot 2026-10-15)
```

To skip the prompt, pass the choice:

```console
$ sudo archix recover --continue
$ sudo archix recover --rollback
$ sudo archix recover --abandon
```

Pick continue if the interruption was something outside archix, like a power cut. Pick rollback if you don't trust the new package set. Pick abandon if you want to inspect and fix things by hand; afterwards `archix diff` shows how the system differs from the lock, and the next `archix switch` reinstalls whatever the lock needs. Rollback and abandon both delete the generation the interrupted switch had created, since it never became current.

A recovery rollback is journaled too. If it gets interrupted in turn, `archix recover` picks it up the same way.

Continue and rollback both finish like a switch. They run the [hooks](/docs/guides/hooks/) of the state they end in, and every `on_change` script of it, since archix can't tell which files the interrupted switch had already rewritten; that's why those scripts must be idempotent. Soft failures are reported as on a switch, ending with `error: generation 43 (snapshot 2026-10-15) is current, but 1 step failed (listed above)` or `error: the rollback finished, but 1 step failed (listed above)`, and a non-zero exit.

## Interrupted pacman transactions

Power loss in the middle of a pacman transaction is the classic way to break Arch: a stale lock file, half-extracted packages, a local database that disagrees with the disk.

`recover` handles this before anything else. It first makes sure pacman isn't running; if it is, it stops and asks you to wait. Then it removes the stale `/var/lib/pacman/db.lck`.

Next it repairs pacman's local database. A crash can tear an entry, leaving a package's directory without its name or version. archix rewrites such an entry from the package file the switch was installing, `pacman's database entry for glibc-2.44-1 was torn by the crash; rewrote it from /var/cache/archix/pkg/glibc-2.44-1-x86_64.pkg.tar.zst`, and an entry it can't rewrite is moved to `/var/lib/archix/torn-db` and its package reinstalled over its files.

Then it clears the way for pacman. Files the cut-off pacman run had already extracted belong to no package yet, and pacman would refuse to install over them. archix moves them to `/var/lib/archix/crash-leftovers/<time>` instead of overwriting anything: `moved 214 files left behind by the interrupted pacman run to /var/lib/archix/crash-leftovers/1792051200`.

Last, it checks that pacman starts at all. pacman loads its own libraries, libcurl and libarchive among them, from packages it installs, so a run killed halfway can leave them at mixed versions, and pacman then fails before doing anything, with something like `symbol lookup error: /usr/lib/libcurl.so.4: undefined symbol: ...`. archix then unpacks pacman's dependency closure from the package files it's recovering to, without pacman, the way [rescue mode](#rescue-mode) does: `pacman couldn't start, its libraries left at mixed versions by the interruption; unpacked the 70 packages it runs on without it`. If pacman still doesn't start, recover stops and points you at the current generation's rescue entry.

If the install step was the one interrupted, continuing reinstalls every package of the transaction, even ones pacman already reports as installed, so half-extracted ones are replaced. The verify step then checks the result. Prepare already downloaded everything to `/var/cache/archix/pkg`, so continuing needs no network.

### A power cut inside pacman

pacman doesn't fsync. A power cut in the middle of a large update can leave the files it was writing half-written, systemd itself included, and the next boot can freeze before anything could run `archix recover`.

So when archix manages booting, its initramfs checks for that. The rescue hook that carries the static archix is in every image archix builds, and on every boot, before switching to the real root, it looks for a journal there. If the switch stopped inside pacman, while installing or removing packages, it restores the booted generation's core packages from the cache, as [rescue mode](#rescue-mode) does, plus every package the switch was changing, and puts back the booted kernel's modules:

```
archix: the switch to generation 43 (snapshot 2026-10-15) (started at unix time 1792051200) was interrupted while installing packages, inside pacman; restoring the packages it was changing so the system can boot
```

It does this once per interrupted switch, and leaves the journal alone. The machine boots the generation it was running before, the switch shows as unfinished, and `archix recover` finishes or undoes it.

A VM test suite does this to a real Arch install: it updates it from the 2026-06-01 snapshot to 2026-10-10, glibc 2.43 to 2.44 and linux 7.0 to 7.2, cuts the power halfway, and checks that the next boot repairs itself and `archix recover --continue` finishes the update with no file missing.

## Rescue mode

Some breakage stops pacman itself: a half-installed glibc, a deleted libalpm. Then nothing that relies on pacman can repair the system, including rollback.

For that, when archix manages booting, the initramfs carries a static archix, and the newest two generations get a rescue entry:

```
  Arch Linux (archix, generation 43)
  Arch Linux (archix, generation 42)
› Arch Linux (archix rescue, generation 43)
  Arch Linux (archix rescue, generation 42)
```

Booting it adds `archix.rescue` to the kernel command line. Before switching to the real root, the initramfs runs `archix rescue`, which unpacks that entry's generation's core packages straight from the cache, read from `archix.generation` on the command line, without pacman. The core set is what systemd, pacman, archix and every installed kernel need to run, their whole dependency closure down to glibc, recorded when the generation was activated; a generation from an archix too old to record it gets a fixed list instead. If a switch is unfinished, rescue also restores every package that switch was changing. It keeps config files you edited, saving the package's version as `.pacnew`, and rewrites the packages' entries in pacman's database the way pacman writes them, with the packaged files' backup hashes, so the next upgrade still notices your edits. For each kernel it restored, it runs `depmod`, as pacman's hook would, so that kernel's modules load. Then the system boots, and rescue's last line tells you how to finish with the regular tools:

```
36 packages restored; boot normally, then run `sudo archix rollback 43` to put everything else back
```

It also records what it did in `/var/lib/archix/last-rescue`, such as `generation 43, 36 packages restored, at unix time 1792051200`.

Rolling back to the current generation puts back whatever differs from it, packages, files and services alike. If a switch was interrupted, run `sudo archix recover` instead.

That's what makes a broken pacman repairable without a live USB. It works with both systemd-based and busybox initramfs images. The rescue hook copies the static binary that the [`archix-git`](/docs/installation/#about-archix-git) AUR package installs at `/usr/lib/archix/archix-static`; rescue entries are only built when that file exists. If you built archix by hand, build it yourself from the source tree:

```console
$ cargo build --release --target x86_64-unknown-linux-musl
$ sudo install -Dm755 target/x86_64-unknown-linux-musl/release/archix /usr/lib/archix/archix-static
```

In CI, a QEMU VM with OVMF firmware deletes libalpm to break pacman, boots the rescue entry, and checks that pacman works again.

### From a live USB

If the machine can't boot any of its own entries, the same static binary works from a live USB against the mounted system:

```console
# mount /dev/nvme0n1p2 /mnt
# mount /dev/nvme0n1p1 /mnt/boot
# /mnt/usr/lib/archix/archix-static rescue --root /mnt
```

Without `--generation`, rescue restores the generation on the kernel command line, and with none there, as from a live USB, the current one. `--generation N` restores another generation's core packages, and `--package NAME` adds packages beyond the core set. You shouldn't need it often. It's a fallback for accidents the built-in boot entries can't cover, not a normal recovery path.
