---
title: Safety
description: What a switch checks before it changes anything, the snapshot it takes first, how it applies in place, what happens when a step fails, who can change what a switch applies, and how managed files keep your edits.
sidebar:
  order: 5
---

`archix switch` applies your config to the running system, in place, with no reboot. That makes order matter: everything that can refuse refuses before anything changes, a [snapshot](/docs/concepts/snapshots/) of the root is taken next, and only then does the switch apply. This page explains what a switch checks, what it does, what happens when something goes wrong, and who decides what it applies. [Limitations and failure modes](/docs/concepts/limitations/) covers what it doesn't guarantee, and [What falls back by itself](/docs/concepts/boot-and-fallback/#what-falls-back-by-itself) what boots when an update breaks the system.

## Prepare, then apply

`archix switch` runs in two phases.

**Prepare** does everything that can fail without touching the system:

- captures the config directory as its owner and evaluates it in a sandbox, as an unprivileged throwaway user ([below](#the-config-directory)),
- resolves the lock with pacman itself, so dependencies are consistent, and shows new AUR build recipes for review,
- checks that no locked package is built for a newer x86-64 level than this CPU runs, which happens when another machine locked the host, and stops if one is, leaving the lock for `archix lock` on this machine to fix ([CPU levels](/docs/guides/third-party-repos/#cpu-levels)),
- offers to adopt packages that are newer than the lock, and settles packages installed with pacman since the last switch by the [drift](/docs/concepts/pacman-and-yay/#drift) policy,
- checks that every managed file can replace what's there now, keeping a backup ([below](#managed-files-backups-and-hand-edits)),
- checks a sudoers drop-in with `visudo` and tmpfiles lines with `systemd-tmpfiles`, on copies of what the switch would write, so a bad sudoers file never reaches `/etc`,
- with the [firewall](/docs/guides/environment-and-firewall/#firewall) on and `nft` installed, checks the generated rules with `nft -c -f`,
- checks that the ESP can hold a new boot image,
- lists the changes, refuses [unsafe downgrades](#unsafe-downgrades) and [partial downgrades](/docs/concepts/generations/#partial-downgrades) unless you allow them, shows the plan and the config diff since the last generation archix applied, and asks you to confirm,
- stores every managed file's contents, and downloads files declared with a `url`, checking each against its `sha256`,
- builds every AUR package in a clean chroot,
- downloads every package and checks it against the lock,
- checks that no two packages ship the same file, and that no incoming file is already there without an owner ([below](#packages-that-ship-the-same-file)),
- has pacman check every incoming package's signature, and checks that the filesystems have room for the incoming files ([below](#signatures-and-disk-space)).

If anything fails here, a typo, a network error, an AUR build that won't compile, a bad signature, a full disk, a declined prompt, archix stops with `nothing was changed`, and no snapshot is taken. Most problems surface in this phase.

**Apply** then changes the running system. With [rollbacks](/docs/reference/options/#rollbackenable) on, it first takes a read-only snapshot of the root, which holds the system as it was before:

```console
$ sudo archix switch
upgrade    linux 7.2.8.arch1-1 -> 7.2.9.arch1-1
upgrade    mesa 1:25.3.0-1 -> 1:25.3.1-1
:: the plan since the last generation archix applied
package ~ linux 7.2.8.arch1-1 -> 7.2.9.arch1-1
package ~ mesa 1:25.3.0-1 -> 1:25.3.1-1
apply 3 changes? [Y/n]
snapshot 12 holds the system as it was before this change
:: applying generation 43
...
the running kernel was replaced: reboot to run the new one (the running one's modules stay loadable until then)
:: building the boot images
switched to generation 43 (snapshot 2026-10-15)
```

The apply runs in this order, on the root itself:

1. **The package transaction.** pacman installs and removes packages directly, keyring packages first. Install scriptlets and the packages' own pacman hooks run as with plain pacman; archix's [own hooks](/docs/concepts/pacman-and-yay/#archixs-pacman-hooks) recognise the transaction as archix's and stay out of it. The running kernel's modules are kept loadable until you reboot, and DKMS is reconciled.
2. **Users, groups and passwords**, then home directories for new users.
3. **Files and units.** Services the config turns off are disabled, managed files are written, file capabilities set, services the config turns on enabled, and archix's own files, the repos in `pacman.conf`, `fstab` and the hibernation guard written.
4. **Secrets** are decrypted into `/run/archix/secrets`.
5. **Services.** On the booted system, `systemctl daemon-reload`, units turned off are stopped, units turned on are started, and units whose files the switch replaced are restarted with `try-restart`, except the ones a restart would cut you off from, such as the display manager and `sshd`, which are listed for later; [`restart_if_changed`](/docs/reference/options/#restart_if_changed) changes that per unit.
6. **Hooks.** [`on_change` scripts and hooks](/docs/guides/hooks/) run.

Then the generation is recorded and made current, the boot images are built from the root as it is now, older snapshots are pruned, and the boot menu, the rescue entry and boot counting are written; see [Boot and fallback](/docs/concepts/boot-and-fallback/). A new kernel runs from the next boot; everything else takes effect now.

### Packages that ship the same file

pacman only finds out that two packages own the same file once it's installing them. archix checks first. Once every package is downloaded or built, it reads each incoming package's file list and compares it with the other incoming packages and with the installed packages the switch keeps:

```console
error: pacman would refuse these packages over files that are already taken; nothing was changed. Drop one of each pair from the config, or move the stray files away:
  usr/lib/udev/rules.d/70-wooting.rules is in both wooting-udev and wootility
```

Packages that are leaving don't count, and that includes the ones pacman swaps out. An incoming package's `conflicts` and `replaces` name packages that go when it arrives, and an installed package whose `conflicts` or `replaces` names an incoming one goes too, as when CachyOS's `zlib-ng-compat` and Arch's `zlib` replace each other.

It also looks at the root. An incoming file that already exists there, with no installed package owning it, is what pacman reports as "exists in filesystem", and archix refuses it the same way, as `usr/bin/foo from foo exists on disk, and no package owns it`. Files a package lists as `backup`, its config files, don't count, since pacman installs those as `.pacnew` instead.

A [revert](/docs/concepts/generations/#reverting) runs the same check.

### Signatures and disk space

archix asks pacman and the disk before it changes anything, once every package is downloaded or built:

- **Signatures.** pacman loads every incoming package with `pacman -U --print`, which checks signatures and keys exactly as the install will, and installs nothing. A package it refuses, signed by an unknown, expired or revoked key or with a bad signature, stops the switch with `pacman refuses these packages, so nothing was changed:` and pacman's own errors, plus a line from gpg's reading of the signature that names the package and the signer, since pacman only says `required key missing from keyring`: ``linux-7.2.9.arch1-1-x86_64.pkg.tar.zst is signed by Jan Packager <jan@archlinux.org>, whose key expired on 2026-10-01; a newer keyring package from its repo (archlinux-keyring for Arch's) renews it``.
- **Disk space.** Like pacman, archix counts per mount. Every incoming file's size, from the package's `.MTREE` and rounded up to whole blocks, counts on the mount it lands on, with a cushion of 5% of the filesystem, at most 128 MiB, for pacman's extraction and the new boot images. The versions it replaces free nothing, since the snapshot taken before the switch keeps them. Otherwise it names the mount: ``/ has 96 MiB free, but this needs about 412 MiB; nothing was changed. Free some space (`archix gc`, `pacman -Sc`) and try again``. btrfs can show free space and still fail writes once data chunks fill the device, so on a btrfs mount with under 1 GiB unallocated, archix first hands empty data chunks back with `btrfs balance start -dusage=0`, and if metadata still has no room it stops: ``btrfs on / has room for data but almost none for metadata (12 MiB of metadata free, 1 MiB unallocated), so writes would fail halfway; nothing was changed. Run `sudo btrfs balance start -dusage=10 /` and try again``.
- **Downloads** are counted first: before any starts, the official packages a switch has to download are sized from the snapshot's repo databases, and if `/var/cache/archix/pkg` can't hold them, the switch stops with ``/var/cache/archix/pkg has 12 MiB free, but this needs about 380 MiB; nothing was changed``. Third-party packages and AUR builds from a package cache aren't sized ahead; if the disk fills while they download, the switch stops there with ``/var/cache/archix/pkg ran out of space while downloading packages; nothing was changed. Free some space (`archix gc`, `pacman -Sc`) and try again``. A download cut short is never kept under the package's real name.
- **Inodes.** On filesystems that report free inodes, which btrfs doesn't, the root has to have enough for every incoming file, plus a margin: ``/ has 6120 free inodes, but this transaction takes about 48300; nothing was changed. Free some (`archix gc`, or delete many small files), or give the filesystem more inodes``.

`switch`, `revert`, `install`, `remove` and `upgrade` run these checks. The snapshots archix keeps have a free-space floor of their own, [`rollback.min_free`](/docs/reference/options/#rollbackmin_free); see [Retention](/docs/concepts/snapshots/#retention). The ESP has its own check, for the boot image: ``nothing was changed: the ESP at /efi has 40 MiB free (12 MiB more once older snapshots' entries go), but a new boot image needs about 70 MiB; free up space on it``.

A revert installs packages that were verified when they were first installed, by the sha256 the generation pinned and without their `.sig` file, so a packager key revoked since doesn't block going back.

### Keyrings

Packages that ship a pacman keyring, the files `pacman-key --populate` reads from `/usr/share/pacman/keyrings`, get two rules: `archlinux-keyring`, and third-party ones such as `cachyos-keyring` and `chaotic-keyring`.

- **They go in first.** A newer snapshot can bring packages signed by a packager that only its newer `archlinux-keyring` knows. Plain pacman refuses those until you update the keyring on its own. archix runs the signature check above against a copy of pacman's keyring with the incoming keyrings already populated into it, and installs the keyring packages in a transaction of their own first, so pacman checks the rest against the keys they add. If the copy can't be made, the check warns and uses the installed keyring.
- **They never go back.** An older keyring trusts packager keys that were revoked since, and lacks the ones added since. A switch or a revert that would downgrade one keeps the installed version instead: `keeping archlinux-keyring 20260909-1 rather than going back to 20260602-1: an older keyring trusts keys revoked since`.

### One archix at a time

Only one archix changes the system at a time. `switch`, `install`, `remove`, `upgrade` and bare `archix`, `rollback`, `revert`, `user unlock`, `gc`, `snapshot create`, `delete` and `setup`, `resync`, `init`, `secure-boot setup`, `revoke` and `rotate-pcr-key`, `secrets init` and `rotate-machine-key` and the `--machine` forms of `secrets edit` and `rekey`, and the boot-time checks take an exclusive lock on `/var/lib/archix/running`. A command that finds it held by archix's own boot checks, which a counted boot runs for a moment every minute, waits up to two minutes, `archix: waiting for archix unblessed to finish`; any other holder stops it at once. The other way round, archix's boot checks wait for a run of yours, however long it takes, rather than fail, so a switch you start in a counted boot's first minutes never makes a healthy boot fall back:

```console
error: another archix is already running (pid 4242, `archix switch`); wait for it to finish
```

Commands that only read or write the config, `status`, `diff`, `search`, `eval`, `options`, `generations`, `snapshot list`, `lock` and `update`, don't take it.

archix also checks pacman before it starts. If any process holds pacman's lock open, `switch` refuses with `pacman is running; wait for it to finish`. A `db.lck` that's left over with nothing holding it is refused too, and archix tells you why.

Packages are downloaded to `/var/cache/archix/pkg`, and every file is checked against the `sha256` in the lock before it's used. Files already in pacman's own cache are reused. A file that doesn't match its checksum stops the switch here. The same goes for files declared with [`archix.fetchurl`](/docs/reference/lua-api/#archixfetchurl):

```console
error: https://downloads.example.org/hello/1.2.0/hello-x86_64 has sha256 0c1f3a5e7b9d2f4a6c8e0b1d3f5a7c9e2b4d6f8a0c1e3b5d7f9a2c4e6b8d0f1a, but the config says 5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03
```

The tools archix runs and reads the output of, pacman among them, run with `LC_MESSAGES=C`, so they answer in English whatever your locale. Started from a login session, the commands that change the system re-run themselves in a systemd scope of their own (`archix-<pid>.scope`), so logind ending the session, as it does when an ssh connection drops with `KillUserProcesses=yes`, doesn't reach them.

### When a switch refuses outright

Some states stop a switch before it evaluates anything, each with `nothing was changed`:

- **A try boot.** When the running system is a snapshot booted read-only from the menu, a switch would be lost at the next boot: ``this boot is snapshot 9 in try mode, read-only with a throwaway layer, so a switch would be lost at the next boot; `archix rollback 9` to keep it, or reboot into the current system; nothing was changed``. See [Try mode](/docs/concepts/boot-and-fallback/#try-mode).
- **A pending rollback.** After `archix rollback N`, until the reboot: ``rollback to snapshot 9 is pending; reboot, or `archix rollback --cancel`; nothing was changed``.
- **A root archix didn't produce**, such as one a snapper rollback put in place, until `sudo archix resync` adopts it; see [A root archix didn't produce](/docs/concepts/snapshots/#a-root-archix-didnt-produce).

## What takes effect when

Everything a switch applies takes effect as it runs: packages, `/etc` files, users and groups, services, [`sysctl`](/docs/reference/options/#sysctl) settings, [udev rules](/docs/reference/options/#udev_rules), which it reloads and replays for every device, the firewall, dotfiles and Flatpak apps. Services start, stop and restart as described [above](#prepare-then-apply). Some things wait:

- **A new kernel** runs from the next boot. Until then the running kernel's modules stay loadable, and the switch says `the running kernel was replaced: reboot to run the new one (the running one's modules stay loadable until then)`. Before its transaction, the switch copies them aside, and it puts them back after: `:: kept the running kernel's modules, so they still load until you reboot`. A plain pacman transaction that names the running kernel's package does the same, through archix's pacman hooks: `:: put the running kernel's modules back, so they still load until a new image boots`. Once another kernel runs, the next package change drops them, unless their package owns that release again.
- **[`modprobe`](/docs/reference/options/#modprobe) options** apply when a module next loads; a switch doesn't reload modules, so for one already loaded they apply from the next boot.
- **Mounts, swap, zram and tmpfiles lines** apply from the next boot: a switch writes `/etc/fstab` and `/etc/tmpfiles.d/archix.conf`, but doesn't mount or create anything.
- **Units archix doesn't restart**, the display manager, dbus, logind, NetworkManager, sshd, gettys and the desktop session's own units unless [`restart_if_changed`](/docs/reference/options/#restart_if_changed) says otherwise, run their old code until you log out or reboot.

[Flatpak apps](/docs/guides/flatpak-and-appimages/) are installed, pinned and removed once the rest of the switch has applied, even when the same switch brings `flatpak` itself.

## The config directory

Whoever can write the config directory decides what the next `sudo archix switch` applies as root. If the config lives in your home and your account is compromised, malware running as you can edit it and wait. archix doesn't pretend to stop that, but it makes sure you see it and that nothing else about the config runs as root:

- **You see what changes.** Every switch shows the plan, what it changes in packages, files, units, users and groups since the last generation archix applied, and the config's own diff since then, from root's copy, before it asks. `--noconfirm` skips only the question. A config with uncommitted changes gets `warning: /home/me/config has uncommitted changes`.
- **git runs as the owner.** archix captures the config with git as the directory's owner, with that user's groups, `no_new_privs` and a cleared environment, in a throwaway repository, never as root and never through `safe.directory`, so a hostile `.git/config` runs nothing as root. Files git ignores aren't part of the capture, so a config that reads one, through `require`, `archix.read_file` or a `source`, fails, naming the file: `... gitignored, so not in the capture of the config, which is what archix evaluates: a generation must be reproducible from its tree. Stop ignoring ..., or keep a secret in it as an encrypted file (secrets.items)`. A file its owner can't read fails the switch with git's error naming it, and `/home/me/config must be readable by its owner me, who archix reads it as`.
- **Evaluation is sandboxed.** root unpacks the captured tree into a directory of its own, checks that its tree id is the one captured, and evaluates it in a child process under a transient UID from systemd's `DynamicUser` range that no account or process has, so your own processes can't trace it, with `no_new_privs`, 60 seconds of CPU, 2 GiB of memory and a 2-minute timeout: ``evaluating the config took longer than 120 s, so it was stopped; nothing was changed``. The Lua it runs has `table`, `string`, `utf8`, `math` and `coroutine`, and no `io`, `os`, `load` or `dofile`; `require` and [`archix.read_file`](/docs/reference/lua-api/#archixread_file) stay inside the config, and a link out of it is refused. What comes back is a plan, and root applies exactly that plan.
- **root keeps its own copy.** Each applied generation's config tree, lock and evaluated plan are stored in a repository only root can read, `/.archix/config.git` with rollbacks on, else `/var/lib/archix/config.git`. [Revert](/docs/concepts/generations/#reverting) and [rollback](/docs/concepts/snapshots/#keeping-a-snapshot) apply from that copy, checked against the hashes the generation recorded, and never evaluate old Lua again. Your repository gets `refs/archix/generations/<n>` pointing at each applied tree, written as you, for browsing only; archix never reads it back.
- **Files archix writes into the config**, `installed.lua` and `archix.lock`, are written as the directory's owner, through a temporary file and a rename, never through a symlink.

For a machine where that's too much trust, keep the config where only root can write. `archix init` asks `Where should the config live?`; the default, `/etc/archix`, is root's, and any other directory you name gets `/etc/archix` linked to it.

## Secure Boot

With Secure Boot enforcing, an unsigned boot image would be refused by the firmware. So a switch or revert whose images wouldn't be signed, with `boot.secure_boot = false` or with sbctl holding no keys, refuses before it applies anything: ``Secure Boot is on, but this switch's boot images wouldn't be signed (sbctl has no keys), so the firmware would refuse them; nothing was changed. Run `sudo archix secure-boot setup`, or set boot.secure_boot = true if sbctl already has keys``. Once the boot images are built, and before the boot menu points at them, archix checks them and the systemd-boot loader on the ESP for a signature. An unsigned one leaves the menu as it was, and the switch lists it as a [soft failure](#hard-and-soft-failures): ``Secure Boot is on, but 7.2.9-arch1-1-aedff12d49d2.efi isn't signed, so the firmware would refuse it; the menu still starts the old images. Run `sudo archix secure-boot setup` (or set boot.secure_boot = true if sbctl already has keys)``. Inside a chroot, where the firmware is the host's, the check only warns. See [Secure Boot](/docs/guides/secure-boot/).

## Unsafe downgrades

Some packages migrate their data when a new major version starts, so the older version can't read it again: PostgreSQL, MariaDB, MySQL and a few more. A downgrade across such a version can leave a service that won't start, or worse. `switch` and `revert` list those downgrades before the prompt and ask, defaulting to no:

```console
unsafe downgrades: the older versions may refuse or damage data the newer ones wrote
  postgresql 17.2-1 -> 16.6-1 goes back from major version 17 to 16, and postgresql migrates its data between them
downgrade anyway? [y/N]
```

Pressing Enter stops with `aborted; nothing was changed`. `--noconfirm` refuses them, ``refusing unsafe downgrades with --noconfirm; to go ahead, run `archix switch` or `archix revert` with --allow-unsafe-downgrade``, and `--allow-unsafe-downgrade` goes ahead without asking. Add your own boundaries with [`unsafe_downgrades`](/docs/reference/options/#unsafe_downgrades). See [Downgrades that may break data](/docs/concepts/generations/#downgrades-that-may-break-data) for the full list.

Keeping a snapshot with `archix rollback N`, or booting one from the menu, doesn't ask: it restores the snapshot's root as it is, and data in `/var/lib` that a newer version migrated stays migrated.

## Interrupted switches

A switch stopped during prepare, by Ctrl-C, `kill -9`, a full disk or a power cut, has changed nothing. Once it applies, it changes the running system step by step, and a step that fails can't be undone by the switch itself. It says which steps were applied, that the new generation wasn't recorded, and which snapshot holds the system as it was before:

```console
error: generation 43 wasn't recorded, and generation 42 stays current; snapshot 12 holds the system as it was before (`archix snapshot list`): files failed after these were applied: the package transaction, users and groups, passwords, home directories; the system is part way between the two states: ...
```

From there, fix the cause and run `archix switch` again, which applies the whole generation, or go back to the snapshot with [`archix rollback 12`](/docs/concepts/snapshots/#keeping-a-snapshot), which makes it the root from the next boot.

With rollbacks on, and from the boot after the first switch on, when there's a snapshot to fall back to, the switch marks itself unfinished before it applies anything ([before that, a cut has no fallback](/docs/concepts/limitations/#power-cuts)), and a switch that fails part way keeps the mark and says so: `...; the system is marked as part way, so its next counted boot doesn't count as good and falls back to the slot, unless a switch finishes first`. A switch that fails before changing anything drops the mark and says nothing was changed; a package transaction that got as far as pacman's commit is never reported as having changed nothing.

A power cut or a crash while a switch applies leaves the same mark. On systemd-boot with boot counting, the next boot fails its health check and falls back by itself to the slot, the system from before the switch, in try mode; see [What falls back by itself](/docs/concepts/boot-and-fallback/#what-falls-back-by-itself). Keep it with `archix rollback N`, or boot the current entry again and run `archix switch` again. On GRUB, Limine or ext4, pick the snapshot the switch took in the boot menu, or use the [rescue entry](/docs/guides/recovery/#without-a-live-usb).

`installed.lua` and `archix.lock` are written only once the switch has applied; see [Declined and killed switches](#declined-and-killed-switches).

### Upgrading archix itself

archix is one of the packages a switch installs, so a switch can replace archix while it runs. The running archix finishes the switch. Its pacman hooks and boot units call the archix binary by its path, so from then on they run the new one, and every switch, even one with nothing else to do, writes them again. Generation records, snapshot records and stored plans carry a format version, and every newer archix reads every older format, so a snapshot's own initramfs, or a frozen rescue entry, can read records a newer archix wrote. An older archix refuses a plan in a newer format: `the plan is format 2, newer than this archix reads (1); update archix`.

## Atomic writes

Managed files are written to a temporary file and renamed into place, with their mode and owner already set. A rename is atomic, so a file is either the old version or the new one, never half of each.

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

archix only replaces a directory it has never managed, and only once. It refuses when an earlier backup of that path still exists, `/home/me/.config/nvim is a directory and an earlier backup of it exists; move one of them away`, and when archix managed a file at that path in the current generation: `/home/me/.config/nvim exists and is not a file or symlink; archix won't replace it`. Both `switch` and `revert` check this before they change anything, so a refusal is a plain error. Move the directory or the old backup aside and run the command again.

Users and groups archix created are removed when they leave the config, and their home directories are kept. A new user without a password in the config gets a hint, ``created user me; set a password with `passwd me` ``. Keeping a snapshot carries accounts made since it was taken; see [Accounts after a keep](/docs/concepts/snapshots/#accounts-after-a-keep).

The same goes for the [firewall](/docs/guides/environment-and-firewall/#firewall): it lives in its own `inet archix` table, and turning it off deletes only that table, never the ones Docker, Tailscale or libvirt manage.

archix is strict about what *it* manages, and only that. It doesn't revert changes to `/etc` files it never managed, and it doesn't disable services it never managed. Doing that blindly could, for example, delete users that packages create. `archix init` captures the files and services you'd changed, which makes them managed.

Every switch also stores the contents of every file its generation manages in `/var/lib/archix/blobs`, so a revert never finds them missing.

### What init captures and never captures

`archix init` looks for the `/etc` files you changed: package config files edited since they were installed (it compares them with the checksums in pacman's database), and files nobody owns in well-known directories such as `/etc/systemd/system`, `/etc/sudoers.d`, `/etc/sysctl.d`, `/etc/modprobe.d`, `/etc/udev/rules.d`, `/etc/X11/xorg.conf.d`, `/etc/pacman.d/hooks` and `/etc/polkit-1/rules.d`. It sorts what it finds into three groups:

- **Secrets** are never shown or copied, only counted: `· 3 files that hold secrets stay out of the config and aren't listed`. That covers SSH host keys, NetworkManager connections, wpa_supplicant, iwd, WireGuard, OpenVPN and PPP configs, `/etc/ssl/private`, `/etc/cryptsetup-keys.d`, `krb5.keytab`, `smbpasswd`, `ipsec.secrets`, and any path containing `secret`, `password`, `passwd`, `token`, `credential`, `keyfile`, `.key`, `.pem`, `.p12` or `.pfx`. A file also counts as a secret by what's in it: private keys, including PGP private key blocks, GitHub, GitLab, Slack and Anthropic access tokens, AWS secret keys, OAuth refresh tokens, and `psk=` or `password=` lines. The full list is `secret_content` in `src/capture.json`. Files over 1 MiB aren't read, and neither is a directory past its first 200 files.
- **Worth managing** files are pre-selected: `/etc/fstab`, `/etc/crypttab`, `/etc/mkinitcpio.conf`, `/etc/sudoers`, `/etc/pacman.d/mirrorlist`, `/etc/makepkg.conf`, `/etc/hosts`, `sshd_config`, the systemd, NetworkManager, bluetooth and display manager configs, and everything hand-written in the directories above.
- **The rest**, mostly files that package install scripts change, such as `/etc/shells`, are off by default.

init's Config files step shows them all in one checklist, `Config files archix manages`, each tagged with why it was found: `recommended` for the ones worth managing, then `changed from its package's version` or `not from any package`. The recommended ones start ticked. `space` toggles a file, `a` and `n` select all or none, and `r` goes back to the recommended set. `--defaults` takes the recommended ones. Chosen files are copied into `files/<host>/` and declared with their mode, owner and group.

It never captures files that archix or other tools own: `/etc/pacman.conf`, `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow` and their backups, `/etc/machine-id`, `/etc/resolv.conf`, `/etc/mtab`, `/etc/ld.so.cache`, `/etc/kernel/cmdline`, pacman's keyring, the extracted CA certificates, and archix's own files.

Since init only declares what's already there, the first switch after it changes no packages. It records generation 1, writes the boot entries, and with rollbacks on lays the root out for snapshots, which start after the next reboot; dotfiles you chose to import are replaced with links into the config. init's Result section lists those under `! the first switch would still change these:`.

## Hard and soft failures

Not every failure means the same thing.

**Hard failures** stop the switch. Before the apply, that's a refusal, and nothing was changed. During the apply, it's a step that failed: a pacman error, a file that can't be written, a user that can't be created. The switch stops there, the new generation isn't recorded, and the error says what was applied and which snapshot holds the system from before, as in [Interrupted switches](#interrupted-switches).

**Soft failures** don't stop the switch: a unit that can't be enabled, started or restarted, a hook or `on_change` script that fails, a Flatpak app that can't be installed, pinned or removed, a boot image that doesn't build, `gc`. The new generation is recorded and current, archix lists what failed and exits non-zero, like NixOS does:

```console
switched to generation 43 (snapshot 2026-10-15)
failed: the boot images weren't built, so the boot menu still starts the old ones: ...
error: generation 43 is applied and current, but 1 step failed (listed above)
```

See [Activation hooks](/docs/guides/hooks/#failures) for failing hooks.

## Declined and killed switches

Nothing a switch changes in your config is written before the switch has applied. `install` and `remove` evaluate the edited `installed.lua` from memory, as does a switch that adds [drifted packages](/docs/concepts/pacman-and-yay/#drift), and the new `archix.lock` stays in memory too. So declining, a failed check, a failed build, a kill or a power cut before the apply leaves both files byte for byte as they were.

archix runs as root, but files it writes in your config are written as the directory's owner, so `git commit` keeps working after `sudo archix install`.
