---
title: Generations and rollback
description: What a generation records, how a switch applies it in place, how `archix revert` goes back to an older one and how that differs from keeping a snapshot, and how old generations are cleaned up.
sidebar:
  order: 3
---

Every `archix switch` that changes something creates a generation: a numbered record of what the system looks like after that switch. The switch applies it to the running system in place, live, with no reboot. Going back means applying an older generation's record again with [`archix revert`](#reverting), or keeping a [snapshot](/docs/concepts/snapshots/) of the whole root with [`archix rollback N`](/docs/concepts/snapshots/#keeping-a-snapshot).

## What a generation contains

The record lives in `/var/lib/archix/generations/<n>/generation.json`, and `/var/lib/archix/generations/current` holds the number of the current one, written durably. The record holds:

- **The packages.** The snapshot date and every package's exact version, file and sha256, including the sha256 of each AUR package it built.
- **The managed files, users, groups and services.** File contents are stored once by sha256 in `/var/lib/archix/blobs`, so identical files across generations cost nothing extra. Files downloaded with [`archix.fetchurl`](/docs/reference/lua-api/#archixfetchurl), AppImages included, are stored the same way.
- **The hooks.** Every [hook](/docs/guides/hooks/) and `on_change` script.
- **The Flatpak apps.** The remote, ref and commit each declared app was pinned to.
- **The settings it was applied with.** `ignored_packages`, the generation limits, the boot settings and the `rollback` table, so revert and `gc` work without evaluating your config.
- **The config's git commit**, if your config directory is a git repo, and whether the tree had uncommitted changes.
- **A format version.** An archix that finds a record written in a newer format than it knows won't apply it: `generation 44's record was written by a newer archix (record format 2; this archix knows up to 1); upgrade archix`.

A switch that changes nothing prints `nothing to do: the system matches generation 42 (snapshot 2026-10-15)` and creates none, though it still runs the generation's [hooks](/docs/guides/hooks/).

### Root's own copy of the config

A switch evaluates a copy of your config, not the directory you edit; see [Safety](/docs/concepts/safety/). Each generation's config tree, its lock and the plan it evaluated to are stored in a repository only root can write, `/.archix/config.git` on a root [laid out for rollbacks](/docs/concepts/snapshots/#the-layout), or `/var/lib/archix/config.git` otherwise. Revert and keep apply from that copy only, after checking it against the hashes the record stores, and apply the stored plan rather than evaluating old Lua with a newer archix. A copy that doesn't check out refuses before anything changes: `generation 41 can't be read back from root's repository; nothing was changed`.

In your own config repository, the switch points `refs/archix/generations/<n>` at the tree it applied, so you can browse what each generation was built from:

```console
$ git log --oneline refs/archix/generations/42
$ git diff refs/archix/generations/41 refs/archix/generations/42
```

archix never reads those refs back, and never commits for you.

### What a switch shows first

Before its prompt, a switch lists the changes it makes, then the plan since the last generation archix applied, then the config's diff since then, from root's copy of both:

```console
$ sudo archix switch
install    htop 3.4.1-1
:: the plan since the last generation archix applied
package + htop 3.4.1-1
:: the config since the last generation archix applied
...
apply 2 changes? [Y/n]
```

The plan diff shows packages, files, units, users and groups added (`+`), changed (`~`) and removed (`-`), never a secret or a password hash. On a first switch there's no earlier tree to compare with: `:: no generation applied by archix yet; this one records config tree 3f2a9c1e07b4`. Only `--noconfirm` skips the prompt.

## Listing generations

```console
$ archix generations
    40  2026-10-02 18:40 UTC  snapshot 2026-10-02  210 packages  config 8c1f0e2a7b
    41  2026-10-09 09:12 UTC  snapshot 2026-10-09  212 packages  config 3f2a9c1e07
*   42  2026-10-15 08:55 UTC  snapshot 2026-10-15  214 packages  config 5d7b21c4e9+uncommitted
snapshots, by the state each holds:
     3  2026-10-09 09:11 UTC  switch    system as of tree 8c1f0e2a7b4d (generation 40), before switch to tree 3f2a9c1e07b4
     4  2026-10-12 19:30 UTC  pacman    system as of tree 3f2a9c1e07b4 (generation 41), before pacman transaction
*    5  2026-10-15 08:54 UTC  switch    system as of tree 3f2a9c1e07b4 (generation 41), before switch to tree 5d7b21c4e9a0
```

`*` marks the current generation, the one the system was last brought to, and the snapshots that hold it. A snapshot is taken before its change, so it describes the system it contains, "system as of tree Y", and its label says what came next. On ext4 or with rollbacks off, there are no snapshots, and only the generations are listed.

## Reverting

`archix revert` applies an earlier generation's record live, in place: the previous generation by default, or the one you name, which works forwards too.

```console
$ sudo archix revert
reverting to generation 41 (snapshot 2026-10-09), applying its record live, in place
downgrade  linux 7.2.9.arch1-1 -> 7.2.8.arch1-1
downgrade  mesa 1:25.3.1-1 -> 1:25.3.0-1
revert to generation 41 (snapshot 2026-10-09)? [Y/n]
:: applying generation 41 (snapshot 2026-10-09)
reverted to generation 41 (snapshot 2026-10-09). archix.lock is unchanged, so the next `archix switch` returns to the config
$ sudo archix revert 40
$ sudo archix revert 42
```

- **What it puts back** is everything the generation declares: packages at their locked versions, from [archix's package cache](#the-package-cache) or the Arch Linux Archive, the managed files, users, groups, services and hooks, and the Flatpak apps at their pinned commits. AUR and local packages go back to the generation's own build, by its sha256. Packages are installed by the sha256 the generation pinned, without checking their signatures against today's keyring again, so a packager key revoked since doesn't block going back; see [Signatures and disk space](/docs/concepts/safety/#signatures-and-disk-space).
- **What it leaves** is everything the generation doesn't declare: packages installed with pacman and kept untracked, `/var/lib` data, home directories, `npm -g` and `/opt` installers.
- **Before it applies**, with rollbacks on, it snapshots the root, labelled `before revert to generation 41`, so the system as it was stays one [keep](/docs/concepts/snapshots/#keeping-a-snapshot) away.
- **When it fails partway**, it can't be undone: it says what was applied, `generation 42 stays current; snapshot 6 holds the system as it was before (`archix snapshot list`)`.

Naming the current generation changes nothing: `generation 42 is current already`. Revert doesn't evaluate your config, so it works even when the config is broken. It refuses in [try mode](/docs/concepts/boot-and-fallback/#try-mode), while a keep waits for its reboot, and on a root another tool replaced, as a switch does.

Revert changes the system, not your config. Your `init.lua` and `archix.lock` still describe the newer generation, so the next `archix switch` applies them again. That's deliberate: archix never rewrites your config behind your back. Before switching again, fix the config, check out an older commit, or move the lock back with `archix update --to`.

### Switching to an older commit

Checking out an older commit of your config and running `archix switch` goes back too, as a new generation. It needs the same packages: the switch fetches each one from archix's package cache or the Arch Linux Archive, and refuses before anything is applied if one can't be fetched.

### Partial downgrades

Packages you installed with pacman or an AUR helper and keep untracked aren't in the lock, so a switch that downgrades declared packages could leave them broken. Before applying, the switch checks every untracked package against the target lock: its declared dependencies, and the libraries its binaries load. A break refuses with the list:

```console
$ sudo archix switch
this switch downgrades declared packages, which would break packages archix doesn't track:
  obs-plugin-foo needs qt6-base>=6.10, but qt6-base would be 6.9.3-1
declare them, remove them, or pass --allow-partial to switch anyway; nothing was changed
```

`--allow-partial` goes ahead regardless. `archix revert` doesn't run this check.

### Downgrades that may break data

Packages go back; their data doesn't. Data in `/var/lib`, such as a database, stays as it is on a revert and on a keep, and some programs migrate the data they keep there when a new major version first starts, so the old version can't read it afterwards, or damages it. PostgreSQL refuses to start on a data directory a newer major version wrote. Going back across such a version is unsafe, so archix asks first, and the answer defaults to no:

```console
$ sudo archix revert 40
reverting to generation 40 (snapshot 2026-10-02), applying its record live, in place
downgrade  postgresql 17.2-1 -> 16.6-1
downgrade  postgresql-libs 17.2-1 -> 16.6-1
unsafe downgrades: the older versions may refuse or damage data the newer ones wrote
  postgresql 17.2-1 -> 16.6-1 goes back from major version 17 to 16, and postgresql migrates its data between them
downgrade anyway? [y/N]
error: aborted; nothing was changed
```

archix knows these packages, and how many version parts make their major version:

| Packages | Major version | Unsafe |
| --- | --- | --- |
| `postgresql`, `forgejo`, `nextcloud`, `redis`, `valkey` | The first part | `17.2-1` back to `16.6-1` |
| `mariadb`, `mysql`, `percona-server`, `gitea`, `mattermost` | The first two parts | `11.4.3-1` back to `11.2.1-1` |
| `chromium`, `google-chrome`, `microsoft-edge-stable-bin` | The first part | `141.0.7390.54-1` back to `140.0.7339.207-1` |
| `brave-bin`, `vivaldi` | The first two parts | `1:1.84.10-1` back to `1:1.83.120-1` |
| `firefox`, `firefox-developer-edition`, `firefox-esr`, `librewolf`, `thunderbird` | Every part: any older build | `146.0.1-1` back to `146.0-1` |

The browsers don't migrate a database, but each refuses a profile that a newer version used: Firefox and Thunderbird any newer build, the Chromium-based ones a newer major version. The question names that, `firefox 146.0.1-1 -> 146.0-1 goes back to an older build, and firefox won't open a profile a newer build used`.

Downgrades within a major version, such as `17.2-1` to `17.1-3`, pass without a question. For anything else, mark the versions yourself with [`unsafe_downgrades`](/docs/reference/options/#unsafe_downgrades): a downgrade from at or above a listed version to below it asks too, `lemon 3.54.0-1 -> 3.53.4-1 crosses 3.54, which unsafe_downgrades marks`. Each generation stores its `unsafe_downgrades`, so a revert checks both the generation it leaves and the one it goes back to, every version either lists for a package, without evaluating your config.

Answer `y` only once the data is taken care of, such as restored from a dump the older version can read. `--noconfirm` refuses unsafe downgrades: ``refusing unsafe downgrades with --noconfirm; to go ahead, run `archix switch` or `archix revert` with --allow-unsafe-downgrade``. `--allow-unsafe-downgrade` still lists them, but doesn't ask. A [switch](/docs/reference/cli/#switch) asks the same way, for example after `archix update --to` an earlier date. `install`, `remove` and `upgrade` don't take the flag.

## Revert and keep

archix has two ways back, and they replace different things:

| | `archix revert [G]` | `archix rollback N` (keep) |
| --- | --- | --- |
| Goes back to | Generation G's record | Snapshot N, the whole root as it was |
| When | Now, live, in place | From the next boot |
| Puts back | What G declares: packages, files, users, services, hooks, Flatpak apps | Everything on `@`: declared or not, `/etc` edits, untracked packages, `/usr` and `/opt` |
| Leaves as it is | Everything G doesn't declare, `/var/lib` and home | `/var/lib` (but package-owned files there) and home; accounts made since are carried over |
| Packages from | archix's package cache and the Arch Linux Archive | The snapshot itself; nothing is downloaded |
| Needs | Any filesystem | btrfs with rollbacks on |
| Undo | Revert again, or switch | `archix rollback --cancel` before the reboot; afterwards, keep the snapshot of kind replaced |

Revert is the one to reach for after a config change you regret. Keep is the one for a system that a package update broke in ways the config doesn't describe. See [Keeping a snapshot](/docs/concepts/snapshots/#keeping-a-snapshot). Picking a snapshot in the boot menu, or a fallback after a failed boot, boots it in try mode and changes nothing until you keep it; see [Boot and fallback](/docs/concepts/boot-and-fallback/).

## The package cache

A revert needs the exact package files of the generation it goes to. Official packages can come from the Arch Linux Archive, but CachyOS builds, AUR builds and other packages the Archive doesn't serve can't be fetched again. So every switch keeps those in archix's own package cache, `/.archix/packages` on a root laid out for rollbacks, else `/var/cache/archix/pkg`, outside `paccache`'s reach. It also keeps every package of the newest two generations, the official ones included, so a revert to yesterday works offline.

Files are taken from pacman's cache as reflinks where the filesystem can, else hard links, else copies, so they rarely cost space twice. `archix status` shows the size:

```console
package cache: 812 MiB on disk in /.archix/packages, for reverts and rollbacks (reflinks and hard links share blocks with pacman's cache)
```

A file goes when no kept generation locks it any more. On a root short of space, official packages of older generations, which the Archive can serve again, go before archix warns; see [Retention](/docs/concepts/snapshots/#retention). A revert that needs a package that's gone, or can't fetch one, refuses before anything is applied. On CachyOS, whose repos keep only the last few builds, reverts reach back only as far as archix has been installed.

## Limits and `gc`

Generation records cost their package files in the cache. archix cleans up after itself:

| What | Kept |
| --- | --- |
| Generation records | The current one, the newest [`generations.keep`](/docs/reference/options/#generationskeep) (10), and every one younger than `generations.keep_days` (30) |
| Package cache | Files a kept generation locks that the Archive doesn't serve, and every file of the newest two |
| Managed-file contents (downloads included), snapshot databases | Only what kept generation records need, plus snapshot databases fetched in the last day, which an `archix lock` running at the same time may be using |
| When | `gc` runs after every switch and revert |

Snapshots have limits of their own; see [Retention](/docs/concepts/snapshots/#retention). `archix init` asks about the generation limits. To clean up by hand, for example after lowering one:

```console
$ sudo archix gc
deleted generation records 30, 31
freed 812.4 MiB: 46 cached package files, 3 snapshot databases
```

A generation whose record has been collected can't be reverted to, because its package files may be gone. If you want to keep one longer, raise the limits before it ages out.
