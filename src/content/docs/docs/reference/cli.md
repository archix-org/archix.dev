---
title: CLI reference
description: Every archix subcommand, pacman-style alias and flag, with examples and exit behaviour.
sidebar:
  order: 1
---

```
archix [--config <dir>] [--host <name>] <command> [flags]
archix [--config <dir>] [--host <name>] <terms>...
```

Running `archix` with no command does a full upgrade, like `yay`. See [`upgrade`](#upgrade--syu).

## Search and install: `archix <terms>`

Like `yay <terms>`, words that aren't a command are a search. archix searches the configured repos at the lock's snapshot and the AUR, numbers the results with the best match as 1, printed last, right above the prompt, and installs what you pick:

```console
$ archix btop
3 aur/btop-gpu-git 1.4.5.r12.g5d3e2a1-1 (+4 0.12)
    A monitor of resources, with GPU support
2 aur/btop-git 1.4.5.r3.gc1a2b3d-1 (+12 0.35)
    A monitor of resources
1 extra/btop 1.4.5-1
    A monitor of system resources, bpytop ported to C++
==> Install which? Numbers (1 3), ranges (2-5), ^ to leave one out: 1
...
install    btop 1.4.5-1
a new deployment with its own boot entry; it boots at the next restart
apply 2 changes? [Y/n]
```

Matching is the same as [`search`](#search--ss): every term has to appear in the name or the description. Results are ranked: an exact name first, then names starting with the first term, then names containing a term, then matches in the description only. Within each group, repo packages come before AUR ones, then they're sorted by name.

The prompt takes numbers (`1 2 3`), ranges (`1-3`) and exclusions (`^4`, `^2-3`), separated by spaces or commas. Exclusions on their own mean all but those, and an empty answer installs nothing. Anything else stops with `"x" isn't a number, a range like 1-3 or an exclusion like ^4`, or `6 is outside 1-5`. With no results, it prints `no packages match btpo` and exits.

The picks go through [`install`](#install--s): they're declared in `lua/hosts/<host>/installed.lua`, then archix switches, with the usual preview and `apply N changes?` prompt, and puts `installed.lua` back if that fails. Run as a normal user, archix calls sudo itself for that part, like yay does, running `sudo` on its own binary with `--config <dir> --host <host> install`, `--noconfirm` if you gave it, and `-- <picks>`.

Commands always win over terms: `archix status` runs `status`, and a typo such as `archix swtich` searches for "swtich". `--noconfirm` can go anywhere among the terms; it's taken out of the search and skips the switch's question. Bare `archix`, with no terms, is still the full upgrade.

`ax` is a short alias for `archix`. The AUR packages install it as a symlink, `/usr/bin/ax`, and it takes every command and flag below: `sudo ax switch`, `ax diff`, `sudo ax -Syu`. Help and usage errors use the name you typed, such as `Usage: ax [OPTIONS] [COMMAND]`, while archix's own messages, `--version` included, still say `archix`. See [Installation](/docs/installation/#ax-the-short-name).

## Root

Commands that change the system need root: `install`, `remove`, `switch`, `upgrade` (and `-Syu` and bare `archix`), `rollback`, `gc`, `hooks`, `verify`, `eject`, `init`, `secure-boot setup`, `secure-boot revoke` (but not `--dry-run`) and `secure-boot rotate-pcr-key`. Run as your user, they stop before touching anything, `installed.lua` included, and give you the line to run, the way you typed it:

```console
$ archix -S htop
error: `archix install` changes the system, so it needs root; run it with sudo: sudo archix -S htop
```

Only one of these runs at a time; see [Commands that change the system](#commands-that-change-the-system). Any other error that comes from a missing permission gets the same hint, such as `archix lock` on a root-owned `/etc/archix`: `hint: that needs root; run it with sudo: sudo archix lock`. Reading doesn't need root: `archix <terms>`, `search`, `diff`, `status`, `eval`, `options`, `generations` and `secure-boot check` work as your user, and so do `lock` and `update` on a config you own. `archix <terms>` asks sudo for the install itself.

Every command except `init`, `options`, `rollback`, `gc`, `hooks`, `verify`, `generations`, `eject`, `prefetch`, `cache push` and `secure-boot` reads the config. `options` works without one, listing every option against an empty config. With no `init.lua` in the config directory, the others stop with:

```console
error: there's no archix config at /etc/archix yet; run `sudo archix init` to write one that describes this machine, or point --config at yours
```

## Global flags

These work with every command.

| Flag | Default | Description |
| --- | --- | --- |
| `--config <dir>` | `/etc/archix` | The config directory containing `init.lua`. Also read from the `ARCHIX_CONFIG` environment variable. |
| `--host <name>` | the machine's hostname | Evaluate as this host instead. Sets `archix.hostname`. |
| `--help` | | Show help for archix or a command. |
| `--version` | | Print the archix version. |

```console
$ archix --config ~/archix --host laptop diff
$ ARCHIX_CONFIG=~/archix archix eval
```

`--host` changes which host's config is evaluated and locked. It doesn't reach other machines: switching over ssh is not supported.

## Setting up

### `init`

Write a config that describes this machine as it is, and a lock pinned to exactly the package versions that are installed. Needs root. A config written inside your home, such as `sudo archix init` with `~/archix-config` as the location, ends up owned by you, the whole tree including the `.git` init creates, so you can edit and commit without `sudo`. Exactly: init hands the tree to the owner of the config directory's parent, when that isn't root. The Config step also has `Write files read-only, like the Nix store (apps can't change them; a switch puts them back)`, off by default and with `--defaults`; on, init writes `mutable_files = false` (see [Read-only files](/docs/guides/dotfiles-and-apps/#read-only-files)). Picked dotfiles are imported as live links, or as copies for files their app replaces on save; see [Bringing your existing dotfiles in](/docs/guides/dotfiles-and-apps/#bringing-your-existing-dotfiles-in).

```console
$ sudo archix init
```

`init` opens a full-screen wizard with nine steps: Welcome, Config, Packages, System, Config files, Boot, Generations, Dotfiles and Review. It asks where the config should live, whether to make it a git repository, what to do with orphaned packages and with packages you later install with plain pacman or yay, which settings, users, groups and services to take over (four checklists, all ticked), which config files to manage (a checklist, each file tagged `recommended`, `changed from its package's version` or `not from any package`), whether to boot every generation as its own deployment from a systemd-boot entry, how many generations to keep, and which dotfiles to bring in (recommended ones start ticked). Every answer starts at the recommended one.

Focus moves one line at a time, over every option, setting, text field, checklist entry and button; `›` marks it.

| Key | What it does |
| --- | --- |
| `↑` `↓`, `tab`, `shift-tab` | Move the focus to the next or previous line. |
| `enter`, `space` | Act on the focused line: pick an option, switch a setting or checklist entry, start editing a text field, or press a button. |
| `←` `→` | Previous or next step, like the buttons; `→` is refused while a text field is invalid, and on Review it only focuses `Write the config ›`. |
| `End` | Focus the `Next` button. |
| `esc` | Previous step (while editing a text field, put the old value back instead). |
| `a`, `n` | Select all or none in a checklist. |
| `r` | Reset a checklist to its recommended entries. |
| `q`, `ctrl-c` | Quit without writing anything. |

Text fields start empty and show their default in gray, `/etc/archix (default)`, `10 (default)` and `30 (default)`; left empty, the default applies. They're edited in place: `enter` starts editing, `enter` keeps the value and `esc` puts the old one back, and only while editing do typed keys go into the field. A typed config location must be an absolute path and typed generation limits whole numbers, or `Next` and `→` refuse to leave the step. Steps with nothing to answer, Welcome and Review, start on `Next`, and `↑` `↓` scroll them; the others start on their first field. Quitting prints `init was cancelled; nothing was written`.

Nothing is written until `Write the config ›` on Review. Then the wizard closes, and init writes the config and the lock and shows what the first switch would change:

```
── Lock ────────────────────────────────────────
  · finding the archive snapshot closest to what's installed
  ✓ 212 packages locked to the 2026-10-09 snapshot
  · held at their installed version instead: mesa
  ✓ 2 Flatpak apps locked to their installed commits

── Result ──────────────────────────────────────
  ✓ the system matches the lock and the config: the first switch changes nothing

  Config written to /etc/archix
    init.lua                          loads this host
    lua/hosts/desktop/init.lua        settings, users, services and files
    lua/hosts/desktop/installed.lua   packages; archix -S and -R edit it
    files/desktop/                    copies of the managed config files
    archix.lock                       the exact package versions

  Next: look it over, then run sudo archix switch
```

System-wide Flatpak apps are declared too, each locked to the commit it's installed at; `--user` apps are left out, and so are apps whose remote is gone. `init` changes nothing on the system except the config directory. The first `sudo archix switch` lays the system out as deployment 1, with a boot entry of its own, without changing any packages; see [Atomic deployments](/docs/concepts/atomicity/#requirements) for what it needs. Imported dotfiles are the exception: the first switch replaces them with links, and the Result section lists them under `! the first switch would still change these:`. Secret files are never shown or copied, only counted. See the [quick start](/docs/quick-start/) for every step.

| Flag | Description |
| --- | --- |
| `--defaults` | Take every recommended answer without opening the wizard: everything in System, the recommended config files, no dotfiles (it writes no `home` section and touches nothing in your home), and a git repository (`git init`, `git add -A`) when git is installed. It needs no terminal, and prints what it captured in Packages, System, Config files, Boot and Generations sections before Lock and Result. The Boot section says whether the first switch can lay the system out as deployments, and why not when it can't, such as `the first switch can't lay this system out as deployments: the ESP is mounted at /boot, in /boot, which each deployment keeps its own copy of; mount it at /efi instead (in /etc/fstab, and boot.esp = "/efi"), then switch again`. Config files lists the recommended files it chose, then `· N more files left unmanaged; the wizard (archix init without --defaults) lets you pick them`. Its output includes `! their remote is gone, so archix leaves these Flatpak apps alone: ...` when that applies. Without it, `init` needs a terminal: `` `archix init` asks questions; run it in a terminal, or pass --defaults to take every default ``. |
| `--repack PACKAGE` | Rebuild this package, which no repo or the AUR has, as a recipe in `pkgbuilds/` from its installed files, as ticking it in the wizard does. Its files stay in `/var/cache/archix/sources`, out of the config, and its builds out of package caches; a config file you edited goes in as the package shipped it, from a cached package file, or not at all. Can be given more than once. Without it, `--defaults` rebuilds nothing and leaves such packages in `ignored_packages`. |

`init` refuses to run if the config directory already has an `init.lua`.

## Commands that change the system

Only one of these runs at a time: each takes an exclusive lock on `/var/lib/archix/running`, and a second one stops with `` another archix is already running (pid 4242, `archix switch`); wait for it to finish ``. See [Safety](/docs/concepts/safety/#one-archix-at-a-time).

These need root. Run without it, they stop before touching anything, with the command's name and your exact command line: `` `archix switch` changes the system, so it needs root; run it with sudo: sudo archix switch --noconfirm ``. See [Root](#root).

### `switch`

Build the next deployment from the config and the lock, and record it as a new generation. It boots at the next restart; the running system doesn't change. See [Atomic deployments](/docs/concepts/atomicity/).

```console
$ sudo archix switch
resolving 87 declared packages against the 2026-10-09 snapshot
+ htop 3.4.1-1
213 packages locked
wrote /etc/archix/archix.lock
install    htop 3.4.1-1
a new deployment with its own boot entry; it boots at the next restart
run hooks  tailscale, when the deployment boots
apply 2 changes? [Y/n]
downloading htop-3.4.1-1-x86_64.pkg.tar.zst
:: building deployment 2 from deployment 1
...
hibernation is off until you reboot into deployment 2: resuming would start its kernel on the running system's memory image; suspend still works
switched to generation 2 (snapshot 2026-10-09); its deployment boots at the next restart
deployment 2 is committed and boots at the next restart; changes to /etc made before then are carried over to it
```

`switch` re-locks at the locked date if the config changed, lists the package, file, user, group, service and Flatpak changes, and asks before building. A package built from a recipe in the config that changed while its version didn't is listed as `reinstall  backup-scripts 1.2.0-1 (a build from a different recipe)`, and rebuilt and reinstalled at the same version. Then it downloads files declared with a `url` into the store (each checked against its `sha256`), builds AUR packages and the config's own [recipes and patched packages](/docs/guides/local-pkgbuilds-and-patches/) in a clean chroot, downloads every package to `/var/cache/archix/pkg` (each file checked against the lock's `sha256`, and hard-linked rather than copied when pacman's cache already has it), and builds the deployment: packages, users, groups, services, managed `/etc` files and the boot entry. Once it's committed, `switch` writes the files outside the deployment, such as dotfiles, and runs their `on_change` scripts, starts the [`/etc` merge](/docs/concepts/safety/#changes-to-etc-before-the-reboot), which carries changes you make to `/etc` into the new deployment until the reboot, turns hibernation off until then, syncs Flatpak apps and runs `gc`. Nothing is restarted: the new generation takes effect when you reboot. See [Safety](/docs/concepts/safety/#what-takes-effect-when).

The very first switch lays the running system out as deployment 1 first, `lay out   the system as deployment 1: /usr, /etc, /opt, /boot and pacman's database move into /archix/deploy/1, with a boot entry of its own`, then builds deployment 2 if the config changes anything. It refuses before changing anything when archix doesn't manage booting, the ESP is in `/boot`, or `/boot`, `/usr`, `/opt`, `/var`, `/var/lib`, `/var/lib/pacman` or `/archix` is a separate filesystem; see [Requirements](/docs/concepts/atomicity/#requirements).

With [`hooks`](/docs/guides/hooks/) declared, the preview has a `run hooks` line listing them in the order they'll run, at the new deployment's first boot. Hooks aren't counted in `apply N changes?`.

With [Flatpak apps](/docs/guides/flatpak-and-appimages/) declared, `switch` locks new apps to their remote's current commit, then, once the deployment is committed, adds the remotes they use, locks apps whose remote it just added, installs missing apps, pins every app to its locked commit with `flatpak update --commit=`, and uninstalls the apps that left the config:

```console
flatpak    install org.mozilla.firefox (0bb3a9e40c1f)
flatpak    com.spotify.Client 5f8806197a2b -> 81c2d4e6f0a3
flatpak    remove org.gimp.GIMP
```

When the same switch brings the `flatpak` package, which the running system doesn't have yet, the apps wait for the new deployment: `flatpak    install org.mozilla.firefox at the new deployment's first boot, which brings flatpak`, and once committed, `flatpak comes with deployment 2, so deployment 2's first boot installs them once the network is up: org.mozilla.firefox`. `archix-flatpak.service` then adds the remotes, installs, locks and pins them after `network-online.target`, and tries again at the next boot if that fails.

A Flatpak step that fails is a soft failure, like a unit that can't be enabled. Before it asks, `switch` also checks that no managed file would replace something it can't back up, and stops with a plain error if one would; see [Safety](/docs/concepts/safety/#managed-files-backups-and-hand-edits). Once everything is downloaded and built, and before it builds the deployment, it refuses packages that ship the same file or a file that's already there with no owner, packages pacman's signature check rejects, and a deployment the root filesystem or the ESP has no room or inodes for; see [Signatures and disk space](/docs/concepts/safety/#signatures-and-disk-space). A switch that fails or is killed while building leaves the running system and its boot entries as they were.

If the system already matches, it prints `nothing to do: the system matches generation 2 (snapshot 2026-10-09)`, builds nothing and runs the hooks right away, like NixOS runs activation scripts on every switch. Any other change, even to a setting such as `ignored_packages`, builds a new deployment.

`switch` refuses to start while pacman, pamac, PackageKit or anything else holds pacman's lock open, or while a stale pacman lock (`/var/lib/pacman/db.lck`) exists. It also refuses a lock with packages built for a newer x86-64 level than this CPU runs, which another machine locked, and asks for `archix lock` here; see [`lock`](#lock). If installed packages are newer than the lock, as after a `pacman -Syu` between `archix init` and the first switch, it offers to adopt them first:

```console
the system is newer than the lock for 3 packages (linux, mesa, vulkan-radeon), probably from pacman or an AUR helper
adopt them into the lock? (no reverts them to the lock) [Y/n]
```

With the default `undeclared_packages = "remove"`, it also offers to declare explicit packages pacman installed since the last switch. Yes adds them to `installed.lua` and switches with them; no leaves them out:

```console
installed with pacman or an AUR helper but not declared: htop ncdu
declare them in installed.lua? (no removes them) [Y/n]
```

Downgrades that may break data, such as PostgreSQL going back a major version, are listed after the preview and asked about separately, defaulting to no:

```console
unsafe downgrades: the older versions may refuse or damage data the newer ones wrote
  postgresql 17.2-1 -> 16.6-1 goes back from major version 17 to 16, and postgresql migrates its data between them
downgrade anyway? [y/N]
```

Pressing Enter stops with `aborted; nothing was changed`. See [Downgrades that may break data](/docs/concepts/generations/#downgrades-that-may-break-data) and the [`unsafe_downgrades`](/docs/reference/options/#unsafe_downgrades) option.

See [pacman and yay](/docs/concepts/pacman-and-yay/#adopting) and [Safety](/docs/concepts/safety/).

| Flag | Description |
| --- | --- |
| `--noconfirm` | Don't ask. Builds without the `apply N changes?` question, adopts newer packages, declares packages installed with pacman or yay, trusts new AUR build recipes without showing them, and takes the nearest newer version when a locked AUR version can't be built. It refuses unsafe downgrades: ``refusing unsafe downgrades with --noconfirm; to go ahead, run `archix switch` or `archix rollback` with --allow-unsafe-downgrade``. With the default `third_party_fallback = "ask"`, a locked third-party package that's gone fails the switch rather than being swapped for another; see [The fallback chain](/docs/guides/third-party-repos/#the-fallback-chain). |
| `--allow-unsafe-downgrade` | Go back across versions that may not read newer data (postgresql majors and the like) without asking. They're still listed. |
| `--accept-nearest newer\|older` | When a locked AUR version can't be built, take the nearest newer or older one without asking. Overrides the `missing_versions` option. |

### `upgrade`, `-Syu`

The yay-style full upgrade. Bare `archix` and `archix -Syu` do the same.

```console
$ sudo archix -Syu
$ sudo archix
```

It moves the lock to the newest Arch Linux Archive snapshot, as `archix update` does, moves AUR packages to the newest commit of their build recipes, showing each changed recipe for review, and then switches. The new deployment boots at the next restart. pacman can't upgrade a deployment in place, so it doesn't run `pacman -Syu`.

`archix -Syu <packages>` also declares the packages, like `archix install` with the upgrade.

| Flag | Description |
| --- | --- |
| `--noconfirm` | Skip archix's questions, as in `switch`. |

`-Sy` on its own is refused: ``archix doesn't sync databases on their own; use -Syu to upgrade, or `archix update` to move the lock``. To move the lock without building anything, use [`update`](#update), then `switch`.

### `install`, `-S`

Declare packages and install them, in a new deployment that boots at the next restart.

```console
$ sudo archix install zen-browser-bin
$ sudo archix -S neovim ripgrep
```

The names go into `lua/hosts/<host>/installed.lua`, a plain Lua file archix imports automatically and you can edit too, and then archix switches. A name a recipe in `pkgbuilds/` builds under exactly that name comes from there; other names are looked up in your configured repos, then in `pkgbuilds/` by what recipes `provides`, then in the AUR, and new AUR build recipes are shown for review. A name found nowhere stops it: `` `foo` is in none of the configured repos, pkgbuilds/ or the AUR ``. A name the config already declares somewhere else isn't added again: `tree is already declared in the config; not adding it to lua/hosts/box/installed.lua`, and the switch installs it from there. `installed.lua` and `archix.lock` are written only when the switch commits, so declining, a failed switch or a config that stops evaluating leaves both as they were; see [Declined and killed switches](/docs/concepts/safety/#declined-and-killed-switches).

| Flag | Description |
| --- | --- |
| `--path <dir>` | Edit the `installed.lua` in this directory instead. It must be inside the config's `lua/` directory, or archix refuses. archix prints the module name to import, such as `modules.dev.installed`, since only the host's own `installed.lua` is imported automatically. |
| `--temp` | Install without declaring anything, in a new deployment built on the current one. The package shows up as undeclared, and the next `switch` leaves it out unless you declare it by then. It needs a system already laid out as deployments: ``a temporary install builds on the current deployment, and this system isn't laid out as deployments yet; run `archix switch` first``. |
| `--noconfirm` | Don't ask. |

```console
$ sudo archix install --temp sl
install    sl 5.05-4
install 1 package without declaring them, in a new deployment that boots at the next restart? [Y/n]
:: building deployment 43 from deployment 42
...
installed sl without declaring it, in deployment 43, which boots at the next restart; the next `archix switch` drops it unless it's declared
deployment 43 is committed and boots at the next restart; changes to /etc made before then are carried over to it
```

### `remove`, `-R`

Take packages out of `installed.lua` and uninstall them, in a new deployment that boots at the next restart.

```console
$ sudo archix remove zen-browser-bin
$ sudo archix -R neovim
```

archix only edits `installed.lua`, never your own Lua. A package declared elsewhere is refused: `htop isn't in /etc/archix/lua/hosts/desktop/installed.lua; it's declared elsewhere in the config, so remove it there`.

| Flag | Description |
| --- | --- |
| `--path <dir>` | Edit the `installed.lua` in this directory instead. |
| `--noconfirm` | Don't ask. |

### `rollback`

Make an earlier generation current, the previous one or the one you name. It boots at the next restart.

Rolling back to the current generation rewrites its boot entry, default and fallback boot image when the ESP has none, as after the ESP was lost, and changes nothing else.

```console
$ sudo archix rollback
generation 41 (snapshot 2026-10-09) still has its deployment: rolling back points current and the boot default at it, nothing is reinstalled
downgrade  mesa 1:25.3.1-1 -> 1:25.3.0-1
downgrade  vulkan-radeon 1:25.3.1-1 -> 1:25.3.0-1
run hooks  tailscale, when the deployment boots
roll back to generation 41 (snapshot 2026-10-09)? [Y/n]
rolled back to generation 41 (snapshot 2026-10-09); it boots at the next restart. archix.lock is unchanged, so the next `archix switch` returns to the config
```

When the generation's deployment is kept, rollback checks its files against its packages as [`verify`](#verify) does, repairing what differs from the package cache or stopping with `nothing was rolled back` when it can't, then makes it systemd-boot's default entry and current, and nothing is reinstalled. When it was dropped, rollback builds it again from the nearest kept deployment, the newer one on a tie, and the generation's exact package files, from the cache or the Arch Linux Archive, and commits it like a switch: `generation 38 (snapshot 2026-09-18)'s deployment was dropped: rebuilding it from deployment 41 and the package cache`. Packages go back by the sha256 the generation pinned, without checking their signatures against today's keyring again, so a key revoked since doesn't block the rollback, and like `switch`, a rebuild checks first that no two packages ship the same file and that the disk has room. A package built from one of the config's recipes at the same version, but from a different recipe, is listed as `reinstall  backup-scripts 1.2.0-1 (a build from a different recipe)`, and that generation's cached build goes back.

Either way, the running system's users and passwords are carried into the generation's `/etc` by name (see [Users and passwords across deployments](/docs/concepts/safety/#users-and-passwords-across-deployments)), the `/etc` merge carries later changes to `/etc` until the reboot, files outside the deployment, fetched ones included, come back from the store right away, and Flatpak apps are pinned back to the generation's commits, which are downloaded from their remote, with apps the newer generation added uninstalled. The generation's [hooks](/docs/guides/hooks/) run when it boots. Downgrades that may break data are listed and asked about first, `downgrade anyway? [y/N]`, as on a switch. `rollback` doesn't evaluate the config, so it works even when the config is broken. `archix rollback 42` moves forward again. Naming the current generation prints `generation 42 is current already; nothing was changed`. See [Generations and rollback](/docs/concepts/generations/).

| Argument or flag | Description |
| --- | --- |
| `[generation]` | The generation number. `archix generations` lists them. |
| `--noconfirm` | Don't ask. Refuses [unsafe downgrades](/docs/concepts/generations/#downgrades-that-may-break-data) unless `--allow-unsafe-downgrade` is given too. |
| `--allow-unsafe-downgrade` | Go back across versions that may not read newer data (postgresql majors and the like) without asking. They're still listed. |

### `gc`

Delete the deployments [`deployments.keep`](/docs/reference/options/#deploymentskeep) leaves out, with their boot entries, old generation records and the cached files no kept generation needs: package files, snapshot databases and managed-file contents. It also runs after every switch.

```console
$ sudo archix gc
deleted deployments 39
deleted generation records 30, 31
freed 812.4 MiB: 46 cached package files, 3 snapshot databases
```

It never drops the booted, current or boot-default deployment, the one the [fallback image](/docs/concepts/atomicity/#the-fallback-boot-image) boots, or the one the last switch or rollback chose, and keeps the newest others up to `deployments.keep` (2). Like `switch` and `rollback`, it refuses unless `boot.esp` is a mounted FAT filesystem.

### `hooks`

Run again, or drop, the [hooks](/docs/guides/hooks/#failures) of a deployment that failed at three boots, which no longer run at boot. Both take a deployment number, the running one by default.

```console
$ sudo archix hooks retry
deployment 46's hooks ran
$ sudo archix hooks skip
dropped the hooks waiting for deployment 46; they won't run
```

`retry` runs them on the running deployment, and only that one: `deployment 47 isn't the one running, so its hooks run at its boot`. If one fails again, they stay waiting: `hook link: exited with exit status: 1; they stay waiting, `archix hooks skip` drops them`. With none waiting, both say `no hooks wait for deployment 46`.

### `verify`

Check each kept deployment's `/usr`, `/opt` and `/boot` against its packages, file by file: type, link target, mode, owner, size and sha256, as `pacman -Qkk` does. Files a package lists as `backup` aren't checked.

```console
$ sudo archix verify
deployment 41: matches its packages
deployment 42: 1 files differ from its packages:
  /usr/bin/sudo (sudo): mode 4777, not 4755
error: deployment 42 changed; `sudo archix verify --repair` restores them
```

Up to 20 files are listed per deployment. Packaged files that are missing, as pacman.conf's `NoExtract` leaves them, are counted on their own line and don't fail the check. While the last check found differences, `archix status` says so: ``deployment 42 differs from its packages in 1 files at the last check; `sudo archix verify --repair` restores them``. A [rollback](#rollback) to a kept deployment runs the same check first.

| Flag | Description |
| --- | --- |
| `--repair` | Restore what differs from the package files in archix's cache, pacman's cache or archix's own AUR builds, each written under a new name and renamed into place. A package that isn't cached stops it before anything changes: ``sudo-1.9.17.p2-1-x86_64 isn't in the package cache, so it can't be repaired; download it into /var/cache/archix/pkg and try again``. | Records go when they have no deployment and fall outside the newest [`generations.keep`](/docs/reference/options/#generationskeep) (10) and `generations.keep_days` (30), using the limits the current generation was applied with.

### `eject`

Stop archix managing the machine, and leave plain Arch behind. On a system laid out as deployments, the deployment you're running becomes the root's own `/usr`, `/etc`, `/opt`, `/boot` and pacman database, at once and without a reboot, and `/archix` goes:

```console
$ sudo archix eject
archix eject: archix stops managing this machine, which becomes plain Arch
  system    deployment 42, which runs now, becomes the root's own /usr, /etc, /opt, /boot and pacman database; the running system switches to them at once
  boot      a UKI that mkinitcpio's preset writes to EFI/Linux, signed with sbctl, the default instead of archix's entries, which go with the fallback UKI
  deletes   /archix and every deployment in it
  removes   archix's boot units, the deployment initramfs hook and units, the pacman hooks and the login notices; pacman.conf's CheckSpace is back on
  files     stay as they are, made writable again (--restore-originals puts the originals back)
  keeps     packages, users, groups, services and pacman.conf
  keeps     /var/lib/archix and /var/cache/archix (--purge deletes them)
eject? [Y/n]
...
this machine is plain Arch now and archix no longer manages it; restart when convenient. `pacman -Rns archix` removes archix itself; `archix switch` would lay the machine out as deployments again
```

It copies the deployment's trees into the root's own, as hardlinks on ext4 and reflinks on btrfs, takes archix's integration out of them, then hands the ESP over: it adds a `default_uki` line for `EFI/Linux/arch-<kernel>.efi` to the kernel's mkinitcpio preset, runs `mkinitcpio -p`, signs the UKI when sbctl has keys, makes it `loader.conf`'s default, and deletes archix's entries, boot images and fallback image. With the ESP mounted at `/boot`, it writes plain `arch-<kernel>.conf` entries instead. Then the running system moves off the deployment's mounts onto the root's own trees, and `/archix` is deleted.

The deployment that runs has to be `current`, since that's what you've been using: with a switch waiting for its reboot, it refuses with ``deployment 43 is current, but this boot runs deployment 42; restart into 43 (or `archix rollback 42`) and eject then; nothing was changed``. It also refuses when this boot didn't come from a deployment.

`eject` records how far it got in `/var/lib/archix/eject`. If it's cut off, by a kill or a power cut, the machine still boots, and `sudo archix eject` again finishes it: `archix eject: finishing the eject to plain Arch from deployment 42 that was cut off`. A failure says the same: `the eject stopped partway; the machine still boots, and running `sudo archix eject` again finishes it`.

On a system not laid out as deployments yet, before the first switch, it does the rest: hands the ESP over the same way, takes off archix's boot services, initramfs hook, pacman hooks and login notices, makes managed files writable again, and leaves packages, users, groups, services, the firewall and `pacman.conf` as they are. `eject` doesn't evaluate the config.

| Flag | Description |
| --- | --- |
| `--noconfirm` | Don't ask. |
| `--restore-originals` | Put back the files archix replaced, from its backups, and delete the ones it created, as taking them out of the config would; `/etc/kernel/cmdline` stays, as mkinitcpio reads the kernel command line from it. An eject that was cut off keeps the choice it started with. |
| `--purge` | Also delete `/var/lib/archix` and `/var/cache/archix`. |

`archix recover` was removed with the move to deployments; running it says why and exits.

### `rescue`

The prompt the [rescue entry](/docs/guides/recovery/#without-a-live-usb-the-rescue-entry) opens, from an initramfs that mounts nothing; on a running system it refuses. It first asks for the password of root or of a wheel or sudo user from the current deployment's `/etc/shadow` (three tries, then the machine restarts), and on an encrypted root for the LUKS passphrase before that. Its commands are `list`, `use N`, `check [--repair-btrfs]`, `verify [--repair]`, `password USER [--all]`, `restore-etc PATH`, `eject`, `open DEVICE [NAME]` (a LUKS volume by passphrase, never a token), `shell`, `reboot` and `poweroff`; each takes `--root`, `--rootflags` and `--esp` when the defaults don't fit, such as `--root /dev/mapper/NAME` after `open`. Run from a live USB with `--root-dir`, it asks for no login. See [Recovering a broken system](/docs/guides/recovery/).

### `mount-deployment`

Run from a live USB as a deployment's static archix, `ROOT/archix/deploy/<n>/usr/lib/archix/archix-static mount-deployment ROOT [N]`: binds deployment N's `/usr`, `/opt` and `/boot` read-only and its `/etc`, pacman database and DKMS state read-write over the mounted root, so `arch-chroot ROOT` can run archix. N defaults to current; a deployment missing a tree is refused, with the complete ones listed. `umount -R ROOT` undoes it. See [With a live USB](/docs/guides/recovery/#with-a-live-usb).

### `secure-boot`

Set up, check, revoke and rotate the PCR key. None of the subcommands evaluates the config. See [Secure Boot](/docs/guides/secure-boot/).

#### `secure-boot setup`

Create `sbctl` keys if needed and enroll them, with Microsoft's, while the firmware is in setup mode. Needs root.

```console
$ sudo archix secure-boot setup
enrolled your Secure Boot keys; the next `archix switch` signs its boot entries with them
then turn Secure Boot on in the firmware settings
```

If the firmware isn't in setup mode, it creates the keys but can't enroll them, and says `the keys exist, but the firmware isn't in setup mode, so they can't be enrolled`, then ``reboot into the firmware settings, reset or clear the Secure Boot keys (setup mode), then run `sudo archix secure-boot setup` again``. Keys are enrolled in the firmware's `db` with `sbctl enroll-keys --microsoft`; archix doesn't use shim or MOK.

| Flag | Description |
| --- | --- |
| `--no-microsoft` | Leave Microsoft's keys out; GPU option ROMs and Windows may then fail to start, and some boards won't boot at all. It warns: `warning: without Microsoft's keys, GPU option ROMs and Windows may fail to start, and some boards won't boot at all`. |

#### `secure-boot check`

Show whether Secure Boot is on and keys are enrolled, one `  ok  ` or `  no  ` line per requirement. It exits non-zero if any isn't met, and doesn't need root.

```console
$ archix secure-boot check
  ok  Secure Boot is on
  ok  the firmware isn't in setup mode (keys are enrolled)
```

| Flag | Description |
| --- | --- |
| `--anticheat` | Also check what Easy Anti-Cheat Linux v2 requires of the boot: a kernel signed with a well-known key, integrity lockdown, kexec disabled, the BPF LSM, the IOMMU without passthrough, a TPM 2.0 and a measured boot, and no out-of-tree or unsigned kernel modules. It ends with `Easy Anti-Cheat Linux v2 would refuse this boot: N requirements aren't met` when any is unmet. See [Anti-cheat requirements](/docs/guides/secure-boot/#anti-cheat-requirements). |

#### `secure-boot revoke`

Experimental, tested on OVMF only: add the boot images archix signed and no longer uses to the firmware's `dbx`, so it refuses to start them. Needs root, except with `--dry-run`.

```console
$ sudo archix secure-boot revoke
2 boot images archix signed and no longer uses:
  4a49f7139881a5bd  7.2.8-arch1-1-aedff12d49d2.efi
  9c03b1e27f5d6a40  7.2.8-arch1-1-protected-31b0c9e4d2aa.efi
dbx has 431 entries, 0 of them archix's; this adds 2 (124 bytes)
add 2 hashes to dbx? [y/N] y
revoked 2 boot images; the firmware refuses them from now on (dbx from before: /var/lib/archix/boot/dbx-before-1792512000)
```

archix records the Authenticode hash of every UKI it signs, and revokes the ones no file on the ESP is any more, never a hash any `.efi` file on the ESP has. The update is signed with sbctl's KEK and appended to `dbx` under archix's own owner GUID, `e85b82c6-e8da-40eb-b968-e918df35dec1`, so Microsoft's entries and any other stay as they are. It stops before writing when that would make more than 128 of archix's entries, when the firmware doesn't report its free variable space or would have less than 64 KiB left, when the KEK isn't a key file (sbctl can keep it in a TPM or a YubiKey), or when `efitools` isn't installed. It backs `dbx` up first, and checks the hashes are there after. With nothing to revoke it says `nothing to revoke: every boot image archix signed is still on the ESP or already in dbx`. [`boot.revoke_unused`](/docs/reference/options/#bootrevoke_unused) does this after every switch. See [Revoking old boot images](/docs/guides/secure-boot/#revoking-old-boot-images).

| Flag | Description |
| --- | --- |
| `--dry-run` | List what it would revoke and stop. Without `dbx` to read, it lists as if `dbx` were empty. |
| `--noconfirm` | Don't ask before writing `dbx`. |

#### `secure-boot rotate-pcr-key`

Experimental, tested with swtpm only: replace systemd's PCR policy signing key in every deployment, and bind the TPM2 slot of each LUKS2 volume you name to the new key. Needs root and `openssl`.

```console
$ sudo archix secure-boot rotate-pcr-key /dev/nvme0n1p2
wrote a new PCR policy signing key to /etc/systemd in all 2 deployments
bound /dev/nvme0n1p2's TPM2 slot to the new key
run `archix switch` before the next restart: until a UKI built with the new key boots, TPM2-bound volumes unlock only with a passphrase or recovery key, and UKIs built with the old key no longer unlock them
```

It makes a new 2048-bit RSA key, writes `/etc/systemd/tpm2-pcr-private-key.pem`, readable by root only from the start, and `tpm2-pcr-public-key.pem` into every deployment's `/etc`, and re-enrolls each volume with `systemd-cryptenroll --wipe-slot=tpm2 --tpm2-public-key-pcrs=11`, which asks for its passphrase or recovery key, or reads `$PASSWORD`. A volume that fails to bind is named, with the `systemd-cryptenroll` line to run for it; the key stays in place. The next `switch` builds a new deployment even if nothing else changed, `update     the PCR policy key: the new deployment's UKIs carry policies signed with the one in /etc/systemd`, and its UKIs carry policies signed with the new key. See [TPM-bound secrets](/docs/guides/secure-boot/#tpm-bound-secrets).

| Argument | Description |
| --- | --- |
| `[devices]...` | LUKS2 volumes to re-enroll, such as `/dev/nvme0n1p2`. Without any, it only replaces the key. |

## Commands that read the system or the config

These don't change the system and don't need root. As a normal user, archix keeps its download cache in `~/.cache/archix` (or `$XDG_CACHE_HOME/archix`); as root it uses `/var/cache/archix`.

### `diff`

Show what a switch would change to make this machine match the lock and the config, and what the machine carries that the config doesn't declare.

```console
$ archix diff
snapshot 2026-10-15: 214 packages
install    htop 3.4.1-1
upgrade    mesa 1:25.3.0-1 -> 1:25.3.1-1
downgrade  linux 7.2.9.arch1-1 -> 7.2.8.arch1-1
remove     libnl-legacy 1.1.4-1 (not in the lock)
mark       python-pip as a dependency
create     /etc/sysctl.d/80-swappiness.conf
update     /etc/issue (edited outside archix; your version gets saved)
enable     sshd.service
```

A package from one of the config's own recipes whose recipe changed at the same version shows as `reinstall  backup-scripts 1.2.0-1 (a build from a different recipe)`.

When everything matches, it prints `the system matches the lock and the config`. If installed packages are newer than the lock, it says so first: ``3 packages are newer than the lock; `archix switch` will offer to adopt them instead of downgrading``. If the lock is out of date with the config, `diff` resolves a fresh lock in memory, prints `note: archix.lock is out of date for desktop; showing a fresh resolution`, and shows the result without saving it. Packages in `ignored_packages` never appear. With `undeclared_packages = "warn"`, undeclared packages are listed on their own line instead of as removals:

```console
not declared, kept because undeclared_packages = "warn": htop ncdu
```

Files declared with a `url` are compared by their `sha256`, so `diff` downloads nothing. Flatpak changes are listed as in [`switch`](#switch). `diff` never adds Flatpak remotes, so an app that can't be locked yet, because flatpak or its remote isn't there, shows as `flatpak    install <id>: ...` with the reason, until the switch that adds it. System-wide Flatpak apps that aren't in the config, and weren't in the current generation, are listed on their own line. archix leaves them alone:

```console
flatpak apps archix doesn't manage: com.valvesoftware.Steam
```

#### Carried but not declared

Last, `diff` names what this machine carries that the config doesn't declare. The first switch copies the live `/usr`, `/opt`, `/boot` and `/etc` into deployment 1, and every later deployment starts as a copy of the one before, so a file nobody declared stays on this machine through every switch and rollback. A machine built from the same config won't have it. archix warns about these files and leaves them alone:

```console
carried on this machine but not in the config (a machine built from this config won't have them):
  no package's, in /usr and /opt:
    /opt/foo/ (1342 files)
    /usr/bin/kubectl
  no package's, in /etc:
    /etc/systemd/system/backup.service.d/ (1 file)
    /etc/udev/rules.d/99-keyboard.rules
  edited from what their package shipped, in /etc:
    /etc/ssh/sshd_config
  secrets, which no config carries:
    /etc/NetworkManager/system-connections/ (3 files)
    /etc/ssh/ssh_host_ed25519_key
    /var/lib/sbctl/keys/
```

- **In `/usr` and `/opt`**: files and symlinks no installed package owns, in the deployment the next switch starts from. A directory no package owns is one line with its file count. `/usr/local` isn't listed, nor files pacman hooks make again on any machine with the same packages, such as depmod's `modules.*`, DKMS's modules, icon and font caches and `__pycache__`.
- **In `/etc`**: files and symlinks no package owns, and package config files (pacman's backup files) edited since pacman installed them, that [`files`](/docs/reference/options/#files) doesn't declare. A directory no package owns is again one line. What [`init`](#init) never offers is left out here too: `.pacnew`, `.pacsave` and other leftovers, generated files such as `/etc/ld.so.cache`, the account files, systemctl's service links, archix's own files and the config. Edited config files of packages in `ignored_packages` aren't listed.
- **Secrets**: credentials `init` never copies into a config, found by path or content, such as SSH host keys, Wi-Fi and VPN connections and TLS keys, with a secret directory as one line, plus those outside `/etc`, DKMS's `mok.key` and sbctl's keys.

Paths the config declares, in `files`, `units`, `sysctl` and the other options that write files, never appear. Each kind shows its first 10 lines, then `and 12 more`; `--all` lists them all. Run as a normal user, `diff` can't read all of `/etc`: a file it can't read is listed with the secrets, and package config files it couldn't check for edits are counted, as in ``2 package config files only root can read weren't checked for edits; `sudo archix diff` checks them``.

| Flag | Description |
| --- | --- |
| `--all` | List everything carried but not declared, not only the first 10 lines of each kind. |

### `status`

Show the current generation, the booted one when they differ, changes made outside archix, the lock, pending changes and what the machine carries that the config doesn't declare.

```console
$ archix status
current generation: 42 (snapshot 2026-10-15)
booted generation: 41; generation 42 boots at the next restart
deployment 42 is committed and boots at the next restart; changes to /etc made before then are carried over to it
3 updates are waiting for a reboot (deployment 42): linux 7.2.8.arch1-1 -> 7.2.9.arch1-1, mesa 1:25.3.0-1 -> 1:25.3.1-1, htop 3.4.1-1; security-relevant, so reboot soon: linux
hibernation is off until you reboot into deployment 42: resuming would start its kernel on the running system's memory image; suspend still works
2 /etc changes carried into the pending deployment 42, 0 conflicts (.pacnew)
  last pass: 2026-10-15 09:12:40 UTC
Secure Boot: on
desktop is locked to snapshot 2026-10-15 (214 packages, 1 pinned off the snapshot)
the system matches the lock
```

The `booted generation` line and the notes under it appear while a committed deployment waits for the next boot: the updates waiting for it, with kernels, `glibc`, `openssl`, `openssh`, `systemd` and `sudo` named as security-relevant, the [hibernation guard](/docs/concepts/atomicity/#a-switch), and what the [`/etc` merge](/docs/concepts/safety/#changes-to-etc-before-the-reboot) has carried into it, with a line for each conflict and each change it couldn't carry. Once the deployment has booted, the same lines say what was carried from the one before, and whether its last pass at shutdown was cut off.

Other lines appear when they apply: a verify that found changed files (see [`verify`](#verify)), hooks or `systemd-sysusers` failing at boot (`deployment 46's hooks failed at 2 of 3 boots; the next boot runs them again`), a [health check](/docs/reference/options/#boothealth_check) or boot count that fell back to the previous deployment, and ``warning: systemd-boot's LoaderEntryDefault variable (the boot menu's default, or `bootctl set-default`) names archix-40.conf and overrides archix's default; the next switch clears it, or `sudo bootctl set-default ''` now``. The lock line counts the [pins](/docs/concepts/lockfile/#pins), packages held at a version other than the snapshot's, when there are any. When the snapshot is older than [`lock_warn_after_days`](/docs/reference/options/#lock_warn_after_days), 30 by default, the next line warns: ``warning: archix.lock is at the 2026-09-10 snapshot, 38 days old; updates released since aren't installed. `archix update` moves it to the newest snapshot``. `switch` prints the same warning before it starts.

On a machine that booted through UEFI, the `Secure Boot:` line says `on`, `off`, or ``setup mode (no keys enrolled; `sudo archix secure-boot setup` enrolls them)``. The last line compares the lock with the current deployment: `the system matches the lock`, or `2 changes pending; see `archix diff``. If the config changed since the lock was written, it adds ``the config changed since the lock was written; `archix lock` or `archix switch` will re-resolve``. That includes editing a recipe in `pkgbuilds/` or a file listed in `patches`, not just the Lua.

Packages archix builds from a VCS checkout, AUR packages and recipes named `-git`, `-hg`, `-svn`, `-bzr`, `-fossil` or `-darcs`, get their own line while the lock has no pinned sources for them, as before their first build: ``1 package builds from a VCS source archix.lock has no commit for yet, so its next build takes upstream's newest code, whose git commit is then recorded: neovim-git``. `archix lock` and `update` mark them the same way, `+ neovim-git 0.12.0.r310.g1a2b3c4-1 (builds from a VCS source; its first build records the commit)`. Packages in `ignored_packages` that aren't installed are named too, as `switch` names them: `ignored_packages lists 1 package that isn't installed here, and archix never installs ignored ones: nvidia-dkms-custom`. That's what a second machine sees when it shares a config `init` wrote on another.

`status` ends with the same [carried but not declared](#carried-but-not-declared) section as `diff`, and takes the same `--all` flag.

### `generations`

List generations. The current one, which the next boot starts, is marked with `*`; those with a kept deployment say `deployment`, the running one `booted`, and those with only their record `record`.

```console
$ archix generations
    40  2026-10-02 18:40 UTC  snapshot 2026-10-02  210 packages  config 8c1f0e2a7b  record
    41  2026-10-09 09:12 UTC  snapshot 2026-10-09  212 packages  config 3f2a9c1e07  deployment, booted
*   42  2026-10-15 08:55 UTC  snapshot 2026-10-15  214 packages  config 5d7b21c4e9+uncommitted  deployment
```

`+uncommitted` marks a generation switched from a config with uncommitted changes. `generations` doesn't evaluate the config.

### `search`, `-Ss`

Search the locked snapshot's repos, including your third-party repos, and the AUR. Every term has to match the name or the description.

```console
$ archix search zen browser
aur/zen-browser-bin 1.16.3b-1 (+412 18.53)
    Zen Browser, a Firefox-based browser focused on privacy
$ archix -Ss wezterm
extra/wezterm 20240203.110809.5046fc22-3 [installed]
    GPU-accelerated cross-platform terminal emulator and multiplexer written in Rust
```

Results are ranked as for [`archix <terms>`](#search-and-install-archix-terms), best first: an exact name, then names starting with the first term, then names containing a term, then description matches, with repo packages before AUR ones within each group. AUR results show their votes and popularity. `[installed]` marks installed packages and `(out of date)` flagged AUR packages. To pick from the results and install, leave out `search`: `archix zen browser`.

### `lock`

Resolve the config into `archix.lock`, keeping the host's snapshot date and the locked AUR commits.

```console
$ archix lock
resolving 87 declared packages against the 2026-10-09 snapshot
+ htop 3.4.1-1
213 packages locked
wrote /etc/archix/archix.lock
```

If the lock already matches, it prints `archix.lock is up to date for desktop (snapshot 2026-10-09)` and changes nothing, unless it has packages built for a newer x86-64 level than this CPU supports, as when another machine locked the host: then `lock` resolves it again for this machine, config change or not. Only `lock` does that. `switch` refuses such a lock, ``archix.lock for laptop has packages built for a newer CPU level than this machine runs, so it was locked elsewhere: zlib (x86_64_v4), bash (x86_64_v4). Run `archix lock` on this machine; nothing was changed``, and leaves `archix.lock` alone; see [CPU levels](/docs/guides/third-party-repos/#cpu-levels). New AUR build recipes are shown for review before they're locked.

Flatpak apps keep their locked commits. A newly declared app is locked to its remote's current commit, `flatpak org.mozilla.firefox at 0bb3a9e40c1f`. `lock` never adds Flatpak remotes, so if flatpak isn't installed yet, or the app's remote hasn't been added, the app is locked by the next switch instead, and `lock` says so: ``note: org.mozilla.firefox: flatpak isn't installed yet; the next `archix switch` locks it``.

| Flag | Description |
| --- | --- |
| `--noconfirm` | Trust new AUR build recipes without showing them. |

### `update`

Move the host's lock to the newest archive snapshot, or to a date you choose, move AUR packages to their newest commits and Flatpak apps to their remote's current commits, and re-resolve. It applies nothing.

```console
$ archix update
resolving 87 declared packages against the 2026-10-15 snapshot
snapshot 2026-10-09 -> 2026-10-15
+ libdisplay-info 0.3.0-1
~ linux 7.2.8.arch1-1 -> 7.2.9.arch1-1
~ mesa 1:25.3.0-1 -> 1:25.3.1-1
- libnl-legacy 1.1.4-1
213 packages locked
wrote /etc/archix/archix.lock
```

`+` is a new package, `~` a changed version and `-` a package that's no longer needed. Changed AUR build recipes are shown as a diff for review in between. Flatpak apps move to their remote's current commit, with a line for each one that changed, such as `flatpak com.spotify.Client at 81c2d4e6f0a3`. An app whose remote can't be reached keeps its locked commit: `warning: couldn't check com.spotify.Client on flathub; keeping its locked commit`.

| Flag | Description |
| --- | --- |
| `--to YYYY-MM-DD` | Snapshot date to move to, instead of the newest. Works in both directions. |
| `--noconfirm` | Trust new AUR build recipes without showing them. |

Follow it with `sudo archix switch` to apply. See [The lockfile](/docs/concepts/lockfile/).

`lock` and `update` write to the config directory, so you need write access to it: `sudo` for a root-owned `/etc/archix`, nothing for a config in your own repo.

### `eval`

Print the evaluated config as JSON. Useful to check what your modules merged into, and to script against.

```console
$ archix eval | jq .repos
[
  "core",
  "extra",
  "multilib"
]
```

### `options`

List every declared option: its path, type, default, description and the file that declared it.

```console
$ archix options
ignored_packages
    type: list of str
    default: []
    Installed packages archix leaves alone and never reports as undeclared.
    declared in <archix>/packages.lua
packages
    type: list of str
    default: []
    Packages to install: from the configured repositories, else from the AUR. Their dependencies are installed too and recorded in archix.lock.
    declared in <archix>/packages.lua
...
```

| Flag | Description |
| --- | --- |
| `--json` | Print a JSON array of `{ path, type, default, description, declared_in }` objects instead. |

### `prefetch`

Download a file and print the [`archix.fetchurl`](/docs/reference/lua-api/#archixfetchurl) line to paste into the config, with the file's sha256 filled in.

```console
$ archix prefetch https://downloads.example.org/hello/1.2.0/hello-x86_64
archix.fetchurl { url = "https://downloads.example.org/hello/1.2.0/hello-x86_64", sha256 = "5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03" }
```

Paste it as the value of a [`files`](/docs/reference/options/#files) or [`home.files`](/docs/reference/options/#homefiles) entry, or take the `url` and `sha256` for an [`appimages`](/docs/reference/options/#appimages) entry. `prefetch` doesn't evaluate the config. A URL that isn't there is an error: `https://... doesn't exist`.

Run as root, it also keeps the download in archix's store, so the next switch doesn't fetch it again. As a normal user, it deletes the file once it has printed the hash.

### `cache push`

Copy this machine's package files, their signatures and its AUR build records into a directory that other machines list in [`package_caches`](/docs/reference/options/#package_caches).

```console
$ sudo archix cache push /mnt/nas/archix
copied 214 files to /mnt/nas/archix
```

It copies everything in `/var/cache/archix/pkg`, the files archix downloaded and the packages it built, plus the build records under `aur-builds/`, and skips files the directory already has at the same size. Builds of private recipes, the ones `init --repack` writes, are never copied, since their files may not be yours to share. Each file is written under a temporary name and renamed, so a machine reading the cache never sees half of one. Serve the directory over HTTP to share it beyond a network mount. `cache push` doesn't evaluate the config. See [Package caches](/docs/concepts/lockfile/#package-caches).

## Internal commands

Hidden commands are run by files archix installs, not by you: `archix hook-record`, from the pacman hook; `archix boot-check` and `archix boot-complete`, from `archix-boot.service` and `archix-boot-complete.service`; `archix deploy-mount`, from the initramfs hook, which mounts the deployment the boot belongs to; `archix deploy-activate`, from `archix-deploy-activate.service`, which applies a deployment's `/var` changes at its first boot and runs its hooks; `archix etc-merge`, from `archix-etc-merge.service` and, with `--boot`, `archix-etc-merge-boot.service`; `archix health-check`, from `archix-health-check.service`; `archix deploy-flatpak`, from `archix-flatpak.service`; `archix bless`, from a drop-in for `systemd-bless-boot.service`, which renames a counted entry to its good name and flushes it under that name, as a FAT rename isn't on disk until then; `archix pacman-guard`, from the pacman hook that stops plain pacman on a deployment; and `archix eject-here`, which `eject` runs in a private mount namespace.

## Exit behaviour

- `0` when the command succeeded.
- `1` on errors, with each error printed to stderr as `error: ...`. That includes config errors (every one is reported, not just the first), resolution failures, a declined prompt, and failures before a deployment is committed, which leave the running system and its boot entries untouched.
- Non-zero after a switch or rollback with soft failures: nothing is undone, the steps that failed (units that can't be enabled, files outside the deployment, Flatpak apps, `gc`) are listed, and the last line says what did happen, such as `error: generation 43 is committed and boots at the next restart, but 2 steps failed (listed above)`. A switch that had nothing to do but whose hooks failed ends with `error: nothing was changed, but 1 step failed (listed above)`, and a rollback with `error: generation 41 (snapshot 2026-10-09) is current and boots at the next restart, but 1 step failed (listed above)`. See [Safety](/docs/concepts/safety/#hard-and-soft-failures).

Piping output works as expected: `archix diff | head` exits quietly when `head` closes the pipe.
