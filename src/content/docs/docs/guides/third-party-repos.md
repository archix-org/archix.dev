---
title: Third-party repos
description: How archix declares and locks CachyOS, chaotic-aur and other third-party repos, and what happens when a locked version can't be restored.
sidebar:
  order: 3
---

Many Arch-based systems install packages from repos that Arch doesn't run. CachyOS has its optimized `cachyos-v3`, `cachyos-core-v3` and `cachyos-extra-v3` repos and its kernels. EndeavourOS has `endeavouros`. Plenty of people add chaotic-aur for prebuilt AUR packages.

archix supports all of them. They need a little more care than the official repos, though, and on CachyOS that care matters for most of your system. This page explains why, and what archix does about it.

## Declaring a third-party repo

Third-party repos are declared in your config with [`custom_repos`](/docs/reference/options/), like everything else:

```lua title="lua/modules/cachyos.lua"
return {
  custom_repos = {
    {
      name = "cachyos-core-v3",
      servers = { "https://mirror.cachyos.org/repo/x86_64_v3/$repo" },
      keys = { "F3B607488DB35A47" },
      before_official = true,
    },
    {
      name = "cachyos",
      servers = { "https://mirror.cachyos.org/repo/$arch/$repo" },
      keys = { "F3B607488DB35A47" },
      before_official = true,
    },
  },
}
```

| Field | Default | Meaning |
| --- | --- | --- |
| `name` | | The repo's name, as in `pacman.conf`. It can't clash with an official repo. |
| `servers` | | Server URLs, tried in order. `$repo` and `$arch` are expanded as pacman does; `$arch_v3` and `$arch_v4` become `x86_64_v3` and `x86_64_v4`. |
| `keys` | `{}` | Fingerprints of the repo's signing keys. archix imports and locally signs any that pacman's keyring doesn't have yet, at switch. |
| `keyserver` | `"hkps://keyserver.ubuntu.com"` | Where those keys are fetched from. |
| `sig_level` | `"Required DatabaseOptional"` | The repo's `SigLevel`. |
| `before_official` | `false` | List the repo before `core` and `extra`, so its packages win. CachyOS's repos need this. |

From the declaration, archix writes the repo's section into `/etc/pacman.conf`, in a [block it manages](/docs/concepts/pacman-and-yay/#third-party-repos-in-pacmanconf), so plain pacman sees the same repos. If pacman has no database for a new repo yet, the switch downloads it, so pacman doesn't warn `database file for 'cachyos' does not exist` on every run. Removing a repo from your config removes its section, and its packages. A fresh machine gets the same repos, with the same keys, on its first switch.

You rarely write the declaration yourself. `archix init` imports every third-party repo in your current `pacman.conf`: its servers, including the ones in `Include`d mirrorlists, its `SigLevel`, and whether it's listed before the official repos. On a CachyOS install, that's where your kernel, and often your mesa, glibc and more, come from.

`init` doesn't know which keys belong to which repo, so their signing keys stay in pacman's keyring, where they already are, and the declaration has no `keys`. That works on this machine. List the fingerprints under `keys` if you want archix to import them on other machines too.

The official repos are a separate, simpler option: [`repos`](/docs/reference/options/) picks which of `core`, `extra`, `multilib` and the testing repos you use. Those are always locked against the Arch Linux Archive, on CachyOS too. Repos resolve in `pacman.conf` order: the official ones at the lock's snapshot date, third-party ones at their live state.

### CPU levels

CachyOS builds its packages for x86-64 microarchitecture levels, and labels them `x86_64_v3` or `x86_64_v4` instead of `x86_64`. When archix resolves, it accepts `x86_64` plus every level your CPU supports, as reported by `/lib/ld-linux-x86-64.so.2 --help`, so those repos resolve, and a CPU without v4 never gets v4 packages. Server URLs may use `$arch_v3` or `$arch_v4`, as CachyOS's mirrorlists do: `$arch` is replaced by `x86_64`, so they become `x86_64_v3` and `x86_64_v4`.

The levels come from the CPU of the machine that resolves the lock, not the host it's for. Locking several hosts on one machine is still fine, as long as each host's CPU can run what its lock holds, and archix checks that. Before changing anything, a switch looks at the architecture of every locked package, and if any is built for an x86-64 level this CPU doesn't support, it stops and leaves `archix.lock` as it is:

```console
error: archix.lock for laptop has packages built for a newer CPU level than this machine runs, so it was locked elsewhere: zlib (x86_64_v4), bash (x86_64_v4). Run `archix lock` on this machine; nothing was changed
```

The fix is `archix lock` on that machine. It treats such a lock as out of date even when the config hasn't changed, and resolves it again for its own CPU. So a desktop with x86-64-v4 can lock a v3 laptop's host; the laptop's switch refuses it, `archix lock` there re-locks it for v3, and you commit the result. A switch never rewrites the lock it's applying for this reason, and `diff` and `status` don't mention it.

## Three kinds of repo, three ways to lock

archix locks each kind of package with the strongest guarantee its source can give.

| | Official repos | AUR | Third-party repos |
| --- | --- | --- | --- |
| Examples | `core`, `extra`, `multilib` | any AUR package | `cachyos-v3`, `chaotic-aur`, `endeavouros` |
| Locked by | Arch Linux Archive snapshot date, plus exact version and sha256 | Build recipe commit | Exact version and sha256 of the package file |
| Where the file comes from | The ALA, for any date and any version | Built in a clean chroot from the locked commit, or a package cache's build of it | archix's own cached copy, a package cache, or the repo's servers |
| Restorable on a fresh machine | Always | As long as the commit is there and builds | Only while the repo or a package cache still has that version |
| When it can't be restored | Doesn't happen | The nearest-version prompt | The fallback chain |

The official repos are the easy case. The Arch Linux Archive keeps every day of the repos and every package version ever published, so any lock can be restored anywhere.

The AUR is nearly as good. A recipe commit is permanent unless the package is deleted, and the build is redone from that commit at the locked snapshot date.

Third-party repos are the hard case.

## Why third-party versions can go missing

Third-party repos have no dated archive. Most of them, CachyOS included, keep only the current version of each package. When a new build lands, the old one is gone from the mirrors.

So archix keeps its own copy. Every package file a generation takes from a third-party repo stays in `/var/cache/archix/pkg`, checked against the sha256 in the lock, and `gc` keeps it for as long as a kept generation needs it. That includes packages archix didn't install itself, such as the ones `archix init` found on the machine: the switch that makes the generation copies them from pacman's cache, or downloads them while the repo still has them. On that machine, rollback and reinstalls work exactly as they do for official packages: the file is right there.

When the copy can't be made, the switch says so and goes on: `warning: couldn't keep a copy of linux-cachyos 6.17.2-2, so a rollback to this generation needs cachyos to still have it`. A rollback that rebuilds a generation and finds no copy asks the servers `pacman.conf` lists for that repo.

The gap is a different machine, or the same machine after a reinstall. Its cache is empty. If the repo has moved on since you made the lock, the exact `linux-cachyos 6.17.2-2` in your lock doesn't exist anywhere archix can reach. No tool can fix that without an archive the repo doesn't provide. archix is upfront about it and gives you a predictable way forward.

You can provide that archive yourself. Run `archix cache push` on the machine that has the files, into a directory your other machines list in [`package_caches`](/docs/reference/options/#package_caches), and they fetch the locked file from there, checked against the lock's `sha256`, before they ever ask the repo. See [Package caches](/docs/concepts/lockfile/#package-caches).

## The fallback chain

When a locked third-party package is gone, not cached and not on the repo's servers, archix falls back in this order:

1. **The repo's current version.** This keeps you on the same optimizations, for example still the `x86-64-v3` build.
2. **The AUR.** For packages that also exist there, such as most of chaotic-aur, archix locks the AUR package, shows its recipe for review, and builds it in a clean chroot.
3. **The official repos.** For CachyOS rebuilds of official packages, such as `mesa` from `cachyos-extra-v3`, archix installs Arch's own build from the locked snapshot. You lose the CachyOS optimizations for that package but keep a working, locked system.

Whatever it picks is written back to the lock, so the next machine gets the same answer.

### Asking first

Falling back changes what you run, so by default archix asks. The prompt defaults to falling back:

```console
$ sudo archix switch
...
linux-cachyos 6.17.2-2 is gone from cachyos-core-v3
fall back to its current version, the AUR, or the official repos? [Y/n]
using linux-cachyos 6.17.4-1 from cachyos-core-v3
```

Press Enter and archix uses the fallback. Answer `n` and the switch stops before anything is installed: `aborted; nothing was changed`.

The [`third_party_fallback`](/docs/reference/options/) option sets the behaviour, and when you have third-party repos, `archix init` asks about it in its Packages step, as `If a locked package disappears from a third-party repo`:

| Value | Behaviour |
| --- | --- |
| `"ask"` | Ask each time, defaulting to falling back. With `--noconfirm`, there's no one to ask, so the switch fails. This is the default. |
| `"fallback"` | Fall back without asking, `--noconfirm` or not. |
| `"fail"` | Stop the switch: `linux-cachyos 6.17.2-2 is gone from cachyos-core-v3; third_party_fallback = "fail"`. |

An unattended run never swaps what you run or rewrites the lock on its own. With `"ask"` and `--noconfirm`, a timer or script running `archix switch --noconfirm` stops before anything changes:

```console
linux-cachyos 6.17.2-2 is gone from cachyos-core-v3
error: linux-cachyos 6.17.2-2 is gone from cachyos-core-v3; nothing was changed. Run without --noconfirm to choose a replacement, or set third_party_fallback = "fallback"
```

Set `"fallback"` if unattended runs should fall back too.

The setting lives in your config, so it applies to every machine that uses the config.

## Tips for CachyOS

- **Let `archix init` import the repos.** On CachyOS, the third-party repos aren't extras. They supply the kernel and many core packages.
- **Add `keys` before you use the config on a second machine.** On the first one, the keys are already in pacman's keyring.
- **Switch the second machine soon after locking, or share a package cache.** Third-party versions are most likely to still exist right after the lock was made. The longer you wait, the more packages hit the fallback chain, unless the first machine pushed them to a [package cache](/docs/concepts/lockfile/#package-caches) the second one lists.
- **Keep the default prompt at first.** Watching what falls back where tells you how much of your system depends on CachyOS-only builds.
- **Expect conflicting packages to swap.** CachyOS replaces some core packages with its own, such as `zlib-ng-compat` for `zlib`, which declares `conflicts=zlib`. Before anything changes, a switch checks that no two packages ship the same file, and it counts a package another one conflicts with or replaces as leaving, in either direction, the way pacman swaps them. So moving between `zlib` and `zlib-ng-compat` works, and a real clash stops the switch with nothing changed; see [Safety](/docs/concepts/safety/#prepare-then-build).
- **Commit the lock after a fallback.** The chosen versions are written back, and committing them saves your other machines from the same question.
- **Package scripts get no network while staging.** CachyOS's pacman runs every install scriptlet and hook in a network namespace of its own, which it can't create inside archix's staging root. archix detects such a pacman, by whether it takes `--disable-sandbox-network`, and runs the whole transaction in a network namespace of its own instead, with only loopback, as it does for installing an AUR build's dependencies. So a pacman hook with `NetworkAccess = allowed` has no network while a switch builds the deployment. See [Limitations](/docs/concepts/limitations/#switching).
