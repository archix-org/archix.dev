---
title: pacman and yay
description: Why plain pacman and yay can't change a system laid out as deployments, the pacman-style commands that do, how adopting works, and how undeclared and ignored packages are treated.
sidebar:
  order: 4
---

Once archix has made its first switch, the running system's `/usr`, `/opt` and `/boot` are read-only: each switch builds a new [deployment](/docs/concepts/atomicity/) beside the running one, and the next boot starts it. So plain pacman and yay can't change the system any more. Everything that reads keeps working, `pacman -Q`, `-Ss`, `-Si`, `-F` and `-Sw` among them, but `pacman -S`, `pacman -Syu`, `pacman -R` and `yay` fail, since they'd write into the running deployment, which older deployments share files with. A pacman hook archix installs in every deployment, `00-archix-deployment.hook`, stops them before they start and says what to run instead:

```console
$ sudo pacman -S htop
...
archix: this system runs from a deployment whose /usr, /opt and /boot are read-only, so pacman, and AUR helpers like yay that run it, can't change its packages. Change them with archix, which builds a new deployment that boots at the next restart: `archix install <package>` (or `archix -S`) to add one, `archix remove <package>` (or `archix -R`) to take one out, `archix upgrade` (or `archix -Syu`) to update everything, and `archix switch` to apply your config. pacman -Q, -Ss, -Si, -F and -Sw work as before. See https://archix.dev/docs/concepts/pacman-and-yay/
```

pacman's own disk space check, `CheckSpace`, would otherwise stop the transaction first with `Partition /usr is mounted read only`, so a deployment's `pacman.conf` has it commented out; archix checks space itself. [`eject`](/docs/reference/cli/#eject) turns it back on.

Change packages through archix instead. It has the commands you already type:

| Instead of | Run | What it does |
| --- | --- | --- |
| `pacman -S htop`, `yay -S zen-browser-bin` | `archix -S htop` | Declares it in your config and builds a deployment with it |
| `pacman -Syu`, `yay` | `archix -Syu`, or bare `archix` | Moves the lock to the newest snapshot and AUR recipes, then switches |
| `pacman -R htop` | `archix -R htop` | Takes it out of your config and builds a deployment without it |
| `pacman -S` to try something | `archix install --temp htop` | Builds a deployment with it, undeclared; the next switch drops it |

Each of those takes effect at the next boot. There's no way to install a package into the running system: that's what keeps the deployments you'd roll back to intact.

What isn't a package can go in `/usr/local`, which stays writable: an installer run with `curl | sh`, `sudo make install`, `sudo npm install -g`, or a binary copied into `/usr/local/bin` works at once, in every deployment. It isn't part of any deployment, so no rollback undoes it, and `/usr/local/bin` comes before `/usr/bin` in `PATH`. A package that ships files into `/usr/local`, which Arch's packaging rules forbid, is refused by the switch. See [What `/usr/local` holds](/docs/concepts/atomicity/#what-usrlocal-holds).

## The pacman hook

archix installs `/etc/pacman.d/hooks/archix.hook`, which runs `archix hook-record` after every pacman transaction archix didn't run itself, and records it in `/var/lib/archix/outside-changes.json`:

```console
$ sudo pacman -Syu
...
:: Running post-transaction hooks...
(1/1) Recording package changes for archix...
archix: recorded 3 packages changed outside archix; `archix switch` will offer to adopt them
```

The hook never fails or changes a pacman transaction; it only takes notes. `archix status` reports them, and the next successful switch clears them. That matters for changes made before archix laid the system out, between `archix init` and the first switch.

## Adopting

On the next `archix switch`, if installed packages are newer than the lock, as after a `pacman -Syu` between `archix init` and the first switch, archix asks:

```console
$ sudo archix switch
the system is newer than the lock for 3 packages (linux, mesa, vulkan-radeon), probably from pacman or an AUR helper
adopt them into the lock? (no reverts them to the lock) [Y/n]
```

- **Yes** moves the lock to match the system. The switch then proceeds from there, and the upgrade you did with pacman becomes part of a generation you can roll back.
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
a new deployment with its own boot entry; it boots at the next restart
apply 2 changes? [Y/n]
```

Run as your user, archix asks sudo for the install itself. Commands win over terms, so `archix status` is never a search. See [`archix <terms>`](/docs/reference/cli/#search-and-install-archix-terms).

`-Sy` on its own, and `-S` or `-Ss` without arguments, fail with a hint: archix never syncs databases without upgrading. See the [CLI reference](/docs/reference/cli/#install--s) for every flag.

### `archix -Syu`

If you want the yay experience, use the full upgrade:

```console
$ sudo archix -Syu
```

It moves the lock to the newest Arch Linux Archive snapshot, moves AUR packages to the newest commit of their build recipes (showing each changed recipe for review), and switches. Running `archix` with no arguments does the same, the way `yay` does. It's `archix update` followed by `archix switch`, in one command. pacman can't upgrade a deployment in place, so `-Syu` no longer runs `pacman -Syu` against your live mirrors.

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

Adopting covers packages archix already tracks. It doesn't cover a package that isn't in your config at all, such as one you installed with pacman before the first switch. It's installed, but it isn't in the lock. The [`undeclared_packages`](/docs/reference/options/) option decides what happens to it, and `archix init` asks you to choose in its Packages step: `Later, when you install something with plain pacman or yay`.

**`"remove"`** is the default: `switch offers to declare it, and removes it if you say no`. The next switch lists the explicitly installed packages the pacman hook saw arrive since the last switch, and asks:

```console
$ sudo archix switch
installed with pacman or an AUR helper but not declared: htop ncdu
declare them in installed.lua? (no removes them) [Y/n]
```

- **Yes** adds them to [`installed.lua`](#installedlua) and switches with them in the lock, the same as if you'd run `archix install`.
- **No** removes them.

`--noconfirm` declares them. Anything else that isn't in the lock, such as a package installed before the hook could see it or one that `archix install --temp` installed, is left out of the next deployment. The system is exactly what the config says, which is the guarantee that makes rollback and a fresh reinstall meaningful.

This covers packages only. Changes to `/etc` files and services that archix never managed are left alone; see [Safety](/docs/concepts/safety/#managed-files-backups-and-hand-edits).

**`"warn"`** is `switch leaves it installed and only lists it`. Undeclared packages are listed in `archix diff` and `archix switch` instead:

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

To try something without declaring it, `sudo archix install --temp <pkg>` builds a deployment with it, undeclared on purpose, which boots at the next restart. archix ran that transaction itself, so switch doesn't offer to declare it: the next switch leaves it out unless you've declared it by then.

## Ignoring packages everywhere

Some packages you manage by hand on purpose. List them in [`ignored_packages`](/docs/reference/options/), and archix never touches or reports them, whatever `undeclared_packages` says:

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

Repos with `before_official = true` go in the first block, the rest in the second. archix rewrites only these blocks, in each new deployment's `/etc/pacman.conf`. A hand-written section for a repo your config declares is taken over, so it's never listed twice. Removing a repo from your config removes it from `pacman.conf`. Everything else in the file, including the official repos and their mirrorlists, is yours.
