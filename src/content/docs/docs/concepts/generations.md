---
title: Generations and rollback
description: What a generation and its deployment contain, how archix rolls back from a running system or from the boot menu, and how old deployments and generations are cleaned up.
sidebar:
  order: 3
---

Every `archix switch` that changes something creates a generation: a numbered, complete record of what the system looks like after that switch, and a [deployment](/docs/concepts/atomicity/) that holds it, ready to boot. Rolling back means booting an older generation again.

## What a generation contains

The record lives in `/var/lib/archix/generations/<n>/generation.json`, and the deployment in `/archix/deploy/<n>`. The record holds:

- **The packages.** The snapshot date and every package's exact version, file and sha256, including the sha256 of each AUR package it built.
- **The managed files, users, groups and services.** File contents are stored once by sha256 in `/var/lib/archix/blobs`, so identical files across generations cost nothing extra. Files downloaded with [`archix.fetchurl`](/docs/reference/lua-api/#archixfetchurl), AppImages included, are stored the same way.
- **The hooks.** Every [hook](/docs/guides/hooks/) and `on_change` script.
- **The Flatpak apps.** The remote, ref and commit each declared app was pinned to.
- **The settings it was applied with.** `undeclared_packages`, `ignored_packages`, the generation and deployment limits and the boot settings, so rollback and `gc` work without evaluating your config.
- **The config's git commit**, if your config directory is a git repo, and whether the tree had uncommitted changes.
- **A format version.** An archix that finds a record written in a newer format than it knows won't rewrite it, or rebuild that generation's dropped deployment, since it would drop what it doesn't know: `generation 44's record was written by a newer archix (record format 2; this archix knows up to 1); boot a deployment whose archix is as new, or upgrade archix`. Booting an older deployment boots its older archix, which reads newer records and rolls back to kept deployments as usual.

The deployment is the installed result: `/usr`, `/opt`, `/boot`, `/etc`, pacman's database and DKMS state as that switch left them. `gc` drops old deployments sooner than old records; a generation whose deployment is gone can still be rolled back to, by building it again from its record.

A switch that changes nothing prints `nothing to do: the system matches generation 42 (snapshot 2026-10-15)` and creates none, though it still runs the generation's [hooks](/docs/guides/hooks/). Any other switch builds a new deployment, even one that only changes a setting.

```console
$ archix generations
    40  2026-10-02 18:40 UTC  snapshot 2026-10-02  210 packages  config 8c1f0e2a7b  record
    41  2026-10-09 09:12 UTC  snapshot 2026-10-09  212 packages  config 3f2a9c1e07  deployment, booted
*   42  2026-10-15 08:55 UTC  snapshot 2026-10-15  214 packages  config 5d7b21c4e9+uncommitted  deployment
```

`*` marks the current generation, the one the next boot starts. `deployment` marks generations whose deployment is kept, `booted` the one running now, and `record` those that only have their record. Here, generation 42 is committed but the machine hasn't rebooted into it yet. `archix status` says the same: `booted generation: 41; generation 42 boots at the next restart`.

## Rolling back from a running system

```console
$ sudo archix rollback
generation 41 (snapshot 2026-10-09) still has its deployment: rolling back points current and the boot default at it, nothing is reinstalled
downgrade  linux 7.2.9.arch1-1 -> 7.2.8.arch1-1
downgrade  mesa 1:25.3.1-1 -> 1:25.3.0-1
roll back to generation 41 (snapshot 2026-10-09)? [Y/n]
rolled back to generation 41 (snapshot 2026-10-09); it boots at the next restart. archix.lock is unchanged, so the next `archix switch` returns to the config
```

`rollback` goes to the previous generation, or the one you name; that works forwards too:

```console
$ sudo archix rollback 40
$ sudo archix rollback 42
```

- **When the generation's deployment is kept**, rollback makes it systemd-boot's default entry and `current`, and nothing is reinstalled. First it checks the deployment's `/usr`, `/opt` and `/boot` against its packages, as [`archix verify`](/docs/reference/cli/#verify) does, `:: checking deployment 41's files against its packages`, and repairs what differs from the package cache; if it can't, nothing is rolled back: `deployment 41 was changed outside archix; nothing was rolled back: ...`. The deployment boots as it was, with its own packages, `/etc`, kernel and modules, and the running system's users and passwords carried into its `/etc` by name; see [Users and passwords across deployments](/docs/concepts/safety/#users-and-passwords-across-deployments).
- **When its deployment was dropped**, rollback builds it again, from the nearest kept deployment and the generation's exact package files, from the cache or from the Arch Linux Archive, then commits it like a switch: `generation 38 (snapshot 2026-09-18)'s deployment was dropped: rebuilding it from deployment 41 and the package cache`. That takes as long as an update. AUR and local packages go back to the generation's own build, by its sha256. Packages are installed by the sha256 the generation pinned, without checking their signatures against today's keyring again, so a packager key revoked since doesn't block going back; see [Signatures and disk space](/docs/concepts/safety/#signatures-and-disk-space). The one exception is generation 1 when it records the system as archix found it, before a first switch that changed something: it has no record to rebuild from, so once its deployment is gone, rolling back to it stops with `generation 1 is the system as archix found it, and its deployment was dropped, so it can't be rebuilt; nothing was changed`.

Either way, the generation takes effect at the next boot, and its [hooks](/docs/guides/hooks/) run then. Files outside the deployment, such as dotfiles, are put back right away, and Flatpak apps are pinned back to the generation's commits, downloaded from their remote, with apps the newer generation added uninstalled:

```console
flatpak    com.spotify.Client 81c2d4e6f0a3 -> 5f8806197a2b
flatpak    remove org.gimp.GIMP
```

If a Flatpak step fails, it's listed like any [soft failure](/docs/concepts/safety/#hard-and-soft-failures), and the rest of the rollback stands. Apps archix never managed, and `--user` apps, are never touched.

Naming the current generation changes nothing: `generation 42 is current already; nothing was changed`.

`rollback` doesn't evaluate your config, so it works even when the config is broken.

### Downgrades that may break data

Packages roll back; their data doesn't. Data in `/var`, such as a database, is shared by every deployment, and some programs migrate the data they keep there when a new major version first starts, so the old version can't read it afterwards, or damages it. PostgreSQL refuses to start on a data directory a newer major version wrote. Rolling back across such a version is unsafe, so archix asks first, and the answer defaults to no:

```console
$ sudo archix rollback 40
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

Downgrades within a major version, such as `17.2-1` to `17.1-3`, pass without a question. For anything else, mark the versions yourself with [`unsafe_downgrades`](/docs/reference/options/#unsafe_downgrades): a downgrade from at or above a listed version to below it asks too, `lemon 3.54.0-1 -> 3.53.4-1 crosses 3.54, which unsafe_downgrades marks`. Each generation stores its `unsafe_downgrades`, so a rollback checks both the generation it leaves and the one it goes back to, every version either lists for a package, without evaluating your config.

Answer `y` only once the data is taken care of, such as restored from a dump the older version can read. `--noconfirm` refuses unsafe downgrades: ``refusing unsafe downgrades with --noconfirm; to go ahead, run `archix switch` or `archix rollback` with --allow-unsafe-downgrade``. `--allow-unsafe-downgrade` still lists them, but doesn't ask. A [switch](/docs/reference/cli/#switch) asks the same way, for example after `archix update --to` an earlier date. `install`, `remove` and `upgrade` don't take the flag.

Rollback changes the system, not your config. Your `init.lua` and `archix.lock` still describe the newer generation, so the next `archix switch` will apply them again. That's deliberate: archix never rewrites your config behind your back. Before switching again, fix the config, revert the lock in git, or move it back with `archix update --to`.

## Rolling back from the boot menu

Every kept deployment has its own boot entry. When a generation won't boot, or boots to a broken desktop, pick an older one in systemd-boot:

```
  Arch Linux (archix, generation 42)
› Arch Linux (archix, generation 41)
```

systemd-boot records the entry it started, `archix-41.conf`, in the `LoaderEntrySelected` EFI variable. archix's initramfs hook reads it and mounts deployment 41 before the system starts, so it boots with the same packages, `/etc`, kernel and modules it had, DKMS modules included; `/var` and your home are shared and stay as they are now. Nothing is reinstalled, and pacman doesn't run. Early in boot, `archix-boot.service` makes 41 current and systemd-boot's default, so the next reboot comes back to it:

```
archix: booted generation 41, but 42 was current; 41 is current now
```

It stays current until your next switch or rollback, and the running system's users and passwords are carried into its `/etc` by name. The deployment the last switch or rollback chose, 42 here, stays too: `gc` doesn't drop it while you run an older one. An entry archix has no record of writing, such as one copied or edited on the ESP, boots `current` instead.

A generation that can't boot at all doesn't need you at the menu. With systemd-boot 260 or newer, a new generation's entry gets [`boot.tries`](/docs/reference/options/#boottries) boots, three by default, to reach `boot-complete.target`. Each boot that doesn't get that far counts one down, and after the last one systemd-boot starts the last deployment that booted instead. archix makes it current and leaves a note for your next login: `archix: generation 42 didn't finish booting in the tries it had, so systemd-boot started generation 41, which is current now`. When the failed deployment ran newer versions of packages that migrate their data, such as PostgreSQL, the note adds a `WARNING` naming them. On servers, [`boot.health_check`](/docs/reference/options/#boothealth_check) can fail a boot that got that far, but whose units, ports or network aren't up.

Picking an older entry doesn't ask about [unsafe downgrades](#downgrades-that-may-break-data), and data in `/var` that a newer version migrated stays migrated.

## Boot entries

archix needs `boot.loader = "systemd-boot"`, and writes a boot entry for each kept deployment. Your `kernel_params` must contain `root=`. archix declares the packages it needs for this, the `boot.kernel` package, `mkinitcpio` and `systemd-ukify`, plus `sbctl` with `boot.secure_boot = true` and the kernel's headers when a DKMS package is declared, so you don't list them yourself.

- Each entry is a small file on the ESP, `/efi/loader/entries/archix-42.conf`, titled "Arch Linux (archix, generation 42)", that starts a unified kernel image (UKI) in `/efi/EFI/archix/`. A UKI is built with `ukify` from the deployment's own kernel and a fresh `mkinitcpio` initramfs made inside it, and named after its kernel and a hash of what it's built from: the kernel, microcode, initramfs, command line and os-release, and the [PCR signing key](/docs/guides/secure-boot/#tpm-bound-secrets) when there is one, as in `7.2.9-arch1-1-aedff12d49d2.efi`. Deployments whose UKI comes out the same share one. The UKI's command line doesn't name a generation; the initramfs reads it from the entry systemd-boot started. A UKI is deleted once no entry starts it.
- A new deployment's entry carries systemd-boot's boot counter, `archix-42+3.conf`, and `loader.conf` names it as `preferred`, with `default` on the newest deployment that has booted, which `gc` never drops. Once a boot of it reaches `boot-complete.target`, `systemd-bless-boot` renames it `archix-42.conf`, and `archix-boot-complete.service` then makes it the `default`; after three that don't, it's `archix-42+0-3.conf`, and systemd-boot passes it over. archix reads whether the running systemd-boot can count, 260 or newer, from its `LoaderFeatures` variable, and writes plain `archix-42.conf` entries otherwise, which `switch`, `status` and `secure-boot check` say: ``boot counting is off: the systemd-boot that started this boot is older than 260, so a generation that doesn't boot isn't passed over; `sudo bootctl update` installs the one your systemd package has``. See [`boot.tries`](/docs/reference/options/#boottries).
- Every other kernel package installed in the deployment, such as `linux-lts` beside `boot.kernel = "linux"`, gets its own UKI and entry, `archix-42-linux-lts.conf`, titled "Arch Linux (archix, generation 42, linux-lts)". Only the `boot.kernel` entry is counted. DKMS builds its modules for each kernel, so each entry boots its kernel with its own build.
- The entry is written while the deployment is committed, so a switch that fails never gets one.
- One more copy of the current deployment's UKI, `EFI/Linux/archix-fallback-42.efi`, is listed by systemd-boot as "archix (fallback)" and starts when `loader.conf` names no entry it has, as when a crash lost `loader/entries`. It boots `current`, and follows it on every switch, rollback and boot that changes it. See [The fallback boot image](/docs/concepts/atomicity/#the-fallback-boot-image).
- Before a switch or rollback builds anything, archix checks that `boot.esp` is a mounted FAT filesystem, the one systemd-boot started from when it says, and that it can hold the new UKIs: one per kernel, the protected one, and the fallback copy. When space runs short at the commit, entries of deployments `deployments.keep` would drop go first. If it doesn't fit, the switch stops before changing anything: ``the ESP at /efi has 152 MiB free (40 MiB more after dropping the deployments it may), but this switch's boot entries need about 210 MiB; nothing was changed. Free up space on it, or set deployments.keep lower``. 1 GiB holds a switch beside the booted and current deployments; see [Storage](/docs/concepts/atomicity/#storage).
- Whenever archix writes `loader.conf`, it clears the `LoaderEntryDefault` and `LoaderEntryPreferred` EFI variables, which the boot menu or `bootctl set-default` set and which override `loader.conf`, and says so: `archix: cleared LoaderEntryDefault (archix-40.conf), set in the boot menu or with bootctl, which overrode loader.conf and would have kept archix-42+3.conf from booting`. A one-shot entry from `bootctl set-oneshot` is left alone and reported.
- With [`boot.protected_entry = true`](/docs/reference/options/#bootprotected_entry), each deployment also gets a protected entry, `archix-42-protected.conf`, titled "Arch Linux (archix protected, generation 42)", with the kernel command line anti-cheat software asks for. See [The protected entry](/docs/guides/secure-boot/#the-protected-entry).

Entries archix doesn't build, such as one for memtest86+, go in [`boot.extra_entries`](/docs/reference/options/#bootextra_entries), written to `<esp>/loader/entries/`.

GRUB and Limine are not supported, and archix refuses to switch without systemd-boot: they boot a kernel from `/boot` under one name that every deployment would share. See [Requirements](/docs/concepts/atomicity/#requirements).

### Secure Boot

archix signs every UKI with `sbctl` when `boot.secure_boot` is `true`, or by default whenever `sbctl` has keys. It also signs systemd-boot itself, so you can keep Secure Boot enforcing without lanzaboote or a hand-written hook. `sudo archix secure-boot setup` creates the keys and enrolls them while the firmware is in setup mode. Turning Secure Boot on counts as a settings change, so the next switch writes a newly signed entry.

With Secure Boot enforcing, archix never commits an entry the firmware would refuse: a switch whose entry wouldn't be signed is refused before it builds anything, and the built image and the loader are checked for a signature before the commit. See [Secure Boot](/docs/guides/secure-boot/).

Booting older deployments, rollback by repointing and by rebuilding, Secure Boot enforcing with keys enrolled by `archix secure-boot setup`, the protected entry and refusing an unsigned switch are tested by booting archix-managed deployments in QEMU VMs with OVMF firmware, on ext4 and btrfs. A second VM suite covers rollback across a real glibc and kernel update: from the boot menu and at runtime, after `gc`, killed and reset halfway, on a full disk, and with a broken glibc in a new deployment.

## Limits and `gc`

Deployments cost disk space, mostly what changed between them, and generation records cost their package files in the cache. archix cleans up after itself:

| What | Default |
| --- | --- |
| Deployments | [`deployments.keep`](/docs/reference/options/#deploymentskeep), 2: the booted, current and boot-default ones, the one the fallback image boots and the one the last switch or rollback chose always, then the newest others, up to that many |
| Boot entries | One for each kept deployment and installed kernel, plus a protected one with `boot.protected_entry`, and the fallback image of `current` |
| Generation records | The current one, those with a deployment, the newest 10, and every one younger than 30 days |
| Package cache, managed-file contents (downloads included), snapshot databases | Only what kept generation records need, plus snapshot databases fetched in the last day, which an `archix lock` running at the same time may be using |
| When | `gc` runs after every switch, and after a rollback that rebuilds |

With `deployments.keep = 2`, the deployment you're running and the previous one stay, so a rollback is instant. With 1, the previous deployment stays only until the new one has booted, and older generations are rebuilt from the cache. Set [`generations.keep`](/docs/reference/options/#generationskeep) and `generations.keep_days` to change how far back a rollback can rebuild; `archix init` asks about them. To clean up by hand, for example after lowering a limit:

```console
$ sudo archix gc
deleted deployments 39
deleted generation records 30, 31
freed 812.4 MiB: 46 cached package files, 3 snapshot databases
```

A generation whose record has been collected can't be rolled back to, because its package files may be gone. If you want to keep one longer, raise the limits before it ages out.
