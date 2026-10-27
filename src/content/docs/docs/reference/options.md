---
title: Options reference
description: The core archix options, packages, repos, programs and files, and how to list every option available on your system.
sidebar:
  order: 2
---

This page documents every option archix ships: packages and repositories, the AUR, third-party repos, generations and deployments, the system (files, users, services and more), the environment, the firewall, activation hooks, booting, AppImages, Flatpak apps, your home directory and the `programs.*` modules. Your installed archix is still the authoritative list, because `archix options` is generated from the declarations themselves, and it includes options your own modules declare.

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

Point it at a mirror you run, or a caching proxy in front of the archive, when the archive is slow or far away. Every snapshot database and official package comes from there, including `/packages/` for pins. `switch` records the URL, so `rollback`, which doesn't evaluate the config, uses the same one. When it's down or lacks a file, the [`archive_mirrors`](#archive_mirrors) are tried in order.

```lua
archive_url = "https://ala-mirror.example.org",
```

### `archive_mirrors`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/packages.lua` |

Copies of the archive with its layout, tried in order when archive_url is down or lacks a file, for locking, downloads and rollbacks.

A file is looked for on `archive_url` first, then at the same path on each mirror. Once `archive_url` fails to answer, as opposed to not having a file, the rest of the run goes straight to the mirrors, so a dead archive costs one round of retries, not one per package: `warning: downloading https://ala-mirror.example.org/repos/2026/10/10/core/os/x86_64/core.db: ...; trying https://archive.archlinux.org/repos/2026/10/10/core/os/x86_64/core.db`. The resolver's pacman configuration lists every mirror as a `Server` after the archive. `switch` records the list, so `rollback` uses it too. Every file is still checked against the lock's sha256, wherever it comes from.

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

Caches are tried in order, before the archive and the third-party repos' servers, for every package file a lock, switch or temporary install needs. An AUR package, or a build of one of the config's recipes, is reused from a cache that has a build record for the same recipe commit and snapshot, instead of being built again, as long as it's the build `archix.lock` records; another build is passed over: `warning: the package cache at file:///mnt/nas/archix has a build of yay that doesn't match archix.lock; building it here instead`. A cache that fails is skipped with a warning. `switch` records the list, so `rollback` uses it too. Fill a cache with [`archix cache push`](/docs/reference/cli/#cache-push), or have every switch fill it with [`package_cache_push`](#package_cache_push); see [Package caches](/docs/concepts/lockfile/#package-caches).

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

After every switch that makes a generation, archix copies whatever the directory doesn't already hold and says `copied 3 files to /mnt/nas/archix`. A cache that's unmounted or read-only doesn't fail the switch, whose deployment is already committed: `warning: couldn't copy this machine's packages to the package cache at file:///mnt/nas/archix: ...`. List the same directory in [`package_caches`](#package_caches) on the machines that read from it.

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

With `"remove"`, the next switch offers to declare explicit packages that plain pacman or an AUR helper installed, as before the first switch (`declare them in installed.lua? (no removes them)`), and leaves every other package that isn't in the lock out of the new deployment, so the system is exactly what the config says. `--noconfirm` declares them. With `"warn"`, archix keeps undeclared packages and lists them in `archix diff` and `archix switch`.

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

Use it for packages you deliberately leave as they are, such as one you built by hand before archix took over; each new deployment carries them over unchanged. A package you build yourself can instead go in [`pkgbuilds/`](/docs/guides/local-pkgbuilds-and-patches/), so it's locked and rolled back like the rest. An ignored package is never removed, whatever `undeclared_packages` says, and never shows up in `diff`. Like `packages`, the list concatenates across modules. `archix init` puts foreign packages that aren't on the AUR here.

```lua title="lua/modules/common.lua"
return {
  ignored_packages = { "nvidia-dkms-custom" },
}
```

Ignoring only affects packages that aren't in the lock. A package you also declare in `packages` is managed as usual.

### `unsafe_downgrades`

| | |
| --- | --- |
| Type | `attrs of list of str` |
| Default | `{}` |
| Declared in | `<archix>/packages.lua` |

Versions per package that a downgrade mustn't cross without asking, on top of archix's own list (postgresql and mariadb major versions and a few more). A switch or rollback going back across one asks, defaulting to no; `--noconfirm` refuses unless `--allow-unsafe-downgrade` is given.

A downgrade from at or above a listed version to below it counts, for any package. Each generation stores the option, so a rollback checks it without evaluating the config. archix's own list covers `postgresql`, `forgejo`, `nextcloud`, `redis` and `valkey` by their first version part, and `mariadb`, `mysql`, `percona-server`, `gitea` and `mattermost` by their first two. See [Downgrades that may break data](/docs/concepts/generations/#downgrades-that-may-break-data).

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

They build in a throwaway copy of the current deployment, staged like a new one, with their build dependencies installed into it from the locked snapshot; the copy is discarded afterwards, and the running system doesn't change. Use it for packages that don't build in a chroot.

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

Number of newest generation records gc always keeps; a generation whose deployment is gone can still be rolled back to, rebuilt from the package cache.

### `generations.keep_days`

| | |
| --- | --- |
| Type | `int` |
| Default | `30` |
| Declared in | `<archix>/generations.lua` |

gc also keeps the record of every generation younger than this many days.

`archix gc` runs after every switch. Besides these, it keeps the records of the current and booted generations and of every generation with a deployment, then deletes cached packages, file contents and snapshot databases no kept record needs. See [Generations](/docs/concepts/generations/#limits-and-gc).

```lua
generations = { keep = 5, keep_days = 14 },
```

### `deployments.keep`

| | |
| --- | --- |
| Type | `int` |
| Default | `2` |
| Declared in | `<archix>/generations.lua` |

How many deployments gc keeps, newest first, counting the booted, current and boot-default ones, which it never drops; each kept one boots from its own entry and rolls back instantly. 1 keeps the previous deployment only until the new one has booted.

It's at least 1; `0` is an error. With `2`, the deployment you run and the one before it stay, so a rollback is a repoint and booting the previous entry works. A kept deployment costs the files the next one changed: about one update's worth, 0.9 GB for a week of updates and 2.9 GB for four months on ext4 or btrfs, less with btrfs compression. With `1`, older generations are rebuilt from the package cache when you roll back to them, as slowly as an update. Raise it to keep more instant rollbacks, and mind the ESP: each kept deployment has its boot images there. See [Storage](/docs/concepts/atomicity/#storage).

```lua
deployments = { keep = 3 },
```

## System

Every setting in this section renders to files archix manages. Their contents are stored with each generation, so `archix rollback` restores them along with the packages. Files under `/etc`, `/usr` and `/opt` are written into the new deployment and take effect at the next boot; files elsewhere are written once the switch has committed. A file archix replaces is backed up and comes back when it leaves the config; a managed file edited by hand is saved before it's overwritten, and `diff` flags it beforehand. See [Safety](/docs/concepts/safety/).

### `files`

| | |
| --- | --- |
| Type | `attrs of submodule` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

Files archix writes, keyed by absolute path; give one of `text`, `source`, `link`, `url` or `merge`. Files in `/etc`, `/usr`, `/opt` and `/boot` go into the new deployment and take effect at the next boot; files elsewhere are written once the switch commits. Files archix replaces are backed up and restored when they leave the config.

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
| `on_change` | `null or lines` | `null` | A bash script run as root when a switch or rollback changed this file: for a file in `/etc`, `/usr`, `/opt` or `/boot`, at the new deployment's first boot, before networking and before the [hooks](#hooks); for a file elsewhere, which is written right away, right after it's written. |
| `mutable` | `null or bool` | `null` | Write the file read-only (`false`) or writable (`true`); by default as [`mutable_files`](#mutable_files) says. |
| `merge` | `null or submodule` | `null` | Set only these keys in a file the app also writes, keeping the rest of it, like plasma-manager. Leaving the config leaves the keys as they are. `format` is one of `"ini"`, `"kconfig"` or `"json"`, *required*; `values` (`attrs of any`, default `{}`) holds the keys: sections (kconfig groups nest) of keys, or a JSON object deep-merged into the file. |

Give exactly one of `text`, `source`, `link`, `url` and `merge`; otherwise it's an error, `files.<path>: give exactly one of text, source, link, url or merge`. `text` is a `lines` value, so several modules can each add lines to the same file. A `source` path can't leave the config directory.

`on_change` runs when the switch or rollback changed the file: when its entry differs from the one in the generation the system is leaving (the first switch that manages it, new contents, mode, owner, group or link target, a rollback to a generation where it was different), or when the file on disk didn't match and archix wrote it again, such as a hand-edited file put back or a deleted one recreated. It doesn't run when only the script changes, or when the file leaves the config. For a file in the deployment, it runs at the first boot of the new deployment, before the network starts, and again at the next boot if it or a hook failed, up to three boots. For a file outside it, such as in `/srv`, it runs on the running system right after the switch writes the file. Make the script idempotent. A failing script is a soft failure, listed as `failed: on_change of <path>: ...`. See [Activation hooks](/docs/guides/hooks/#on_change).

A `url` needs a `sha256` of 64 hex digits; without one, `diff` and `switch` stop with `files.<path>: url needs sha256`. The hash is what's locked: `diff` compares the file on disk against it without downloading anything, and `switch` downloads the file after you confirm, before it changes anything. A download whose hash differs stops the switch: `<url> has sha256 <actual>, but the config says <sha256>`. The contents go into the same store as every other managed file, so rollback restores them even after the URL is gone.

With `mutable = false`, or [`mutable_files = false`](#mutable_files), the file is written without its write bits, so `"0644"` becomes `0444` and `"0755"` becomes `0555`. Links aren't affected.

`merge` sets keys in a file archix doesn't own, such as one an app writes too: archix edits only the declared keys and leaves the app's own. The app's keys never show up in `archix diff`; a declared key that changed does, and the next switch sets it back. JSON files may be JSONC, with comments and trailing commas, but archix drops the comments when it writes the file back; a file that can't take its keys stops the switch before anything changes. A new file gets `0644` and `owner`/`group`; an existing one keeps its mode and owner. When the entry leaves the config, the file keeps its keys and is neither restored nor deleted. See [Setting single keys](/docs/guides/dotfiles-and-apps/#setting-single-keys).

archix doesn't write under `/usr` (except `/usr/local`, which is part of each deployment), `/proc`, `/sys`, `/dev`, `/run`, `/tmp`, `/home` or `/boot`: `files: <path> is in a directory archix doesn't write to; home files go in home.files, and boot entries in boot.extra_entries`. Files in a home directory go in [`home.files`](#homefiles), and systemd-boot entries in [`boot.extra_entries`](#bootextra_entries).

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

System units to enable (`true`) or disable (`false`) in the next deployment; they start or stop at the next boot.

archix enables and disables them in the new deployment's `/etc`, after its unit files are in place, and doesn't start, stop or restart anything on the running system. A unit that can't be enabled doesn't undo the switch: the deployment is still committed, the failure is listed, and archix exits non-zero.

```lua
services = { ["sshd.service"] = true, ["bluetooth.service"] = true, ["cups.service"] = false },
```

### `user_services`

| | |
| --- | --- |
| Type | `attrs of bool` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

User units to enable (`true`) or disable (`false`) for every user in the next deployment, with `systemctl --global`; they start or stop at the next boot.

```lua
user_services = { ["pipewire-pulse.socket"] = true },
```

### `users`

| | |
| --- | --- |
| Type | `attrs of submodule` |
| Default | `{}` |
| Declared in | `<archix>/system.lua` |

Users archix creates and keeps in step; ones it created are removed (home kept) when they leave the config. Passwords are never managed: set them with `passwd`.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `uid` | `null or int` | `null` | Picked by `useradd` when null. |
| `group` | `null or str` | `null` | Primary group; by default one named after the user. |
| `groups` | `list of str` | `[]` | Supplementary groups. |
| `shell` | `path` | `"/bin/bash"` | |
| `home` | `null or path` | `null` | The home directory; `/home/<name>` when null. |
| `description` | `str` | `""` | The GECOS field. |
| `system` | `bool` | `false` | A system user: low uid, no home directory. |

Declaring a user that already exists brings it in line with every field, defaults included: a declared user is authoritative, so `groups` is the complete list of supplementary groups, and `shell` falls back to `/bin/bash`. When you declare an existing user by hand, list all of its groups and its shell. `archix init` captures existing users with their real groups, shell, home and description, so declaring them changes nothing. Users that archix didn't create are never deleted. Users are created in the new deployment's `/etc`, and their home directories once it's committed. One archix created and you declare again later without a `uid` gets back the uid it had, so the files it owned are still its own. Groups work the same way.

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

## Environment

See [Environment and firewall](/docs/guides/environment-and-firewall/) for a walkthrough.

### `environment.variables`

| | |
| --- | --- |
| Type | `attrs of any` |
| Default | `{}` |
| Declared in | `<archix>/environment.lua` |

Environment variables for every user: login shells get them from `/etc/profile.d/archix-environment.sh`, and systemd user services and graphical sessions from `/etc/environment.d/90-archix.conf`, from the next boot on. Lists are joined with `:`.

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

Shell aliases for every user's login shells, from `/etc/profile.d/archix-aliases.sh`, from the next boot on; [`programs.zsh.shell_aliases`](#programszsh) covers every interactive zsh.

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

An nftables firewall in its own `inet archix` table, loaded by `archix-firewall.service`; like every change to it, it applies at the next boot. Incoming traffic is dropped unless it answers outgoing traffic or something below allows it; other tables (Docker's, Tailscale's) are left alone.

With `true`, archix declares the `nftables` package, writes the rules to `/etc/nftables-archix.conf`, and writes and enables `archix-firewall.service`, which loads them before the network starts. Rule changes take effect at the next boot, like the rest of the deployment; turning it off disables the service, so the next boot has no `archix` table. archix doesn't use `nftables.service`, whose stop flushes every table.

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

Scripts archix runs at the first boot of every deployment a switch or rollback makes current. See [Activation hooks](/docs/guides/hooks/) for a walkthrough.

### `hooks`

| | |
| --- | --- |
| Type | `attrs of submodule` |
| Default | `{}` |
| Declared in | `<archix>/hooks.lua` |

Scripts run at the first boot of the deployment every switch or rollback commits, like NixOS activation scripts: early, once its users exist and its /var changes are applied, before networking, D-Bus and logins. A switch with nothing to build runs them right away. They run once per switch, so make them idempotent; a failing hook is reported on the console and at the next login, runs again at the next boot, and doesn't undo the switch.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `run` | `lines` | *required* | The bash script to run; it runs with `set -e` and `pipefail`. |
| `user` | `null or str` | `null` | Run as this user, in their home; root by default. At boot nobody is logged in, so there's no session bus; a switch with nothing to build gives them theirs if they're logged in. |
| `after` | `list of str` | `[]` | Hooks that must run before this one. |

Hooks run at the first boot of the deployment, after the [`on_change`](#files) scripts, from `archix-deploy-activate.service`: after local filesystems are mounted and `systemd-sysusers` has run, before `sysinit.target`, so before the network and before anyone logs in. They also run right away on a switch that has nothing to build. Every switch and rollback that makes a deployment current records them for its boot, and the preview shows them on a `run hooks` line, in order: by `after`, then by name. An `after` that names no hook is an error, `hooks.<name>.after names "<hook>", which isn't a hook`, and so is a cycle, which names the hooks in it, such as `hooks b -> c -> b wait on each other in a cycle`, or `hook a lists itself in after`.

Each script runs as `bash -e -o pipefail -c` in a clean environment: `PATH=/usr/local/sbin:/usr/local/bin:/usr/bin`, `HOME`, `USER` and `LOGNAME`, and `LANG` if archix has one. Root hooks run in `/`. With `user`, the hook runs as that user without opening a login session, in their home (or `/` if it doesn't exist), with `XDG_RUNTIME_DIR` and `DBUS_SESSION_BUS_ADDRESS` set while they're logged in, which at boot they aren't.

A failing hook is a soft failure: nothing is undone, the boot goes on, and archix prints `archix: at boot, hook <name>: exited with exit status: 1; it runs again at the next boot (try 1 of 3)` on the console and leaves the same note for your next login. Until all of them succeed, every hook and `on_change` script waiting for that deployment runs again at the next boot, up to three boots; then they wait for [`archix hooks retry` or `archix hooks skip`](/docs/reference/cli/#hooks).

```lua
users = { alice = { groups = { "wheel" } } },
hooks = {
  ["backup-dir"] = { run = "install -d -m 0750 -o alice /srv/backups" },
  ["flatpak-overrides"] = { run = "flatpak override --system --filesystem=xdg-download com.spotify.Client", after = { "backup-dir" } },
},
```

## Boot and kernel

archix boots every generation as its own [deployment](/docs/concepts/atomicity/), from its own systemd-boot entry, so `boot.loader = "systemd-boot"` is required. Each entry is `loader/entries/archix-<n>.conf` on the ESP, starting a unified kernel image in `EFI/archix/`, built with `ukify` from the deployment's kernel and a fresh `mkinitcpio` initramfs; deployments whose image comes out the same share one. Every other kernel package installed beside `boot.kernel`, such as `linux-lts`, gets an entry and image of its own, `archix-<n>-linux-lts.conf`. See [Boot entries](/docs/concepts/generations/#boot-entries). Booting an older entry boots that deployment, with the same packages, kernel and `/etc`.

With `boot.loader` set, archix declares the packages it needs for this itself: the `boot.kernel` package, `mkinitcpio`, `systemd-ukify` and `dosfstools`, whose `fsck.vfat` lets the ESP be checked at boot after a power cut, plus `sbctl` when `boot.secure_boot = true`. You don't list them in `packages`. With the default `secure_boot = null`, signing depends on sbctl being installed with keys, so keep `sbctl` declared yourself if you rely on that.

Each kept deployment can also get a protected entry with [`boot.protected_entry`](#bootprotected_entry). See [Secure Boot](/docs/guides/secure-boot/).

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

This is the whole command line of every boot entry, so it must contain `root=`; archix refuses to switch without it. The generation an entry boots isn't on the command line; archix reads it from the entry systemd-boot started.

### `boot.loader`

| | |
| --- | --- |
| Type | `null or one of "systemd-boot"` |
| Default | `null` |
| Declared in | `<archix>/boot.lua` |

The boot loader archix writes each deployment's boot entry for; a switch needs it, as every generation boots as its own deployment.

With `null`, `switch` refuses: ``archix boots every generation as its own deployment from a systemd-boot entry, so it needs boot.loader = "systemd-boot" (with boot.esp and kernel_params); nothing was changed``. GRUB and Limine aren't supported.

### `boot.esp`

| | |
| --- | --- |
| Type | `path` |
| Default | `"/efi"` |
| Declared in | `<archix>/boot.lua` |

Where the EFI system partition is mounted; not /boot, which each deployment keeps its own copy of.

The first switch refuses an ESP at or inside `/boot`: ``the ESP is mounted at /boot, in /boot, which each deployment keeps its own copy of; mount it at /efi instead (in /etc/fstab, and boot.esp = "/efi"), then switch again``. To move it, mount the ESP at `/efi` in `/etc/fstab` instead of `/boot`, and copy the kernel, initramfs and microcode images it held into the now empty `/boot` on the root filesystem, where their packages expect them; `EFI/` and `loader/` stay on the ESP.

archix writes the ESP's line in each deployment's `/etc/fstab` so that a power cut can't leave it broken or readable by everyone:

- It's mounted on access and unmounted after a minute idle (`x-systemd.automount,x-systemd.idle-timeout=60`), so a forced reboot rarely catches it mounted.
- It's checked at boot (`fs_passno` 2), so `fsck.fat` repairs a cluster chain a power cut left half freed. Without the check, the kernel remounts the ESP read-only at the first broken chain.
- It's readable by root only (`fmask=0077,dmask=0077`), unless the line already sets `umask`, `fmask` or `dmask`. systemd-boot's random seed lives on the ESP.
- An ESP with no line gets one, `UUID=… /efi vfat umask=0077,x-systemd.automount,x-systemd.idle-timeout=60 0 2`, as systemd's own automatic mount can't put it at `/efi` once each deployment binds its own `/boot`.

When the config manages `/etc/fstab`, archix leaves it alone and the switch warns instead, for each of these it finds missing.

### `boot.kernel`

| | |
| --- | --- |
| Type | `str` |
| Default | `"linux"` |
| Declared in | `<archix>/boot.lua` |

The kernel package boot entries use, such as `linux-lts`, `linux-zen` or `linux-cachyos`. archix declares it for you, and `archix init` sets it to the package of the kernel you're running. Each deployment has its own kernel and modules, so an older entry boots with its own after the kernel package was upgraded.

This kernel's entry is the main one, the one boot counting, the protected entry and the fallback image use. Any other kernel package you declare gets its own entry too, titled "Arch Linux (archix, generation 42, linux-lts)". When the lock has `dkms`, it also gets the `-headers` package of every `linux*` kernel in it that the repos have headers for, so DKMS builds its modules for each kernel.

### `boot.microcode`

| | |
| --- | --- |
| Type | `list of str` |
| Default | `[]` |
| Declared in | `<archix>/boot.lua` |

Microcode images in /boot loaded before the initramfs, such as amd-ucode.img.

Paths are relative to the deployment's `/boot`, where the `amd-ucode` and `intel-ucode` packages put them.

### `boot.tries`

| | |
| --- | --- |
| Type | `int` |
| Default | `3` |
| Declared in | `<archix>/boot.lua` |

How many boots systemd-boot tries a new deployment's entry before it falls back to the last deployment that booted, which archix then makes current; a boot that reaches boot-complete.target counts as good. Needs systemd-boot 260 or newer on the ESP; 0 turns boot counting off.

A new deployment's entry is written as `archix-12+3.conf`, and `loader.conf` names it as `preferred`, with `default` left on the newest deployment that has booted. systemd-boot counts a try down at every boot of it, and once a boot reaches `boot-complete.target`, `systemd-bless-boot` renames it `archix-12.conf` and it counts as good. After three boots that never got that far, it's `archix-12+0-3.conf`, which systemd-boot passes over for `default`. That boot starts deployment 11 as it was, archix makes it current, and leaves a note at the next login: `archix: generation 12 didn't finish booting in the tries it had, so systemd-boot started generation 11, which is current now`.

Counting needs systemd-boot to count tries for a UKI entry, which came in 258, and `preferred`, which came in 260; archix reads what the running systemd-boot supports from its `LoaderFeatures` variable, and with an older one writes plain entries as before. A deployment that boots far enough but is broken past that point, such as a desktop that doesn't start, counts as good, unless [`boot.health_check`](#boothealth_check) says otherwise.

On a counted try, if the initramfs can't mount the deployment or its first-boot steps fail, archix reboots after 10 seconds, so the next try or the fallback happens without anyone at the machine.

### `boot.health_check`

| | |
| --- | --- |
| Type | options, below |
| Declared in | `<archix>/boot.lua` |

What a new deployment's boot has to pass, besides reaching `boot-complete.target`, before it counts as good. For servers and headless machines; off by default.

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `boot.health_check.units` | `list of str` | `[]` | For servers and headless machines, off by default: systemd units that must be active, such as "sshd.service", before a new deployment's boot counts as good. Every target here needs boot.tries > 0. A target that still fails after boot.health_check.timeout marks the deployment's entry bad, so the next boot starts the previous deployment, and the login notes and `archix status` say which target failed and when. |
| `boot.health_check.listening` | `list of int` | `[]` | TCP ports something on this machine must listen on, such as 22 for sshd, before a new deployment's boot counts as good. |
| `boot.health_check.reachable` | `list of str` | `[]` | Hosts that must answer before a new deployment's boot counts as good: "host:port" must accept a TCP connection, a bare host must answer a ping, and "gateway" is the default IPv4 gateway. A host that's down for a reason outside the deployment (the router, a captive portal, a laptop on a train) fails the check just the same and falls back from a good update, so list only what is always up when this machine boots; switching again once it answers brings the update back. |
| `boot.health_check.login` | `bool` | `false` | Each user archix declares must be able to log in: `su -s <shell> <user> -c true` must succeed. Without it, a new deployment's boot still doesn't count as good while a declared user's login shell is missing or not in /etc/shells. |
| `boot.health_check.timeout` | `int` | `120` | Seconds the health check keeps retrying its targets after boot before it fails. |
| `boot.health_check.reboot_on_failure` | `bool` | `false` | Reboot as soon as the health check fails, so the previous deployment starts with nobody at the machine; otherwise the failed deployment keeps running until the next restart. |

`archix-health-check.service` runs before `boot-complete.target`, on counted boots only, and retries every target every two seconds until `timeout`. With `reachable` set, it waits for `network-online.target`. The declared users' login shells are always checked, with or without other targets. A target that still fails marks the entry bad, as `systemd-bless-boot` would, so the next boot starts the previous deployment:

```
archix: deployment 7 failed its health check "TCP port 22 listening" (nothing listens on TCP port 22); its entry is marked bad, so the next boot starts the previous deployment
```

That boot makes the previous deployment current, and says why at the next login and in `archix status`: `archix: deployment 7 fell back to 6 because the health check "TCP port 22 listening" failed at 2026-10-21 10:03:05 UTC (nothing listens on TCP port 22); switch again once it passes`. If the failed boot ran newer versions of packages that migrate their data, the note adds a `WARNING` naming them, as their services may have migrated data the older versions can't read; see [Downgrades that may break data](/docs/concepts/generations/#downgrades-that-may-break-data).

Any target needs [`boot.tries`](#boottries) above 0, since the fallback is systemd-boot's boot counting: ``boot.health_check needs boot.tries > 0: a failed check falls back by systemd-boot's boot counting; nothing was changed``.

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

When `gc` drops a deployment, its entry and any UKI no other entry uses come off the ESP, and this puts their hashes in `dbx` right after, in the same switch, with every check [`archix secure-boot revoke`](/docs/reference/cli/#secure-boot-revoke) makes. If revoking fails, the switch is still done, and says why: `revoking boot images no entry uses: ...`.

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

`true` also declares the `sbctl` package. When signing, archix also signs systemd-boot's own binary. Changing this builds a new deployment, so turning Secure Boot on re-signs. `sudo archix secure-boot setup` creates and enrolls the keys.

With Secure Boot enforcing, a switch whose entry wouldn't be signed, because this is `false` or `sbctl` has no keys, is refused before anything is built. See [Secure Boot](/docs/guides/secure-boot/).

### `boot.extra_entries`

| | |
| --- | --- |
| Type | `attrs of lines` |
| Default | `{}` |
| Declared in | `<archix>/boot.lua` |

More systemd-boot entries archix writes to the ESP's loader/entries, keyed by file name, such as `["memtest86+.conf"] = "title Memtest86+\nefi /memtest86+/memtest.efi"`. With Secure Boot on, whatever they start must be signed too.

Each one becomes `<boot.esp>/loader/entries/<name>`, a managed file like any other, so it's removed when it leaves the config. Names must end in `.conf`, since systemd-boot reads only those: `boot.extra_entries: "memtest" must end in .conf, as systemd-boot reads only those`. What an entry starts has to be on the ESP, since systemd-boot reads nothing else. A package that installs an EFI image into `/boot`, as `memtest86+-efi` does, puts it in the deployment's `/boot`, so copy it to the ESP, `/efi/memtest86+/memtest.efi` here. archix doesn't sign what an entry starts; with Secure Boot on, sign it yourself, for example with `sbctl sign -s /efi/memtest86+/memtest.efi`.

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

Also write an 'archix protected' entry for each deployment with the kernel command line Easy Anti-Cheat Linux v2 asks for (integrity lockdown, kexec off, the BPF LSM, IOMMU without passthrough). It can't make the kernel well-known-signed; `archix secure-boot check --anticheat` shows what's still missing.

The entry is `archix-<n>-protected.conf`, kept with its deployment and signed like the rest. It adds `lockdown=integrity sysctl.kernel.kexec_load_disabled=1 lsm=landlock,lockdown,yama,integrity,bpf iommu.passthrough=0` to the command line. See [The protected entry](/docs/guides/secure-boot/#the-protected-entry).

### `boot.rescue_entry`

| | |
| --- | --- |
| Type | `bool` |
| Default | `true` |
| Declared in | `<archix>/boot.lua` |

Keep an 'archix rescue' entry in the boot menu, after the deployments and never the default. It boots the kernel of the newest deployment that booted fine, with an initramfs of its own that mounts no deployment and opens [`archix rescue`](/docs/reference/cli/#rescue), so a system no deployment boots can be repaired without a live USB. It's refreshed at each switch when its source changes, kept in `EFI/archix-rescue/`, signed with Secure Boot like the rest, never given a TPM policy, and never removed by `gc` or revoked while in use. It's a root shell for whoever is at the console, as a live USB is; turn it off on a machine whose firmware is locked against other boot media. See [Recovering a broken system](/docs/guides/recovery/).

## Removed options

These options are gone. Setting one is a config error that names what replaced it, such as `option was removed (set in lua/hosts/desktop/init.lua): every switch builds a new deployment beside the running one, so no snapshot is taken; deployments.keep sets how many stay bootable`.

| Option | Why |
| --- | --- |
| `boot.snapshots` | Every switch builds a new deployment beside the running one, so no snapshot is taken; [`deployments.keep`](#deploymentskeep) sets how many stay bootable. |
| `boot.entries` | Every kept deployment has one boot entry, so [`deployments.keep`](#deploymentskeep) sets how many there are. |
| `restart_if_changed` | A switch takes effect at the next boot, so archix restarts nothing. |

## AppImages

See [Flatpak and AppImages](/docs/guides/flatpak-and-appimages/) for a walkthrough.

### `appimages`

| | |
| --- | --- |
| Type | `attrs of submodule` |
| Default | `{}` |
| Declared in | `<archix>/appimages.lua` |

AppImages to download, keyed by command name: installed to `/opt/archix/appimages`, linked into `/usr/local/bin` and added to app menus. Declares `fuse2`, which AppImages need.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `url` | `str` | *required* | Where to download the AppImage. |
| `sha256` | `str` | *required* | Its sha256; `archix prefetch <url>` prints it. |
| `name` | `null or str` | `null` | The name in app menus; the attribute name by default. |
| `icon` | `null or str` | `null` | An icon name or path for the menu entry. |
| `categories` | `list of str` | `[]` | Menu categories, such as `Office` or `Development`. |
| `desktop` | `bool` | `true` | Add an entry to app menus. |

Each entry becomes three managed files, so it's locked by its hash and rolls back like any other [`files`](#files) entry:

| File | What it is |
| --- | --- |
| `/opt/archix/appimages/<name>.AppImage` | The download, mode `0755`. |
| `/usr/local/bin/<name>` | A link to it, so `<name>` runs from your `PATH`. |
| `/usr/local/share/applications/archix-<name>.desktop` | The menu entry, unless `desktop = false`. |

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

Flatpak app ids to install system-wide, such as `org.mozilla.firefox`; write `remote:id` for a remote other than `flatpak.remote`. Each is locked to a commit in `archix.lock` and installed in `/var/lib/flatpak` once the switch commits; when that switch brings flatpak itself, the new deployment's first boot installs them, once the network is up.

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

Files in the user's home, keyed by path relative to it (a leading `~/` is allowed); written once a switch commits, without waiting for the boot.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `text` | `null or lines` | `null` | The file's contents, written as given. |
| `source` | `null or str` | `null` | A file or directory in the config directory, as a path relative to it. |
| `live` | `bool` | `false` | Symlink to `source` in the config directory instead of copying it, so apps that write their own config edit your repo. |
| `link` | `null or str` | `null` | Make the file a symlink to this target. |
| `url` | `null or str` | `null` | Download the file from here; needs `sha256`. `archix.fetchurl { url, sha256 }` builds both. |
| `sha256` | `null or str` | `null` | The sha256 the download from `url` must have; `archix prefetch <url>` prints it. |
| `executable` | `bool` | `false` | Write the file with mode `0755` instead of `0644`. |
| `on_change` | `null or lines` | `null` | A bash script run as `home.user` right after a switch or rollback writes a change to this file, which it does once the deployment is committed, so a reload signal reaches the running app; when the same switch creates `home.user`, it runs at the new deployment's first boot instead. |
| `mutable` | `null or bool` | `null` | Write the file read-only (`false`) or writable (`true`); by default as [`mutable_files`](#mutable_files) says. |
| `merge` | `null or submodule` | `null` | Set only these keys in a file the app also writes, keeping the rest of it, like plasma-manager. Leaving the config leaves the keys as they are. `format` is one of `"ini"`, `"kconfig"` or `"json"`, *required*; `values` (`attrs of any`, default `{}`) holds the keys: sections (kconfig groups nest) of keys, or a JSON object deep-merged into the file. |

Give exactly one of `text`, `source`, `link`, `url` and `merge`. A `source` is copied, and a directory `source` needs `live = true`. `live` only works with `source`; with `text`, `link` or `url` it's an error: `home.files.<path>: live = true needs source, the file in the config to link to`. A `url` works as it does in [`files`](#files): the hash is locked, and the download happens at switch. Paths can't leave the home or the config directory.

`mutable` and `merge` work as in [`files`](#files): a read-only file is written `0444`, or `0555` with `executable`, and a merged file is owned by `home.user`. A `live` link stays writable, since it points into your config.

When a path is a directory archix didn't manage before, such as an existing `~/.config/nvim` you now link into the config, it's moved to `/var/lib/archix/users/<user>/backups` and comes back when the entry leaves the config. See [Dotfiles and apps](/docs/guides/dotfiles-and-apps/).

Home files are written once the switch's deployment is committed, without waiting for the reboot, by a process running as `home.user`, so links in the home can't redirect root's writes. Merged `kconfig` files take KDE's `<file>.lock` while archix sets their keys, and every merged file is swapped in at once, so an app that saves in between gets its change merged again rather than lost. `on_change` runs when the switch or rollback changed the file, as in [`files`](#files), but right after the write, as `home.user`, in the environment of a [user hook](/docs/guides/hooks/#user-hooks). A `live` entry is a link, and archix never writes what's behind it, so editing that file never runs `on_change`.

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
