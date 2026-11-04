---
title: pacman and yay
description: Plain pacman, yay, paru and pamac keep working beside archix; the pacman hooks archix ships, the snapshot before each transaction, what the next switch does with packages installed outside it, strict mode, and the pacman-style commands.
sidebar:
  order: 4
---

Arch stays Arch. `/usr` and `/opt` are writable, and plain pacman, yay, paru and pamac keep working beside archix: `pacman -S htop`, `pacman -Syu` and `yay -S zen-browser-bin` change the running system as they always have. archix doesn't wrap pacman. It ships a few pacman hooks that snapshot the root before each transaction, keep the boot menu in step after it, and note what changed, so the next `archix switch` can bring it into your config.

You can also change packages through archix, with the commands you already type:

| Instead of | Run | What it does |
| --- | --- | --- |
| `pacman -S htop`, `yay -S zen-browser-bin` | `archix -S htop` | Declares it in your config and installs it |
| `pacman -Syu`, `yay` | `archix -Syu`, or bare `archix` | Moves the lock to the newest snapshot and AUR recipes, then switches |
| `pacman -R htop` | `archix -R htop` | Takes it out of your config and uninstalls it |

Both take effect right away. The difference is what's recorded: archix's commands change your config and the lock, so the change is part of a generation you can [revert](/docs/concepts/generations/#reverting) and reproduce on another machine; plain pacman changes only this machine, until the next switch settles it by the [drift](#drift) policy.

## archix's pacman hooks

Every switch, even one with nothing else to do, writes four hooks into `/usr/share/libalpm/hooks`, beside the ones packages ship:

| Hook | When | What it does |
| --- | --- | --- |
| `00-archix-removal.hook` | Before a transaction that removes archix | Notes pacman's PID, so [strict mode](#strict-mode) lets that transaction through |
| `00-archix-strict.hook` | Before every transaction | Runs `/usr/lib/archix/strict-hook.sh`, a small shell script that refuses the transaction while strict mode is on or a [rollback is pending](/docs/concepts/snapshots/#keeping-a-snapshot), and lets it through otherwise |
| `01-archix-snapshot.hook` | Before every transaction, after the strict hook and before every other hook, mkinitcpio's included | Snapshots the root, with rollbacks on |
| `95-archix-post.hook` | After every transaction, after DKMS's and mkinitcpio's hooks | Rebuilds the boot images if the kernel or initramfs changed, resets boot counting, and records the packages that changed |

They skip transactions archix runs itself, which recognise themselves by an environment variable. A plain transaction looks like this:

```console
$ sudo pacman -S htop
...
:: Running pre-transaction hooks...
(1/2) archix: checking strict mode...
(2/2) archix: snapshotting the root before this transaction...
archix: snapshot 13 holds the system as it was before this transaction
:: Processing package changes...
(1/1) installing htop
:: Running post-transaction hooks...
(1/1) archix: recording the transaction and updating the boot menu...
archix: recorded 1 package changed outside archix; `archix switch` handles them as the config's drift policy says
archix: the boot menu is up to date, and the current system's entry has 2 tries again
```

Each hook checks first that archix is still installed, so removing archix, or deleting its binary, never wedges pacman; the post hook then deletes archix's hooks itself.

### The snapshot before a transaction

With [rollbacks](/docs/reference/options/#rollbackenable) on and the root laid out for them, `01-archix-snapshot.hook` takes a read-only snapshot of the root before pacman changes anything, of kind `pacman`, labelled `before pacman transaction`. Before the transaction starts, it also gives the current system's boot entry its [tries](/docs/reference/options/#boottries) and the fallback slot its entry, and marks the transaction unfinished until the post hook has flushed the root, so a boot after a power cut mid-transaction doesn't count as good and falls back to the system from before it. See [Snapshots before plain pacman](/docs/concepts/snapshots/#snapshots-before-plain-pacman) and [What falls back by itself](/docs/concepts/boot-and-fallback/#what-falls-back-by-itself).

The hook makes sure the snapshot has room first. It estimates what the transaction installs, without subtracting what it replaces, since the snapshot keeps the old files, plus a margin of a tenth of that, at least 64 MiB. pacman's own check subtracts the old files, so it would pass where the snapshot can't. When the root would end up below [`rollback.min_free`](/docs/reference/options/#rollbackmin_free), or below the 1024 MiB kept for a rollback, it says so, and on a terminal it asks:

```console
archix: warning: this transaction needs about 2310 MiB with the snapshot that keeps what it replaces, and the root has 9120 MiB free; after it 7% would be left, below rollback.min_free = 10%
archix: [a]bort, [p]rune old snapshots, or [s]till try (snapshot and install anyway)? [a]
```

- **Abort**, the default, stops the transaction before it changes anything: ``archix: aborted the transaction: the root is short of space for it and its snapshot; nothing was changed. Free space, delete snapshots with `archix snapshot delete N`, or set rollback.low_space``.
- **Prune** deletes the oldest snapshots it may, but only while a safe recovery point stays, a snapshot of a system that booted fine with its boot entry, and the transaction goes ahead if it then fits: `archix: deleted snapshots 4, 5 to make room; snapshot 9, the fallback slot, stays`. Without a safe recovery point, or still short after pruning, it aborts.
- **Still try** takes the snapshot and installs anyway. It's offered only while the 1024 MiB kept for a rollback would still be free.

With no terminal to ask, as under pamac, `pacman --noconfirm` or an unattended update, [`rollback.low_space`](/docs/reference/options/#rollbacklow_space) decides: `"prune"` by default, `"abort"`, or `"skip"`, which lets the transaction run without a snapshot: `archix: no snapshot before this transaction (rollback.low_space = "skip")`. What the hook decided goes to the journal, tagged `archix`, and `archix status` shows the last ten such events.

With rollbacks off, on ext4, or during a [try boot](/docs/concepts/boot-and-fallback/#try-mode), there's no snapshot. On a root [archix didn't produce](/docs/concepts/snapshots/#a-root-archix-didnt-produce), the hook never blocks pacman: it skips the snapshot with `archix: warning: no snapshot before this transaction: ...` and the reason.

### After a transaction

`95-archix-post.hook` flushes the root, then:

- when the transaction changed the kernel or the initramfs, rebuilds the boot images, `archix: the kernel or initramfs changed; building the boot images`, and refreshes the rescue entry;
- prunes snapshots by the [retention](/docs/concepts/snapshots/#retention) settings;
- writes the boot menu again and gives the current system's entry its tries, so the next boot is counted and [falls back](/docs/concepts/boot-and-fallback/) by itself if it doesn't come up;
- records the packages that changed in `/var/lib/archix/outside-changes.json`, which `archix status` reports and the next switch settles by the [drift](#drift) policy.

On Limine, archix writes no boot entries, so the post hook only records the transaction.

### Masked and shadowed hooks

A hook in a `HookDir`, `/etc/pacman.d/hooks` by default, replaces a packaged hook of the same name, and a link to `/dev/null` there masks it. `archix status`, the login notice and a desktop notification say so for archix's hooks:

```console
archix: archix's pacman hook 01-archix-snapshot.hook is masked by /etc/pacman.d/hooks/01-archix-snapshot.hook, so plain pacman runs take no snapshot; remove that file
```

A hook in `/usr/share/libalpm/hooks` that's missing, changed or linked to `/dev/null` is reported the same way, and the next switch writes it again.

## Drift

A package you install with plain pacman or an AUR helper is on the machine but not in your config. [`drift`](/docs/reference/options/#drift) decides what the next switch does with explicitly installed packages the post hook saw arrive since the last switch, and `archix init` asks:

| `drift` | The next switch |
| --- | --- |
| `"ask"`, the default | Asks whether to add them to `installed.lua`; `--noconfirm` adds them |
| `"add"` | Adds them to `installed.lua` without asking |
| `"keep"` | Keeps them installed and untracked: never removed, never declared |

```console
$ sudo archix switch
installed with pacman or an AUR helper since the last switch, not declared: htop ncdu
add them to installed.lua? (no keeps them installed, untracked) [Y/n]
adding them to installed.lua, as drift = "ask" says
```

- **Adding** writes the names into the host's [`installed.lua`](#installedlua) and switches with them in the lock, the same as if you'd run `archix install`. `installed.lua` is a generated data file your config imports; archix writes it as the config directory's owner, never through a symlink, and only once the switch has applied.
- **Keeping** says `keeping them installed and untracked: no switch removes them or what they need`, and remembers them. A [revert](/docs/concepts/generations/#partial-downgrades) still checks that it doesn't break them.

If the config doesn't pick up `installed.lua`, adding can't settle the drift, and the switch refuses: ``htop ncdu went into /etc/archix/lua/hosts/desktop/installed.lua, but the config still doesn't declare them with it, so adding them can't settle their drift. Declare them in packages yourself, or set drift = "keep"; nothing was changed``.

```lua title="lua/hosts/desktop/init.lua"
return {
  drift = "keep",
}
```

Drift covers explicitly installed packages. Packages newer than the lock are [adopted](#adopting), and anything else that isn't in the lock is handled by [`undeclared_packages`](#undeclared-packages).

## Strict mode

Strict mode is for machines where every package change should go through the config. With [`strict = true`](/docs/reference/options/#strict) and a switch, plain pacman and every front end on libalpm, AUR helpers and pamac included, refuse every transaction archix doesn't run:

```console
$ sudo pacman -S htop
...
:: Running pre-transaction hooks...
(1/2) archix: checking strict mode...
archix: strict mode is on, so packages change only through archix (`archix install`, `archix remove`, `archix upgrade`); set strict = false and switch to turn it off, or remove archix with pacman -R archix
error: failed to commit transaction (failed to run transaction hooks)
```

```lua title="lua/hosts/server/init.lua"
return {
  strict = true,
}
```

It's a guardrail, not enforcement. archix's own transactions are recognised by an environment variable anyone with root can set, and root can always remove the hook. What it stops is the habit, `pacman -Syu` out of muscle memory on a machine you meant to keep declarative.

It's built so it can never wedge pacman. The hook runs a small shell script, not archix, and lets every transaction through once:

- strict mode is turned off and switched;
- archix is removed: `pacman -R archix` works while strict mode is on, through `00-archix-removal.hook`, which notes pacman's PID for the strict script, which ignores it once that pacman is gone;
- archix's binary, its state in `/var/lib/archix`, or the config that turned strict mode on is gone.

The same hook refuses package changes, strict mode or not, while a [rollback is pending](/docs/concepts/snapshots/#keeping-a-snapshot), since the root it would change is replaced at the next boot: ``archix: rollback to snapshot 9 is pending; reboot, or `archix rollback --cancel` ``. In a [try boot](/docs/concepts/boot-and-fallback/#try-mode), a hook of the try boot's own refuses them too.

## Adopting

On the next `archix switch`, if installed packages are newer than the lock, as after a `pacman -Syu`, archix asks:

```console
$ sudo archix switch
the system is newer than the lock for 3 packages (linux, mesa, vulkan-radeon), probably from pacman or an AUR helper
adopt them into the lock? (no reverts them to the lock) [Y/n]
```

- **Yes** moves the lock to match the system. The switch then proceeds from there, and the upgrade you did with pacman becomes part of a generation you can revert to.
- **No** reverts the system to the lock. Packages go back to their locked versions.

`--noconfirm` adopts. `archix diff` mentions it ahead of time: ``3 packages are newer than the lock; `archix switch` will offer to adopt them instead of downgrading``.

Adopting re-resolves your config at the lock's date and at the days around your manual upgrade, and keeps the snapshot that matches the most installed versions. Live mirrors don't update in lockstep with the archive, so a `pacman -Syu` rarely matches one day exactly. Anything still newer is [pinned](/docs/concepts/lockfile/#pins) to its exact installed version, fetched from the Arch Linux Archive's per-package history and listed under `pinned`. Nothing is downgraded.

```console
adopt them into the lock? (no reverts them to the lock) [Y/n] y
snapshot 2026-10-09 -> 2026-10-15
~ linux 7.2.8.arch1-1 -> 7.2.9.arch1-1
~ mesa 1:25.3.0-1 -> 1:25.3.1-1
~ vulkan-radeon 1:25.3.0-1 -> 1:25.3.1-1
214 packages locked
wrote /etc/archix/archix.lock
```

AUR packages upgraded by yay or paru are adopted too: archix finds the commit whose build recipe builds the installed version, and locks that.

[Patched packages](/docs/guides/local-pkgbuilds-and-patches/#patched-packages) are adopted the same way: when pacman replaced your patched build with Arch's newer one, adopting moves the lock to that version and applies your patches to it, so the switch goes on to `upgrade    hyprland 0.52.0-1 -> 0.52.0-1.1`. Declining puts your patched build of the locked version back.

Pins survive `archix lock` at the same date. `archix update` to a new snapshot drops them.

## The pacman-style commands

archix is also an AUR helper, with the flags you already type:

| Command | Same as |
| --- | --- |
| `archix -S <pkg>` | `archix install <pkg>` |
| `archix -Ss <terms>` | `archix search <terms>` |
| `archix -R <pkg>` | `archix remove <pkg>` |
| `archix -Syu` | `archix upgrade` |
| `archix -Syu <pkg>` | `archix upgrade`, also installing `<pkg>` |
| `archix` | `archix upgrade` |
| `archix <terms>` | `archix search <terms>`, then install the numbers you pick |

Like `yay <terms>`, a search with no command lists numbered results, the best match as 1 next to the prompt, and installs what you pick through `archix install`, declaring it in `installed.lua`:

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

Run as your user, archix asks sudo for the install itself. Commands win over terms, so `archix status` is never a search. See [`archix <terms>`](/docs/reference/cli/#search-and-install-archix-terms).

`-Sy` on its own, and `-S` or `-Ss` without arguments, fail with a hint: archix never syncs databases without upgrading. See the [CLI reference](/docs/reference/cli/#install--s) for every flag.

### `archix -Syu`

If you want the yay experience, use the full upgrade:

```console
$ sudo archix -Syu
```

It moves the lock to the newest Arch Linux Archive snapshot, moves AUR packages to the newest commit of their build recipes (showing each changed recipe for review), and switches. Running `archix` with no arguments does the same, the way `yay` does. It's `archix update` followed by `archix switch`, in one command. It upgrades to the archive snapshot the lock moves to, not to your live mirrors, which is what makes the result reproducible.

### `installed.lua`

`archix install` declares before it installs. The package name goes into `lua/hosts/<host>/installed.lua`, a plain Lua file that's yours as much as archix's:

```lua title="lua/hosts/desktop/installed.lua"
-- This host's packages. Edit freely; `archix install` and `archix remove` add and remove names here too.
return {
  packages = {
    "htop",
    "zen-browser-bin",
  },
}
```

archix imports it automatically whenever it exists, so you don't have to. `archix remove` takes names out of it. archix never rewrites your own Lua: a package you declared elsewhere is refused by `remove`, with a pointer to where it's declared. `archix init` puts the packages it captures in this file, so `install` and `remove` work on them from day one.

`archix install --path <dir>` edits the `installed.lua` in another directory under `lua/`, for example a module shared by several hosts. Import it yourself; archix prints the name to use.

## Undeclared packages

[Drift](#drift) covers packages the post hook saw you install explicitly. Anything else that's installed but in neither the lock nor your config, such as a package installed before archix's hooks were there, is handled by [`undeclared_packages`](/docs/reference/options/#undeclared_packages). Packages that drift keeps untracked, and [ignored packages](#ignoring-packages-everywhere), never count.

**`"remove"`** is the default: the next switch removes them, so the system is exactly what the config says, which is what makes a revert and a fresh reinstall meaningful. The switch lists each removal before it asks.

This covers packages only. Changes to `/etc` files and services that archix never managed are left alone; see [Safety](/docs/concepts/safety/#managed-files-backups-and-hand-edits).

**`"warn"`** leaves them installed and only lists them, in `archix diff` and `archix switch`:

```lua title="lua/hosts/desktop/init.lua"
return {
  undeclared_packages = "warn",
}
```

```console
$ archix diff
snapshot 2026-10-15: 214 packages
the system matches the lock and the config
not declared, kept because undeclared_packages = "warn": htop ncdu
```

To keep one, add it to `packages` or run `archix install htop`. To get rid of it, switch back to `"remove"`.

## Ignoring packages everywhere

Some packages you manage by hand on purpose. List them in [`ignored_packages`](/docs/reference/options/#ignored_packages), and archix never touches or reports them, whatever `drift` and `undeclared_packages` say:

```lua title="lua/modules/common.lua"
return {
  ignored_packages = { "nvidia-dkms-custom" },
}
```

Because it's in your config, the decision is visible, committed and the same on every machine that imports the module. `archix init` lists installed packages that are in neither the repos nor the AUR here, and, if you choose, orphaned dependencies.

## Third-party repos in `pacman.conf`

So that plain pacman sees the same repos as archix, archix keeps your declared [third-party repos](/docs/guides/third-party-repos/) in `/etc/pacman.conf`, inside marked blocks around the official repos:

```ini title="/etc/pacman.conf"
[options]
HoldPkg = pacman glibc
Architecture = auto

# BEGIN archix custom repos (before the official repos)
[cachyos]
SigLevel = Required DatabaseOptional
Server = https://mirror.cachyos.org/repo/$arch/$repo

# END archix custom repos (before the official repos)
[core]
Include = /etc/pacman.d/mirrorlist

[extra]
Include = /etc/pacman.d/mirrorlist

# BEGIN archix custom repos (after the official repos)
[chaotic-aur]
SigLevel = Required DatabaseOptional
Server = https://cdn-mirror.chaotic.cx/$repo/$arch

# END archix custom repos (after the official repos)
```

Repos with `before_official = true` go in the first block, the rest in the second. archix rewrites only these blocks, at every switch. A hand-written section for a repo your config declares is taken over, so it's never listed twice. Removing a repo from your config removes it from `pacman.conf`. Everything else in the file, including the official repos and their mirrorlists, is yours.
