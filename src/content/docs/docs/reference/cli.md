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
apply 2 changes? [Y/n]
```

Matching is the same as [`search`](#search--ss): every term has to appear in the name or the description. Results are ranked: an exact name first, then names starting with the first term, then names containing a term, then matches in the description only. Within each group, repo packages come before AUR ones, then they're sorted by name.

The prompt takes numbers (`1 2 3`), ranges (`1-3`) and exclusions (`^4`, `^2-3`), separated by spaces or commas. Exclusions on their own mean all but those, and an empty answer installs nothing. Anything else stops with `"x" isn't a number, a range like 1-3 or an exclusion like ^4`, or `6 is outside 1-5`. With no results, it prints `no packages match btpo` and exits.

The picks go through [`install`](#install--s): they're declared in `lua/hosts/<host>/installed.lua`, then archix switches, with the usual preview and `apply N changes?` prompt, and puts `installed.lua` back if that fails. Run as a normal user, archix calls sudo itself for that part, like yay does, running `sudo` on its own binary with `--config <dir> --host <host> install`, `--noconfirm` if you gave it, and `-- <picks>`.

Commands always win over terms: `archix status` runs `status`, and a typo such as `archix swtich` searches for "swtich". `--noconfirm` can go anywhere among the terms; it's taken out of the search and skips the switch's question. Bare `archix`, with no terms, is still the full upgrade.

`ax` is a short alias for `archix`. The AUR packages install it as a symlink, `/usr/bin/ax`, and it takes every command and flag below: `sudo ax switch`, `ax diff`, `sudo ax -Syu`. Help and usage errors use the name you typed, such as `Usage: ax [OPTIONS] [COMMAND]`, while archix's own messages, `--version` included, still say `archix`. See [Installation](/docs/installation/#ax-the-short-name).

## Root

Commands that change the system need root: `install`, `remove`, `switch`, `upgrade` (and `-Syu` and bare `archix`), `rollback`, `revert`, `user unlock`, `gc`, `snapshot create`, `snapshot delete`, `snapshot setup`, `resync`, `init`, `secrets init`, `secrets rotate-machine-key`, `secrets edit --machine`, `secrets rekey --machine`, `secure-boot setup`, `secure-boot revoke` and `secure-boot rotate-pcr-key`. Run as your user, they stop before touching anything, `installed.lua` included, and give you the line to run, the way you typed it:

```console
$ archix -S htop
error: `archix install` changes the system, so it needs root; run it with sudo: sudo archix -S htop
```

`rescue` and `chroot` mount filesystems, so they need root too: `` `archix rescue` mounts filesystems, so it needs root ``.

Only one command that changes the system runs at a time; see [Commands that change the system](#commands-that-change-the-system). Any other error that comes from a missing permission gets the same hint, such as `archix lock` on a root-owned `/etc/archix`: `hint: that needs root; run it with sudo: sudo archix lock`. Reading doesn't need root: `archix <terms>`, `search`, `diff`, `status`, `eval`, `options`, `generations`, `snapshot list` and `secure-boot check` work as your user, and so do `lock` and `update` on a config you own. `archix <terms>` asks sudo for the install itself.

Every command except `init`, `options`, `rollback`, `revert`, `user`, `gc`, `generations`, `snapshot`, `resync`, `chroot`, `rescue`, `prefetch`, `cache push`, `secure-boot`, `secrets init`, `secrets pubkey` and `secrets rotate-machine-key` reads the config. `options` works without one, listing every option against an empty config. With no `init.lua` in the config directory, the others stop with:

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

`init` opens a full-screen wizard with nine steps: Welcome, Config, Packages, System, Config files, Boot, Generations, Dotfiles and Review. It asks where the config should live, whether to make it a git repository, what to do with orphaned packages and with packages you later install with plain pacman or yay (the [`drift`](/docs/reference/options/#drift) policy), which settings, users, groups and services to take over (four checklists, all ticked), which config files to manage (a checklist, each file tagged `recommended`, `changed from its package's version` or `not from any package`), whether archix manages booting with systemd-boot, how many generations to keep, whether to take [rollback snapshots](/docs/concepts/snapshots/), and which dotfiles to bring in (recommended ones start ticked). Every answer starts at the recommended one.

Some questions appear only where they apply. On btrfs, Rollbacks is on by default: `on: a btrfs snapshot before every switch and pacman run, to roll back to or boot (recommended)`; on ext4 the Review step says rollbacks aren't available. A root that is btrfs's top level gets `move it into an @ subvolume at the boot after the first switch, so rollbacks work` (see [A root at the top level](/docs/concepts/snapshots/#a-root-at-the-top-level)). With snap-pac installed, init offers to keep both or to mask snap-pac's pacman hooks, never on Limine. On a Limine machine, the Boot step offers `move to systemd-boot for automatic fallback: Limine stays installed and first until a boot through systemd-boot counts as good` (see [Moving from Limine to systemd-boot](/docs/concepts/boot-and-fallback/#moving-from-limine-to-systemd-boot)). Review also runs the boot health check once and shows what it would conclude on this machine now.

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

System-wide Flatpak apps are declared too, each locked to the commit it's installed at; `--user` apps are left out, and so are apps whose remote is gone. `init` changes nothing on the system except the config directory. The first `sudo archix switch` applies the config in place; with rollbacks on, it also lays the root out for them, `set up     rollbacks: the @archix and @varlib subvolumes and their fstab lines; snapshots start after the next reboot`, and snapshots start after that reboot (see [The layout](/docs/concepts/snapshots/#the-layout)). Imported dotfiles are the exception: the first switch replaces them with links, and the Result section lists them under `! the first switch would still change these:`. Secret files are never shown or copied, only counted. See the [quick start](/docs/quick-start/) for every step.

| Flag | Description |
| --- | --- |
| `--defaults` | Take every recommended answer without opening the wizard: everything in System, the recommended config files, no dotfiles (it writes no `home` section and touches nothing in your home), and a git repository (`git init`, `git add -A`) when git is installed. It needs no terminal, and prints what it captured in Packages, System, Config files, Boot and Generations sections before Lock and Result. The Boot section says what booting will look like, and what has to change first when archix's boot entries can't work yet, such as ``install systemd-boot (`bootctl install`)`` or `add the root= kernel parameter to kernel_params`. Config files lists the recommended files it chose, then `· N more files left unmanaged; the wizard (archix init without --defaults) lets you pick them`. Its output includes `! their remote is gone, so archix leaves these Flatpak apps alone: ...` when that applies. Without it, `init` needs a terminal: `` `archix init` asks questions; run it in a terminal, or pass --defaults to take every default ``. |
| `--repack PACKAGE` | Rebuild this package, which no repo or the AUR has, as a recipe in `pkgbuilds/` from its installed files, as ticking it in the wizard does. Its files stay in `/var/cache/archix/sources`, out of the config, and its builds out of package caches; a config file you edited goes in as the package shipped it, from a cached package file, or not at all. Can be given more than once. Without it, `--defaults` rebuilds nothing and leaves such packages in `ignored_packages`. |

`init` refuses to run if the config directory already has an `init.lua`.

## Commands that change the system

Only one of these runs at a time: each takes an exclusive lock on `/var/lib/archix/running`, and a second one stops with `` another archix is already running (pid 4242, `archix switch`); wait for it to finish ``. See [Safety](/docs/concepts/safety/#one-archix-at-a-time).

These need root. Run without it, they stop before touching anything, with the command's name and your exact command line: `` `archix switch` changes the system, so it needs root; run it with sudo: sudo archix switch --noconfirm ``. See [Root](#root).

### `switch`

Apply the lock and the config to the running system, in place: packages, files, users, groups and services change now, with no reboot. With [rollbacks](/docs/concepts/snapshots/) on, a snapshot of the root is taken first, so the system as it was before can be booted or kept.

```console
$ sudo archix switch
resolving 87 declared packages against the 2026-10-09 snapshot
+ htop 3.4.1-1
213 packages locked
wrote /etc/archix/archix.lock
install    htop 3.4.1-1
:: the plan since the last generation archix applied
package + htop 3.4.1-1
:: the config since the last generation archix applied
...
apply 2 changes? [Y/n]
downloading htop-3.4.1-1-x86_64.pkg.tar.zst
snapshot 12 holds the system as it was before this change
:: applying generation 42
...
:: building the boot images
switched to generation 42 (snapshot 2026-10-09)
```

`switch` re-locks at the locked date if the config changed, lists the package, file, user, group, service and Flatpak changes, and asks before applying. A package built from a recipe in the config that changed while its version didn't is listed as `reinstall  backup-scripts 1.2.0-1 (a build from a different recipe)`, and rebuilt and reinstalled at the same version.

Before the question it shows two diffs, both against the last generation archix applied, so a change you didn't make yourself shows up before it reaches root. `:: the plan since the last generation archix applied` lists what the evaluated config changes, one line each, such as `package + htop 3.4.1-1`, `package ~ mesa 1:25.3.0-1 -> 1:25.3.1-1`, `file + /etc/motd (0644 root:root)`, `unit + sshd.service enabled` or `user + alice (uid 1001, groups wheel, shell /usr/bin/zsh)`, with no secret or password hash in it, up to 200 lines. `:: the config since the last generation archix applied` is the diff of the config tree itself, from root's own copy; the first switch says `:: no generation applied by archix yet; this one records config tree 5d7b21c4e9f0` instead. `--noconfirm` skips the question, not the diffs. root evaluates a captured copy of the config, never your working tree, and applies exactly the plan it showed; see [Safety](/docs/concepts/safety/).

After you confirm, it downloads files declared with a `url` into the store (each checked against its `sha256`), builds AUR packages and the config's own [recipes and patched packages](/docs/guides/local-pkgbuilds-and-patches/) in a clean chroot, and downloads every package to `/var/cache/archix/pkg` (each file checked against the lock's `sha256`). Everything that can refuse does so before anything changes: packages that ship the same file or a file that's already there with no owner, packages pacman's signature check rejects, a sudoers rule `visudo` refuses, tmpfiles lines `systemd-tmpfiles` refuses, a managed file that would replace something it can't back up, an ESP with no room for a new boot image, and, with Secure Boot enforcing, boot images that wouldn't be signed; see [Signatures and disk space](/docs/concepts/safety/#signatures-and-disk-space). A refused switch changes nothing and takes no snapshot.

Then it applies, in this order: the [snapshot](/docs/concepts/snapshots/#when-snapshots-are-taken), `snapshot 12 holds the system as it was before this change`; with rollbacks on, the current entry's tries, the fallback slot's entry and the [unfinished mark](/docs/concepts/boot-and-fallback/#boot-counting-and-the-health-check), so a switch cut off from here on falls back by itself; the package transaction, run by pacman on the root itself; users, groups and passwords; home directories; the managed files and file capabilities; enabling and disabling services; `/etc/fstab`; secrets, decrypted into `/run/archix/secrets`; then, on the running system, `systemctl daemon-reload`, stopping units that were disabled, starting units that were enabled, and restarting units whose files the transaction replaced, as [`restart_if_changed`](/docs/reference/options/#restart_if_changed) allows. Then the `on_change` scripts and [hooks](/docs/guides/hooks/) run, the generation is recorded and made current, the boot images are built and the boot menu written, the root is flushed and the mark dropped, Flatpak apps are synced, and `gc` runs. A kernel the transaction replaced keeps running until you reboot: `the running kernel was replaced: reboot to run the new one (the running one's modules stay loadable until then)`.

A switch that fails partway can't be undone by itself: the error says what was applied and that the generation wasn't recorded, and with rollbacks on, which snapshot holds the state before, ``generation 42 wasn't recorded, and generation 41 stays current; snapshot 12 holds the system as it was before (`archix snapshot list`)``. With rollbacks on it adds `; the system is marked as part way, so its next counted boot doesn't count as good and falls back to the slot, unless a switch finishes first`; a switch that failed before changing anything drops the mark instead. Run `archix switch` again once the cause is fixed, or [keep](#rollback) that snapshot.

The first switch with [`rollback.enable`](/docs/reference/options/#rollbackenable) on a root that isn't laid out for rollbacks yet lays it out instead of taking a snapshot: `set up     rollbacks: the @archix and @varlib subvolumes and their fstab lines; snapshots start after the next reboot`. On a root that is btrfs's top level, it refuses until the root has moved into `@`, ``rollback.enable is set, but / is btrfs's top level, not a subvolume, so a snapshot of it can't become the root; `archix init` or `archix snapshot setup --move-root` moves it into an `@` subvolume at the next boot. Set rollback.enable = false to switch without rollbacks until then; nothing was changed``. On ext4 it warns and switches without snapshots. See [The layout](/docs/concepts/snapshots/#the-layout).

With [`hooks`](/docs/guides/hooks/) declared, the preview has a `run hooks` line listing them in the order they'll run. Hooks aren't counted in `apply N changes?`.

With [Flatpak apps](/docs/guides/flatpak-and-appimages/) declared, `switch` locks new apps to their remote's current commit, then, once the system is applied, adds the remotes they use, locks apps whose remote it just added, installs missing apps, pins every app to its locked commit with `flatpak update --commit=`, and uninstalls the apps that left the config:

```console
flatpak    install org.mozilla.firefox (0bb3a9e40c1f)
flatpak    com.spotify.Client 5f8806197a2b -> 81c2d4e6f0a3
flatpak    remove org.gimp.GIMP
```

A Flatpak step that fails is a soft failure, like a unit that can't be enabled.

If the system already matches, it prints `nothing to do: the system matches generation 42 (snapshot 2026-10-09)`, takes no snapshot, and runs the hooks, like NixOS runs activation scripts on every switch.

`switch` refuses to start while pacman, pamac, PackageKit or anything else holds pacman's lock open, or while a stale pacman lock (`/var/lib/pacman/db.lck`) exists. It refuses in [try mode](/docs/concepts/boot-and-fallback/#try-mode), `` this boot is snapshot 9 in try mode, read-only with a throwaway layer, so a switch would be lost at the next boot; `archix rollback 9` to keep it, or reboot into the current system; nothing was changed ``, while a [rollback](#rollback) waits for its reboot, `` rollback to snapshot 9 is pending; reboot, or `archix rollback --cancel`; nothing was changed ``, and on a root another tool put in place, until [`resync`](#resync). It also refuses a lock with packages built for a newer x86-64 level than this CPU runs, which another machine locked, and asks for `archix lock` here; see [`lock`](#lock). If installed packages are newer than the lock, as after a `pacman -Syu` between `archix init` and the first switch, it offers to adopt them first:

```console
the system is newer than the lock for 3 packages (linux, mesa, vulkan-radeon), probably from pacman or an AUR helper
adopt them into the lock? (no reverts them to the lock) [Y/n]
```

Explicit packages installed with pacman or an AUR helper since the last switch are handled by [`drift`](/docs/reference/options/#drift). With the default `drift = "ask"`, yes adds them to `installed.lua` and switches with them; no keeps them installed and untracked, so no switch removes them or what they need:

```console
installed with pacman or an AUR helper since the last switch, not declared: htop ncdu
add them to installed.lua? (no keeps them installed, untracked) [Y/n]
```

See [Drift](/docs/concepts/pacman-and-yay/#drift).

Downgrades that may break data, such as PostgreSQL going back a major version, are listed after the preview and asked about separately, defaulting to no:

```console
unsafe downgrades: the older versions may refuse or damage data the newer ones wrote
  postgresql 17.2-1 -> 16.6-1 goes back from major version 17 to 16, and postgresql migrates its data between them
downgrade anyway? [y/N]
```

Pressing Enter stops with `aborted; nothing was changed`. See [Downgrades that may break data](/docs/concepts/generations/#downgrades-that-may-break-data) and the [`unsafe_downgrades`](/docs/reference/options/#unsafe_downgrades) option.

A downgrade of declared packages that would break packages archix doesn't track, by a declared dependency or a library their binaries load, is refused before the question:

```console
this switch downgrades declared packages, which would break packages archix doesn't track:
  ...
declare them, remove them, or pass --allow-partial to switch anyway; nothing was changed
```

See [Partial downgrades](/docs/concepts/generations/#partial-downgrades).

| Flag | Description |
| --- | --- |
| `--noconfirm` | Don't ask. Applies without the `apply N changes?` question, adopts newer packages, adds drifted packages when `drift = "ask"`, trusts new AUR build recipes without showing them, and takes the nearest newer version when a locked AUR version can't be built. It refuses unsafe downgrades: ``refusing unsafe downgrades with --noconfirm; to go ahead, run `archix switch` or `archix revert` with --allow-unsafe-downgrade``. With the default `third_party_fallback = "ask"`, a locked third-party package that's gone fails the switch rather than being swapped for another; see [The fallback chain](/docs/guides/third-party-repos/#the-fallback-chain). |
| `--allow-unsafe-downgrade` | Go back across versions that may not read newer data (postgresql majors and the like) without asking. They're still listed. |
| `--accept-nearest newer\|older` | When a locked AUR version can't be built, take the nearest newer or older one without asking. Overrides the `missing_versions` option. |
| `--without-secrets` | Commit even though secrets don't decrypt on this machine yet: those are left out, and what uses them fails until a switch after `archix secrets rekey`. |
| `--allow-partial` | Downgrade declared packages even though that breaks packages archix doesn't track, by their dependencies or the libraries they load. |

### `upgrade`, `-Syu`

The yay-style full upgrade. Bare `archix` and `archix -Syu` do the same.

```console
$ sudo archix -Syu
$ sudo archix
```

It moves the lock to the newest Arch Linux Archive snapshot, as `archix update` does, moves AUR packages to the newest commit of their build recipes, showing each changed recipe for review, and then switches, in place.

`archix -Syu <packages>` also declares the packages, like `archix install` with the upgrade.

| Flag | Description |
| --- | --- |
| `--noconfirm` | Skip archix's questions, as in `switch`. |

`-Sy` on its own is refused: ``archix doesn't sync databases on their own; use -Syu to upgrade, or `archix update` to move the lock``. To move the lock without applying anything, use [`update`](#update), then `switch`.

### `install`, `-S`

Declare packages and install them, with a switch.

```console
$ sudo archix install zen-browser-bin
$ sudo archix -S neovim ripgrep
```

The names go into `lua/hosts/<host>/installed.lua`, a plain Lua file archix imports automatically and you can edit too, and then archix switches. A name a recipe in `pkgbuilds/` builds under exactly that name comes from there; other names are looked up in your configured repos, then in `pkgbuilds/` by what recipes `provides`, then in the AUR, and new AUR build recipes are shown for review. A name found nowhere stops it: `` `foo` is in none of the configured repos, pkgbuilds/ or the AUR ``. A name the config already declares somewhere else isn't added again: `tree is already declared in the config; not adding it to lua/hosts/box/installed.lua`, and the switch installs it from there. `installed.lua` and `archix.lock` are written only when the switch succeeds, so declining, a failed switch or a config that stops evaluating leaves both as they were; see [Declined and killed switches](/docs/concepts/safety/#declined-and-killed-switches).

To install something without declaring it, use plain pacman or an AUR helper: [`drift`](/docs/reference/options/#drift) decides what the next switch does with it.

| Flag | Description |
| --- | --- |
| `--path <dir>` | Edit the `installed.lua` in this directory instead of `lua/hosts/<host>/`. It must be inside the config's `lua/` directory, or archix refuses. archix prints the module name to import, such as `modules.dev.installed`, since only the host's own `installed.lua` is imported automatically. |
| `--noconfirm` | Don't ask. |

### `remove`, `-R`

Take packages out of `installed.lua` and uninstall them, with a switch.

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

Keep snapshot N: it becomes the root, `@`, from the next boot, with accounts made since carried over, and the running system is kept as a snapshot of kind `replaced`, with a boot entry of its own. `--cancel` undoes it before that boot. Needs [rollbacks](/docs/concepts/snapshots/) on btrfs. See [Keeping a snapshot](/docs/concepts/snapshots/#keeping-a-snapshot).

```console
$ sudo archix rollback 12
keeping snapshot 12: 12  2026-10-15 08:55 UTC  switch    system as of tree 3f2a9c1e07ab (generation 41), before switch to tree 5d7b21c4e9f0
from the next boot it is the root; the running system is kept as a snapshot of kind replaced. Package changes and /etc edits made before that reboot are lost
/var/lib stays: package-owned files there go back to the snapshot's versions where unchanged, and files of packages installed since move to /var/lib/.archix-moved/12
keep snapshot 12? [Y/n]
snapshot 12 is now @, from the next boot; the running system is snapshot 14
archix: rollback to snapshot 12 is pending: it becomes the root at the next boot. Package changes and /etc edits made before that reboot are lost; `sudo archix rollback --cancel` keeps the running system instead
```

Snapshot numbers come from [`snapshot list`](#snapshot) or [`generations`](#generations). The keep is applied from root's own verified copy of the generation the snapshot holds, so a changed copy refuses it before anything changes. archix clones the snapshot, carries accounts into the clone, and swaps it with `@` in one rename, so the root keeps the name `@`. Until the reboot, `switch`, `revert` and pacman refuse: ``archix: rollback to snapshot 12 is pending; reboot, or `archix rollback --cancel` ``. The login message and `archix status` show the pending notice. After the reboot, the kept snapshot's generation is current, and its config tree is the one `archix status` names.

- **Accounts.** Declared users follow the snapshot's config. Users, groups and passwords created or changed since are carried over by name from the `@` being replaced. An account deleted since comes back locked, with no password, expired and with a nologin shell, so its UID stays reserved: ``alice was deleted since snapshot 12, so it comes back locked (no password, expired, nologin shell); `archix user unlock alice` enables a declared one``. See [Accounts after a keep](/docs/concepts/snapshots/#accounts-after-a-keep).
- **`/var/lib`**, on its own subvolume, isn't rolled back. Package-owned files there go back to the snapshot's versions where you haven't changed them, and files of packages installed since are moved aside to `/var/lib/.archix-moved/<N>/`. See [`/var/lib` after a keep](/docs/concepts/snapshots/#varlib-after-a-keep).
- **From a [try boot](/docs/concepts/boot-and-fallback/#try-mode)**, the boot of an older snapshot from the menu, `rollback N` keeps the snapshot you're trying; accounts are carried from the `@` it replaces, not from the throwaway `/etc`: `snapshot 12 is now @, from the next boot; the @ it replaces is snapshot 14`.
- **On Limine**, which boots the kernel and initramfs on the ESP, it refuses a snapshot whose kernel or initramfs differs from the current one: `Limine boots the kernel and initramfs on the ESP, and the snapshot's kernel differs from the current one's, so it would boot with the wrong one: install the snapshot's kernel first, or switch to systemd-boot; nothing was changed`.

It refuses while another keep is pending, `` rollback to snapshot 12 is pending already; reboot, or `archix rollback --cancel` first; nothing was changed ``, for a snapshot that isn't there, `` there's no snapshot 30; `archix snapshot list` shows the ones there are ``, and on a root another tool put in place, until [`resync`](#resync). `rollback` doesn't evaluate the config, so it works even when the config is broken. It changes neither your config nor `archix.lock`.

`rollback` keeps a whole snapshot. To apply an earlier generation's declared state live instead, use [`revert`](#revert); [Revert and keep](/docs/concepts/generations/#revert-and-keep) compares them.

```console
$ sudo archix rollback --cancel
the rollback to snapshot 12 is cancelled: the running system stays the root, and snapshot 14 is gone again
```

`--cancel` swaps the running root back, drops the snapshot the keep filed it as, and puts `/var/lib` and the boot menu back as they were. With nothing pending it says `no rollback is pending; nothing was changed`. A keep or `--cancel` that a power cut stopped halfway is finished or dropped at the next boot.

| Argument or flag | Description |
| --- | --- |
| `[SNAPSHOT]` | The snapshot number. `archix snapshot list` lists them. Required unless `--cancel` is given. |
| `--cancel` | Undo the pending rollback: the running system stays the root. |
| `--noconfirm` | Don't ask. |

### `revert`

Apply an earlier generation's record live, in place, the previous one by default: its packages, from archix's cache and the archive, its files, users and services. What it doesn't declare stays as it is, and the config and `archix.lock` stay as they are.

```console
$ sudo archix revert
reverting to generation 41 (snapshot 2026-10-09), applying its record live, in place
downgrade  mesa 1:25.3.1-1 -> 1:25.3.0-1
downgrade  vulkan-radeon 1:25.3.1-1 -> 1:25.3.0-1
revert to generation 41 (snapshot 2026-10-09)? [Y/n]
snapshot 13 holds the system as it was before this change
:: applying generation 41 (snapshot 2026-10-09)
...
reverted to generation 41 (snapshot 2026-10-09). archix.lock is unchanged, so the next `archix switch` returns to the config
```

`revert` applies the generation's stored plan, never re-evaluating its old Lua, after reading it back from root's own copy and checking it: `generation 41 can't be read back from root's repository; nothing was changed`. Packages go back by the sha256 the generation pinned, from archix's [package cache](/docs/concepts/generations/#limits-and-gc) or the Arch Linux Archive, without checking their signatures against today's keyring again, so a key revoked since doesn't block it; a package that can't be fetched refuses before anything changes. Like `switch`, it checks first that no two packages ship the same file and that the disk has room, takes a snapshot with rollbacks on, and refuses in try mode, while a rollback is pending, and on a root another tool put in place. Downgrades that may break data are listed and asked about first, `downgrade anyway? [y/N]`, as on a switch. `revert` doesn't evaluate the config, so it works even when the config is broken. `archix revert 42` moves forward again. Naming the current generation prints `generation 42 is current already`. See [Generations and rollback](/docs/concepts/generations/).

| Argument or flag | Description |
| --- | --- |
| `[GENERATION]` | The generation number. `archix generations` lists them. |
| `--noconfirm` | Don't ask. Refuses [unsafe downgrades](/docs/concepts/generations/#downgrades-that-may-break-data) unless `--allow-unsafe-downgrade` is given too. |
| `--allow-unsafe-downgrade` | Go back across versions that may not read newer data (postgresql majors and the like) without asking. They're still listed. |

### `user unlock`

Enable a declared account a rollback brought back locked: its declared shell and password, not expired.

```console
$ sudo archix user unlock alice
alice is unlocked with shell /usr/bin/zsh; it has no password yet: set one with `passwd alice`
```

With a [`hashed_password`](/docs/reference/options/#users) declared, it says `alice is unlocked with its declared password and shell /usr/bin/zsh`. An account no rollback locked is refused, `alice isn't an account a rollback brought back locked; nothing was changed`, and so is one the current config doesn't declare: `alice isn't declared in the current config, so it stays locked; declare it and switch, or remove it with userdel`. See [Accounts after a keep](/docs/concepts/snapshots/#accounts-after-a-keep).

### `snapshot`

The root's btrfs snapshots: taking, listing and deleting them, and laying the root out for rollbacks. None of the subcommands evaluates the config. See [Snapshots](/docs/concepts/snapshots/).

#### `snapshot create`

Take a read-only snapshot of the root now, of kind `manual`. Retention never drops it.

```console
$ sudo archix snapshot create --note "before the GPU driver swap"
snapshot 15 taken: /.archix/snapshots/15
```

| Flag | Description |
| --- | --- |
| `--note <text>` | A note to keep with it, shown by `archix snapshot list`. |

It needs the root laid out for rollbacks, with `@archix` mounted at `/.archix`: `` /.archix isn't mounted, so a snapshot's record wouldn't survive a rollback; run `sudo archix snapshot setup` and reboot ``.

#### `snapshot list`

List the snapshots, oldest first, with what took each and the state it holds. Doesn't need root.

```console
$ archix snapshot list
  11  2026-10-09 09:12 UTC  pacman    system as of tree 3f2a9c1e07ab (generation 41), before pacman transaction
  12  2026-10-15 08:55 UTC  switch    system as of tree 3f2a9c1e07ab (generation 41), before switch to tree 5d7b21c4e9f0
  15  2026-10-16 19:02 UTC  manual    system as of tree 5d7b21c4e9f0 (generation 42), taken by hand  (before the GPU driver swap)
```

The kinds are `switch`, `pacman`, `manual` and `replaced`, the root a [rollback](#rollback) replaced. Each snapshot is taken before its change, so it describes what it contains, the system as of the tree before, labelled with what came next. With none, it says `no snapshots yet`.

#### `snapshot delete`

Delete a snapshot and its boot entry.

```console
$ sudo archix snapshot delete 11
snapshot 11 deleted
```

A number that isn't there is refused: `there's no snapshot 11`.

#### `snapshot setup`

Lay the root out for rollbacks: `/.archix` and `/var/lib` as subvolumes of their own, `@archix` and `@varlib`, with pacman's database, DKMS's and archix's state kept on the root under `/var/rollback` and bound over `/var/lib`. `/var/lib` moves at the next boot. A switch with [`rollback.enable`](/docs/reference/options/#rollbackenable) does this by itself.

```console
$ sudo archix snapshot setup
the root is laid out for rollbacks once you reboot: the next boot moves /var/lib into its own subvolume before any service starts, so services' data stays put when you roll back
```

| Flag | Description |
| --- | --- |
| `--move-root` | On a root that is btrfs's top level: move it into an `@` subvolume at the next boot, in the initramfs; other subvolumes at the top level stay as they are. |

```console
$ sudo archix snapshot setup --move-root
the root moves into an @ subvolume at the next boot, in the initramfs: don't power off while it says so. Other subvolumes at the top level stay as they are, and get fstab lines to stay mounted where they are now, which the next switch asks you to declare in file_systems; after that boot, the next switch lays the root out for rollbacks
```

The move happens in archix's initramfs, so archix has to manage booting first: ``the move happens in the initramfs, which doesn't have archix's hook yet: let archix manage booting (boot = { loader = "systemd-boot" }) and switch first; nothing was changed``. On a root that isn't the top level it says ``/ isn't btrfs's top level, so there's nothing to move; nothing was changed``. See [A root at the top level](/docs/concepts/snapshots/#a-root-at-the-top-level).

### `resync`

Adopt a root archix didn't produce, after a snapper or limine-snapper-sync rollback or a restore: check its layout, carry accounts over from the root it replaced when that can be found, and record it. Until then, `switch`, `revert` and `rollback` refuse, and `archix status` and the boot say why: ``the root subvolume (…) isn't the one archix last produced (…): another tool (a snapper or limine-snapper-sync rollback, a restore) replaced it, so archix's records describe another root; on Limine every fallback goes through limine-snapper-restore, so this is expected there. Switches and keeps refuse until `sudo archix resync` adopts it``.

```console
$ sudo archix resync
...
carried users, groups and passwords over from the root it replaced (/@)
recorded the root; switches and keeps work again
```

It finds the replaced root by the subvolume UUID archix recorded. When that's gone, it carries nothing rather than guess: `the root it replaced (…) isn't among the subvolumes any more (another tool may have moved or deleted it), so no accounts were carried over; check users and passwords`. On a root archix produced it says `the root is the one archix last produced; nothing to resync`. See [A root archix didn't produce](/docs/concepts/snapshots/#a-root-archix-didnt-produce).

### `gc`

Delete old generation records and the cached files no kept generation or snapshot needs: package files, snapshot databases and managed-file contents. It also runs after every switch and revert.

```console
$ sudo archix gc
deleted generation records 30, 31
freed 812.4 MiB: 46 cached package files, 3 snapshot databases
```

Records go when they fall outside the newest [`generations.keep`](/docs/reference/options/#generationskeep) (10) and `generations.keep_days` (30), using the limits the current generation was applied with. Snapshots themselves follow [retention](/docs/concepts/snapshots/#retention), applied after every switch and plain pacman transaction. A retired secrets identity no kept generation needs goes too: `deleted the retired secrets identity age1…, which no kept generation needs`.

### `rescue`

The prompt the [rescue entry](/docs/guides/recovery/#without-a-live-usb) opens, from an initramfs that mounts nothing of the layout; on a running system it refuses and points at `archix rollback`. It first asks for the password of root or of a wheel or sudo user from the root's `/etc/shadow` (three tries, then the machine restarts), and on an encrypted root for the LUKS passphrase before that. Run from a live USB with `--root-dir`, it asks for no login. With no command it opens the prompt.

| Command | Description |
| --- | --- |
| `check [--repair-btrfs]` | Check and repair the root filesystem and the ESP with fsck; unmounts them first. On btrfs, `--repair-btrfs` repairs with `btrfs check --repair`, which can make some damage worse; by default btrfs is only checked. |
| `password USER` | Set USER's password in the root's `/etc`, read twice from the console. |
| `open DEVICE [NAME]` | Open a LUKS volume with its passphrase as `/dev/mapper/NAME` (`rescue-data` by default); never with a TPM, FIDO2 or PKCS#11 token. |
| `shell` | Open a shell; `exit` comes back to the prompt. |
| `rollback N` | Roll back to snapshot N: `archix rollback N` on the root with its layout mounted, so it becomes `@` from the next boot. Only with rollbacks on btrfs; hidden otherwise, and refused with `this root has no rollbacks (it isn't btrfs, or rollbacks were never set up), so there's no snapshot to go back to`. |
| `reboot` | Unmount everything and restart. |
| `poweroff` | Unmount everything and power off. |

| Flag | Description |
| --- | --- |
| `--root <DEVICE>` | The root filesystem's device, or `UUID=`, `PARTUUID=`, `LABEL=`; by default `root=` from the kernel command line. Give `/dev/mapper/NAME` after `open`. |
| `--rootflags <OPTIONS>` | Mount options for the root, as `rootflags=` gives them (`subvol=@` for a btrfs root in a subvolume). |
| `--esp <DEVICE>` | The ESP's device, or `PARTUUID=`; by default the partition systemd-boot started from. |
| `--root-dir <DIR>` | A root already mounted here, as from a live USB, instead of the one rescue mounts. |
| `--esp-dir <DIR>` | With `--root-dir`, where its ESP is mounted if not where `boot.esp` says under it. |

See [Recovering a broken system](/docs/guides/recovery/).

### `chroot`

Repair from a live USB: mount the root on a device as it boots, `@archix`, `@varlib`, the binds from `/var/rollback` and the ESP included, and `arch-chroot` into it. Needs `arch-install-scripts` on the live system.

```console
# archix chroot /dev/nvme0n1p2
# archix chroot /dev/mapper/root pacman -Qkk linux
```

pacman's database lives on the root under `/var/rollback/pacman` and is bound over `/var/lib/pacman` at boot, so a plain `mount` and `arch-chroot` would show pacman an empty directory with a permanent `db.lck` in it, and pacman refuses there. `chroot` mounts it the way the boot does, runs the command (a shell by default), and unmounts everything when it ends. Open an encrypted root with `cryptsetup open` first and pass `/dev/mapper/<name>`.

The ESP goes where [`boot.esp`](/docs/reference/options/#bootesp) says, `/efi` by default, so `archix rollback N` inside writes its boot menu without a manual `mount`. It mounts the device the root's `fstab` names there, else its first `vfat` line; with no `fstab` line, the only EFI system partition `lsblk` lists. With several or none it refuses, naming them, `the root's fstab has no ESP line and there are 2 EFI system partitions (/dev/nvme0n1p1, /dev/sda1); name the root's with --esp DEVICE`. See [From a live USB](/docs/guides/recovery/#from-a-live-usb).

| Argument | Description |
| --- | --- |
| `<DEVICE>` | The root's btrfs device, such as `/dev/nvme0n1p2` or an opened `/dev/mapper/root`. |
| `[COMMAND]...` | The command to run inside, a shell by default. |
| `--esp <DEVICE>` | The ESP's device, when the root's fstab doesn't say and there's more than one EFI system partition. |

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

Experimental, tested with swtpm only: replace systemd's PCR policy signing key in `/etc`, and bind the TPM2 slot of each LUKS2 volume you name to the new key. Needs root and `openssl`.

```console
$ sudo archix secure-boot rotate-pcr-key /dev/nvme0n1p2
wrote a new PCR policy signing key to /etc/systemd
bound /dev/nvme0n1p2's TPM2 slot to the new key
run `archix switch` before the next restart: until a UKI built with the new key boots, TPM2-bound volumes unlock only with a passphrase or recovery key, and UKIs built with the old key no longer unlock them
```

It makes a new 2048-bit RSA key, writes `/etc/systemd/tpm2-pcr-private-key.pem`, readable by root only from the start, and `tpm2-pcr-public-key.pem`, and re-enrolls each volume with `systemd-cryptenroll --wipe-slot=tpm2 --tpm2-public-key-pcrs=11`, which asks for its passphrase or recovery key, or reads `$PASSWORD`. A volume that fails to bind is named, with the `systemd-cryptenroll` line to run for it; the key stays in place. The next `switch` builds UKIs whose policies it signs with the new key. See [TPM-bound secrets](/docs/guides/secure-boot/#tpm-bound-secrets).

| Argument | Description |
| --- | --- |
| `[devices]...` | LUKS2 volumes to re-enroll, such as `/dev/nvme0n1p2`. Without any, it only replaces the key. |

### `secrets`

Declared [secrets](/docs/reference/options/#secretsitems): this machine's key, editing, setting, re-encrypting and checking them. Secrets are age files in the config directory, decrypted at boot, and by every switch, into `/run/archix/secrets`, a ramfs.

| Command | Description |
| --- | --- |
| `init [--replace]` | Make this machine's age identity in `/var/lib/archix/secrets` and print the key for `secrets.machine_key`. Needs root. `--replace` makes a new identity even though one exists; the old one is retired, kept while a kept generation needs it. |
| `pubkey` | Print this machine's age recipient. |
| `edit NAME [--machine]` | Decrypt a secret into an editor on a private ramfs and encrypt what you save to every recipient the config names. `--machine` decrypts with this machine's own key (root) instead of yours. |
| `set NAME [--file PATH]` | Encrypt a value from `--file` or stdin as a secret, for piping from a password manager. |
| `rekey [NAMES]... [--machine]` | Re-encrypt secrets (all by default) to the recipients every host's config names now, as after adding a host or an admin. `--machine` decrypts with this machine's own key and its retired ones (root) instead of yours. |
| `check` | Show for each secret who it should be encrypted to against who it was, whether this machine decrypts it, and what uses it. |
| `rotate-machine-key [--compromised]` | Make a new machine identity; the old one is retired until no kept generation needs it (`gc` drops it then). Needs root. `--compromised` deletes the old key at once instead of retiring it. |

## Commands that read the system or the config

These don't change the system and don't need root. As a normal user, archix keeps its download cache in `~/.cache/archix` (or `$XDG_CACHE_HOME/archix`); as root it uses `/var/cache/archix`.

### `diff`

Show what a switch would change to make this machine match the lock and the config, and what it carries that the config doesn't declare.

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

When everything matches, it prints `the system matches the lock and the config`. If installed packages are newer than the lock, it says so first: ``3 packages are newer than the lock; `archix switch` will offer to adopt them instead of downgrading``. If the lock is out of date with the config, `diff` resolves a fresh lock in memory, prints `note: archix.lock is out of date for desktop; showing a fresh resolution`, and shows the result without saving it. Packages in `ignored_packages`, and packages [drift](/docs/concepts/pacman-and-yay/#drift) keeps untracked, never appear as removals. With `undeclared_packages = "warn"`, undeclared packages are listed on their own line instead of as removals:

```console
not declared, kept because undeclared_packages = "warn": htop ncdu
```

Files declared with a `url` are compared by their `sha256`, so `diff` downloads nothing. Flatpak changes are listed as in [`switch`](#switch). `diff` never adds Flatpak remotes, so an app that can't be locked yet, because flatpak or its remote isn't there, shows as `flatpak    install <id>: ...` with the reason, until the switch that adds it. System-wide Flatpak apps that aren't in the config, and weren't in the current generation, are listed on their own line. archix leaves them alone:

```console
flatpak apps archix doesn't manage: com.valvesoftware.Steam
```

| Flag | Description |
| --- | --- |
| `--all` | List everything carried but not declared, not only the first lines of each kind. |

### `status`

Show the lock, the current generation, the snapshots and boot entries, pending changes and what the machine carries that the config doesn't declare.

```console
$ archix status
current generation: 42 (tree 5d7b21c4e9f0, snapshot 2026-10-15)
package cache: 412 MiB on disk in /.archix/packages, for reverts and rollbacks (reflinks and hard links share blocks with pacman's cache)
snapshot  11  2026-10-09 09:12 UTC  pacman    system as of tree 3f2a9c1e07ab (generation 41), before pacman transaction
snapshot  12  2026-10-15 08:55 UTC  switch    system as of tree 3f2a9c1e07ab (generation 41), before switch to tree 5d7b21c4e9f0  (fallback slot)
boot entries: archix-current.conf archix-snapshot-11.conf archix-snapshot-12.conf
Secure Boot: on
desktop is locked to snapshot 2026-10-15 (214 packages, 1 pinned off the snapshot)
the system matches the lock
```

The first lines are notices, when they apply: the [try mode](/docs/concepts/boot-and-fallback/#try-mode) notice on a boot of an older snapshot, a root another tool put in place (see [`resync`](#resync)), a [rollback](#rollback) waiting for its reboot, a health check or boot count that fell back, a move from Limine to systemd-boot in progress (`moving from Limine to systemd-boot: each switch has the firmware try systemd-boot (entry 0003) once; it goes first once a boot through it counts as good, and until then Limine stays first`), archix's pacman hooks masked or shadowed (`pacman hook, ...`), and packages [drift](/docs/concepts/pacman-and-yay/#drift) keeps untracked: `2 packages installed outside archix kept untracked (drift): htop ncdu`.

Under the current generation come the size of [`/var/lib/.archix-moved`](/docs/concepts/snapshots/#varlib-after-a-keep) when a keep moved files aside, the package cache archix keeps for reverts and rollbacks, every snapshot with the state it holds, the fallback slot marked, and the boot entries on the ESP. Other lines appear when they apply: ``warning: systemd-boot's LoaderEntryDefault variable (the boot menu's default, or `bootctl set-default`) names archix-snapshot-11.conf and overrides archix's default; the next switch clears it, or `sudo bootctl set-default ''` now``, a warning when no account can log in to the [rescue entry](/docs/guides/recovery/#without-a-live-usb), `warning: snapshots 11, 12 should have boot entries, but the ESP has none or a torn one; the next switch or pacman transaction writes them again` when a [power cut](/docs/concepts/boot-and-fallback/#power-cuts-and-the-esp) left an entry missing, and `3 packages were changed outside archix in 2 transactions since the last switch`. The lock line counts the [pins](/docs/concepts/lockfile/#pins), packages held at a version other than the snapshot's, when there are any. When the snapshot is older than [`lock_warn_after_days`](/docs/reference/options/#lock_warn_after_days), 30 by default, the next line warns: ``warning: archix.lock is at the 2026-09-10 snapshot, 38 days old; updates released since aren't installed. `archix update` moves it to the newest snapshot``. `switch` prints the same warning before it starts.

On a machine that booted through UEFI, the `Secure Boot:` line says `on`, `off`, or ``setup mode (no keys enrolled; `sudo archix secure-boot setup` enrolls them)``. The last line compares the lock with the system: `the system matches the lock`, or ``2 changes pending; see `archix diff` ``. If the config changed since the lock was written, it adds ``the config changed since the lock was written; `archix lock` or `archix switch` will re-resolve``. That includes editing a recipe in `pkgbuilds/` or a file listed in `patches`, not just the Lua.

Packages archix builds from a VCS checkout, AUR packages and recipes named `-git`, `-hg`, `-svn`, `-bzr`, `-fossil` or `-darcs`, get their own line while the lock has no pinned sources for them, as before their first build: ``1 package builds from a VCS source archix.lock has no commit for yet, so its next build takes upstream's newest code, whose git commit is then recorded: neovim-git``. `archix lock` and `update` mark them the same way, `+ neovim-git 0.12.0.r310.g1a2b3c4-1 (builds from a VCS source; its first build records the commit)`. Packages in `ignored_packages` that aren't installed are named too, as `switch` names them: `ignored_packages lists 1 package that isn't installed here, and archix never installs ignored ones: nvidia-dkms-custom`. That's what a second machine sees when it shares a config `init` wrote on another.

| Flag | Description |
| --- | --- |
| `--all` | List everything carried but not declared, not only the first lines of each kind. |

### `generations`

List generations, the current one marked with `*`, then the snapshots by the state each holds.

```console
$ archix generations
    40  2026-10-02 18:40 UTC  snapshot 2026-10-02  210 packages  config 8c1f0e2a7b
    41  2026-10-09 09:12 UTC  snapshot 2026-10-09  212 packages  config 3f2a9c1e07
*   42  2026-10-15 08:55 UTC  snapshot 2026-10-15  214 packages  config 5d7b21c4e9+uncommitted
snapshots, by the state each holds:
    11  2026-10-09 09:12 UTC  pacman    system as of tree 3f2a9c1e07ab (generation 41), before pacman transaction
    12  2026-10-15 08:55 UTC  switch    system as of tree 3f2a9c1e07ab (generation 41), before switch to tree 5d7b21c4e9f0
*   15  2026-10-16 19:02 UTC  manual    system as of tree 5d7b21c4e9f0 (generation 42), taken by hand  (before the GPU driver swap)
```

A generation is a record of what a switch or revert applied; a snapshot is the root as it was before a change, so each one describes the generation it contains, not the one that came after. `*` on a snapshot marks one holding the current generation. `+uncommitted` marks a generation switched from a config with uncommitted changes. On ext4 or with rollbacks off there are no snapshots, and only the generations are listed. `generations` doesn't evaluate the config. See [Generations and rollback](/docs/concepts/generations/).

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

## Removed commands

These went with deployments, when `archix switch` started applying in place.

| Command | What to use instead |
| --- | --- |
| `eject` | Nothing to undo: the system runs in place, as plain Arch with archix's files on it. `pacman -R archix` works even in strict mode, and archix's pacman hooks let everything through once its binary is gone. `archix eject` is now a search for "eject". |
| `mount-deployment` | [`chroot`](#chroot), which mounts the root with its layout from a live USB. |
| `verify` | `pacman -Qkk`, which reads the root's own pacman database. |
| `hooks retry`, `hooks skip` | Hooks run during the switch, so a failing one is reported there; run `archix switch` again once it's fixed. |
| `install --temp` | Plain pacman or an AUR helper; [`drift`](/docs/reference/options/#drift) decides what the next switch does with the package. |
| `rescue use`, `rescue list`, `rescue verify`, `rescue restore-etc`, `rescue eject` | `rescue rollback N`, `rescue shell`. |
| `rollback [generation]`, applying a generation | [`revert [generation]`](#revert). `rollback N` now keeps snapshot N. |
| `recover` | Running it says `` `archix recover` is gone: a switch that fails partway says what it applied, and with rollbacks on a snapshot holds the system as it was before; run `archix switch` again `` and exits. |

## Internal commands

Hidden commands are run by files archix installs, not by you: `archix pacman-hook pre` and `post`, from archix's pacman hooks in `/usr/share/libalpm/hooks`, which snapshot the root before a plain transaction and rebuild the boot image and reset boot counting after it; `archix boot-check`, from `archix-boot.service`; `archix boot-complete`, from `archix-boot-complete.service`, once systemd-boot has counted a boot as good; `archix boot-mount`, in the initramfs, which mounts the snapshot the started entry names, read-only with a throwaway layer; `archix health-check`, from `archix-health-check.service`; `archix unblessed`, from `archix-unblessed.timer`, which reboots a counted boot stuck past [`boot.unblessed_timeout`](/docs/reference/options/#bootunblessed_timeout); `archix bless`, from a drop-in for `systemd-bless-boot.service`, which counts the booted entry as good and flushes it under its new name, as a FAT rename isn't on disk until then; `archix notice`, which prints the try-mode notice at login and, with `--desktop`, sends it as a desktop notification; `archix split-var-lib`, from `archix-split-var-lib.service`, which moves `/var/lib` into its own subvolume before it's mounted; `archix swapfile`, which makes a declared swap file before it's swapped on; `archix emergency-hint`, from a drop-in for `emergency.service`; `archix secrets activate`, from `archix-secrets.service`; `archix evaluate-isolated`, which a root command runs under a transient uid to evaluate a captured config; and `archix write-file`, which root runs as the config directory's owner to write a file there without following a symlink.

## Exit behaviour

- `0` when the command succeeded.
- `1` on errors, with each error printed to stderr as `error: ...`. That includes config errors (every one is reported, not just the first), resolution failures, a declined prompt, and refusals before anything is applied, which leave the system as it was and take no snapshot.
- Non-zero after a switch or revert with soft failures: nothing is undone, the steps that failed (units that can't be enabled or started, hooks, `on_change` scripts, Flatpak apps, the boot menu, `gc`) are listed, and the last line says what did happen, such as `error: generation 43 is applied and current, but 2 steps failed (listed above)`. A switch that had nothing to do but whose hooks failed ends with `error: nothing was changed, but 1 step failed (listed above)`, and a revert with `error: generation 41 (snapshot 2026-10-09) is applied and current, but 1 step failed (listed above)`. See [Safety](/docs/concepts/safety/#hard-and-soft-failures).
- Non-zero after a switch whose package transaction or another hard step failed partway: what was applied stays, the generation isn't recorded, and the error names the snapshot that holds the system as it was before.

Piping output works as expected: `archix diff | head` exits quietly when `head` closes the pipe.
