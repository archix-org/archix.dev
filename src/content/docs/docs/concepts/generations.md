---
title: Generations and rollback
description: What a generation contains, how archix rolls back from a running system or from the boot menu, and how old generations are cleaned up.
sidebar:
  order: 3
---

Every `archix switch` that changes something creates a generation: a numbered, complete record of what the system looked like after that switch. Rolling back means making the system match an older generation again.

## What a generation contains

Generations live in `/var/lib/archix/generations/<n>/generation.json`, and `/var/lib/archix/current` points at the current one. Each one records:

- **The packages.** The snapshot date and every package's exact version, file and sha256, including the sha256 of each AUR package it built.
- **The managed files, users, groups and services.** File contents are stored once by sha256 in `/var/lib/archix/blobs`, so identical files across generations cost nothing extra. Files downloaded with [`archix.fetchurl`](/docs/reference/lua-api/#archixfetchurl), AppImages included, are stored the same way. Every switch that records a generation stores the contents it needs, including one with nothing else to apply, so rolling back to any kept generation finds its files.
- **The hooks.** Every [hook](/docs/guides/hooks/) and `on_change` script, so a rollback runs the ones the generation was built with.
- **The Flatpak apps.** The remote, ref and commit each declared app was pinned to.
- **The settings it was applied with.** `undeclared_packages`, `ignored_packages`, the generation limits and the boot settings, so rollback and `gc` work without evaluating your config.
- **The config's git commit**, if your config directory is a git repo, and whether the tree had uncommitted changes.
- **Its btrfs snapshot**, on a btrfs root.

A switch that only changes settings, such as `ignored_packages` or a boot option, still creates a generation, so turning on Secure Boot gives you newly signed entries. A switch that changes nothing prints `nothing to do: the system matches generation 42 (snapshot 2026-10-15)` and creates none, though it still runs the generation's [hooks](/docs/guides/hooks/). A switch with nothing to apply but a new generation to record, such as the first one after `archix init` when archix doesn't manage booting, prints `switched to generation 1 (snapshot 2026-10-09); no package changes were needed`.

```console
$ archix generations
    40  2026-10-02 18:40 UTC  snapshot 2026-10-02  210 packages  config 8c1f0e2a7b
    41  2026-10-09 09:12 UTC  snapshot 2026-10-09  212 packages  config 3f2a9c1e07
*   42  2026-10-15 08:55 UTC  snapshot 2026-10-15  214 packages  config 5d7b21c4e9+uncommitted
```

`*` marks the current generation. A generation only becomes current in the last step of a switch, after everything else succeeded, by atomically replacing the `current` link. An interrupted switch never becomes current.

## Rolling back from a running system

```console
$ sudo archix rollback
downgrade  linux 7.2.9.arch1-1 -> 7.2.8.arch1-1
downgrade  mesa 1:25.3.1-1 -> 1:25.3.0-1
roll back to generation 41 (snapshot 2026-10-09) with 2 changes? [Y/n]
:: installing packages
...
:: making the new generation current
rolled back to generation 41 (snapshot 2026-10-09); archix.lock is unchanged, so the next `archix switch` returns to the config
```

This makes the system match the previous generation: its exact package versions are reinstalled, from the cache when they're there and from the Arch Linux Archive otherwise, and its files, users, groups and services are applied again. AUR and local packages go back to the generation's own build, by its sha256, even when today's build has the same version. Packages are installed by the sha256 the generation pinned, without checking their signatures against today's keyring again, so a packager key revoked since doesn't block going back; see [Signatures and disk space](/docs/concepts/safety/#signatures-and-disk-space). When archix manages booting, the generation you rolled back to becomes systemd-boot's default, so the next reboot doesn't go forward again. Units still using files the rollback replaced are [restarted](/docs/concepts/safety/#restarting-what-uses-replaced-files), as on a switch. Files come from the store, so a file declared with a `url` comes back even when the URL is gone. Its [hooks](/docs/guides/hooks/) run last, once it's current and its Flatpak apps are pinned, after the `on_change` scripts of files the rollback changed, under `:: running activation hooks` as on a switch. It goes through the same journal as a switch, so an interrupted rollback is as recoverable as an interrupted switch. See [Safety](/docs/concepts/safety/).

Flatpak apps follow last, once the generation is current. Each app is pinned back to the commit the generation recorded, and apps the newer generation added are uninstalled:

```console
flatpak    com.spotify.Client 81c2d4e6f0a3 -> 5f8806197a2b
flatpak    remove org.gimp.GIMP
```

Old commits are downloaded from their remote, so this part needs the network and a remote that still has the commit. If it fails, the failure is listed like a service that didn't start, and the rest of the rollback stands. Apps archix never managed, and `--user` apps, are never touched.

To go to a specific generation, name it. That works forwards too:

```console
$ sudo archix rollback 40
$ sudo archix rollback 42
```

Naming the current generation repairs it. `archix rollback 42` with 42 current prints `generation 42 is current; putting back whatever differs from it`, then reinstalls drifted packages, rewrites changed files, users and services, and reruns its hooks. That's also how you finish after [rescue mode](/docs/concepts/safety/#rescue-mode).

`rollback` doesn't evaluate your config, so it works even when the config is broken. It refuses while a switch is unfinished; `archix recover --rollback` is the way back from that.

### Downgrades that may break data

Packages roll back; their data doesn't. Some programs migrate the data they keep on disk when a new major version first starts, and the old version can't read it afterwards, or damages it. PostgreSQL refuses to start on a data directory a newer major version wrote. Rolling back across such a version is unsafe, so archix asks first, and the answer defaults to no:

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

Answer `y` only once the data is taken care of, such as restored from a dump the older version can read. `--noconfirm` refuses unsafe downgrades: ``refusing unsafe downgrades with --noconfirm; to go ahead, run `archix switch` or `archix rollback` with --allow-unsafe-downgrade``. `--allow-unsafe-downgrade` still lists them, but doesn't ask. A [switch](/docs/reference/cli/#switch) asks the same way, for example after `archix update --to` an earlier date, and so does `archix recover --rollback`, which takes `--allow-unsafe-downgrade` too. `install`, `remove` and `upgrade` don't take the flag; with `--noconfirm` they adopt newer installed versions instead of going back, so they rarely downgrade at all.

Rollback changes the system, not your config. Your `init.lua` and `archix.lock` still describe the newer generation, so the next `archix switch` will apply them again. That's deliberate: archix never rewrites your config behind your back. Before switching again, fix the config, revert the lock in git, or move it back with `archix update --to`.

## Rolling back from the boot menu

When archix manages booting, every generation gets its own boot entry. When a generation won't boot, or boots to a broken desktop, pick an older one in systemd-boot:

```
  Arch Linux (archix, generation 42)
› Arch Linux (archix, generation 41)
  Arch Linux (archix, generation 40)
  Arch Linux (archix rescue, generation 42)
  Arch Linux (archix rescue, generation 41)
```

systemd-boot records the entry it started, `archix-41.conf`, in the `LoaderEntrySelected` EFI variable, and that's how archix knows which generation booted. Early in boot, once local filesystems are mounted and before `sysinit.target`, `archix-boot.service` sees that 42 is current and rolls back to 41 from the cache, like `archix rollback 41`:

```
archix: booted generation 41, but 42 is current; rolling back to 41
```

A generation that can't boot at all doesn't need you at the menu. With systemd-boot 260 or newer, a new generation's entry gets [`boot.tries`](/docs/reference/options/#boottries) boots, three by default, to reach `boot-complete.target`. Each boot that doesn't get that far counts one down, and after the last one systemd-boot starts the last generation that booted instead; archix rolls back to it the same way and leaves a note for your next login: `archix: generation 42 didn't finish booting in the tries it had, so systemd-boot started generation 41, and archix rolled back to it`.

You land in generation 41, and it stays current until your next switch. It also becomes systemd-boot's default entry, so a plain reboot comes back to 41 instead of going forward to 42 and rolling back again. Nothing has started yet that early, so the boot-time rollback skips restarting units, and the services it starts or stops are queued with `systemctl --no-block` rather than waited for: a unit waiting for `sysinit.target` would otherwise wait for the rollback, which waits for it. They run as the boot carries on. This works the same on every filesystem.

Flatpak commits come from their remote, and the network isn't up that early in boot, so the boot service leaves Flatpak apps as they are and says so:

```
archix: Flatpak apps aren't pinned at boot; run `sudo archix rollback 41` once you're online
```

[Hooks](/docs/guides/hooks/#when-they-run) and `on_change` scripts are skipped too, since nobody is logged in yet:

```
archix: hooks don't run at boot; the next switch or `sudo archix rollback 41` runs them once you're online
```

The `on_change` scripts it skipped are recorded, script and all, in `/var/lib/archix/pending-on-change`, and the next switch, rollback or recovery runs them, even if its own config dropped them. See [Activation hooks](/docs/guides/hooks/).

Naming the current generation puts back whatever still differs, Flatpak apps included, and runs its hooks.

A rollback from the boot menu doesn't ask about [unsafe downgrades](#downgrades-that-may-break-data). You picked the entry because the newer generation didn't work, and there's no one to answer that early, so it goes ahead, the way NixOS does. If it went back across one, archix leaves a note that's shown at every login until the next switch or rollback: in login shells, by `/etc/profile.d/archix.sh`, and as a desktop notification, through `notify-send`:

```
archix: the boot-menu rollback to generation 41 went back across versions that may not read newer data; check these services:
  postgresql 17.2-1 -> 16.6-1 goes back from major version 17 to 16, and postgresql migrates its data between them
```

### Older kernels keep booting

An older entry needs its kernel's modules, which pacman deletes when it upgrades the kernel. archix keeps a hardlinked copy of each entry's modules in `/var/lib/archix/kernels`, refreshed at every switch so it carries the DKMS modules built since, and the boot service puts the booted kernel's modules back. A modules directory without its `modules.dep` counts as gone too: that's what pacman leaves when files no package owns, such as DKMS's build directories, keep it from deleting the directory, and nothing loads from it. archix restores the copy into it and keeps what was left there. udev and `systemd-modules-load` may already have run without them by then, so after restoring them it restarts `systemd-modules-load.service` and re-triggers udev, and devices whose drivers were missing get another go. `gc` deletes the copies of kernels no kept boot entry uses.

When the rollback reinstalls a DKMS package, DKMS's pacman hook rebuilds its module for the booted kernel, which needs that kernel's headers. Declaring a DKMS package with `boot.loader` set also declares `boot.kernel`'s headers, so each generation has the ones for its own kernel. A VM test declares only `v4l2loopback-dkms`, rolls it back with linux 7.2 to 7.0 from the boot menu, and loads the module DKMS rebuilt.

DKMS won't install a build over a module file of the same version that's already there, and the kept copy archix put back is one. It would then list the module as only `built`, and leave its files behind when the package goes. So after pacman installs, archix sets those copies aside and has DKMS install its build: `dkms status` shows the module `installed`, and uninstalling the package removes it. A second VM test does this with NVIDIA's drivers. It rolls `nvidia-open-dkms` 615.78.08 back to 610.43.02 with linux 7.2 to 7.0 from the boot menu. It goes back to a generation without the driver at runtime and from the boot menu, and builds the closed `nvidia-580xx-dkms` from the AUR and rolls that back too. Each time, the module DKMS built for the running kernel is the generation's version, and no `nvidia` module file is left once the driver is gone. The VM has no NVIDIA GPU, so the test checks the modules DKMS built, not that they drive a card.

That's separate from what happens to the *running* kernel during a switch. archix also copies the running kernel's modules into `/var/lib/archix/kernels` before pacman replaces its package, and puts them back in `/usr/lib/modules` so they keep loading until you reboot. Both are temporary: once that kernel isn't running, archix deletes the put-back directory and the copy, at the next switch or rollback, or at the next boot from an archix boot entry. A copy a boot entry still uses stays. See [The running kernel](/docs/concepts/safety/#the-running-kernel).

### When pacman itself is broken

Rolling back at boot needs a working pacman and glibc to reinstall packages. If a bad update broke those, boot the generation's **archix rescue** entry instead. Rescue mode restores the core packages from the cache without pacman, in the initramfs, before the system starts. [Safety](/docs/concepts/safety/#rescue-mode) has the details.

If a switch was interrupted, the boot service only reports it and changes nothing; run `sudo archix recover`.

## Boot entries

Set `boot.loader = "systemd-boot"` (`archix init` offers it when systemd-boot is installed) and archix writes a boot entry for each generation. Your `kernel_params` must contain `root=`. archix declares the packages it needs for this, the `boot.kernel` package, `mkinitcpio` and `systemd-ukify`, plus `sbctl` with `boot.secure_boot = true` and the kernel's headers when a DKMS package is declared, so you don't list them yourself.

- Each entry is a small file, `/boot/loader/entries/archix-42.conf`, titled "Arch Linux (archix, generation 42)", that starts a unified kernel image (UKI) in `/boot/EFI/archix/`. A UKI is built with `ukify` from a fresh `mkinitcpio` initramfs and named after its kernel and a hash of what it's built from: the kernel, microcode, initramfs, command line and os-release, and the [PCR signing key](/docs/guides/secure-boot/#tpm-bound-secrets) when there is one, as in `7.2.9-arch1-1-aedff12d49d2.efi`. mkinitcpio builds the same initramfs from the same files, so generations whose kernel and initramfs didn't change share one UKI, and a switch that only adds packages writes no new one; ten such switches leave one 74 MiB UKI and ten entries. The UKI's command line doesn't name a generation, so it can be shared, and it stays embedded and signed under Secure Boot. A UKI is deleted once no entry starts it.
- The kernel is the one the installed kernel package owns, per pacman's database, not one archix kept around for the running kernel, so a switch after a kernel update without a reboot still gets the new kernel.
- A new generation's entry carries systemd-boot's boot counter, `archix-42+3.conf`, and `loader.conf` names it as `preferred`, with `default` on the newest generation that has booted, which pruning and `gc` never take. Once a boot of it reaches `boot-complete.target`, `systemd-bless-boot` renames it `archix-42.conf`, and `archix-boot-complete.service` then makes it the `default`; after three that don't, it's `archix-42+0-3.conf`, and systemd-boot passes it over. archix reads whether the running systemd-boot can count, 260 or newer, from its `LoaderFeatures` variable, and writes plain `archix-42.conf` entries otherwise, which `switch`, `status` and `secure-boot check` say: ``boot counting is off: the systemd-boot that started this boot is older than 260, so a generation that doesn't boot isn't passed over; `sudo bootctl update` installs the one your systemd package has``. See [`boot.tries`](/docs/reference/options/#boottries).
- The entry is written in the step before the generation becomes current, after the result has been verified, so a switch that fails never gets one.
- Only the newest [`boot.entries`](/docs/reference/options/#bootentries) generations (5 by default) keep an entry, and fewer if the ESP runs short of space. `gc` prunes the rest. The entries of the current generation and of the generation you booted are never pruned.
- Before a switch changes anything, archix checks that the ESP can hold the new entries. It estimates their size from the newest existing UKI, as if this switch needed new ones, times one for the normal entry plus one each for a rescue and a protected entry, plus a margin, and compares that with the free space plus the UKIs that removing older archix entries may free, which leaves out the current generation's and the booted one's. When space runs short, writing the entry removes the oldest entries, but never the new one, the current generation's or the one you booted. With no entry yet, as on the first switch with boot entries, the estimate is the size of `boot.kernel`'s `vmlinuz` plus 64 MiB for each entry, plus a margin, against the free space where the ESP directory will be. If it doesn't fit, the switch stops: `the ESP at /boot has 38 MiB free (112 MiB more after removing the older archix entries it may), but this switch's boot entries need about 196 MiB; nothing was changed. Free up space on it, or set boot.entries lower`.
- The newest two generations also get a rescue entry, `archix-42-rescue.conf`, starting a rescue UKI with `archix.rescue` on its command line, titled "Arch Linux (archix rescue, generation 42)". Rescue entries need the static binary at `/usr/lib/archix/archix-static`, which the AUR packages install; see [Installation](/docs/installation/#the-static-rescue-binary).
- With [`boot.protected_entry = true`](/docs/reference/options/#bootprotected_entry), the newest two generations also get a protected entry, `archix-42-protected.conf`, titled "Arch Linux (archix protected, generation 42)", with the kernel command line anti-cheat software asks for. See [The protected entry](/docs/guides/secure-boot/#the-protected-entry).

Entries from before UKIs were shared, `/boot/EFI/Linux/archix-41.efi` with `archix.generation=41` on their command line, keep booting and are pruned like the others.

Entries archix doesn't build, such as one for memtest86+, go in [`boot.extra_entries`](/docs/reference/options/#bootextra_entries), written to `<esp>/loader/entries/`. `files` refuses paths under `/boot`, so that's the way to add them.

GRUB and Limine are not supported. With them, rollback from a running system still works, but archix doesn't write generation boot entries.

### Secure Boot

archix signs every UKI with `sbctl` when `boot.secure_boot` is `true`, or by default whenever `sbctl` has keys. It also signs systemd-boot itself, so you can keep Secure Boot enforcing without lanzaboote or a hand-written hook. `sudo archix secure-boot setup` creates the keys and enrolls them while the firmware is in setup mode. Turning Secure Boot on counts as a settings change, so the next switch writes a newly signed entry.

With Secure Boot enforcing, archix never makes an entry the firmware would refuse the default: a switch whose entry wouldn't be signed is refused before anything changes, and every built entry and the loader are checked for a signature before the entry is copied to the ESP. See [Secure Boot](/docs/guides/secure-boot/).

Booting older generations, Secure Boot enforcing with keys enrolled by `archix secure-boot setup`, the protected entry, refusing an unsigned switch, and the rescue entry are tested by booting archix-managed generations in a QEMU VM with OVMF firmware in CI. A second VM suite covers rollback itself: from the boot menu and at runtime across a real glibc and kernel update, after `gc`, killed halfway, on a full disk, and staying the default across a reboot.

## btrfs snapshots

On a btrfs root, archix takes a read-only snapshot of `/` right before applying each switch, in `/.archix-snapshots/<generation>-<time>`. Turn it off with `boot.snapshots = false`.

These snapshots are manual recovery points. Rollback, at the command line or from the boot menu, never restores them; it reinstalls packages and files the same way on every filesystem. A snapshot is there for when you want something back by hand: copy a file out of it, or, from a live USB, make a writable snapshot of it and boot that as your root. It only covers the subvolume mounted at `/`, not nested ones such as a separate `/home`. Each snapshot is deleted with its generation.

## Limits and `gc`

Generations cost disk space, mostly in the package cache. archix cleans up after itself:

| What | Default |
| --- | --- |
| Generations kept | The current one, the newest 10, and every one younger than 30 days |
| Boot entries | The newest 5, and fewer if the ESP runs out of space |
| Rescue entries | The newest 2 |
| Protected entries | The newest 2, with `boot.protected_entry` |
| Package cache, managed-file contents (downloads included), snapshot databases | Only what kept generations need, plus snapshot databases fetched in the last day, which an `archix lock` running at the same time may be using |
| Preserved kernel modules in `/var/lib/archix/kernels` | Only the kernels of kept boot entries |
| Running-kernel modules kept during a switch | Deleted once that kernel isn't running, at the next switch or rollback, or at the next boot from an archix boot entry, unless a boot entry still uses the copy |
| btrfs snapshots | Deleted with their generation |
| When | `gc` runs after every switch |

Set [`generations.keep`](/docs/reference/options/) and `generations.keep_days` to change the limits; `archix init` asks about them. To clean up by hand, for example after lowering a limit:

```console
$ sudo archix gc
deleted generations 30, 31
freed 812.4 MiB: 46 cached package files, 3 snapshot databases
```

`gc` never removes the current generation. A generation that has been collected can't be rolled back to, because its package files may be gone. If you want to keep one longer, raise the limits before it ages out.
