---
title: Coming from NixOS
description: Map NixOS concepts to archix, understand what works differently, and port an existing NixOS and Home Manager config.
sidebar:
  order: 9
---

You already know the ideas behind archix: a declarative config, a lockfile, generations and rollback, a typed module system with priorities. This page maps what you know onto archix, then covers what's different and how to port a config.

## Concept map

| NixOS | archix |
| --- | --- |
| `configuration.nix` | `init.lua` |
| Modules with `imports`, `options`, `config` | Modules with `imports`, `options`, `config` |
| `flake.lock` | `archix.lock` |
| `nixos-rebuild boot` | `archix switch`, which always takes effect at the next boot |
| `nixos-rebuild switch --rollback` | `archix rollback` |
| `nix-env --list-generations` | `archix generations` |
| `nixos-rebuild dry-activate` | `archix diff` |
| `nix flake update` | `archix update` |
| `nix eval` of a config | `archix eval` |
| `nixos-option`, the options search | `archix options` |
| `lib.mkDefault` | `archix.default(v)` |
| `lib.mkForce` | `archix.force(v)` |
| `lib.mkOverride n` | `archix.override(n, v)` |
| `null` | `archix.null` |
| `lib.mkOption { type = ...; }` | `archix.option { type = ... }` |
| `types.listOf`, `attrsOf`, `nullOr`, `enum`, `submodule` | `archix.types.list_of`, `attrs_of`, `nullable`, `enum`, `submodule` |
| `builtins.readFile ./x` | `archix.read_file("x")` |
| A derivation in `pkgs/*.nix` | A PKGBUILD in `pkgbuilds/<dir>/` |
| `overrideAttrs (old: { patches = old.patches ++ [ ./fix.patch ]; })` | `patches.<package> = { "patches/fix.patch" }` |
| `pkgs.fetchurl { url; sha256; }` | `archix.fetchurl { url = ..., sha256 = ... }`, as a file in `files` or `home.files` |
| `nix-prefetch-url` | `archix prefetch` |
| `services.flatpak` with nix-flatpak's `packages` | `flatpak.apps`, locked to commits in `archix.lock` |
| `networking.hostName` per host | `archix.hostname`, from the machine |
| Home Manager `programs.*` | `programs.*`, ported from Home Manager |
| `home.file` | `home.files` |
| `home.file` being read-only in the Nix store | `mutable_files = false` |
| `config.lib.file.mkOutOfStoreSymlink` | `home.files.<path> = { source = ..., live = true }` |
| plasma-manager's `programs.plasma.configFile` | `home.files.<path>.merge` with `format = "kconfig"` |
| plasma-manager's `immutable = true` | `mutable_files = false`, which writes merged kconfig keys as `Key[$i]` |
| `environment.etc` | `files` |
| `systemd.services`, `systemd.user.services` | `units`, `services`, `home.systemd` |
| `users.users` | `users` |
| `environment.variables`, `environment.sessionVariables` | `environment.variables` |
| `environment.shellAliases` | `environment.shell_aliases` |
| `networking.firewall.enable` | `firewall.enable` |
| `networking.firewall.allowedTCPPorts`, `allowedUDPPorts` | `firewall.allowed_tcp_ports`, `allowed_udp_ports` |
| `networking.firewall.allowedTCPPortRanges`, `allowedUDPPortRanges` | `firewall.allowed_tcp_port_ranges`, `allowed_udp_port_ranges` |
| `networking.firewall.trustedInterfaces` | `firewall.trusted_interfaces` |
| `networking.firewall.allowPing` | `firewall.allow_ping` |
| `networking.firewall.checkReversePath` | `firewall.check_reverse_path` |
| `networking.firewall.extraInputRules` | `firewall.extra_input_rules` |
| `system.activationScripts`, Home Manager's `home.activation` | `hooks` |
| Home Manager's `home.file.<path>.onChange` | `home.files.<path>.on_change`, and `on_change` on `files` |
| `nix-collect-garbage` | `archix gc` |
| nixpkgs | The Arch repos, the AUR and third-party repos |
| Generations in the boot menu | Deployments in the systemd-boot menu |
| The store's system closures | Deployments in `/archix/deploy` |

The priorities use the same numbers: option defaults are 1500, `default` is 1000, plain values 100, `force` 50. If you've reasoned about `mkOverride` before, the same reasoning applies.

## What's different

### No derivations

archix installs Arch's binary packages. There's no store, no derivations, no build closure to evaluate, and nothing to wait for while nixpkgs catches up. The filesystem is a normal FHS layout, so binaries from the internet, language toolchains and proprietary apps just work.

Custom builds still live in your config. A package you'd write as a derivation in `pkgs/*.nix` becomes a PKGBUILD in `pkgbuilds/<dir>/`, and declaring its name is enough. An `overrideAttrs` that adds `patches` becomes [`patches.<package>`](/docs/reference/options/#patches): archix rebuilds Arch's own recipe for the locked version with your patches. Both are locked by a hash of the recipe and built in the same clean chroot as AUR packages. Other changes to how an official package is built, such as different configure flags, mean copying its recipe into `pkgbuilds/`. See [Local PKGBUILDs and patches](/docs/guides/local-pkgbuilds-and-patches/).

### A lock of versions, not of inputs

`flake.lock` pins the source of nixpkgs, and everything follows from evaluation. `archix.lock` pins outputs: an Arch Linux Archive date, plus the exact version and checksum of every installed package. The effect is the same, reproducible installs, but the lock is also readable as a list of what's on the machine. One lockfile covers all hosts, and each host can sit on its own date.

### Lua, not Nix

Lua is eager. Where Nix gets `config.foo` references from laziness, archix re-runs function modules until the merged config stops changing:

```nix
# NixOS
config = lib.mkIf config.my.desktop.enable { environment.systemPackages = [ pkgs.firefox ]; };
```

```lua
-- archix
config = function(cfg)
  if not cfg.my.desktop.enable then return {} end
  return { packages = { "firefox" } }
end,
```

There's no `mkIf`, `mkMerge`, `mkBefore` or `mkAfter`. Use plain `if`, return a table, and rely on definition order: a module's own config, then its imports in order. Lists concatenate in that order. See [Configuration](/docs/concepts/configuration/#function-config-and-fixpoint-rounds).

Evaluation is still pure. The sandbox has no `io` or `os`, `pairs` iterates in sorted order, and the only escape hatches are `archix.hostname` and `archix.read_file`.

### A firewall in nft syntax

`firewall` follows `networking.firewall` with the nftables backend: the same options in snake_case, ranges as `{ from = 60000, to = 61000 }`, and `check_reverse_path` as `"strict"`, `"loose"` or `"off"`. It lives in its own `inet archix` table and leaves other tables alone. There's no iptables backend, so `extraCommands` and `extraStopCommands` written for iptables have to be rewritten as nft rules, in `extra_input_rules` for the input chain or `extra_rules` for chains and sets of your own. See [Environment and firewall](/docs/guides/environment-and-firewall/#firewall).

`environment.variables` reaches login shells and systemd user sessions, which covers both `environment.variables` and `environment.sessionVariables`. `environment.shell_aliases` reaches login shells; for every interactive zsh, use `programs.zsh.shell_aliases`.

### Writable files by default

Home Manager's files are symlinks into the read-only Nix store. archix writes real files, writable by default, so apps that save their settings keep working. Set [`mutable_files = false`](/docs/reference/options/#mutable_files) to get the NixOS behaviour: files lose their write bits, and every switch puts them back. For a file you want to keep editing in place, like `mkOutOfStoreSymlink`, use a [live link](/docs/guides/dotfiles-and-apps/#live-links-for-app-written-configs). For KDE and other apps that write their own config, [`merge`](/docs/guides/dotfiles-and-apps/#setting-single-keys) sets single keys the way plasma-manager does, and with `mutable_files = false` it locks kconfig keys with `[$i]`, like plasma-manager's `immutable`.

### Mutable passwords

archix follows NixOS's default of `users.mutableUsers = true` per user: `initial_hashed_password` is `initialHashedPassword`, set when the account is created, and `hashed_password` is `hashedPassword`, set again at every switch, with `archix diff` saying when that resets a password changed with `passwd`. Unlike NixOS, an enforced hash never unlocks an account locked with `passwd -l`, a password set in the rescue entry holds until the declared hash changes, and a rollback carries today's passwords rather than restoring an older generation's. There's no `hashedPasswordFile` yet; a secrets option will supply hashes from an encrypted file. A user without either keeps the password `passwd` gives it. See [Users, SSH and sudo](/docs/guides/users-and-access/).

Encrypted secrets, agenix- or sops-style, are not supported.

### pacman reads, archix writes

On NixOS, imperative changes mostly don't stick. On archix, they can't be made: `/usr`, `/opt` and `/boot` are read-only, so `pacman -S` and `yay` stop with a message pointing to archix, while `pacman -Q`, `-Ss` and `-F` work as usual. Install through archix, `archix -S`, `archix -Syu` or `archix install --temp` to try something undeclared, which, like everything else, takes effect at the next boot. See [pacman and yay](/docs/concepts/pacman-and-yay/).

### One machine at a time

`archix switch` applies to the machine it runs on. You can evaluate, diff and lock another host with `--host`, but remote switch over ssh, like `nixos-rebuild --target-host`, is not supported.

### Bootloaders

systemd-boot only, and archix needs it: every generation boots as its own deployment, from its own entry. GRUB and Limine are not supported. Each deployment gets a UKI, signed with sbctl if you have keys, so Secure Boot works without lanzaboote; `archix secure-boot setup` creates and enrolls the keys. See [Secure Boot](/docs/guides/secure-boot/). Picking an older generation in the boot menu boots its deployment as it was, as on NixOS, on ext4 as well as btrfs.

### Switches take effect at the next boot

`nixos-rebuild switch` activates the new generation on the running system and restarts what changed. archix never changes the running system: a switch builds the next deployment and commits it, and the machine runs the old one until you reboot, like `nixos-rebuild boot`. There's no `restartIfChanged`, `stopIfChanged` or `reloadIfChanged` to get right, and no window where new binaries meet old config. The exceptions are files in your home and Flatpak apps, which a switch writes right away. Changes you make to `/etc` before the reboot are merged into the new deployment, so nothing you set in between is lost. See [Atomic deployments](/docs/concepts/atomicity/).

### Rollback asks before unsafe downgrades

NixOS rolls back without asking, even when an older PostgreSQL can't read the data the newer one migrated. archix asks first, defaulting to no, when `switch` or `rollback` would take PostgreSQL, MariaDB, MySQL and a few others back across a major version, or cross a version you list in [`unsafe_downgrades`](/docs/reference/options/#unsafe_downgrades). `--noconfirm` refuses such downgrades unless you add `--allow-unsafe-downgrade`. Picking an older generation in the boot menu boots it without asking, as on NixOS. See [Downgrades that may break data](/docs/concepts/generations/#downgrades-that-may-break-data).

### Activation hooks run at boot

`hooks` are archix's activation scripts. Like `system.activationScripts`, they run after every switch, including one that changes nothing, so they have to be idempotent, and `after` orders them like `deps`. Each one is a bash script that runs as root, or as a `user` you name, which also covers what Home Manager's `home.activation` does. `on_change` on a `files` or `home.files` entry is Home Manager's `onChange`: it runs only when that file changed, right after the switch writes it for home files, at the first boot for files in `/etc`.

They run at the new deployment's first boot, before the network and before anyone logs in, so a hook can't reach a service, a network or a user session. A failure doesn't stop the boot or undo anything: archix runs the rest of the hooks, prints the failure on the console and at your next login, and runs them all again at the next boot, up to three boots, where Home Manager stops activation at the first failing script. See [Activation hooks](/docs/guides/hooks/).

### Deployments instead of a store

NixOS rolls back by pointing at an older closure that's still in the store. archix keeps the last few generations installed as deployments, 2 by default with [`deployments.keep`](/docs/reference/options/#deploymentskeep), and rolling back to one points the boot entry at it, as instantly. A generation whose deployment `gc` dropped is built again from the nearest kept one and its package files in the cache, which is why `gc` keeps the package files of every kept generation record, and why a collected record can't be rolled back to. Unlike a closure, a deployment doesn't include `/var`: data a newer version migrated stays migrated.

## Porting a config

The NixOS config ports by hand, module by module. These tips make it go faster.

**Start with `archix init`, not a translation.** Let onboarding capture your current Arch install into a working config first. Then bring over the structure and options from your NixOS config, one module at a time, running `archix diff` after each.

**Keep your module structure.** If your NixOS config has `hosts/` and `modules/`, the same split works in `lua/hosts/` and `lua/modules/`. `imports = [ ./common.nix ]` becomes `imports = { "modules.common" }`.

**Translate package names.** Most names match between nixpkgs and Arch, but not all. `pkgs.python3` is `python`, and many nixpkgs sets like `pythonPackages.requests` are `python-requests`. `archix search` finds the Arch name. Anything only in the AUR is still just a name in `packages`.

**Home Manager config ports almost directly.** archix's `programs.*` modules take their option names from Home Manager, in snake_case, and `settings` passes through, so most `programs.<name>` blocks become Lua tables with the same keys. Like Home Manager's per-user config, they apply to one user, which you name with `home.user`:

```nix
programs.kitty = {
  enable = true;
  settings = { font_size = 11; confirm_os_window_close = 0; };
};
```

```lua
users = { me = { groups = { "wheel" } } },
home = { user = "me" },
programs = {
  kitty = {
    enable = true,
    settings = { font_size = 11, confirm_os_window_close = 0 },
  },
},
```

**Your own options port too.** A custom NixOS module with `options.my.thing = mkOption { ... }` becomes:

```lua
local t = archix.types

return {
  options = {
    my = {
      thing = archix.option { type = t.nullable(t.str), default = archix.null, description = "What it does" },
    },
  },
}
```

**Leave services for last.** Packages and dotfiles are easy wins. Services, `/etc` files and kernel parameters touch the boot path, so port them once the rest is stable, and keep the previous generation's boot entry around while you do. See [Dotfiles and apps](/docs/guides/dotfiles-and-apps/) for the ported `programs.*` modules.
