---
title: The AUR
description: Use archix as your AUR helper, with build recipe review, clean chroot builds, a build cache and every package recorded in your config.
sidebar:
  order: 1
---

archix replaces yay and paru. It searches and installs from the official repos, your third-party repos and the AUR, with the flags you already know. The difference is what happens around the install: the package is added to your config, its build recipe is shown to you and locked by commit, and it's built in a clean chroot so the result doesn't depend on what happens to be installed on your machine.

## Install, remove, search

```console
$ archix search zen browser
aur/zen-browser 1.16.3b-1 (+38 2.10)
    Zen Browser, built from source
aur/zen-browser-bin 1.16.3b-1 (+412 18.53)
    Zen Browser, a Firefox-based browser focused on privacy
$ sudo archix install zen-browser-bin
$ sudo archix remove zen-browser-bin
```

The pacman-style aliases do the same:

| Alias | Same as |
| --- | --- |
| `archix -S <pkg>` | `archix install <pkg>` |
| `archix -Ss <terms>` | `archix search <terms>` |
| `archix -R <pkg>` | `archix remove <pkg>` |
| `archix -Syu` | `archix upgrade`: `pacman -Syu`, adopt, newest AUR recipes, switch |
| `archix` | `archix -Syu` |
| `archix <terms>` | Search, pick from a numbered list, and install, like `yay <terms>` |

There's one `packages` list for everything. `install` looks a name up in your configured repos first, official and third-party, and only names they don't have are looked up in the AUR. A recipe of your own in `pkgbuilds/` beats both for a name you declare that it builds under exactly that name; see [Local PKGBUILDs and patches](/docs/guides/local-pkgbuilds-and-patches/).

Search covers the locked snapshot's repos and the AUR's package list, with votes, popularity and out-of-date flags. Exact names rank first, then names starting with your first term, then names containing a term, then description matches, and repo packages come before AUR ones. `archix zen browser`, without `search`, numbers the results and installs the ones you pick; see [`archix <terms>`](/docs/reference/cli/#search-and-install-archix-terms). archix downloads that list (`packages-meta-v1.json.gz`) when it first needs it, refreshes it when it's more than an hour old and on `archix update`, and only loads it when a name isn't in the repos.

## Where `archix install` writes

`archix install` doesn't just install. It declares. The package name goes into `lua/hosts/<host>/installed.lua`, a plain Lua file archix imports automatically and you can edit too, and then archix switches. The next switch, on this machine or a fresh one, installs it again. If you abort or the switch fails, the file is put back as it was.

To put a package somewhere shared instead, pass a directory under the config's `lua/`:

```console
$ sudo archix install --path /etc/archix/lua/modules/dev rust-analyzer
note: import "modules.dev.installed" from a module this host loads, or archix won't see it
```

A directory outside `lua/` is refused, and nothing is installed.

archix only ever edits `installed.lua`. It never rewrites your Lua. So `archix remove` removes packages that `archix install` (or `archix init`) added; a package you declared by hand in `packages` is refused, and stays until you delete it from your file.

You can also skip the command and declare AUR packages in Lua directly, alongside official ones:

```lua title="lua/hosts/desktop/init.lua"
return {
  packages = { "firefox", "zen-browser-bin", "visual-studio-code-bin" },
}
```

To try a package without declaring it, `sudo archix install --temp <pkg>` installs it, built and reviewed the same way, and the next switch removes it unless you've declared it by then.

## How AUR packages are locked

Each AUR package is locked to the git commit of its build recipe, with the version from that commit's `.SRCINFO`:

```toml title="archix.lock"
zen-browser-bin = { version = "1.16.3b-1", repo = "aur", file = "zen-browser-bin-1.16.3b-1-x86_64.pkg.tar.zst", base = "zen-browser-bin", commit = "cd6266c6847e61c71facdd7dbf869b5cf64359be", explicit = true }
```

Dependencies the repos can satisfy, versioned ones included, are resolved from the repos at the lock's snapshot. The rest come from the AUR. An AUR package needed only to build another one is locked with `build_only = true`: it's built, and never installed.

AUR dependencies honour version constraints, `=`, `>=`, `<=`, `>` and `<`, the way pacman does. A package of that name, or one that provides it, only counts if its version meets the constraint. A wanted version without a pkgrel, such as `java-environment>=21`, compares against the pkgver alone, and a provide without a version, such as plain `java-environment`, only meets a dependency without one. A dependency nothing meets stops `lock`, `diff` and `switch` before any build: `` `java-environment=26` is in none of the configured repos, pkgbuilds/ or the AUR ``.

### VCS packages

A VCS package, usually named `-git`, sets its own version while it builds: its `pkgver()` asks the source repository. The locked version comes from `.SRCINFO` and is usually older. archix installs the package at the version the build gave it, and records that version in `archix.lock` and the generation:

```console
recorded the versions these packages gave themselves while building: archix-git 0.1.0.r35.g76a7920-1
```

The commit in the lock is still the recipe's, so the sources a VCS package builds from aren't pinned: a later rebuild of the same commit can produce a newer version. See [Limitations](/docs/concepts/limitations/).

`archix lock` and `archix switch` keep the locked commits. `archix update` and `archix -Syu` move AUR packages to the newest commit. See [The lockfile](/docs/concepts/lockfile/#aur-packages-commits).

## Build recipe review

AUR packages are code from strangers that runs on your machine. archix shows it to you before it's trusted, whenever a recipe enters the lock:

- for a new package, every file the recipe ships, the PKGBUILD first and `.SRCINFO` last, so install scripts and patches are in front of you too;
- for an update, the git diff since the locked commit, `.SRCINFO` included.

`.SRCINFO` matters because archix reads it: the dependencies, sources and PGP keys it locks and builds with come from there, not from running the PKGBUILD.

```console
$ sudo archix install zen-browser-bin
==> zen-browser-bin: new AUR package at cd6266c6847e
==> PKGBUILD
# Maintainer: ...
pkgname=zen-browser-bin
...
==> zen-browser-bin.install
...
==> .SRCINFO
...
trust zen-browser-bin at cd6266c6847e and lock it? [Y/n]
```

Answer `n` and nothing changes: `not locking zen-browser-bin; nothing was changed`.

A commit already in the lock is never shown again. You, or whoever committed the lock, already approved it. That's what lets a second machine install from your lock without asking about every package.

Review happens wherever recipes enter the lock: `install`, `lock`, `update`, `upgrade`, `switch` when the config changed, and `init`. `--noconfirm` skips it for one command, and [`aur.review = false`](/docs/reference/options/) turns it off.

## Clean chroot builds

Every AUR package is built with devtools' `makechrootpkg` in a clean chroot at `/var/lib/archix/chroot`, as the unprivileged `archix-build` user. The chroot is created with `base-devel` at the host's locked snapshot date, and moved whenever the lock's date changes. It's created again from scratch when devtools changes its chroot format. AUR dependencies are built first and passed in. AUR builds need `devtools` and `git` installed.

```console
:: creating the build chroot at the 2026-10-09 snapshot
:: building zen-browser-bin 1.16.3b-1 (in a clean chroot)
```

archix downloads the packages the chroot needs itself, with retries, rather than leaving it to pacman inside the chroot, which gives up after one slow connection to the archive. That covers the chroot's own `base-devel` when it's created or moved, and each build's repo dependencies, including those of the AUR packages the build needs first: they're resolved against the lock's snapshot and downloaded into pacman's cache, `/var/cache/pacman/pkg`, which devtools shares with the chroot, so makepkg installs them from there.

This fixes two classic AUR problems:

- **Missing dependencies.** A PKGBUILD that forgot a dependency fails in the chroot instead of quietly building against whatever you have installed.
- **Unreproducible builds.** The same recipe commit at the same snapshot date builds against the same dependencies, on every machine.

Packages from your config's own [`pkgbuilds/` and `patches`](/docs/guides/local-pkgbuilds-and-patches/) build in the same chroot, the same way, and `build_on_host` works for them too. They're never shown for review: the recipe is already in your config.

The chroot only has the official repos. Your third-party repos stay out of it, since they'd also need their signing keys inside.

### `build_on_host`

Some packages don't build in a chroot: they need hardware access, a running service, or files the chroot doesn't have. List them in [`aur.build_on_host`](/docs/reference/options/), and archix builds them with `makepkg` on the host instead. Their build dependencies are installed from the locked snapshot and removed again afterwards. Use it as an escape hatch, not a default: a host build depends on the state of your machine.

```lua
aur = { build_on_host = { "some-driver-dkms" } },
```

### `skip_check`

Some recipes build fine but their own test suite fails against the locked snapshot, say against a newer toolchain than upstream tested with. A failing `check()` fails the build, and with it the switch. List those package bases in [`aur.skip_check`](/docs/reference/options/#aurskip_check), and archix builds them with `makepkg --nocheck`, in the chroot or on the host. It's paru's `--nocheck`, but per package and in the config, so the choice is declared and reviewed like the rest:

```lua
aur = { skip_check = { "helm-ls" } },
```

## The build cache

Built packages are cached in `/var/cache/archix/pkg`, named by recipe commit and snapshot date, so every build of a version keeps its own file. If neither changed, archix reuses the built package instead of building again. Each generation records the sha256 of the file it installed, so rollback puts back that exact file, even when a later build has the same version.

To build once for several machines, push the builds to a [package cache](/docs/concepts/lockfile/#package-caches) with `archix cache push`, and list it in [`package_caches`](/docs/reference/options/#package_caches) on the others. A machine that finds a build of the same recipe commit at the same snapshot there installs it instead of building: `:: using zen-browser-bin's build from the package cache at file:///mnt/nas/archix/aur-builds/zen-browser-bin/cd6266c6847e61c71facdd7dbf869b5cf64359be-2026-10-09.json`. That build is trusted as the cache's record says, so list only caches you control.

A snapshot move rebuilds a package archix built, at the same commit, exactly when a library it links disappears in the switch: when a package the switch changes or removes stops shipping one of the ELF sonames the build needs. Otherwise it keeps the build it has. See [Rebuilds at the same version](/docs/guides/local-pkgbuilds-and-patches/#rebuilds-at-the-same-version).

When a locked commit can't be built any more, because it's gone from the package's history or the build fails, archix offers the nearest newer version, then the nearest older one, and the lock follows once the version you accepted has built; if it fails too, `archix.lock` is left unchanged. Set [`missing_versions`](/docs/reference/options/) or pass `--accept-nearest newer|older` to decide without asking. See [Missing versions](/docs/concepts/lockfile/#missing-versions). A build that fails while the chroot, the cache or `/tmp` has less than 1 GiB free is blamed on the disk, not the recipe: the switch stops with ``/var/lib/archix/chroot has only 19 MiB free, which is likely why the build failed; nothing was changed``, and no other version is tried.

## How installs run

archix reads pacman's local database directly and runs every transaction through the `pacman` binary. It doesn't reimplement pacman, so hooks, `pacman.conf` options and the local database behave as you expect.
