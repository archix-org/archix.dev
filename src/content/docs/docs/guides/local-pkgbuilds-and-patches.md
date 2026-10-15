---
title: Local PKGBUILDs and patches
description: Build packages from PKGBUILDs kept in your config, and rebuild official packages with your own patches, locked and rolled back like everything else.
sidebar:
  order: 2
---

Sometimes the package you want isn't in any repo or the AUR, or the one in the repos needs a fix that hasn't shipped yet. archix handles both from your config:

- **Local PKGBUILDs.** Put a recipe in `pkgbuilds/<dir>/` and declare the package it builds.
- **Patched packages.** List patch files in [`patches`](/docs/reference/options/#patches), and archix rebuilds Arch's own recipe for that package with them.

Both are locked, built in the same clean chroot as AUR packages, and recorded in every generation.

## Layout

Recipes and patches live in the config directory, next to `lua/`:

```
/etc/archix
├── init.lua
├── archix.lock
├── lua
│   └── hosts
│       └── desktop
│           └── init.lua
├── patches
│   └── hyprland
│       └── 0001-fix-cursor-warp.patch
└── pkgbuilds
    └── backup-scripts
        ├── PKGBUILD
        └── backup.sh
```

Every directory directly under `pkgbuilds/` that contains a `PKGBUILD` is a recipe. `patches/` is only a convention: a patch can be anywhere in the config directory, since you name its path.

## Local PKGBUILDs

A recipe is an ordinary PKGBUILD, with any files it uses next to it:

```bash title="pkgbuilds/backup-scripts/PKGBUILD"
pkgname=backup-scripts
pkgver=1.2.0
pkgrel=1
pkgdesc='Nightly restic backups'
arch=(any)
license=(MIT)
depends=(restic)
source=(backup.sh)
sha256sums=('SKIP')

package() {
  install -Dm755 backup.sh "$pkgdir/usr/bin/backup"
}
```

There's no option to set. Declare a package the recipe builds, and archix finds it:

```lua title="lua/hosts/desktop/init.lua"
return {
  packages = { "backup-scripts" },
}
```

A recipe nothing uses isn't built. One recipe can build several packages, as a split package; declare the ones you want. `pkgbuilds/` belongs to the whole config, so every host can use the same recipes.

### How names resolve

A recipe beats the repos only for a name you declare that it builds under exactly that name. Declare `backup-scripts` and the recipe in `pkgbuilds/backup-scripts/` builds it, even if a repo has a package of that name, so you can replace an official package with your own build.

A recipe that only `provides` a name doesn't take it from the repos. If the repos have that name, they win; to get the recipe's package, declare its real package name instead.

Everything else, dependencies included, resolves in the same order as for AUR packages: the repos first, official and third-party, then your recipes in `pkgbuilds/`, by name or by what they `provides`, then the AUR. So a recipe can supply a dependency the repos don't have, for another recipe or for an AUR package. A name none of them have stops `lock`, `diff` and `switch`:

```console
error: `backup-scriptz` is in none of the configured repos, pkgbuilds/ or the AUR
```

The AUR's package list is only downloaded when a name is in neither the repos nor `pkgbuilds/`.

### Reading the recipe

archix reads each recipe with `makepkg --printsrcinfo`, so a PKGBUILD that computes its version or package list works as it does for makepkg. Run as root, archix runs makepkg as the unprivileged `archix-build` user, since makepkg refuses root. The result is cached by the recipe's content, so an unchanged recipe isn't read again. A recipe makepkg can't read stops the command with `reading pkgbuilds/<dir>/PKGBUILD: makepkg --printsrcinfo failed: ...` and makepkg's own message.

### How it's locked

A local package is locked with `repo = "local"`, the recipe directory as its base, and a hash of the recipe as its commit:

```toml title="archix.lock"
[hosts.desktop.packages]
backup-scripts = { version = "1.2.0-1", repo = "local", file = "backup-scripts-1.2.0-1-any.pkg.tar.zst", base = "local:backup-scripts", commit = "7f3c9a1e5b2d8046c1e9f7a3b5d2c8e0a4f6b1d3c5e7a9f2b4d6c8e0a1f3b5d7", explicit = true }
restic = { version = "0.18.1-1", repo = "extra", file = "restic-0.18.1-1-x86_64.pkg.tar.zst", sha256 = "4b2e8d0f6a1c3e5b7d9f2a4c6e8b0d1f3a5c7e9b2d4f6a8c0e1b3d5f7a9c2e4b", explicit = false }

[hosts.desktop.recipes]
"local:backup-scripts" = "7f3c9a1e5b2d8046c1e9f7a3b5d2c8e0a4f6b1d3c5e7a9f2b4d6c8e0a1f3b5d7"
"local:wifi-firmware" = "builds:5d0b8e2f4a6c1e3b7d9f0a2c4e6b8d1f3a5c7e9b0d2f4a6c8e1b3d5f7a9c0e2b"
```

The hash covers every file in the directory, by path, contents and executable bit, so `chmod +x` on a file counts as a change. makepkg's own output at the top of the directory is left out: `src/`, `pkg/`, `.SRCINFO`, built `*.pkg.tar*` files and `*.log` files. `.git` is left out at any depth. Deeper down, everything counts, so a `files/src/fix.c` in the recipe is part of the hash. You can run `makepkg` in the directory to try a recipe without making the lock stale.

The `recipes` table has an entry for every recipe in `pkgbuilds/`. A recipe the host's lock builds from is recorded by its content hash, as `local:backup-scripts` above. Any other recipe is recorded as `builds:` and a hash of the package names it builds and `provides`, as `local:wifi-firmware` is for a host that doesn't use it. So editing a recipe a host doesn't use leaves that host's lock alone, while adding a recipe, or changing what one builds, makes every host's lock out of date, since it could change how a name resolves. When the lock is out of date, `archix status` says ``the config changed since the lock was written; `archix lock` or `archix switch` will re-resolve``, and the next `lock` or `switch` resolves again. See [The lockfile](/docs/concepts/lockfile/#local-recipes-and-patched-packages).

### How it's built

Local recipes build exactly like AUR packages: with `makechrootpkg` in the [clean chroot](/docs/guides/aur/#clean-chroot-builds) at the lock's snapshot date, or on the host if you list the package in [`aur.build_on_host`](/docs/guides/aur/#build_on_host). The build happens after you confirm and before anything on the system changes:

```console
$ sudo archix switch
install    backup-scripts 1.2.0-1
install    restic 0.18.1-1
apply 2 changes? [Y/n]
:: building local:backup-scripts 1.2.0-1 (in a clean chroot)
:: installing packages
...
```

They're never shown for [review](/docs/guides/aur/#build-recipe-review). The recipe is already in your config, and you reviewed it when you wrote or committed it.

### Rebuilds at the same version

Change a recipe without bumping `pkgver` or `pkgrel`, and the package still has to be rebuilt to reach the system. archix compares the recipe hash the current generation built with the lock's, and when they differ at the same version, `diff` and `switch` list a reinstall:

```console
$ sudo archix switch
resolving 87 declared packages against the 2026-10-09 snapshot
214 packages locked
wrote /etc/archix/archix.lock
reinstall  backup-scripts 1.2.0-1 (a build from a different recipe)
apply 1 change? [Y/n]
```

The package is built from the new recipe and reinstalled at the same version. A package installed at the locked version that archix never built from this recipe, such as one you installed by hand, is replaced the same way.

Builds from local recipes are cached in `/var/cache/archix/pkg` under a file name prefixed with a hash of the recipe, so two builds of the same version don't overwrite each other, and each generation records the file it installed. That's what lets a [rollback](/docs/concepts/generations/) go back across a same-version change: when the generation it returns to used a different recipe, the rollback lists the same `reinstall  backup-scripts 1.2.0-1 (a build from a different recipe)` line and reinstalls that generation's cached build, without building anything.

## Patched packages

To fix an official package, give archix the patch and keep Arch's recipe:

```lua title="lua/hosts/desktop/init.lua"
return {
  packages = { "hyprland" },
  patches = {
    hyprland = { "patches/hyprland/0001-fix-cursor-warp.patch" },
  },
}
```

Paths are relative to the config directory, and can't leave it: `patches.hyprland: ../fix.patch leaves the config directory`. List several patches and they apply in that order.

### What archix does

1. It takes the version the lock has for the package, and clones Arch's recipe for it from `https://gitlab.archlinux.org/archlinux/packaging/packages`, at the tag for that version: the version with `:` replaced by `-` and `~` by `.`.
2. It adds your patches to the recipe's sources, and applies them with `patch -p1` before the recipe's own `prepare()` runs. Each patch goes into whichever top-level directory of `$srcdir` it applies to, so a patch made in the upstream source tree, such as `git format-patch` output, works as it is.
3. It bumps the pkgrel to a sub-release, `0.51.1-2` to `0.51.1-2.1`, so the patched build sorts after Arch's build of the same version.

The first switch replaces Arch's build with yours:

```console
$ sudo archix switch
upgrade    hyprland 0.51.1-2 -> 0.51.1-2.1
apply 1 change? [Y/n]
:: building patch:hyprland 0.51.1-2.1 (in a clean chroot)
...
```

Only official packages can be patched, and only ones in the host's lock, declared or pulled in as a dependency:

```console
error: patches.hyprland: hyprland isn't among this config's packages or their dependencies
error: patches.zen-browser-bin: zen-browser-bin comes from aur, but only packages from Arch's official repos can be patched
```

The same goes for packages from third-party repos, such as `linux-cachyos` from `cachyos-core-v3`, and from your own recipes: archix only has Arch's recipes to patch. A package [pinned](/docs/concepts/lockfile/#pins) from the Arch Linux Archive counts as official. For an AUR package, copy its recipe into `pkgbuilds/` and change it there.

When the patched package comes from a pkgbase that builds several packages, every package of that base in your lock is swapped for the patched build too, so they stay at the same version. Patching `mesa` also rebuilds the `vulkan-radeon` you have from the same recipe.

### How it's locked

A patched package is locked with `repo = "local"`, `patch:` and the pkgbase as its base, and a commit made of Arch's recipe commit and a hash of your patches. The `recipes` table holds the full hash of the patches, keyed by the package you patched:

```toml title="archix.lock"
[hosts.desktop.packages]
hyprland = { version = "0.51.1-2.1", repo = "local", file = "hyprland-0.51.1-2.1-x86_64.pkg.tar.zst", base = "patch:hyprland", commit = "3a9e1c7b5d2f8046e1c3a5b7d9f0e2c4a6b8d0f1+c2e8a4f60b1d3e57", explicit = true }

[hosts.desktop.recipes]
"patch:hyprland" = "c2e8a4f60b1d3e579a1c3e5f7b9d2f4a6c8e0b2d4f6a8c1e3b5d7f9a0c2e4b6d"
```

Editing a patch, adding one or removing one makes the lock out of date, and the next switch rebuilds it at the same version: `reinstall  hyprland 0.51.1-2.1 (a build from a different recipe)`. Take the package out of `patches` and the next switch goes back to Arch's build.

### Updates

The patches follow the package. When `archix update` moves the snapshot to a new version of hyprland, the lock gets the new version, archix clones Arch's recipe at that version's tag, applies your patches to it, and builds `0.52.0-1.1`. A patch upstream has merged, or that no longer fits the new source, stops the build; drop or refresh it and switch again.

A plain `pacman -Syu` replaces your patched build whenever Arch has a newer version: pacman sees `0.52.0-1` as newer than `0.51.1-2.1`. The patches aren't lost. The next switch sees the system is newer than the lock and offers to [adopt](/docs/concepts/pacman-and-yay/#adopting) it:

- **Yes** moves the lock to the transaction's snapshot, as for any adopted upgrade, then applies your patches to the new version. The switch upgrades from Arch's build to yours, `upgrade    hyprland 0.52.0-1 -> 0.52.0-1.1`.
- **No** reverts to the lock, which puts your patched build of the locked version back, `downgrade  hyprland 0.52.0-1 -> 0.51.1-2.1`.

Either way, the patched build is what ends up installed.

## When it fails

Everything here fails before the system changes, in [prepare](/docs/concepts/safety/#prepare-then-apply), so a failed build leaves the packages, files and services as they were.

- **A build fails.** The switch stops with `local:backup-scripts 1.2.0-1 can't be built: ...` and makechrootpkg's output above it. Unlike an AUR package, there's no nearest version to fall back to: the recipe is yours, so fix it and switch again.
- **A patch applies nowhere.** The build fails, after `archix: archix-0-0001-fix-cursor-warp.patch applies to no directory in $srcdir`. archix copies each patch into the recipe as `archix-<n>-<file name>`, numbered in your list's order.
- **Arch's recipe can't be fetched**, because gitlab.archlinux.org is unreachable or has no tag for that version: `could not fetch Arch's recipe for hyprland 0.51.1-2 (https://gitlab.archlinux.org/archlinux/packaging/packages/hyprland.git, tag 0.51.1-2)`. Fetched recipes are cached, so this only needs the network the first time a version is locked.
- **makepkg is missing**: `makepkg not found; local recipes need pacman's makepkg`.
