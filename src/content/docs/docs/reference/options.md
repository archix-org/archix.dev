---
title: Options reference
description: The core archix options, packages, repos, programs and files, and how to list every option available on your system.
sidebar:
  order: 2
---

This page documents every option archix ships: packages and repositories, the AUR, third-party repos, generations and rollbacks, the system (files, users, services and more), the environment, the firewall, activation hooks, booting, AppImages, Flatpak apps, your home directory and the `programs.*` modules. Your installed archix is still the authoritative list, because `archix options` is generated from the declarations themselves, and it includes options your own modules declare.

All of these modules are built in and loaded before your `init.lua`. You never import them.

## Listing every option

```console
$ archix options
```

Each entry shows the option's path, type, default, description and the module that declared it:

```
repos
    type: list of one of "core", "extra", "multilib", "core-testing", "extra-testing", "multilib-testing"
    default: ["core", "extra"]
    Official repositories packages are resolved from, in priority order.
    declared in <archix>/packages.lua
```

Options declared by your own modules appear in the same list, with your file in `declared in`. In paths, `<name>` stands for any key of an `attrs_of` option, and `*` for any element of a list.

For tooling, `--json` prints an array of objects:

```console
$ archix options --json | jq '.[] | select(.path == "packages")'
{
  "path": "packages",
  "type": "list of str",
  "default": [],
  "description": "Packages to install: from the configured repositories, else from the AUR. Their dependencies are installed too and recorded in archix.lock.",
  "declared_in": "<archix>/packages.lua"
}
```

Options with no default, such as a `custom_repos` entry's `name`, have `"default": null` in JSON and no `default:` line in the text output. Give such an option a value whenever you set its parent; this page marks them *required*. A module that reads one while it's unset fails with `has no value and no default`.

Types on this page are written the way `archix options` prints them. `null or str` is `types.nullable(types.str)`; set it to `archix.null` to clear it. See [Types](/docs/reference/lua-api/#types).

## Packages and repositories

### `packages`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/packages.lua` |

Packages to install: from the configured repositories, else from the AUR. Their dependencies are installed too and recorded in `archix.lock`.

List only what you want; dependencies are resolved and locked for you, and marked as dependencies in pacman's database. Names are checked offline against the sync databases and the AUR metadata, so a misspelled name fails before anything is downloaded. `archix install` adds names here through your host's `installed.lua`, a plain Lua file you can also edit by hand.

AUR package names go in the same list. archix looks each name up in your configured repos first, third-party repos included, and only then in the AUR. See [AUR packages](/docs/guides/aur/).

So do packages built from the config's own recipes. A recipe in `pkgbuilds/<dir>/` needs no option: declare a package it builds, by its exact name, and that recipe wins over the repos and the AUR. A recipe that only `provides` a name doesn't beat the repos for it. Dependencies resolve from the repos first, then `pkgbuilds/`, then the AUR. A name found nowhere is an error: `` `foo` is in none of the configured repos, pkgbuilds/ or the AUR ``. See [Local PKGBUILDs and patches](/docs/guides/local-pkgbuilds-and-patches/).

Like every list, `packages` concatenates across modules in definition order. Duplicates are harmless. Program modules add their own packages here, so `programs.kitty.enable = true` is enough to install kitty.

```lua
return {
  packages = { "git", "neovim", "ripgrep", "zen-browser-bin" },
}
```

To replace the list a shared module built up instead of adding to it, force it:

```lua
packages = archix.force { "base", "linux", "linux-firmware" },
```

### `prefer`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/packages.lua` |

Packages to use when a dependency could be met by several, such as which font satisfies `ttf-font`. They're installed as dependencies, not declared.

Every preferred package is locked and installed, marked as a dependency, and it wins over the provider pacman would pick and over the real package of the same name: `prefer = { "rustup" }` meets `rust` with rustup. See [The lockfile](/docs/concepts/lockfile/#official-repos-arch-linux-archive-snapshots).

```lua
prefer = { "rustup", "noto-fonts" },
```

### `patches`

| | |
| --- | --- |
| Type | `attrs of list of str` |
| Default | `{}` |
| Declared in | `<archix>/packages.lua` |

Patch files for official packages, keyed by package name and relative to the config directory. archix rebuilds Arch's own recipe for the locked version with them, applied with `patch -p1` before the recipe's `prepare()`, and adds a sub-release to the pkgrel. Recipes in `pkgbuilds/<dir>/` need no option: declaring a package they build is enough.

archix clones Arch's recipe from gitlab.archlinux.org at the tag for the locked version, and applies each patch, in list order, in whichever top-level directory of `$srcdir` it fits. The patched build's pkgrel is a sub-release, `2` becoming `2.1`, so it sorts after Arch's. Other packages of the same pkgbase in the lock are swapped for the patched build too. It's locked with `repo = "local"` and `base = "patch:<pkgbase>"`, and `archix update` rebuilds it for each new version.

The package has to be in the host's lock, declared or as a dependency, and in Arch's official repos. A package your lock takes from a third-party repo, such as CachyOS's `hyprland`, is replaced by the patched build of Arch's recipe at Arch's version at the locked snapshot; an official or pinned package keeps its version. Otherwise `diff`, `lock` and `switch` stop with `patches.<name>: <name> isn't among this config's packages or their dependencies`, or `patches.<name>: <name> isn't in Arch's official repos, and only Arch's recipes can be patched`. A path can't leave the config directory. A patch that applies nowhere fails the build before the system changes.

```lua title="lua/hosts/desktop/init.lua"
return {
  packages = { "hyprland" },
  patches = {
    hyprland = { "patches/hyprland/0001-fix-cursor-warp.patch" },
  },
}
```

### `repos`

| | |
| --- | --- |
| Type | `list of one of "core", "extra", "multilib", "core-testing", "extra-testing", "multilib-testing"` |
| Default | `["core", "extra"]` |
| Declared in | `<archix>/packages.lua` |

Official repositories packages are resolved from, in priority order.

All of them are locked against the Arch Linux Archive snapshot in `archix.lock`. Add `multilib` for Steam and Wine. The testing repos are for testing; packages there are locked like any other but change often.

The default only applies when no module sets `repos`. As soon as one does, the default is dropped, so always list every repo you want. `repos = { "multilib" }` on its own would leave you without `core` and `extra`:

```lua title="lua/hosts/desktop/init.lua"
return {
  repos = { "core", "extra", "multilib" },
}
```

If several modules set `repos`, their lists concatenate in definition order, and that order is the priority order. Duplicate entries are ignored, and an empty list is an error: packages need at least one repository.

Third-party repos such as CachyOS's are declared with [`custom_repos`](#custom_repos).

### `archive_url`

| | |
| --- | --- |
| Type | `str` |
| Default | `"https://archive.archlinux.org"` |
| Declared in | `<archix>/packages.lua` |

The Arch Linux Archive, or a mirror of it with the same layout (`repos/YYYY/MM/DD/...`), that locks resolve and download from.

Point it at a mirror you run, or a caching proxy in front of the archive, when the archive is slow or far away. Every snapshot database and official package comes from there, including `/packages/` for pins. `switch` records the URL, so `revert`, which doesn't evaluate the config, uses the same one. When it's down or lacks a file, the [`archive_mirrors`](#archive_mirrors) are tried in order.

```lua
archive_url = "https://ala-mirror.example.org",
```

### `archive_mirrors`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/packages.lua` |

Copies of the archive with its layout, tried in order when archive_url is down or lacks a file, for locking, downloads and reverts.

A file is looked for on `archive_url` first, then at the same path on each mirror. Once `archive_url` fails to answer, as opposed to not having a file, the rest of the run goes straight to the mirrors, so a dead archive costs one round of retries, not one per package: `warning: downloading https://ala-mirror.example.org/repos/2026/10/10/core/os/x86_64/core.db: ...; trying https://archive.archlinux.org/repos/2026/10/10/core/os/x86_64/core.db`. The resolver's pacman configuration lists every mirror as a `Server` after the archive. `switch` records the list, so `revert` uses it too. Every file is still checked against the lock's sha256, wherever it comes from.

```lua
archive_url = "https://ala-mirror.example.org",
archive_mirrors = { "https://archive.archlinux.org" },
```

### `package_caches`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/packages.lua` |

Package caches tried before the archive: directories (`file:///mnt/nas/archix`) or HTTP servers holding what `archix cache push` copied. Repo packages and AUR builds are checked against the lock's sha256; an AUR build the lock has no sha256 for yet is taken as the cache's build record says, so list only caches you control.

Caches are tried in order, before the archive and the third-party repos' servers, for every package file a lock, switch or revert needs. An AUR package, or a build of one of the config's recipes, is reused from a cache that has a build record for the same recipe commit and snapshot, instead of being built again, as long as it's the build `archix.lock` records; another build is passed over: `warning: the package cache at file:///mnt/nas/archix has a build of yay that doesn't match archix.lock; building it here instead`. A cache that fails is skipped with a warning. `switch` records the list, so `revert` uses it too. Fill a cache with [`archix cache push`](/docs/reference/cli/#cache-push), or have every switch fill it with [`package_cache_push`](#package_cache_push); see [Package caches](/docs/concepts/lockfile/#package-caches).

```lua
package_caches = { "file:///mnt/nas/archix" },
```

### `package_cache_push`

| | |
| --- | --- |
| Type | `null or str` |
| Default | `null` |
| Declared in | `<archix>/packages.lua` |

A package cache directory (`file:///mnt/nas/archix`) every switch copies this machine's new package files, signatures and AUR build records into, as `archix cache push` does, so other machines get its exact third-party versions and AUR builds. A failed copy only warns.

After every switch that makes a generation, archix copies whatever the directory doesn't already hold and says `copied 3 files to /mnt/nas/archix`. A cache that's unmounted or read-only doesn't fail the switch, which is already applied: `warning: couldn't copy this machine's packages to the package cache at file:///mnt/nas/archix: ...`. List the same directory in [`package_caches`](#package_caches) on the machines that read from it.

```lua
package_caches = { "file:///mnt/nas/archix" },
package_cache_push = "file:///mnt/nas/archix",
```

### `lock_warn_after_days`

| | |
| --- | --- |
| Type | `int` |
| Default | `30` |
| Declared in | `<archix>/packages.lua` |

Warn when `archix.lock`'s snapshot is older than this many days, since fixes released after it aren't installed; `0` never warns.

`archix status` and `archix switch` print the warning, ``warning: archix.lock is at the 2026-09-10 snapshot, 38 days old; updates released since aren't installed. `archix update` moves it to the newest snapshot``, and go on as usual. Raise it for a machine you keep on an old snapshot on purpose.

```lua
lock_warn_after_days = 90,
```

### `undeclared_packages`

| | |
| --- | --- |
| Type | `one of "remove", "warn"` |
| Default | `"remove"` |
| Declared in | `<archix>/packages.lua` |

What switch does with installed packages that aren't in the lock: remove them, or only list them.

With `"remove"`, a switch removes every installed package that isn't in the lock, so the system is exactly what the config says, except packages [`ignored_packages`](#ignored_packages) lists and the ones [`drift`](#drift) keeps untracked. Explicit packages that plain pacman or an AUR helper installed since the last switch go through `drift` first, so with the default `drift = "ask"` the switch asks before anything is removed. With `"warn"`, archix keeps undeclared packages and lists them in `archix diff` and `archix switch`.

`"warn"` is a good setting while you move an existing system into your config. Switch back to the default once nothing is listed any more. See [pacman and yay](/docs/concepts/pacman-and-yay/).

```lua title="lua/hosts/desktop/init.lua"
return {
  undeclared_packages = "warn",
}
```

### `ignored_packages`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/packages.lua` |

Installed packages archix leaves alone and never reports as undeclared.

Use it for packages you deliberately leave as they are, such as one you built by hand before archix took over; switches leave them as they are. A package you build yourself can instead go in [`pkgbuilds/`](/docs/guides/local-pkgbuilds-and-patches/), so it's locked and rolled back like the rest. An ignored package is never removed, whatever `undeclared_packages` says, and never shows up in `diff`. Like `packages`, the list concatenates across modules. `archix init` puts foreign packages that aren't on the AUR here.

```lua title="lua/modules/common.lua"
return {
  ignored_packages = { "nvidia-dkms-custom" },
}
```

Ignoring only affects packages that aren't in the lock. A package you also declare in `packages` is managed as usual.

### `drift`

| | |
| --- | --- |
| Type | `one of "ask", "add", "keep"` |
| Default | `"ask"` |
| Declared in | `<archix>/packages.lua` |

What the next switch does with packages installed explicitly with pacman or an AUR helper since the last one: ask whether to add them to the host's `installed.lua` (with `--noconfirm`, add), add them without asking, or keep them installed but untracked, so no switch removes them or what they depend on. `installed.lua` is a generated data file the config imports; archix writes it as the config directory's owner, never through a symlink.

pacman, yay, paru and pamac keep working on an archix machine, and archix notices what they installed at the next switch: `installed with pacman or an AUR helper since the last switch, not declared: htop ncdu`. With `"ask"`, it then asks `add them to installed.lua? (no keeps them installed, untracked)`; with `"add"`, it says `adding them to installed.lua, as drift = "add" says`; with `"keep"`, or answering no, `keeping them installed and untracked: no switch removes them or what they need`. Untracked packages are listed by `archix status` and left out of reproducing the system on another machine. `archix init` asks which one you want. See [Drift](/docs/concepts/pacman-and-yay/#drift).

```lua
drift = "keep",
```

### `strict`

| | |
| --- | --- |
| Type | `bool` |
| Default | `false` |
| Declared in | `<archix>/packages.lua` |

Strict mode, a guardrail rather than enforcement: plain pacman and the front ends on libalpm (AUR helpers, pamac) refuse every transaction archix doesn't run, through a pacman hook running a small shell script. It lets everything through again once strict is turned off and switched, archix is removed (`pacman -R archix` works while it's on), or archix's binary, its state or this config is gone, so it never wedges pacman. archix's own transactions are recognised by an environment variable anyone can set.

A refused transaction stops in pacman's pre-transaction hooks, before anything changes: ``archix: strict mode is on, so packages change only through archix (`archix install`, `archix remove`, `archix upgrade`); set strict = false and switch to turn it off, or remove archix with pacman -R archix``. It takes effect at the switch that sets it. See [Strict mode](/docs/concepts/pacman-and-yay/#strict-mode).

```lua
strict = true,
```

### `holds`

| | |
| --- | --- |
| Type | `attrs of any` |
| Default | `{}` |
| Declared in | `<archix>/packages.lua` |

Packages kept back while the rest of the lock moves with the snapshot, keyed by package name: an exact version (`"1.1.2-1"`, or `"1.1.2"` for its newest pkgrel), which an official package is fetched at from the Arch Linux Archive and an AUR package is built at from the recipe commit that builds it; a floor (`">= 0.15.4"`), which never pins but refuses a lock below it; or `{ commit = "..." }`, an AUR package's recipe commit. `archix lock` and `archix update` keep them, and refuse, naming both packages, a hold that a dependency (a soname included) of the new snapshot rules out. A hold on a package the lock doesn't have is an error.

```lua
holds = { mesa = "25.2.4-1", ["yt-dlp"] = ">= 2026.09.01" },
```

### `unsafe_downgrades`

| | |
| --- | --- |
| Type | `attrs of list of str` |
| Default | `{}` |
| Declared in | `<archix>/packages.lua` |

Versions per package that a downgrade mustn't cross without asking, on top of archix's own list (postgresql and mariadb major versions and a few more). A switch or revert going back across one asks, defaulting to no; `--noconfirm` refuses unless `--allow-unsafe-downgrade` is given.

A downgrade from at or above a listed version to below it counts, for any package. Each generation stores the option, so a revert checks it without evaluating the config. archix's own list covers `postgresql`, `forgejo`, `nextcloud`, `redis` and `valkey` by their first version part, and `mariadb`, `mysql`, `percona-server`, `gitea` and `mattermost` by their first two. See [Downgrades that may break data](/docs/concepts/generations/#downgrades-that-may-break-data).

```lua
unsafe_downgrades = { lemon = { "3.54" }, ["matrix-synapse"] = { "1.100", "2.0" } },
```

With that, going from `3.54.0-1` back to `3.53.4-1` asks: `lemon 3.54.0-1 -> 3.53.4-1 crosses 3.54, which unsafe_downgrades marks`.

## AUR

See [AUR packages](/docs/guides/aur/) for how AUR packages are locked, reviewed and built.

### `aur.url`

| | |
| --- | --- |
| Type | `str` |
| Default | `"https://aur.archlinux.org"` |
| Declared in | `<archix>/aur.lua` |

The AUR to look packages up in and clone build recipes from.

### `aur.review`

| | |
| --- | --- |
| Type | `bool` |
| Default | `true` |
| Declared in | `<archix>/aur.lua` |

Show each AUR build recipe entering the lock (the PKGBUILD, or a diff) and ask before trusting it.

A new package shows every file its recipe ships, PKGBUILD first, install scripts and patches included; an update shows the git diff, `.SRCINFO` included. A commit already in the lock is never shown again. `--noconfirm` skips the review for one run; `aur.review = false` skips it always.

### `aur.build_on_host`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/aur.lua` |

AUR packages built directly on the host with makepkg instead of in a clean chroot.

They build with makepkg as the build user on the root itself: the build dependencies that are missing are installed from the locked snapshot for the build, then removed again. Use it for packages that don't build in a chroot.

```lua
aur = { build_on_host = { "some-driver-dkms" } },
```

### `aur.require_locked_build`

| | |
| --- | --- |
| Type | `bool` |
| Default | `false` |
| Declared in | `<archix>/aur.lua` |

Install only the AUR and recipe builds archix.lock records: one that no package cache has is refused instead of built here, since a build here differs from it byte for byte.

The lock records the sha256 of the first build of each recipe commit, on whichever machine made it. Builds aren't bit-for-bit reproducible, so another machine building the same pinned sources gets the same code in a different file, and by default installs it and says so: `built zen-browser-bin here, which differs from the build archix.lock records (another machine's, or an earlier one); its sources are pinned, so it's the same code. aur.require_locked_build refuses that instead`. With this set, it takes the recorded build from a [package cache](#package_caches) or stops before changing anything: `archix.lock records a build of zen-browser-bin that no package cache has, and aur.require_locked_build doesn't allow building it here; nothing was changed`. A recipe nobody has built yet still builds.

```lua
aur = { require_locked_build = true },
```

### `aur.skip_check`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/aur.lua` |

AUR packages (by pkgbase) built without running their test suite (`makepkg --nocheck`), for recipes whose `check()` fails against the locked snapshot.

It applies in the chroot and with `build_on_host` alike. Use it when the package builds but its own tests don't pass against the snapshot's toolchain, rather than skipping checks for one run. See [`skip_check`](/docs/guides/aur/#skip_check).

```lua
aur = { skip_check = { "helm-ls" } },
```

### `missing_versions`

| | |
| --- | --- |
| Type | `one of "ask", "newer", "older"` |
| Default | `"ask"` |
| Declared in | `<archix>/aur.lua` |

When a locked AUR version can't be built any more: ask (defaulting to the nearest newer version), or always take the nearest newer or older one.

A version can't be built when its commit is gone from the package's history or its build fails. Whichever version is taken, the lock follows. `switch --accept-nearest newer|older` decides for one run.

## Third-party repositories

See [Third-party repos](/docs/guides/third-party-repos/) for a full walkthrough.

### `custom_repos`

| | |
| --- | --- |
| Type | `list of submodule` |
| Default | `[]` |
| Declared in | `<archix>/packages.lua` |

Third-party repositories. They have no dated archive, so archix keeps a copy of every package it installs from them.

Each entry has these fields:

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `name` | `str` | *required* | Repository name, as in `pacman.conf`. |
| `servers` | `list of str` | *required* | Server URLs, tried in order; `$repo` and `$arch` are expanded like pacman does. `$arch` becomes `x86_64`, so CachyOS's `$arch_v3` and `$arch_v4` become `x86_64_v3` and `x86_64_v4`. |
| `keys` | `list of str` | `[]` | Fingerprints of the repository's signing keys; archix imports and locally signs them. |
| `keyserver` | `str` | `"hkps://keyserver.ubuntu.com"` | Where the `keys` are fetched from. |
| `sig_level` | `str` | `"Required DatabaseOptional"` | The repository's `SigLevel` in `pacman.conf`. |
| `before_official` | `bool` | `false` | List the repository before core and extra, so its packages win (CachyOS's repos need this). |

A name that clashes with an official repo or another custom repo is an error. When resolving, archix accepts packages for `x86_64` and every x86-64 level the CPU supports, so CachyOS's `x86_64_v3` and `x86_64_v4` repos resolve, and a CPU without v4 never gets v4 packages. A switch on an older CPU refuses a lock resolved on a machine with a newer level, and `archix lock` on that machine resolves it again for its own CPU; see [CPU levels](/docs/guides/third-party-repos/#cpu-levels). archix writes a block for each repo into `/etc/pacman.conf`, so plain pacman sees the same repos, and removes the block when the repo leaves the config.

```lua
custom_repos = {
  {
    name = "chaotic-aur",
    servers = { "https://cdn-mirror.chaotic.cx/$repo/$arch" },
    keys = { "3056513887B78AEB" },
  },
},
```

### `third_party_fallback`

| | |
| --- | --- |
| Type | `one of "ask", "fallback", "fail"` |
| Default | `"ask"` |
| Declared in | `<archix>/packages.lua` |

When a locked package from a custom repo is gone: ask (with --noconfirm, fail), fall back without asking, or fail. Falling back tries the repo's current version, then the AUR, then the official repos.

The lock follows whatever replaces the package, and an AUR replacement goes through review. With `"ask"`, `--noconfirm` fails the switch instead, so an unattended run never swaps a package or rewrites the lock: `linux-cachyos 6.17.2-2 is gone from cachyos-core-v3; nothing was changed. Run without --noconfirm to choose a replacement, or set third_party_fallback = "fallback"`. See [The fallback chain](/docs/guides/third-party-repos/#the-fallback-chain).

## Generations

### `generations.keep`

| | |
| --- | --- |
| Type | `int` |
| Default | `10` |
| Declared in | `<archix>/generations.lua` |

Number of newest generation records gc always keeps; `archix revert` applies a kept record live, in place, its packages from archix's cache and the archive.

### `generations.keep_days`

| | |
| --- | --- |
| Type | `int` |
| Default | `30` |
| Declared in | `<archix>/generations.lua` |

gc also keeps the record of every generation younger than this many days.

`archix gc` runs after every switch. Besides these, it keeps the record of the current generation, then deletes cached packages, file contents and snapshot databases no kept record or snapshot needs. See [Generations](/docs/concepts/generations/#limits-and-gc).

```lua
generations = { keep = 5, keep_days = 14 },
```

## Rollbacks

archix's own read-only btrfs snapshots of the root, taken before each switch and each plain pacman transaction, which the boot menu can boot and [`archix rollback N`](/docs/reference/cli/#rollback) can keep. They need the root on a btrfs subvolume; on ext4 there are no rollbacks, and everything else works as usual. See [Snapshots](/docs/concepts/snapshots/) and [Boot and fallback](/docs/concepts/boot-and-fallback/).

### `rollback.enable`

| | |
| --- | --- |
| Type | `bool` |
| Default | `false` |
| Declared in | `<archix>/rollback.lua` |

Take a read-only btrfs snapshot of the root before each switch and each plain pacman transaction, to roll back to or boot. Needs the root on a btrfs subvolume and `archix snapshot setup`; on ext4 there are no rollbacks, and everything else works as usual.

`archix init` asks, and on btrfs writes `rollback = { enable = true }` by default. The first switch with it lays the root out for rollbacks, `@archix` at `/.archix` and `@varlib` at `/var/lib`, as [`archix snapshot setup`](/docs/reference/cli/#snapshot-setup) does, and snapshots start after the next reboot. On ext4 a switch warns and goes on without snapshots; on a root that is btrfs's top level it refuses until the root has moved into `@` (see [A root at the top level](/docs/concepts/snapshots/#a-root-at-the-top-level)).

```lua
rollback = { enable = true },
```

### `rollback.keep_switch`

| | |
| --- | --- |
| Type | `int` |
| Default | `10` |
| Declared in | `<archix>/rollback.lua` |

How many snapshots taken by switches to keep, newest first; ones with a boot entry are never dropped.

### `rollback.keep_pacman`

| | |
| --- | --- |
| Type | `int` |
| Default | `10` |
| Declared in | `<archix>/rollback.lua` |

How many snapshots taken before plain pacman transactions to keep, newest first; ones with a boot entry are never dropped.

Snapshots taken by hand with `archix snapshot create` are never pruned. Retention runs after every switch and plain pacman transaction. See [Retention](/docs/concepts/snapshots/#retention).

### `rollback.min_free`

| | |
| --- | --- |
| Type | `int` |
| Default | `10` |
| Declared in | `<archix>/rollback.lua` |

Below this percentage of free space on the root, archix warns and drops the oldest snapshots it may, never manual ones, ones with a boot entry or the newest.

Besides the percentage, archix keeps 1 GiB free for a keep, which needs room for a writable clone of the snapshot and its metadata. When no snapshot may go and the root is still short, it drops cached packages the archive can serve again, `dropped 512 MiB of cached packages the archive can serve again`, then warns.

### `rollback.low_space`

| | |
| --- | --- |
| Type | `one of "abort", "prune", "skip"` |
| Default | `"prune"` |
| Declared in | `<archix>/rollback.lua` |

What the snapshot before a plain pacman transaction does when the transaction and its snapshot would leave the root below min_free and nobody can be asked (no terminal, --noconfirm, pamac): abort the transaction; prune the oldest snapshots it may, but only while a safe recovery point stays (a snapshot of a system that booted fine, with its boot entry), else abort; or skip the snapshot and let the transaction run. On a terminal it asks instead: abort, prune, or still try.

The estimate is the transaction's full install size, as the snapshot keeps every file the transaction replaces, plus a margin. On a terminal, pacman's output shows the warning and asks `archix: [a]bort, [p]rune old snapshots, or [s]till try (snapshot and install anyway)? [a] `; "still try" isn't offered when the transaction would eat into the 1 GiB kept for a keep. An abort stops pacman before anything changes: ``archix: aborted the transaction: the root is short of space for it and its snapshot; nothing was changed. Free space, delete snapshots with `archix snapshot delete N`, or set rollback.low_space``. What was decided is logged to the journal and shown by `archix status`. See [Short of space](/docs/concepts/snapshots/#short-of-space).

```lua
rollback = { enable = true, low_space = "abort" },
```

### `rollback.entries`

| | |
| --- | --- |
| Type | `int` |
| Default | `5` |
| Declared in | `<archix>/rollback.lua` |

How many of the newest snapshots get a boot entry (snapshots taken by switches, and by pacman transactions that changed the kernel or initramfs), besides the fallback slot: the first snapshot since the last boot that counted as good.

A snapshot with an entry is never pruned. Before writing one, archix checks the ESP has room for its boot image, shared with every other snapshot whose kernel and initramfs are the same: `the ESP at /efi has 40 MiB free (0 MiB more once older snapshots' entries go), but a new boot image needs about 52 MiB; free up space on it`. See [Boot entries](/docs/concepts/boot-and-fallback/#boot-entries).

## System

Every setting in this section renders to files archix manages. Their contents are stored with each generation, so `archix revert` restores them along with the packages, and a snapshot holds them with the rest of the root. A switch writes them on the running system, after the package transaction and before it starts or restarts services. A file archix replaces is backed up and comes back when it leaves the config; a managed file edited by hand is saved before it's overwritten, and `diff` flags it beforehand. See [Safety](/docs/concepts/safety/).

### `files`

| | |
| --- | --- |
| Type | `attrs of submodule` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

Files archix writes, keyed by absolute path; give one of `text`, `source`, `link`, `url` or `merge`. A switch writes them in place, after the package transaction. Files archix replaces are backed up and restored when they leave the config.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `text` | `null or lines` | `null` | The file's contents. A final newline is added if missing. |
| `source` | `null or str` | `null` | A file in the config directory to copy, as a path relative to it. |
| `link` | `null or str` | `null` | Make the file a symlink to this target. |
| `url` | `null or str` | `null` | Download the file from here; needs `sha256`. [`archix.fetchurl`](/docs/reference/lua-api/#archixfetchurl) builds both. |
| `sha256` | `null or str` | `null` | The sha256 the download from `url` must have; [`archix prefetch <url>`](/docs/reference/cli/#prefetch) prints it. |
| `mode` | `str` | `"0644"` | Octal permissions. |
| `owner` | `str` | `"root"` | |
| `group` | `str` | `"root"` | |
| `on_change` | `null or lines` | `null` | A bash script run as root when a switch or revert changed this file, near the end of that switch, before the [hooks](#hooks). |
| `secret` | `null or str` | `null` | Make the file a link to this secret ([`secrets.items`](#secretsitems)), which is decrypted into `/run/archix/secrets` with the secret's owner and mode at every boot and every switch; the root holds only the link. |
| `mutable` | `null or bool` | `null` | Write the file read-only (`false`) or writable (`true`); by default as [`mutable_files`](#mutable_files) says. |
| `merge` | `null or submodule` | `null` | Set only these keys in a file the app also writes, keeping the rest of it, like plasma-manager. Leaving the config leaves the keys as they are. `format` is one of `"ini"`, `"kconfig"` or `"json"`, *required*; `values` (`attrs of any`, default `{}`) holds the keys: sections (kconfig groups nest) of keys, or a JSON object deep-merged into the file. |

Give exactly one of `text`, `source`, `link`, `url`, `merge` and `secret`; otherwise it's an error, `files.<path>: give exactly one of text, source, link, url, merge or secret`. `text` is a `lines` value, so several modules can each add lines to the same file. A `source` path can't leave the config directory.

`on_change` runs when the switch or revert changed the file: when its entry differs from the one in the generation the system is leaving (the first switch that manages it, new contents, mode, owner, group or link target, a revert to a generation where it was different), or when the file on disk didn't match and archix wrote it again, such as a hand-edited file put back or a deleted one recreated. It doesn't run when only the script changes, or when the file leaves the config. It runs on the running system once the switch has written every file and started and restarted its services, before the hooks. Make the script idempotent. A failing script is a soft failure, listed as `failed: on_change of <path>: ...`. See [Activation hooks](/docs/guides/hooks/#on_change).

A `url` needs a `sha256` of 64 hex digits; without one, `diff` and `switch` stop with `files.<path>: url needs sha256`. The hash is what's locked: `diff` compares the file on disk against it without downloading anything, and `switch` downloads the file after you confirm, before it changes anything. A download whose hash differs stops the switch: `<url> has sha256 <actual>, but the config says <sha256>`. The contents go into the same store as every other managed file, so a revert restores them even after the URL is gone.

With `mutable = false`, or [`mutable_files = false`](#mutable_files), the file is written without its write bits, so `"0644"` becomes `0444` and `"0755"` becomes `0555`. Links aren't affected.

`merge` sets keys in a file archix doesn't own, such as one an app writes too: archix edits only the declared keys and leaves the app's own. The app's keys never show up in `archix diff`; a declared key that changed does, and the next switch sets it back. JSON files may be JSONC, with comments and trailing commas, but archix drops the comments when it writes the file back; a file that can't take its keys stops the switch before anything changes. A new file gets `0644` and `owner`/`group`; an existing one keeps its mode and owner. When the entry leaves the config, the file keeps its keys and is neither restored nor deleted. See [Setting single keys](/docs/guides/dotfiles-and-apps/#setting-single-keys).

archix doesn't write under `/usr` (except `/usr/local`), `/proc`, `/sys`, `/dev`, `/run`, `/tmp`, `/home` or `/boot`: `files: <path> is in a directory archix doesn't write to; home files go in home.files, and boot entries in boot.extra_entries`. Files in a home directory go in [`home.files`](#homefiles), and systemd-boot entries in [`boot.extra_entries`](#bootextra_entries).

```lua
files = {
  ["/etc/motd"] = { text = "Managed by archix." },
  ["/etc/sudoers.d/wheel"] = { text = "%wheel ALL=(ALL:ALL) ALL", mode = "0440" },
  ["/etc/X11/xorg.conf.d/30-touchpad.conf"] = { source = "files/30-touchpad.conf" },
  ["/etc/resolv.conf"] = { link = "/run/systemd/resolve/stub-resolv.conf" },
  ["/etc/dconf/db/local.d/00-archix"] = { text = "[org/gnome/desktop/interface]\ncolor-scheme='prefer-dark'", on_change = "dconf update" },
  ["/etc/xdg/kdeglobals"] = { merge = { format = "kconfig", values = { General = { ColorScheme = "BreezeDark" } } } },
  ["/usr/local/bin/hello"] = archix.fetchurl {
    url = "https://downloads.example.org/hello/1.2.0/hello-x86_64",
    sha256 = "5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03",
    mode = "0755",
  },
},
```

### `mutable_files`

| | |
| --- | --- |
| Type | `bool` |
| Default | `true` |
| Declared in | `<archix>/system.lua` |

Whether files archix writes stay writable. false writes them read-only like the Nix store, restores them on every switch, and marks merged kconfig keys [$i]; each file's mutable overrides it.

With `false`, files from `text`, `source` and `url` in [`files`](#files) and [`home.files`](#homefiles) lose their write bits, `0644` becoming `0444`. A file's own `mutable` wins either way. Links and live links aren't affected: a live link points into your config repo, which stays writable. Merged ini and JSON files stay writable for their app, since those keys can't be locked; merged kconfig keys get KDE's lock marker, `ColorScheme[$i]=BreezeDark`, so KDE won't change them. It's called `mutable_files` because `files` is already the option for system files. See [Read-only files](/docs/guides/dotfiles-and-apps/#read-only-files).

```lua
mutable_files = false,
```

### `units`

| | |
| --- | --- |
| Type | `attrs of lines` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

systemd system units, keyed by file name such as `foo.service`. They're written to `/etc/systemd/system`. Declaring a unit doesn't enable it; list it in [`services`](#services) too.

```lua
units = {
  ["backup.service"] = [[
[Unit]
Description=Nightly backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup
]],
},
```

### `user_units`

| | |
| --- | --- |
| Type | `attrs of lines` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

systemd user units for every user, keyed by file name. They're written to `/etc/systemd/user`. For units in one user's home, use [`home.systemd`](#homesystemd).

### `services`

| | |
| --- | --- |
| Type | `attrs of bool` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

System units to enable (`true`) or disable (`false`).

A switch disables units before it writes the files, enables them after their unit files are in place, then runs `systemctl daemon-reload`, stops the units it disabled and starts the ones it enabled. A unit that can't be enabled or started doesn't undo the switch: the rest stays applied, the failure is listed, and archix exits non-zero. Which running units are restarted when their files change is up to [`restart_if_changed`](#restart_if_changed).

```lua
services = { ["sshd.service"] = true, ["bluetooth.service"] = true, ["cups.service"] = false },
```

### `user_services`

| | |
| --- | --- |
| Type | `attrs of bool` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

User units to enable (`true`) or disable (`false`) for every user, with `systemctl --global`, from each user's next login.

```lua
user_services = { ["pipewire-pulse.socket"] = true },
```

### `users`

| | |
| --- | --- |
| Type | `attrs of submodule` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

Users archix creates and keeps in step; ones it created are removed (home kept) when they leave the config. Passwords come from `hashed_password` or `initial_hashed_password`, else set them with `passwd`.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `uid` | `null or int` | `null` | Picked by `useradd` when null. |
| `group` | `null or str` | `null` | Primary group; by default one named after the user. |
| `groups` | `list of str` | `[]` | Supplementary groups. |
| `shell` | `path` | `"/bin/bash"` | |
| `home` | `null or path` | `null` | The home directory; `/home/<name>` when null. |
| `description` | `str` | `""` | The GECOS field. |
| `system` | `bool` | `false` | A system user: low uid, no home directory. |
| `hashed_password` | `null or str` | `null` | The password as a crypt(3) hash (`mkpasswd` prints a yescrypt one), or `"!"` to lock the account; set at every switch, so a password changed with `passwd` goes back to it, and `archix diff` says so. A locked or expired account stays so, and a password set in the rescue entry holds until this hash changes. Anyone who can read the config can try to crack it offline. |
| `initial_hashed_password` | `null or str` | `null` | A crypt(3) hash set only when archix creates the account; `passwd` changes it for good after that. |
| `ssh_authorized_keys` | `list of str` | `[]` | SSH public keys that may log in as the user, one `.pub` line each, written to `/etc/ssh/authorized_keys.d/<user>`; `~/.ssh/authorized_keys` keeps working beside them. Declaring any installs openssh; enable `sshd.service` in [`services`](#services). |

Declaring a user that already exists brings it in line with every field, defaults included: a declared user is authoritative, so `groups` is the complete list of supplementary groups, and `shell` falls back to `/bin/bash`. When you declare an existing user by hand, list all of its groups and its shell. `archix init` captures existing users with their real groups, shell, home and description, so declaring them changes nothing. Users that archix didn't create are never deleted. A switch creates users and groups right after the package transaction, then their home directories. A [rollback](/docs/reference/cli/#rollback) carries accounts made since the snapshot over by name, and brings back accounts deleted since as locked placeholders; see [Accounts after a keep](/docs/concepts/snapshots/#accounts-after-a-keep). One archix created and you declare again later without a `uid` gets back the uid it had, so the files it owned are still its own. Groups work the same way.

```lua
users = {
  me = { groups = { "wheel", "video" }, shell = "/usr/bin/zsh", description = "Me" },
},
```

### `groups`

| | |
| --- | --- |
| Type | `attrs of submodule` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

Groups archix creates and keeps in step.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `gid` | `null or int` | `null` | Picked by `groupadd` when null. |
| `system` | `bool` | `false` | A system group. |

```lua
groups = { media = {}, builders = { gid = 1500 } },
```

### `allow_no_password_login`

| | |
| --- | --- |
| Type | `bool` |
| Default | `false` |
| Declared in | `<archix>/system.lua` |

Let a switch apply a system in which neither root nor a wheel or sudo member has a password, which a switch otherwise refuses when the running system has one, as the [rescue entry](#bootrescue_entry) needs one to open.

### `capabilities`

| | |
| --- | --- |
| Type | `attrs of str` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

File capabilities, keyed by the path of a file a package installs (or the config declares) in `/usr` or `/opt`, in setcap's text form, such as `cap_sys_nice+ep`. A switch sets them after everything that writes or chowns the files, packages included, so an upgrade that replaces the file gets them back; a path that isn't there is refused. Dropping one gives the file back what its package or install script left it.

```lua
capabilities = { ["/usr/bin/gamescope"] = "cap_sys_nice+ep" },
```

### `masked_units`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/system.lua` |

System units to mask, such as `sleep.target`, so nothing can start them: each becomes a link to `/dev/null` in `/etc/systemd/system`. `archix init` captures the units masked there.

```lua
masked_units = { "sleep.target", "suspend.target", "hibernate.target" },
```

### `default_target`

| | |
| --- | --- |
| Type | `null or str` |
| Default | `null` |
| Declared in | `<archix>/system.lua` |

The target the system boots into, such as `multi-user.target` for a machine without a desktop; by default systemd's, `graphical.target`. The [health check](#boothealth_check) waits for it.

### `restart_if_changed`

| | |
| --- | --- |
| Type | `attrs of bool` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

System or user units to always (`true`) or never (`false`) restart when a switch replaces files they use. By default archix restarts them, except the display manager, dbus, logind, NetworkManager, sshd, gettys and the desktop session's own units, which it lists for a logout instead.

archix restarts a running unit when a process of it still maps or holds open a file under `/usr` that the switch's package transaction replaced, or when a package it needs at runtime changed its interpreted code, never the unit archix itself runs in, nor units that already restarted since the switch began. `false` also keeps a unit whose own unit file was rewritten from being try-restarted.

```lua
restart_if_changed = { ["postgresql.service"] = false, ["sshd.service"] = true },
```

### `tmpfiles`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/system.lua` |

tmpfiles.d lines, such as `"d /var/lib/foo 0750 foo foo -"`, written to `/etc/tmpfiles.d/archix.conf`. A switch checks them with `systemd-tmpfiles --dry-run --create` before it changes anything, and a line it refuses fails the switch with nothing changed; systemd applies them at boot. Lines touching archix's own paths are refused.

### `hostname`

| | |
| --- | --- |
| Type | `null or str` |
| Default | `null` |
| Declared in | `<archix>/system.lua` |

Written to `/etc/hostname`. Null leaves the file alone.

### `timezone`

| | |
| --- | --- |
| Type | `null or str` |
| Default | `null` |
| Declared in | `<archix>/system.lua` |

A zoneinfo name such as `Europe/Istanbul`. archix points `/etc/localtime` at it.

### `locales`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/system.lua` |

Locales to generate, such as `en_US.UTF-8`; the first becomes `LANG`.

archix writes `/etc/locale.gen` (the charset is taken from the name, `ISO-8859-1` without one) and `/etc/locale.conf`.

```lua
locales = { "en_US.UTF-8", "tr_TR.UTF-8" },
```

### `keymap`

| | |
| --- | --- |
| Type | `null or str` |
| Default | `null` |
| Declared in | `<archix>/system.lua` |

The console keymap, written to `/etc/vconsole.conf`.

### `sysctl`

| | |
| --- | --- |
| Type | `attrs of any` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

Kernel settings, written to `/etc/sysctl.d/90-archix.conf` and applied at the next boot. Booleans become `1` and `0`.

```lua
sysctl = { ["vm.swappiness"] = 10, ["net.ipv4.ip_forward"] = true },
```

### `kernel_modules`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/system.lua` |

Kernel modules to load at boot, written to `/etc/modules-load.d/archix.conf`.

### `modprobe`

| | |
| --- | --- |
| Type | `attrs of lines` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

modprobe.d snippets, keyed by file name without `.conf`.

```lua
modprobe = { nvidia = "options nvidia_drm modeset=1" },
```

### `udev_rules`

| | |
| --- | --- |
| Type | `attrs of lines` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

udev rules, keyed by file name without `.rules`. They're written to `/etc/udev/rules.d`.

```lua
udev_rules = {
  ["60-ioschedulers"] = [[ACTION=="add|change", KERNEL=="nvme[0-9]n[0-9]", ATTR{queue/scheduler}="none"]],
},
```

## sudo

### `sudo.enable`

| | |
| --- | --- |
| Type | `bool` |
| Default | `false` |
| Declared in | `<archix>/sudo.lua` |

Install sudo and write `/etc/sudoers.d/archix` (mode `0440`), checked with `visudo -c` before it's written; a rule visudo refuses fails the switch with nothing changed.

### `sudo.wheel`

| | |
| --- | --- |
| Type | `bool` |
| Default | `true` |
| Declared in | `<archix>/sudo.lua` |

Let members of wheel run any command as any user.

### `sudo.wheel_needs_password`

| | |
| --- | --- |
| Type | `bool` |
| Default | `true` |
| Declared in | `<archix>/sudo.lua` |

Whether wheel members type their password (`false` gives them `NOPASSWD`).

### `sudo.extra_rules`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/sudo.lua` |

More sudoers lines, such as `"alice ALL=(ALL) NOPASSWD: /usr/bin/systemctl"`.

```lua
sudo = { enable = true, extra_rules = { "alice ALL=(ALL) NOPASSWD: /usr/bin/systemctl" } },
```

## Filesystems and swap

archix writes `/etc/fstab` whole: its own lines for the root and the ESP, and with rollbacks the lines for `@archix`, `@varlib` and the binds from `/var/rollback`, then these. A switch writes it in place; new mounts and swap apply at the next boot. Since a snapshot holds the root, `/etc/fstab` included, keeping a snapshot brings back the fstab it had.

### `file_systems`

| | |
| --- | --- |
| Type | `attrs of submodule` |
| Default | `{}` |
| Declared in | `<archix>/file_systems.lua` |

Filesystems to mount at boot, keyed by mount point.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `device` | `str` | *required* | What to mount: `"UUID=…"`, `"LABEL=…"`, `"PARTUUID=…"`, a `/dev` path, the directory to bind with `options = { "bind" }`, `"tmpfs"`, or a network share such as `"server:/export"`. |
| `fs_type` | `str` | `"auto"` | The filesystem type, such as `ext4`, `btrfs`, `vfat`, `tmpfs` or `nfs`; a bind mount's is written as `none`. |
| `options` | `list of str` | `[]` | Mount options, such as `{ "noatime", "subvol=@data" }` or `{ "bind" }`; none means `defaults`. |
| `check` | `bool` | `false` | fsck the filesystem at boot before mounting it (`fs_passno` 2). |
| `required` | `null or bool` | `null` | Make the boot wait for it and stop in emergency mode without it. By default mounts at `/home`, `/var` and `/srv` and under them are required, as a boot without one would show an empty directory on the root filesystem that programs fill with fresh files, hidden once the mount is back; elsewhere a mount is `nofail`, its device waited for 30 seconds (`x-systemd.device-timeout=30s`, unless `options` set one), so an unplugged or dead disk leaves it unmounted instead of stopping every boot. `false` makes a mount under `/home`, `/var` or `/srv` `nofail` too. |
| `encrypted` | `null or submodule` | `null` | Unlock `device` at boot through an `/etc/crypttab` line archix writes, named after the `/dev/mapper/<name>` this mount's device must be. Fields: `device` (`str`, *required*), the LUKS device, such as `"UUID=…"` of the encrypted partition; `key_file` (`null or str`), a key file to unlock it with, else systemd asks for the passphrase at boot; `options` (`list of str`), crypttab options, such as `{ "discard" }`. |

Mount points that a snapshot of the root has to hold whole, or that archix mounts itself, can't be declared: `/`, `/usr` (but `/usr/local`), `/opt`, `/boot`, `/etc`, `/.archix`, `/var`, `/var/lib`, `/var/lib/pacman`, `/var/lib/dkms` and the ESP, with the reason, such as `/etc can't be declared: /etc stays on the root filesystem, so a snapshot of the root holds it whole`. A required mount that doesn't come up stops the boot in emergency mode with a note naming it; on a counted boot, [`boot.unblessed_timeout`](#bootunblessed_timeout) then reboots it so the fallback starts, and the note at the next login says whether a disk was missing rather than the system broken.

```lua
file_systems = {
  ["/srv/media"] = { device = "UUID=5c1f0e2a-...", fs_type = "ext4", options = { "noatime" } },
  ["/home"] = { device = "/dev/mapper/home", fs_type = "ext4", encrypted = { device = "UUID=9d3b...", options = { "discard" } } },
},
```

### `swap`

| | |
| --- | --- |
| Type | `list of submodule` |
| Default | `[]` |
| Declared in | `<archix>/file_systems.lua` |

Swap partitions and files, written into `/etc/fstab`; they apply at the next boot.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `device` | `null or str` | `null` | A swap partition, such as `"UUID=…"` or a `/dev` path. |
| `file` | `null or path` | `null` | A swap file, such as `/swap/swapfile`. |
| `size` | `null or str` | `null` | For a file: its size, such as `"8G"` or `"512M"`; `archix-swapfile` creates or resizes it at boot before swapon. On btrfs it's made with `btrfs filesystem mkswapfile` in a subvolume of its own, which archix creates when the file's directory is missing, as btrfs can't snapshot a subvolume holding an active swap file. Without a size the file must exist. |
| `priority` | `null or int` | `null` | The swap priority (`pri=`). |
| `options` | `list of str` | `[]` | More swap options, such as `{ "discard" }`. |
| `required` | `bool` | `false` | Make the boot wait for it; by default it's `nofail`. |

A swap file can't be in a tree a snapshot of the root holds whole, `/usr`, `/opt`, `/boot` or `/etc`, nor in `/.archix`, pacman's database, DKMS's state, `/tmp` or the ESP.

```lua
swap = { { file = "/swap/swapfile", size = "8G" } },
```

### `zram`

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `zram.enable` | `bool` | `false` | Swap on compressed RAM at `/dev/zram0` through `zram-generator`, which archix declares; written to `/etc/systemd/zram-generator.conf` and set up at the next boot. |
| `zram.size` | `any` | `null` | `zram-size`: an expression in `ram` (MiB), such as `"ram / 2"` or `"min(ram / 2, 4096)"`, or a number of MiB; by default zram-generator's, `min(ram / 2, 4096)`. |
| `zram.algorithm` | `null or str` | `null` | `compression-algorithm`, such as `zstd` or `lz4`. |
| `zram.priority` | `null or int` | `null` | `swap-priority`; zram-generator's default is 100. |

Declared in `<archix>/file_systems.lua`.

```lua
zram = { enable = true, size = "ram / 2", algorithm = "zstd" },
```

### `drop_mounts`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/file_systems.lua` |

Mount points, and `/dev/mapper/<name>` for `/etc/crypttab` volumes, whose lines a switch may leave out of `/etc/fstab` and `/etc/crypttab` although nothing declares them. Without this a switch refuses to drop a line that's mounted or open now, or for `/home`, `/var` or `/srv` and under them, naming each, so a hand-added line can't vanish with its data. Take an entry out once the system without it has booted.

## Environment

See [Environment and firewall](/docs/guides/environment-and-firewall/) for a walkthrough.

### `environment.variables`

| | |
| --- | --- |
| Type | `attrs of any` |
| Default | `{}` |
| Declared in | `<archix>/environment.lua` |

Environment variables for every user: login shells get them from `/etc/profile.d/archix-environment.sh`, and systemd user services and graphical sessions from `/etc/environment.d/90-archix.conf`, from the next login on. Lists are joined with `:`.

The shell file has `export NAME="value"` lines and `environment.d` has `NAME=value` lines, and both expand `$VAR` and `${VAR}`; numbers and other values are written as text.

```lua
environment = {
  variables = { EDITOR = "nvim", XDG_DATA_DIRS = { "/usr/local/share", "/usr/share" } },
},
```

### `environment.shell_aliases`

| | |
| --- | --- |
| Type | `attrs of str` |
| Default | `{}` |
| Declared in | `<archix>/environment.lua` |

Shell aliases for every user's login shells, from `/etc/profile.d/archix-aliases.sh`, from the next login on; [`programs.zsh.shell_aliases`](#programszsh) covers every interactive zsh.

Each one is written as `alias ll='ls -l'`.

```lua
environment = { shell_aliases = { ll = "ls -l" } },
```

## Firewall

An nftables firewall in its own table. See [Environment and firewall](/docs/guides/environment-and-firewall/#firewall) for a walkthrough and the rules it writes.

### `firewall.enable`

| | |
| --- | --- |
| Type | `bool` |
| Default | `false` |
| Declared in | `<archix>/firewall.lua` |

An nftables firewall in its own `inet archix` table, loaded by `archix-firewall.service`; a change to the rules applies at the next boot. Incoming traffic is dropped unless it answers outgoing traffic or something below allows it; other tables (Docker's, Tailscale's) are left alone.

With `true`, archix declares the `nftables` package, writes the rules to `/etc/nftables-archix.conf`, and writes and enables `archix-firewall.service`, which loads them before the network starts. Turning it on starts the service at that switch. Later rule changes take effect at the next boot, or at once with `sudo systemctl restart archix-firewall.service`; turning it off disables the service, so the next boot has no `archix` table. archix doesn't use `nftables.service`, whose stop flushes every table.

The input chain has policy drop and accepts, in order: established and related traffic, loopback, `trusted_interfaces`, all ICMPv6, DHCPv6 replies from `fe80::/10`, ping when `allow_ping` is on, the allowed ports and ranges, then `extra_input_rules`.

```lua
firewall = {
  enable = true,
  allowed_tcp_ports = { 22, 80, 443 },
  trusted_interfaces = { "tailscale0" },
},
```

### `firewall.allowed_tcp_ports`

| | |
| --- | --- |
| Type | `list of int` |
| Default | `[]` |
| Declared in | `<archix>/firewall.lua` |

TCP ports open to everyone.

Ports must be 1 to 65535: `firewall ports must be 1-65535, not 70000`.

### `firewall.allowed_udp_ports`

| | |
| --- | --- |
| Type | `list of int` |
| Default | `[]` |
| Declared in | `<archix>/firewall.lua` |

UDP ports open to everyone.

### `firewall.allowed_tcp_port_ranges`

| | |
| --- | --- |
| Type | `list of submodule` |
| Default | `[]` |
| Declared in | `<archix>/firewall.lua` |

TCP port ranges open to everyone, as `{ from, to }`.

Both `from` and `to` (`int`, *required*) are 1 to 65535, with `from` no greater than `to`: otherwise `firewall port ranges need from <= to, both 1-65535`.

```lua
firewall = { allowed_tcp_port_ranges = { { from = 8000, to = 8010 } } },
```

### `firewall.allowed_udp_port_ranges`

| | |
| --- | --- |
| Type | `list of submodule` |
| Default | `[]` |
| Declared in | `<archix>/firewall.lua` |

UDP port ranges open to everyone, as `{ from, to }`.

```lua
firewall = { allowed_udp_port_ranges = { { from = 60000, to = 61000 } } },
```

### `firewall.allowed_services`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/firewall.lua` |

Services whose ports, from `firewall.service_ports`, are open to everyone, such as `{ "ssh", "syncthing" }`; each service's rules carry its name as their nftables comment. Naming a service `service_ports` doesn't define is an error.

```lua
firewall = { enable = true, allowed_services = { "ssh", "kdeconnect" } },
```

### `firewall.service_ports`

| | |
| --- | --- |
| Type | `attrs of submodule` |
| Default | the built-in services below |
| Declared in | `<archix>/firewall.lua` |

The ports of each service `firewall.allowed_services` can name. archix defines alvr, gsconnect, home-assistant, http, https, jellyfin, kdeconnect, localsend, mdns, minecraft, mosh, nfs, plex, samba, ssh, steam, steam-dedicated-server, sunshine, syncthing, tailscale, transmission, wireguard and wivrn with `archix.default`, so a plain definition of one of these names replaces it; other names add services. Built-ins meant for the local network (all but http, https, minecraft, mosh, ssh, steam-dedicated-server, tailscale, transmission and wireguard) are open only to LAN sources.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `tcp` | `list of int` | `[]` | TCP ports the service listens on. |
| `udp` | `list of int` | `[]` | UDP ports the service listens on. |
| `tcp_ranges` | `list of submodule` | `[]` | TCP port ranges, as `{ from, to }`. |
| `udp_ranges` | `list of submodule` | `[]` | UDP port ranges, as `{ from, to }`. |
| `sources` | `one of "any", "lan"` | `"any"` | Who may reach the ports: any address, or (`lan`) only private and link-local ones: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`, `fc00::/7` and `fe80::/10`. A public wifi's other guests are on a private network too. |
| `source` | `null or str` | `null` | Where the ports come from, such as the upstream docs. |

```lua
firewall = {
  enable = true,
  allowed_services = { "myapp" },
  service_ports = { myapp = { tcp = { 8443 }, sources = "lan" } },
},
```

### `firewall.trusted_interfaces`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/firewall.lua` |

Interfaces whose incoming traffic is always accepted, such as a VPN's.

### `firewall.allow_ping`

| | |
| --- | --- |
| Type | `bool` |
| Default | `true` |
| Declared in | `<archix>/firewall.lua` |

Answer ICMP echo requests. ICMPv6, which IPv6 needs to work, is always accepted.

### `firewall.check_reverse_path`

| | |
| --- | --- |
| Type | `one of "strict", "loose", "off"` |
| Default | `"strict"` |
| Declared in | `<archix>/firewall.lua` |

Drop packets whose source address wouldn't be routed back out the interface they came in on (strict), out of any interface (loose), or don't check (off). VPNs that route asymmetrically may need loose.

The check runs in an `rpfilter` chain that lets DHCPv4 through. `"off"` leaves that chain out.

### `firewall.extra_input_rules`

| | |
| --- | --- |
| Type | `lines` |
| Default | `""` |
| Declared in | `<archix>/firewall.lua` |

nftables rules added to the end of the input chain, such as `ip saddr 192.168.1.0/24 tcp dport 8080 accept`.

### `firewall.extra_rules`

| | |
| --- | --- |
| Type | `lines` |
| Default | `""` |
| Declared in | `<archix>/firewall.lua` |

More nftables chains or sets inside the archix table.

Once `nftables` is installed, a switch checks the whole ruleset with `nft -c -f` before changing anything, and a mistake stops it: `the firewall rules don't load (check extra_input_rules and extra_rules); nothing was changed:` and nft's error. On the first switch, before `nft` is installed, a mistake shows only when the service fails to load the file at boot.

## Hooks

Scripts archix runs at the end of every switch and revert, on the running system. See [Activation hooks](/docs/guides/hooks/) for a walkthrough.

### `hooks`

| | |
| --- | --- |
| Type | `attrs of submodule` |
| Default | `{}` |
| Declared in | `<archix>/hooks.lua` |

Scripts run at the end of every switch and revert, like NixOS activation scripts: once the packages, users, files and services are applied and the `on_change` scripts have run. A switch with nothing to do runs them too. They run once per switch, so make them idempotent; a failing hook is a soft failure that's listed, and doesn't undo the switch.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `run` | `lines` | *required* | The bash script to run; it runs with `set -e` and `pipefail`. |
| `user` | `null or str` | `null` | Run as this user, in their home; root by default. While they're logged in, the hook gets their session bus. |
| `after` | `list of str` | `[]` | Hooks that must run before this one. |

The preview shows them on a `run hooks` line, in order: by `after`, then by name, and the switch prints `:: running hooks` and each hook's name as it runs. An `after` that names no hook is an error, `hooks.<name>.after names "<hook>", which isn't a hook`, and so is a cycle, which names the hooks in it, such as `hooks b -> c -> b wait on each other in a cycle`, or `hook a lists itself in after`.

Each script runs as `bash -e -o pipefail -c` in a clean environment: `PATH=/usr/local/sbin:/usr/local/bin:/usr/bin`, `HOME`, `USER` and `LOGNAME`, and `LANG` if archix has one. Root hooks run in `/`. With `user`, the hook runs as that user without opening a login session, in their home (or `/` if it doesn't exist), with `XDG_RUNTIME_DIR` and `DBUS_SESSION_BUS_ADDRESS` set while they're logged in.

A failing hook is a soft failure: nothing is undone, the rest of the hooks still run, and the switch lists it, `failed: hook <name>: exited with exit status: 1`, and exits non-zero. Fix the hook and switch again; with nothing else to do, that switch only runs the hooks.

```lua
users = { alice = { groups = { "wheel" } } },
hooks = {
  ["backup-dir"] = { run = "install -d -m 0750 -o alice /srv/backups" },
  ["flatpak-overrides"] = { run = "flatpak override --system --filesystem=xdg-download com.spotify.Client", after = { "backup-dir" } },
},
```

## Boot and kernel

With `boot.loader = "systemd-boot"`, archix writes an entry for the current system, `loader/entries/archix-current.conf` on the ESP, titled "Arch Linux (archix)", and one per snapshot worth booting, `archix-snapshot-<n>.conf`, titled after the state it holds, such as "archix snapshot 12, read-only: 2026-10-15, generation 41, before switch to tree 5d7b21c4e9f0". Each starts a unified kernel image in `EFI/archix/`, built with `ukify` from a kernel and a fresh `mkinitcpio` initramfs, and named by a hash of what it's built from, so snapshots with the same kernel and initramfs share one. Every other kernel package installed beside `boot.kernel`, such as `linux-lts`, gets an entry and image of its own, `archix-current-linux-lts.conf`. With systemd-boot 260 or newer, the current entry is counted, and a change that doesn't boot falls back by itself to the snapshot of how the system last booted fine. Picking a snapshot's entry boots it read-only in [try mode](/docs/concepts/boot-and-fallback/#try-mode). See [Boot and fallback](/docs/concepts/boot-and-fallback/).

GRUB gets the same entries in its own menu, picked by hand; Limine gets none. Without `boot.loader`, archix writes no boot entries and leaves booting to you; switches, reverts and drift tracking work as usual, and so do snapshots, kept from a running system.

With `boot.loader` set, archix declares the packages it needs for this itself: the `boot.kernel` package, `mkinitcpio`, `systemd-ukify` and `dosfstools`, whose `fsck.vfat` lets the ESP be checked at boot after a power cut, plus `sbctl` when `boot.secure_boot = true`. You don't list them in `packages`. With the default `secure_boot = null`, signing depends on sbctl being installed with keys, so keep `sbctl` declared yourself if you rely on that.

The current system can also get a protected entry with [`boot.protected_entry`](#bootprotected_entry). See [Secure Boot](/docs/guides/secure-boot/).

```lua
boot = { loader = "systemd-boot", microcode = { "amd-ucode.img" } },
kernel_params = { "root=UUID=0a1b2c3d-...", "rw", "quiet" },
```

### `kernel_params`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/system.lua` |

Kernel command line parameters, written to `/etc/kernel/cmdline` for boot entries.

This is the whole command line of every boot entry, so it must contain `root=`; with `boot.loader` set, archix refuses to switch without it: `boot.loader is set, but kernel_params has no root=; archix needs the full kernel command line`. The snapshot an entry boots isn't on the command line; archix's initramfs reads it from the entry systemd-boot started. Counted images also get `panic=10`, unless the command line sets `panic=` itself, so a kernel panic, as when a broken glibc kills init, reboots into the next try instead of hanging.

### `boot.loader`

| | |
| --- | --- |
| Type | `null or one of "systemd-boot", "grub", "limine"` |
| Default | `null` |
| Declared in | `<archix>/boot.lua` |

The boot loader archix writes entries for. systemd-boot 260 or newer gets an entry for the current system and for snapshots, with boot counting and automatic fallback; GRUB gets the same entries in its menu, picked by hand; on Limine archix writes no entries and leaves limine.conf to limine-snapper-sync, so a fallback is a manual pick there.

- **`"systemd-boot"`** gives the full set: entries, boot counting, the [fallback slot](/docs/concepts/boot-and-fallback/#the-fallback-slot), the health check, and the [unattended reboot](#bootunblessed_timeout). On a machine Limine boots, setting it starts the [move to systemd-boot](/docs/concepts/boot-and-fallback/#moving-from-limine-to-systemd-boot): systemd-boot goes on the ESP with a firmware entry after Limine's, and only once a boot through it counts as good does it go first. `archix init` offers this on Limine machines.
- **`"grub"`** writes archix's menu to `/boot/grub/archix.cfg`, which `/etc/grub.d/41_archix` makes `grub.cfg` read at boot, so a switch never runs `grub-mkconfig` again after the first. Each entry chainloads the same images systemd-boot's would. GRUB counts no boots, so nothing falls back by itself: you pick a snapshot by hand.
- **`"limine"`** writes no Limine entries. Limine boots the kernel and initramfs on the ESP, which limine-snapper-sync manages, so a fallback goes through its snapper entries, and a [keep](/docs/reference/cli/#rollback) refuses a snapshot whose kernel or initramfs differs from the current one. After a limine-snapper-restore, [`archix resync`](/docs/reference/cli/#resync) adopts the restored root.

[What falls back by itself](/docs/concepts/boot-and-fallback/#what-falls-back-by-itself) compares them.

### `boot.esp`

| | |
| --- | --- |
| Type | `path` |
| Default | `"/efi"` |
| Declared in | `<archix>/boot.lua` |

Where the EFI system partition is mounted.

archix keeps its boot images in `EFI/archix/` and its entries in `loader/entries/` there. It writes the ESP's line in `/etc/fstab` so that a power cut can't leave it broken or readable by everyone:

- It's mounted on access and unmounted after a minute idle (`x-systemd.automount,x-systemd.idle-timeout=60`), so a forced reboot rarely catches it mounted.
- It's checked at boot (`fs_passno` 2), so `fsck.fat` repairs a cluster chain a power cut left half freed. Without the check, the kernel remounts the ESP read-only at the first broken chain.
- It's readable by root only (`fmask=0077,dmask=0077`), unless the line already sets `umask`, `fmask` or `dmask`. systemd-boot's random seed lives on the ESP.
- An ESP with no line gets one, `UUID=… /efi vfat umask=0077,x-systemd.automount,x-systemd.idle-timeout=60 0 2`, as `systemd-gpt-auto-generator` would otherwise mount it at `/boot`.

When the config manages `/etc/fstab`, archix leaves it alone and the switch warns instead, for each of these it finds missing.

### `boot.kernel`

| | |
| --- | --- |
| Type | `str` |
| Default | `"linux"` |
| Declared in | `<archix>/boot.lua` |

The kernel package boot entries use, such as `linux-lts`, `linux-zen` or `linux-cachyos`. archix declares it for you, and `archix init` sets it to the package of the kernel you're running.

This kernel's entry is the main one, the one boot counting and the protected entry use. Any other kernel package you declare gets its own entry too, titled "Arch Linux (archix, linux-lts)". When the lock has `dkms`, it also gets the `-headers` package of every `linux*` kernel in it that the repos have headers for, so DKMS builds its modules for each kernel. A snapshot's entry boots the snapshot's own kernel and modules, so it still boots after the kernel package was upgraded, and a switch or transaction that replaces the running kernel keeps the running one's modules loadable until you reboot.

### `boot.microcode`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/boot.lua` |

Microcode images in /boot loaded before the initramfs, such as amd-ucode.img.

Paths are relative to `/boot`, where the `amd-ucode` and `intel-ucode` packages put them. When mkinitcpio's `microcode` hook already puts it in the initramfs, `archix init` leaves this empty rather than loading it twice.

### `boot.tries`

| | |
| --- | --- |
| Type | `int` |
| Default | `2` |
| Declared in | `<archix>/boot.lua` |

How many boots systemd-boot tries the current system's entry after each change before it falls back, first to the snapshot of how the system last booted fine, booted read-only in try mode; a boot that reaches boot-complete.target and passes the health check counts as good. Needs systemd-boot 260 or newer on the ESP; 0 turns boot counting off.

Every switch, revert and plain pacman transaction that changes something writes the current entry as `archix-current+2.conf` again; plain pacman does it before the transaction, so a power cut halfway still falls back. systemd-boot counts a try down at every boot of it, and once a boot reaches `boot-complete.target` and passes the [health check](#boothealth_check), `systemd-bless-boot` renames it `archix-current.conf` and it counts as good. After two boots that didn't, it's `archix-current+0-2.conf`, which systemd-boot passes over for the next entry in sort order: the [fallback slot](/docs/concepts/boot-and-fallback/#the-fallback-slot), then snapshots from newer to older, each with one try of its own. A fallback boots the snapshot read-only in [try mode](/docs/concepts/boot-and-fallback/#try-mode); [`archix rollback N`](/docs/reference/cli/#rollback) keeps it.

archix reads whether the running systemd-boot can count, 260 or newer, from its `LoaderFeatures` variable, and writes plain entries otherwise, which `switch` and `status` say: ``boot counting is off: the systemd-boot that started this boot is older than 260, so a generation that doesn't boot isn't passed over; `sudo bootctl update` installs the one your systemd package has``. GRUB and Limine don't count boots.

### `boot.unblessed_timeout`

| | |
| --- | --- |
| Type | `int` |
| Default | `10` |
| Declared in | `<archix>/boot.lua` |

Minutes a boot systemd-boot is still counting may take to count as good, or sit in emergency or rescue mode, before it reboots itself so the fallback can start with nobody at the machine; nobody may be logged in. 0 turns it off.

`archix-unblessed.timer`, pulled in early on counted boots, checks every minute; a drop-in for `emergency.target` and `rescue.target` brings it along when the boot lands there. A boot that hangs, never reaches `boot-complete.target`, sits at an emergency shell after a broken `fstab` line, or failed its health check, reboots once this long has passed and nobody is logged in, and the next try or the fallback starts: `archix: ...; rebooting so the boot counter can fall back`. Set it longer for a machine whose boot is slow on purpose, such as one waiting on a network share. See [How long a fallback takes](/docs/concepts/boot-and-fallback/#how-long-a-fallback-takes).

```lua
boot = { loader = "systemd-boot", unblessed_timeout = 20 },
```

### `boot.health_check`

| | |
| --- | --- |
| Type | options, below |
| Declared in | `<archix>/boot.lua` |

What a counted boot has to pass, besides reaching `boot-complete.target`, before it counts as good. `archix-health-check.service` runs it before `boot-complete.target`, on counted boots only. By default a boot counts as good once it reaches its default target, no unit failed that hadn't already failed at the last good boot, and the display manager, when one is enabled, is active. `archix init` runs the check once and shows what it would conclude on this machine now.

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `boot.health_check.default_target` | `bool` | `true` | The boot must reach its default target before it counts as good. |
| `boot.health_check.failed_units` | `bool` | `true` | No unit may have failed that hadn't already failed at the last boot that counted as good. |
| `boot.health_check.display_manager` | `bool` | `true` | When a display manager is enabled (`display-manager.service`), it must be active. |
| `boot.health_check.units` | `list of str` | `[]` | More systemd units that must be active, such as `"sshd.service"`, before a boot counts as good. A target that still fails after `boot.health_check.timeout` marks the entry bad, so the boot after falls back, and the login notes and `archix status` say which target failed and when. |
| `boot.health_check.listening` | `list of int` | `[]` | TCP ports something on this machine must listen on, such as 22 for sshd, before a boot counts as good. |
| `boot.health_check.reachable` | `list of str` | `[]` | Hosts that must answer before a boot counts as good: `"host:port"` must accept a TCP connection, a bare host must answer a ping, and `"gateway"` is the default IPv4 gateway. A host that's down for a reason outside this machine (the router, a captive portal, a laptop on a train) fails the check just the same and falls back from a good update, so list only what is always up when this machine boots; switching again once it answers brings the update back. |
| `boot.health_check.login` | `bool` | `false` | Each user archix declares must be able to log in: `su -s <shell> <user> -c true` must succeed. Without it, a boot still doesn't count as good while a declared user's login shell is missing or not in `/etc/shells`. |
| `boot.health_check.timeout` | `int` | `120` | Seconds the health check keeps retrying its targets after boot before it fails. |
| `boot.health_check.reboot_on_failure` | `bool` | `false` | Reboot as soon as the health check fails, so the fallback starts at once; otherwise `boot.unblessed_timeout` reboots it once nobody is logged in. |

The check retries every target every two seconds until `timeout`. With `reachable` set, it waits for `network-online.target`. Units that were already failing at the last good boot don't count; a switch or transaction that gives the current entry its tries notes the units failing on a good boot first, so a unit that's broken for good doesn't make every update fall back. A boot cut off in the middle of a plain pacman transaction fails the check by itself, so it falls back to the slot instead of counting a half-upgraded system as good. A target that still fails marks the entry bad, as `systemd-bless-boot` would:

```
archix: archix-current.conf failed its health check "TCP port 22 listening" (nothing listens on TCP port 22); its entry is marked bad, so the next boot falls back
```

The fallback says why at the next login and in `archix status`: `archix: the boot of archix-current.conf failed the health check "TCP port 22 listening" at 2026-10-21 10:03:05 UTC (nothing listens on TCP port 22); switch again once it passes`.

Setting anything here needs [`boot.tries`](#boottries) above 0, since the fallback is systemd-boot's boot counting: ``boot.health_check needs boot.tries > 0: a failed check falls back by systemd-boot's boot counting; nothing was changed``.

```lua
boot = {
  loader = "systemd-boot",
  health_check = { units = { "sshd.service" }, listening = { 22 }, reachable = { "gateway" }, reboot_on_failure = true },
},
```

### `boot.revoke_unused`

| | |
| --- | --- |
| Type | `bool` |
| Default | `false` |
| Declared in | `<archix>/boot.lua` |

Experimental, tested on OVMF only: after every switch, add the boot images archix signed and no entry uses any more to dbx, as `archix secure-boot revoke` does, so the firmware refuses them. Needs Secure Boot keys from sbctl as files, and declares efitools.

When a snapshot's entry goes, as retention prunes it, the boot images no other entry uses come off the ESP, and this puts their hashes in `dbx` right after, in the same switch, with every check [`archix secure-boot revoke`](/docs/reference/cli/#secure-boot-revoke) makes. If revoking fails, the switch is still done, and says why: `revoking boot images no entry uses: ...`.

```lua
boot = { loader = "systemd-boot", secure_boot = true, revoke_unused = true },
```

### `boot.secure_boot`

| | |
| --- | --- |
| Type | `null or bool` |
| Default | `null` |
| Declared in | `<archix>/boot.lua` |

Sign boot entries with sbctl; by default, whenever sbctl has keys.

`true` also declares the `sbctl` package. When signing, archix also signs systemd-boot's own binary. Changing this counts as a change, so the next switch builds and signs the boot images again. `sudo archix secure-boot setup` creates and enrolls the keys.

With Secure Boot enforcing, a switch whose boot image wouldn't be signed, because this is `false` or `sbctl` has no keys, is refused before anything changes. See [Secure Boot](/docs/guides/secure-boot/).

### `boot.extra_entries`

| | |
| --- | --- |
| Type | `attrs of lines` |
| Default | `{}` |
| Declared in | `<archix>/boot.lua` |

More systemd-boot entries archix writes to the ESP's loader/entries, keyed by file name, such as `["memtest86+.conf"] = "title Memtest86+\nefi /memtest86+/memtest.efi"`. With Secure Boot on, whatever they start must be signed too.

Each one becomes `<boot.esp>/loader/entries/<name>`, a managed file like any other, so it's removed when it leaves the config. Names must end in `.conf`, since systemd-boot reads only those: `boot.extra_entries: "memtest" must end in .conf, as systemd-boot reads only those`. What an entry starts has to be on the ESP, since systemd-boot reads nothing else. A package that installs an EFI image into `/boot`, as `memtest86+-efi` does, puts it on the root filesystem, so copy it to the ESP, `/efi/memtest86+/memtest.efi` here. archix doesn't sign what an entry starts; with Secure Boot on, sign it yourself, for example with `sbctl sign -s /efi/memtest86+/memtest.efi`.

```lua title="lua/hosts/desktop/init.lua"
return {
  packages = { "memtest86+-efi" },
  boot = {
    loader = "systemd-boot",
    extra_entries = {
      ["memtest86+.conf"] = "title Memtest86+\nefi /memtest86+/memtest.efi",
    },
  },
}
```

### `boot.protected_entry`

| | |
| --- | --- |
| Type | `bool` |
| Default | `false` |
| Declared in | `<archix>/boot.lua` |

Also write an 'archix protected' entry for the current system with the kernel command line Easy Anti-Cheat Linux v2 asks for (integrity lockdown, kexec off, the BPF LSM, IOMMU without passthrough). It can't make the kernel well-known-signed; `archix secure-boot check --anticheat` shows what's still missing.

The entry is `archix-current-protected.conf`, titled "Arch Linux (archix protected)", and signed like the rest. It adds `lockdown=integrity sysctl.kernel.kexec_load_disabled=1 lsm=landlock,lockdown,yama,integrity,bpf iommu.passthrough=0` to the command line. See [The protected entry](/docs/guides/secure-boot/#the-protected-entry).

### `boot.rescue_entry`

| | |
| --- | --- |
| Type | `bool` |
| Default | `true` |
| Declared in | `<archix>/boot.lua` |

Keep an 'archix rescue' entry in the boot menu, after the others and never the default: the current kernel with an initramfs of its own that mounts nothing of the layout and opens `archix rescue`, to check the root and the ESP, reset a password, open a shell or, with rollbacks on btrfs, roll back to a snapshot (`archix rollback N`), without a live USB. Refreshed after each switch, signed when Secure Boot signing is on. Before its prompt or a shell it asks for the password of root or of a wheel (or sudo) user from the root's /etc/shadow, three tries per boot, and on an encrypted root for the LUKS passphrase first, never unlocking with a TPM; with root locked and no wheel password it opens nothing. Under Secure Boot its kernel command line is part of the signed image and can't be edited; without Secure Boot anyone at the console can edit it, as they could boot a live USB.

The entry is `archix-rescue.conf`, with its image in `EFI/archix-rescue/`, never given a TPM policy, and never removed or revoked while in use. It doesn't depend on `@varlib` or the binds, so it opens even when they're what's broken. Whoever logs in can do anything, including resetting passwords. See [Recovering a broken system](/docs/guides/recovery/#without-a-live-usb) and [`archix rescue`](/docs/reference/cli/#rescue).

## Removed options

These options are gone. Setting one is a config error that names what replaced it, such as `option was removed (set in lua/hosts/desktop/init.lua): deployments are gone: a switch applies in place, and rollback.keep_switch sets how many switch snapshots stay`.

| Option | Why |
| --- | --- |
| `deployments.keep` | Deployments are gone: a switch applies in place, and [`rollback.keep_switch`](#rollbackkeep_switch) sets how many switch snapshots stay. |
| `boot.snapshots` | archix takes its own btrfs snapshots: [`rollback.enable`](#rollbackenable) turns them on and `rollback.keep_switch` sets how many switches keep one. |
| `boot.entries` | [`rollback.entries`](#rollbackentries) sets how many snapshots get a boot entry. |

## AppImages

See [Flatpak and AppImages](/docs/guides/flatpak-and-appimages/) for a walkthrough.

### `appimages`

| | |
| --- | --- |
| Type | `attrs of submodule` |
| Default | `{}` |
| Declared in | `<archix>/appimages.lua` |

AppImages to download, keyed by command name: installed to `/opt/archix/appimages`, linked as `/usr/bin/<name>` and added to app menus in `/usr/share/applications`, so a revert, and a snapshot of the root, take them back together; a name a package's `/usr/bin` file has is refused. Declares `fuse2`, which AppImages need.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `url` | `str` | *required* | Where to download the AppImage. |
| `sha256` | `str` | *required* | Its sha256; `archix prefetch <url>` prints it. |
| `name` | `null or str` | `null` | The name in app menus; the attribute name by default. |
| `icon` | `null or str` | `null` | An icon name or path for the menu entry. |
| `categories` | `list of str` | `[]` | Menu categories, such as `Office` or `Development`. |
| `desktop` | `bool` | `true` | Add an entry to app menus. |

Each entry becomes three managed files, so it's locked by its hash and reverts like any other [`files`](#files) entry:

| File | What it is |
| --- | --- |
| `/opt/archix/appimages/<name>.AppImage` | The download, mode `0755`. |
| `/usr/bin/<name>` | A link to it, so `<name>` runs from your `PATH`. |
| `/usr/share/applications/archix-<name>.desktop` | The menu entry, unless `desktop = false`. |

The key has to be usable as a command name: letters, digits, `.`, `_` and `-`. Anything else is an error, such as `appimages: my app isn't a valid command name`.

```lua
appimages = {
  obsidian = {
    url = "https://example.org/releases/Obsidian-1.9.14.AppImage",
    sha256 = "3b9f2c4e8a1d6f0b7c5e2a9d4f1b8c6e3a0d7f5b2c9e4a1d8f6b3c0e7a5d2f9b",
    name = "Obsidian",
    categories = { "Office" },
  },
},
```

## Flatpak

archix installs Flatpak apps system-wide and locks each one to a commit in [`archix.lock`](/docs/concepts/lockfile/#flatpak-apps-commits). Declaring any app also declares the `flatpak` package. See [Flatpak and AppImages](/docs/guides/flatpak-and-appimages/).

### `flatpak.apps`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/flatpak.lua` |

Flatpak app ids to install system-wide, such as `org.mozilla.firefox`; write `remote:id` for a remote other than `flatpak.remote`. Each is locked to a commit in `archix.lock` and installed in `/var/lib/flatpak` once the switch has applied the packages, so a switch that brings flatpak itself installs them too.

Apps that leave the list are uninstalled, but only ones archix managed in the current generation. Apps you installed by hand, and every `--user` installation, are left alone.

```lua
flatpak = {
  apps = { "org.mozilla.firefox", "com.spotify.Client", "myremote:org.example.App" },
},
```

### `flatpak.remote`

| | |
| --- | --- |
| Type | `str` |
| Default | `"flathub"` |
| Declared in | `<archix>/flatpak.lua` |

The remote apps come from unless they name one.

### `flatpak.remotes`

| | |
| --- | --- |
| Type | `attrs of submodule` |
| Default | `{"flathub":{"gpg_verify":true,"url":"https://dl.flathub.org/repo/flathub.flatpakrepo"}}` |
| Declared in | `<archix>/flatpak.lua` |

Remotes archix adds system-wide, keyed by name.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `url` | `str` | *required* | The remote's `.flatpakrepo` file or repository URL. |
| `gpg_verify` | `bool` | `true` | Check the remote's GPG signatures. `false` adds it with `--no-gpg-verify`. |

Flathub is always available, even when you set `remotes` and leave it out. An app naming any other remote that `remotes` doesn't declare is an error: `flatpak.apps: myremote:org.example.App uses remote myremote, which flatpak.remotes doesn't declare`. `switch` adds the remotes your apps use after you confirm, with `flatpak remote-add --system --if-not-exists`, so a remote that already exists under that name is left as it is. Remotes are never removed.

```lua
flatpak = {
  apps = { "org.mozilla.firefox", "myremote:org.example.App" },
  remotes = { myremote = { url = "https://flatpak.example.org/repo/example.flatpakrepo" } },
},
```

## Secrets

Secrets are age files in the config directory, encrypted to this machine's key and to your own, and decrypted into `/run/archix/secrets`, a ramfs that never reaches disk, by `archix-secrets.service` at every boot and by every switch. The config, the generation records and the snapshots only ever hold the encrypted files and links. See [`archix secrets`](/docs/reference/cli/#secrets).

### `secrets.items`

| | |
| --- | --- |
| Type | `attrs of submodule` |
| Default | `{}` |
| Declared in | `<archix>/secrets.lua` |

Secrets by name, decrypted into `/run/archix/secrets/<name>`; `files[path] = { secret = name }` links to one, `archix.secret(name)` puts one into a file's text, and units read `archix.secret_path(name)` through `EnvironmentFile=` or `LoadCredential=`.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `file` | `str` | *required* | The age file in the config directory, such as `secrets/wg0.age`; `archix secrets edit` writes it. |
| `kind` | `one of "file", "crypt_hash"` | `"file"` | `file`, or `crypt_hash` for a password hash that `users.<name>.hashed_password = archix.secret(name)` sets at each boot. |
| `owner` | `str` | `"root"` | |
| `group` | `str` | `"root"` | |
| `mode` | `str` | `"0400"` | |
| `path` | `null or str` | `null` | Also write the secret to this file under `/run` at each boot, for apps that refuse a link, such as a NetworkManager profile in `/run/NetworkManager/system-connections`. |

A secret that doesn't decrypt on this machine fails the switch before anything changes, unless `--without-secrets` is given; a value that turns up in plain text in `/etc` after the switch is reported.

```lua
secrets = {
  machine_key = "age1...",
  admins = { "age1..." },
  items = { wg0 = { file = "secrets/wg0.age", mode = "0400" } },
},
files = { ["/etc/wireguard/wg0.conf"] = { secret = "wg0" } },
```

### `secrets.machine_key`

| | |
| --- | --- |
| Type | `null or str` |
| Default | `null` |
| Declared in | `<archix>/secrets.lua` |

This machine's age recipient (`age1...`), which `sudo archix secrets init` makes and prints; every secret this host declares is encrypted to it.

### `secrets.admins`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/secrets.lua` |

Your own age, age plugin (such as age-plugin-yubikey) or ssh-ed25519 recipients, which every secret is encrypted to as well: the way back to them if a machine's key is lost. Needed once any secret is declared, unless `without_admin = true`.

### `secrets.without_admin`

| | |
| --- | --- |
| Type | `bool` |
| Default | `false` |
| Declared in | `<archix>/secrets.lua` |

Declare secrets with no admin recipient, knowing that losing this machine's key then loses every secret.

### `secrets.allow_hibernation`

| | |
| --- | --- |
| Type | `bool` |
| Default | `false` |
| Declared in | `<archix>/secrets.lua` |

Let the machine hibernate while secrets are declared and a swap isn't encrypted; hibernating writes all of memory, every decrypted secret with it, to that swap.

## Home

archix manages one user's home directory per machine. Set `home.user`, and declare the same user in [`users`](#users) so it exists before its files are written. Files in the home, and directories archix creates for them, are owned by that user.

```lua
users = { me = { groups = { "wheel" }, shell = "/usr/bin/zsh" } },
home = { user = "me" },
```

### `home.user`

| | |
| --- | --- |
| Type | `null or str` |
| Default | `null` |
| Declared in | `<archix>/home.lua` |

The user whose home archix manages: `home.files` and every `programs.*` module apply to them.

Setting `home.files`, or enabling a program, without `home.user` is an error.

### `home.directory`

| | |
| --- | --- |
| Type | `null or path` |
| Default | `null` |
| Declared in | `<archix>/home.lua` |

That user's home directory; defaults to `users.<user>.home`, else `/home/<user>`.

### `home.files`

| | |
| --- | --- |
| Type | `attrs of submodule` |
| Default | `{}` |
| Declared in | `<archix>/home.lua` |

Files in the user's home, keyed by path relative to it (a leading `~/` is allowed); written by every switch, as the user.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `text` | `null or lines` | `null` | The file's contents, written as given. |
| `source` | `null or str` | `null` | A file or directory in the config directory, as a path relative to it. |
| `live` | `bool` | `false` | Symlink to `source` in the config directory instead of copying it, so apps that write their own config edit your repo. |
| `link` | `null or str` | `null` | Make the file a symlink to this target. |
| `url` | `null or str` | `null` | Download the file from here; needs `sha256`. `archix.fetchurl { url, sha256 }` builds both. |
| `sha256` | `null or str` | `null` | The sha256 the download from `url` must have; `archix prefetch <url>` prints it. |
| `executable` | `bool` | `false` | Write the file with mode `0755` instead of `0644`. |
| `on_change` | `null or lines` | `null` | A bash script run as `home.user` right after a switch or revert writes a change to this file, so a reload signal reaches the running app. |
| `mutable` | `null or bool` | `null` | Write the file read-only (`false`) or writable (`true`); by default as [`mutable_files`](#mutable_files) says. |
| `merge` | `null or submodule` | `null` | Set only these keys in a file the app also writes, keeping the rest of it, like plasma-manager. Leaving the config leaves the keys as they are. `format` is one of `"ini"`, `"kconfig"` or `"json"`, *required*; `values` (`attrs of any`, default `{}`) holds the keys: sections (kconfig groups nest) of keys, or a JSON object deep-merged into the file. |

Give exactly one of `text`, `source`, `link`, `url` and `merge`. A `source` is copied, and a directory `source` needs `live = true`. `live` only works with `source`; with `text`, `link` or `url` it's an error: `home.files.<path>: live = true needs source, the file in the config to link to`. A `url` works as it does in [`files`](#files): the hash is locked, and the download happens at switch. Paths can't leave the home or the config directory.

`mutable` and `merge` work as in [`files`](#files): a read-only file is written `0444`, or `0555` with `executable`, and a merged file is owned by `home.user`. A `live` link stays writable, since it points into your config.

When a path is a directory archix didn't manage before, such as an existing `~/.config/nvim` you now link into the config, it's moved to `/var/lib/archix/users/<user>/backups` and comes back when the entry leaves the config. See [Dotfiles and apps](/docs/guides/dotfiles-and-apps/).

Home files are written once the switch has applied the rest, by a process running as `home.user`, so links in the home can't redirect root's writes. Merged `kconfig` files take KDE's `<file>.lock` while archix sets their keys, and every merged file is swapped in at once, so an app that saves in between gets its change merged again rather than lost. `on_change` runs when the switch or revert changed the file, as in [`files`](#files), but right after the write, as `home.user`, in the environment of a [user hook](/docs/guides/hooks/#user-hooks). A `live` entry is a link, and archix never writes what's behind it, so editing that file never runs `on_change`.

```lua
home = {
  user = "me",
  files = {
    [".config/nvim"] = { source = "home/.config/nvim", live = true },
    ["bin/hello"] = { text = "#!/bin/sh\necho hi\n", executable = true },
    [".config/user-dirs.dirs"] = { source = "home/user-dirs.dirs" },
    [".config/kdeglobals"] = { merge = { format = "kconfig", values = { General = { ColorScheme = "BreezeDark" } } } },
    [".config/fontconfig/fonts.conf"] = { source = "home/fonts.conf", on_change = "fc-cache -f" },
    [".local/share/fonts/Inter.ttf"] = archix.fetchurl {
      url = "https://downloads.example.org/fonts/Inter-4.1.ttf",
      sha256 = "9f0c6a3d1b7e24c58a6d0e3f9b1c47a2e8d5f06b3c9a1e7d4f2b8c0a6e3d9f15",
    },
  },
},
```

### `home.systemd`

| | |
| --- | --- |
| Type | `attrs of attrs of any` |
| Default | `{}` |
| Declared in | `<archix>/home.lua` |

The user's systemd units, by kind (`services`, `timers`, `sockets`, `paths`, `targets`) and name, as sections of keys; `Install.WantedBy` creates the `.wants` links.

`slices`, `mounts` and `automounts` work too. Units are written to `~/.config/systemd/user/<name>.<kind>`. A list value repeats the key.

```lua
home = {
  systemd = {
    services = {
      sync = { Unit = { Description = "Sync notes" }, Service = { ExecStart = "/usr/bin/rclone sync notes: %h/notes" } },
    },
    timers = {
      sync = { Timer = { OnCalendar = "hourly" }, Install = { WantedBy = { "timers.target" } } },
    },
  },
},
```

## Programs

Each `programs.<name>` module installs an app and writes its config into `home.user`'s home. They need `home.user`, and they're off until you set `enable = true`.

Every program has a `package` option, the package `enable` adds to [`packages`](#packages). Set it to `archix.null` to configure an app without installing it.

The modules for kitty, git, zsh, starship, fastfetch, `mime_apps` and user units are ported from [Home Manager](https://github.com/nix-community/home-manager), and render the same files Home Manager does. Option names are snake_case versions of Home Manager's: `mouseBindings` is `mouse_bindings`, `signByDefault` is `sign_by_default`. See [Dotfiles and apps](/docs/guides/dotfiles-and-apps/#3-ported-home-manager-modules).

### `programs.kitty`

Writes `~/.config/kitty/kitty.conf`, plus `quick-access-terminal.conf` and `diff.conf` when they have settings.

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `enable` | `bool` | `false` | Install and configure kitty. |
| `package` | `null or str` | `"kitty"` | |
| `font` | `null or submodule` | `null` | `name` (`str`, required) and `size` (`any`): points; 8 renders as 8, 8.5 as 8.500000, like Home Manager. |
| `settings` | `attrs of any` | `{}` | kitty.conf settings; booleans become yes/no. |
| `keybindings` | `attrs of str` | `{}` | `map` lines, keyed by shortcut. |
| `mouse_bindings` | `attrs of str` | `{}` | `mouse_map` lines. |
| `action_aliases` | `attrs of str` | `{}` | `action_alias` lines. |
| `environment` | `attrs of str` | `{}` | `env` lines. |
| `extra_config` | `lines` | `""` | Appended to kitty.conf. |
| `quick_access_terminal_config` | `attrs of any` | `{}` | `quick-access-terminal.conf` settings. |
| `diff_config.settings` | `attrs of any` | `{}` | `diff.conf` settings. |
| `diff_config.keybindings` | `attrs of str` | `{}` | `diff.conf` key mappings. |
| `diff_config.extra_config` | `lines` | `""` | Appended to `diff.conf`. |
| `shell_integration.mode` | `null or str` | `"no-rc"` | kitty's `shell_integration`; `no-rc` is always implied, null turns integration off. |
| `shell_integration.enable_zsh_integration` | `bool` | `true` | Load kitty's shell integration from `~/.zshrc` when `programs.zsh` is enabled. |

```lua
programs = {
  kitty = {
    enable = true,
    font = { name = "JetBrainsMono Nerd Font", size = 11 },
    settings = { background_opacity = "0.95", confirm_os_window_close = 0, enable_audio_bell = false },
    keybindings = { ["ctrl+shift+t"] = "new_tab_with_cwd" },
  },
},
```

### `programs.git`

Writes `~/.config/git/config`, and `ignore`, `attributes` and `allowed_signers` next to it when they have content.

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `enable` | `bool` | `false` | Install and configure git. |
| `package` | `null or str` | `"git"` | |
| `settings` | `any` | `{}` | git config as nested tables: `{ user = { name = ... } }`; deeper levels become `[section "sub"]`. |
| `signing` | `null or submodule` | `null` | Commit signing; fields below. |
| `signing.key` | `null or str` | `null` | Sets `user.signingKey`. |
| `signing.format` | `null or one of "openpgp", "ssh", "x509"` | `null` | Sets `gpg.format` and the matching signer program. |
| `signing.signer` | `null or str` | `null` | The signer program; `/usr/bin/gpg`, `/usr/bin/ssh-keygen` or `/usr/bin/gpgsm` by default, depending on `format`. |
| `signing.sign_by_default` | `null or bool` | `null` | Sets `commit.gpgSign` and `tag.gpgSign`. |
| `signing.allowed_signers` | `lines` | `""` | Written to `~/.config/git/allowed_signers` and set as `gpg.ssh.allowedSignersFile`. |
| `ignores` | `list of str` | `[]` | Lines of the global ignore file. |
| `attributes` | `list of str` | `[]` | Lines of the global attributes file. |
| `includes` | `list of submodule` | `[]` | `path` (`str`, required) and `condition` (`null or str`): an `[include]`, or an `[includeIf "<condition>"]`. |
| `lfs.enable` | `bool` | `false` | Install git-lfs and set up its filter. |
| `lfs.skip_smudge` | `bool` | `false` | Don't download LFS files on checkout. |

Values in `settings` win over the ones `signing` and `lfs` derive.

```lua
programs = {
  git = {
    enable = true,
    settings = {
      user = { name = "Me", email = "me@example.com" },
      init = { defaultBranch = "main" },
      url = { ["git@github.com:"] = { insteadOf = "https://github.com/" } },
    },
    signing = { format = "ssh", key = "/home/me/.ssh/id_ed25519.pub", sign_by_default = true },
    ignores = { ".direnv", "*.swp" },
    includes = { { path = "~/work/.gitconfig", condition = "gitdir:~/work/" } },
  },
},
```

### `programs.zsh`

Writes `~/.zshrc` and `~/.zshenv`, plus `~/.zprofile`, `~/.zlogin` and `~/.zlogout` when they have content. Plugins come from Arch's packages in `/usr/share/zsh/plugins`, installed for you when enabled. To make zsh your login shell, also set `users.<name>.shell = "/usr/bin/zsh"`.

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `enable` | `bool` | `false` | Install zsh and manage `~/.zshrc`. |
| `package` | `null or str` | `"zsh"` | |
| `enable_completion` | `bool` | `true` | Run `completion_init`. |
| `completion_init` | `lines` | `"autoload -U compinit && compinit"` | |
| `default_keymap` | `null or one of "emacs", "viins", "vicmd"` | `null` | |
| `cdpath` | `list of str` | `[]` | |
| `autocd` | `null or bool` | `null` | Sets or unsets `AUTO_CD`. |
| `autosuggestion.enable` | `bool` | `false` | Install and load zsh-autosuggestions. |
| `autosuggestion.strategy` | `list of str` | `["history"]` | `ZSH_AUTOSUGGEST_STRATEGY`. |
| `autosuggestion.highlight` | `null or str` | `null` | `ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE`. |
| `syntax_highlighting.enable` | `bool` | `false` | Install and load zsh-syntax-highlighting. |
| `syntax_highlighting.highlighters` | `list of str` | `[]` | Highlighters besides `main`. |
| `syntax_highlighting.styles` | `attrs of str` | `{}` | `ZSH_HIGHLIGHT_STYLES` entries. |
| `syntax_highlighting.patterns` | `attrs of str` | `{}` | `ZSH_HIGHLIGHT_PATTERNS` entries. |
| `history_substring_search.enable` | `bool` | `false` | Install and load zsh-history-substring-search. |
| `history_substring_search.search_up_key` | `any` | `["^[[A"]` | A key or list of keys. |
| `history_substring_search.search_down_key` | `any` | `["^[[B"]` | A key or list of keys. |
| `history.size` | `int` | `10000` | `HISTSIZE`. |
| `history.save` | `null or int` | `null` | `SAVEHIST`. Defaults to size. |
| `history.path` | `null or str` | `null` | Defaults to `~/.zsh_history`. Relative paths are relative to the home. |
| `history.ignore_patterns` | `list of str` | `[]` | `HISTORY_IGNORE` patterns. |
| `history.append` | `bool` | `false` | `APPEND_HISTORY`. |
| `history.ignore_dups` | `bool` | `true` | `HIST_IGNORE_DUPS`. |
| `history.ignore_all_dups` | `bool` | `false` | `HIST_IGNORE_ALL_DUPS`. |
| `history.save_no_dups` | `bool` | `false` | `HIST_SAVE_NO_DUPS`. |
| `history.find_no_dups` | `bool` | `false` | `HIST_FIND_NO_DUPS`. |
| `history.ignore_space` | `bool` | `true` | `HIST_IGNORE_SPACE`. |
| `history.expire_duplicates_first` | `bool` | `false` | `HIST_EXPIRE_DUPS_FIRST`. |
| `history.extended` | `bool` | `false` | `EXTENDED_HISTORY`. |
| `history.share` | `bool` | `true` | `SHARE_HISTORY`. |
| `set_options` | `list of str` | `[]` | Extra `setopt` options. |
| `shell_aliases` | `attrs of str` | `{}` | |
| `shell_global_aliases` | `attrs of str` | `{}` | `alias -g` aliases. |
| `dir_hashes` | `attrs of str` | `{}` | Named directories, `hash -d`. |
| `session_variables` | `attrs of any` | `{}` | Exported in `~/.zshenv`. |
| `init_content` | `lines` | `""` | Extra lines for `~/.zshrc`; other modules add their shell integration here. |
| `env_extra` | `lines` | `""` | Appended to `~/.zshenv`. |
| `profile_extra` | `lines` | `""` | Contents of `~/.zprofile`. |
| `login_extra` | `lines` | `""` | Contents of `~/.zlogin`. |
| `logout_extra` | `lines` | `""` | Contents of `~/.zlogout`. |

```lua
programs = {
  zsh = {
    enable = true,
    autosuggestion = { enable = true },
    syntax_highlighting = { enable = true },
    history = { size = 50000, ignore_all_dups = true },
    shell_aliases = { ll = "ls -l", g = "git" },
    session_variables = { EDITOR = "nvim" },
    init_content = "bindkey '^R' history-incremental-search-backward",
  },
},
```

### `programs.starship`

Writes `~/.config/starship.toml` when `settings` isn't empty.

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `enable` | `bool` | `false` | Install and configure the starship prompt. |
| `package` | `null or str` | `"starship"` | |
| `settings` | `any` | `{}` | `starship.toml` contents. |
| `enable_zsh_integration` | `bool` | `true` | Start starship from `~/.zshrc` when `programs.zsh` is enabled. |

```lua
programs = {
  starship = {
    enable = true,
    settings = { add_newline = false, character = { success_symbol = "[>](bold green)" } },
  },
},
```

### `programs.fastfetch`

Writes `~/.config/fastfetch/config.jsonc` when `settings` isn't empty.

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `enable` | `bool` | `false` | Install and configure fastfetch. |
| `package` | `null or str` | `"fastfetch"` | |
| `settings` | `any` | `{}` | `config.jsonc` contents. |

### `programs.alacritty`, `programs.foot`, `programs.mpv`

These are thin modules, built with [`archix.program`](/docs/reference/lua-api/#archixprogram). `settings` passes straight through to the app's config file in its format.

| Program | File | Format |
| --- | --- | --- |
| `alacritty` | `~/.config/alacritty/alacritty.toml` | `toml` |
| `foot` | `~/.config/foot/foot.ini` | `ini` |
| `mpv` | `~/.config/mpv/mpv.conf` | `ini` |

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `enable` | `bool` | `false` | Install and configure the program. |
| `package` | `null or str` | the program's name | |
| `settings` | `any` | `{}` | The file's contents, written in the format above. |
| `extra_config` | `lines` | `""` | Text appended to the file. |

```lua
programs = {
  alacritty = { enable = true, settings = { window = { opacity = 0.9 }, font = { size = 11 } } },
  foot = { enable = true, settings = { main = { font = "JetBrains Mono:size=11", pad = "8x8" } } },
  mpv = { enable = true, settings = { hwdec = "auto-safe", ["keep-open"] = "yes" } },
},
```

### `mime_apps`

Default applications per MIME type. Writes `~/.config/mimeapps.list`. It lives at the top level, not under `programs`, and installs nothing.

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `enable` | `bool` | `false` | Manage `~/.config/mimeapps.list`. |
| `associations.added` | `attrs of any` | `{}` | Extra apps per MIME type: a desktop file name or a list of them. |
| `associations.removed` | `attrs of any` | `{}` | Apps removed per MIME type, in the same form. |
| `default_applications` | `attrs of any` | `{}` | Default apps per MIME type, in the same form. |
| `default_application_packages` | `list of submodule` | `[]` | Apps that become the default for every MIME type they declare, after `default_applications`. |
| `default_application_packages.*.desktop` | `str` | *required* | The package's desktop file name, such as `firefox.desktop`. |
| `default_application_packages.*.mime_types` | `list of str` | *required* | The MIME types it handles. |

```lua
mime_apps = {
  enable = true,
  default_applications = {
    ["application/pdf"] = "org.pwmt.zathura.desktop",
    ["x-scheme-handler/https"] = { "firefox.desktop", "chromium.desktop" },
  },
},
```
