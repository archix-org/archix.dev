---
title: Dotfiles and apps
description: Manage dotfiles and app configs with archix, from raw files and format writers to programs.* modules ported from Home Manager.
sidebar:
  order: 4
---

archix manages your home directory with the same switch that manages the system. Dotfiles become part of the generation, so a rollback takes your configs back too.

## Pick the user

archix manages one user's home per machine. Name that user in `home.user`, and declare it in `users` so archix creates the account before writing its files:

```lua title="lua/modules/common.lua"
return {
  users = { me = { groups = { "wheel" }, shell = "/usr/bin/zsh" } },
  home = { user = "me" },
}
```

The home directory is `users.me.home` if you set one, else `/home/me`; `home.directory` overrides both. Everything on this page, `home.files`, `home.systemd` and every `programs.*` module, applies to that user, and setting any of it without `home.user` is an error.

archix runs as root, but files it writes in the home, and the directories it creates for them, belong to the user.

## Home files

`home.files` is keyed by a path relative to the home; a leading `~/` is allowed. Each entry gives exactly one of `text`, `source`, `link`, `url` or `merge`:

```lua
home = {
  files = {
    [".config/user-dirs.dirs"] = { source = "home/user-dirs.dirs" },
    ["bin/backup"] = { text = "#!/bin/sh\nrestic backup ~/notes\n", executable = true },
    [".local/bin/vim"] = { link = "/usr/bin/nvim" },
  },
},
```

| Field | Meaning |
| --- | --- |
| `text` | The file's contents, written as given. |
| `source` | A file in the config directory, copied in. The path is relative to the config directory and can't leave it. |
| `live` | With `source`: link to the file or directory in the config instead of copying it. With `text`, `link` or `url` it's an error. |
| `link` | Make the path a symlink to this target. |
| `url`, `sha256` | Download the file, and refuse it unless it has this sha256. See [Files from a URL](#files-from-a-url). |
| `executable` | Mode `0755` instead of `0644`. |
| `on_change` | A bash script run as the user at the next boot, when a switch or rollback changed the file. See [Run a script when a file changes](#run-a-script-when-a-file-changes). |
| `mutable` | `false` writes the file read-only, `true` keeps it writable; by default as `mutable_files` says. See [Read-only files](#read-only-files). |
| `merge` | Set only some keys in a file the app also writes. See [Setting single keys](#setting-single-keys). |

Copies are real files, not links into a store, so apps see ordinary files they can read and watch. archix stores their contents with the generation and knows exactly what it wrote. If you, or an app, edit a managed file, `archix diff` flags it before the next switch:

```
update     /home/me/.config/kitty/kitty.conf (edited outside archix; your version gets saved)
```

The switch then writes the declared version, and saves yours under `/var/lib/archix/edited/` so nothing is lost. Move the change into your config to keep it.

When archix first takes over a path that already exists, it backs up the original to `/var/lib/archix/backups`. If the entry later leaves the config, archix deletes the file, or puts that original back. See [Safety](/docs/concepts/safety/).

## Run a script when a file changes

Home files live outside the [deployments](/docs/concepts/atomicity/), so a switch writes them as soon as its deployment is committed, without waiting for the reboot. Apps that watch their config pick the change up then; others read it when they next start.

For work that should follow a changed file, give it an `on_change` script, Home Manager's `onChange`. It runs as `home.user`, only when a switch or rollback changed that file, at the first boot of the deployment that switch made, before you log in:

```lua title="lua/modules/desktop.lua"
return {
  users = { me = { groups = { "wheel" } } },
  home = {
    user = "me",
    files = {
      [".config/fontconfig/fonts.conf"] = { source = "home/fonts.conf", on_change = "fc-cache -f" },
    },
  },
}
```

A file a `programs.*` module writes is a `home.files` entry too, so `on_change` works on it without repeating its contents, as `[".config/kitty/kitty.conf"] = { on_change = ... }`. The script runs with `set -e`. Nothing is running yet at that point in the boot, so a signal to reload a running app, such as `pkill -USR1 -x kitty`, reaches nothing: send it yourself after the switch if you want the change right away.

"Changed" means the file's entry differs from the previous generation's, such as new contents, mode or link target, or that archix had to write it again because the file on disk didn't match, such as putting back a file you edited by hand. So it runs after the first switch that writes the file, but never for edits behind a [live link](#live-links-for-app-written-configs): archix doesn't write what's behind the link. A failing script doesn't undo the switch; it's reported at boot and at your next login, and runs again at the next boot. For scripts that should run after every switch, see [Activation hooks](/docs/guides/hooks/).

## Files from a URL

Some files don't belong in your repo: a font, a prebuilt tool, a theme someone else publishes. Declare them by URL and sha256 instead, with `archix.fetchurl`. It works in `home.files` and in the system-wide `files` alike:

```lua title="lua/modules/common.lua"
return {
  users = { me = { groups = { "wheel" } } },
  home = {
    user = "me",
    files = {
      [".local/share/fonts/Inter.ttf"] = archix.fetchurl {
        url = "https://downloads.example.org/fonts/Inter-4.1.ttf",
        sha256 = "9f0c6a3d1b7e24c58a6d0e3f9b1c47a2e8d5f06b3c9a1e7d4f2b8c0a6e3d9f15",
      },
    },
  },
  files = {
    ["/usr/local/bin/hello"] = archix.fetchurl {
      url = "https://downloads.example.org/hello/1.2.0/hello-x86_64",
      sha256 = "5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03",
      mode = "0755",
    },
  },
}
```

You don't work out the hash yourself. `archix prefetch` downloads the file and prints the line to paste:

```console
$ archix prefetch https://downloads.example.org/hello/1.2.0/hello-x86_64
archix.fetchurl { url = "https://downloads.example.org/hello/1.2.0/hello-x86_64", sha256 = "5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03" }
```

The hash is what's locked. `archix diff` compares the file on disk with it and downloads nothing. `archix switch` downloads the file after you confirm, before it changes anything, and refuses it if the hash differs, so a URL that starts serving something else can't slip into your system. The contents are kept in archix's store with the generation, so a rollback brings the file back even after the URL is gone. To move to a new version, change the URL, run `archix prefetch` on it, and paste the new line.

For apps shipped as AppImages, the [`appimages`](/docs/guides/flatpak-and-appimages/#appimages) option does this for you, with a command on your `PATH` and a menu entry.

## Live links for app-written configs

Some files are written by apps on purpose. matugen regenerates color files from your wallpaper; settings dialogs save their state. Managing those as copies would fight the app on every switch.

For those, set `live = true`. The path in your home becomes a symlink into your config directory, the app writes through the link, and the change lands in your config repo where git can see it. Nothing is reported as drift, because the config is the file.

```lua
home = {
  files = {
    [".config/nvim"] = { source = "home/.config/nvim", live = true },
    [".config/matugen"] = { source = "home/.config/matugen", live = true },
  },
},
```

If the path is already a directory, such as the `~/.config/nvim` you had before you moved it into the config, the switch moves that directory to `/var/lib/archix/backups`, contents and all, and puts the link in its place. When the entry leaves the config, the link goes and the directory comes back. This works with a separate `/home` or a btrfs `@home` subvolume too. archix only does this once per path: if an earlier backup of it still exists, the switch stops before changing anything, with `/home/me/.config/nvim is a directory and an earlier backup of it exists; move one of them away`. See [Safety](/docs/concepts/safety/#managed-files-backups-and-hand-edits).

A `source` that's a directory has to be live; archix only copies single files. `live` needs a `source` to link to, so `live = true` with `text`, `link` or `url` makes `diff` and `switch` stop:

```console
error: home.files..config/nvim: live = true needs source, the file in the config to link to
```

## Read-only files

On NixOS, Home Manager's files live in the read-only Nix store, so nothing can change them behind your back. archix can work the same way. Set `mutable_files = false`, and every file archix writes from `text`, `source` or `url` loses its write bits: `0644` becomes `0444`, and an `executable` file `0555` instead of `0755`.

```lua title="lua/modules/common.lua"
return {
  mutable_files = false,
  users = { me = { groups = { "wheel" } } },
  home = {
    user = "me",
    files = {
      [".config/fuzzel/fuzzel.ini"] = { source = "home/fuzzel.ini" },
      [".config/mimeapps.list"] = { source = "home/mimeapps.list", mutable = true },
    },
  },
}
```

An app that tries to save over a read-only file gets an error instead of quietly changing it. If something changes it anyway, `archix diff` flags it and every switch puts the declared contents and mode back, saving the changed copy as for any [hand-edited file](/docs/concepts/safety/#managed-files-backups-and-hand-edits). A file's own `mutable` wins over `mutable_files` in either direction, so `mutable = true` keeps one file writable, as for `mimeapps.list` above, which desktop apps rewrite when you change a default app.

Links aren't affected, and neither are [live links](#live-links-for-app-written-configs): they point into your config repo, which stays writable, so an app writing through one edits your repo as before. Merged files keep their own rules; see [below](#setting-single-keys). The option is called `mutable_files` because `files` is already the map of system files; it covers both `files` and `home.files`.

`archix init` asks about this in its Config step, `Write files read-only, like the Nix store (apps can't change them; a switch puts them back)`, off by default.

## Setting single keys

Some files belong to an app that keeps writing to them: KDE's `kdeglobals` and `kwinrc`, a settings file a GUI saves, an ini file a tool rewrites. Owning the whole file would fight the app, and leaving it alone means your settings aren't in your config. `merge` does what plasma-manager does on NixOS: archix sets only the keys you declare and leaves everything else in the file to the app.

```lua title="lua/modules/desktop.lua"
return {
  users = { me = { groups = { "wheel" } } },
  home = {
    user = "me",
    files = {
      [".config/kdeglobals"] = {
        merge = { format = "kconfig", values = { General = { ColorScheme = "BreezeDark" }, KDE = { SingleClick = false } } },
      },
      [".config/qt6ct/qt6ct.conf"] = {
        merge = { format = "ini", values = { Appearance = { icon_theme = "Papirus-Dark" } } },
      },
      [".config/Code/User/settings.json"] = {
        merge = { format = "json", values = { ["editor.fontSize"] = 14, ["workbench.colorTheme"] = "Default Dark Modern" } },
      },
    },
  },
}
```

`merge` takes the place of `text`, `source`, `link` and `url`. `format` says how to read the file:

- **`kconfig`**, KDE's format: `values` holds groups of keys, and groups nest, so `{ Containments = { ["1"] = { General = { ... } } } }` writes under `[Containments][1][General]`.
- **`ini`**: keys at the top come before any section, then one level of sections. Deeper nesting is an error.
- **`json`**: `values` is deep-merged into the file's object. JSONC works too, as in VS Code's settings and waybar's config: `//` and `/* */` comments and trailing commas are accepted, and `//` inside a string is left alone.

ini and kconfig files are edited line by line. A declared key is replaced where it is, keeping `key = value` spacing if the file uses it; a missing key is added at the end of its group, and a missing group at the end of the file. Every other line, comments included, stays as it was. With an existing `kdeglobals` like this:

```ini title="~/.config/kdeglobals"
[General]
ColorScheme=BreezeClassic
font=Noto Sans,10,-1,5,50,0,0,0,0,0

[KDE]
SingleClick=true
```

the switch leaves:

```ini title="~/.config/kdeglobals"
[General]
ColorScheme=BreezeDark
font=Noto Sans,10,-1,5,50,0,0,0,0,0

[KDE]
SingleClick=false
```

A JSON file is compared by value, so an app that reformats it or reorders its keys hasn't changed anything as far as archix is concerned. When a declared key does differ, archix writes the merged object back, pretty-printed. That rewrite drops the file's comments. A file that can't be read even as JSONC stops the switch before anything changes, `/home/me/.config/Code/User/settings.json can't take its merged keys; nothing was changed: the file isn't valid JSON or JSONC`. ini and kconfig files are checked the same way.

What the app writes is its own business. Its keys never show up in `archix diff`; a declared key it changed does, as `update     /home/me/.config/kdeglobals`, and the next switch sets it back. archix creates a missing file with mode `0644`, owned by `home.user`, and leaves an existing file's mode and owner as they are. When the entry leaves your config, the file keeps whatever keys it has, and archix neither restores nor deletes it.

### Locked keys

With read-only files, by `mutable_files = false` or the entry's `mutable = false`, archix can't make a merged file read-only, since the app still writes the rest of it. For kconfig, it does the next best thing: every declared key gets KDE's immutable marker, so KDE itself won't change it:

```ini title="~/.config/kdeglobals"
[General]
ColorScheme[$i]=BreezeDark
```

ini and JSON have no such marker, so those keys can't be locked, and the files stay writable for their app.

## User units

`home.systemd` writes the user's own systemd units into `~/.config/systemd/user`, by kind and name. Each unit is a table of sections, rendered like Home Manager's `systemd.user.*`. `Install.WantedBy` creates the `.wants` link, so the unit is enabled without running `systemctl --user enable`:

```lua
home = {
  systemd = {
    services = {
      ["notes-sync"] = {
        Unit = { Description = "Sync notes" },
        Service = { Type = "oneshot", ExecStart = "/usr/bin/rclone sync notes: %h/notes" },
      },
    },
    timers = {
      ["notes-sync"] = {
        Timer = { OnCalendar = "hourly", Persistent = true },
        Install = { WantedBy = { "timers.target" } },
      },
    },
  },
},
```

The kinds are `services`, `timers`, `sockets`, `paths`, `targets`, `slices`, `mounts` and `automounts`. For units every user gets, use the system-wide `user_units` and `user_services` options.

## Three layers

How much you write depends on how much help an app needs. archix gives you three layers, from raw to fully modelled.

### 1. Format writers

Any app, no module needed. Pick the writer for the app's config format from `archix.formats`, give it a Lua table, and use the result as a file's `text`:

```lua title="lua/modules/fuzzel.lua"
local f = archix.formats

return {
  packages = { "fuzzel" },
  home = {
    files = {
      [".config/fuzzel/fuzzel.ini"] = {
        text = f.ini {
          main = { font = "JetBrains Mono:size=11", terminal = "foot" },
          colors = { background = "1e1e2eff" },
        },
      },
    },
  },
}
```

renders:

```ini title="~/.config/fuzzel/fuzzel.ini"
[colors]
background=1e1e2eff

[main]
font=JetBrains Mono:size=11
terminal=foot
```

Keys come out sorted, so the rendered file is identical on every run. The writers are `toml`, `json`, `yaml`, `ini`, `systemd`, `git_ini`, `key_value`, `kconfig` (KDE's format) and `kdl`, each matching the generator Home Manager uses. See [Format writers](/docs/reference/lua-api/#format-writers) for their options and quirks, such as numbers in INI files.

For a file that's easier to keep as it is, copy it with `source`, or link it with `live = true`.

### 2. `archix.program`

When you configure the same app on several machines, give it a module. `archix.program` defines one in a single line:

```lua title="lua/modules/fuzzel.lua"
return archix.program "fuzzel" { file = ".config/fuzzel/fuzzel.ini", format = "ini" }
```

Import it, and every host gets `programs.fuzzel`:

```lua title="lua/hosts/laptop/init.lua"
return {
  imports = { "modules.fuzzel" },
  config = {
    programs = {
      fuzzel = {
        enable = true,
        settings = { main = { font = "JetBrains Mono:size=10", terminal = "foot" } },
      },
    },
  },
}
```

`enable` installs the package, which defaults to the program's name, and writes the file. `settings` passes straight through to the writer, so any key the app understands works, including ones added in the app's latest release. `extra_config` appends raw text. Because these are real options, priorities work: a shared module can set `settings` with `archix.default`, and a host can override one value with `archix.force`.

archix ships alacritty, foot and mpv this way. Only define a program archix doesn't already ship; declaring an option twice is an error.

### 3. Ported Home Manager modules

Some apps need more than pass-through: options that interact, or files that depend on each other. For those, archix ships `programs.*` modules ported from [Home Manager](https://github.com/nix-community/home-manager) (MIT, credited):

| Module | Writes |
| --- | --- |
| `programs.kitty` | `~/.config/kitty/kitty.conf`, plus `diff.conf` and `quick-access-terminal.conf` |
| `programs.git` | `~/.config/git/config`, `ignore`, `attributes`, `allowed_signers` |
| `programs.zsh` | `~/.zshrc`, `~/.zshenv`, and `~/.zprofile`, `~/.zlogin`, `~/.zlogout` when used |
| `programs.starship` | `~/.config/starship.toml` |
| `programs.fastfetch` | `~/.config/fastfetch/config.jsonc` |
| `mime_apps` | `~/.config/mimeapps.list` |
| `home.systemd` | User units and their `.wants` links |

```lua title="lua/modules/common.lua"
return {
  users = { me = { groups = { "wheel" } } },
  home = { user = "me" },
  programs = {
    kitty = { enable = true, font = { name = "JetBrainsMono Nerd Font", size = 12 } },
    git = { enable = true, settings = { user = { name = "Me", email = "me@example.com" } } },
    zsh = { enable = true, autosuggestion = { enable = true } },
    starship = { enable = true },
  },
}
```

The modules work together the way they do in Home Manager. kitty's and starship's shell integrations go into `programs.zsh.init_content` when zsh is enabled, and zsh's plugins come from Arch's own packages in `/usr/share/zsh/plugins`, installed for you. Parts that only make sense on Nix are left out.

If you're coming from Home Manager, your config translates closely. Option names are snake_case versions of Home Manager's, and the structure is the same:

| Home Manager | archix |
| --- | --- |
| `programs.kitty.mouseBindings` | `programs.kitty.mouse_bindings` |
| `programs.kitty.shellIntegration.mode` | `programs.kitty.shell_integration.mode` |
| `programs.git.signing.signByDefault` | `programs.git.signing.sign_by_default` |
| `programs.zsh.initContent` | `programs.zsh.init_content` |
| `xdg.mimeApps.defaultApplications` | `mime_apps.default_applications` |
| `systemd.user.timers.<name>` | `home.systemd.timers.<name>` |
| `home.file.<path>` | `home.files.<path>` |
| `home.file.<path>.onChange` | `home.files.<path>.on_change` |

`archix options` lists every option, and the [options reference](/docs/reference/options/#programs) documents them all.

Ports are checked, not just translated. archix's test suite carries Home Manager's own test cases for these modules, with Home Manager's expected output files copied verbatim. archix renders the same input config with its port and compares the result byte for byte. The few differences that are inherent, such as Nix store paths becoming `/usr/bin` paths, are spelled out per case.

A weekly job compares Home Manager's upstream modules and tests with the commit each port was made from. When one changes, it opens an issue listing the upstream commits and files, so the port can be brought up to date and checked against the new tests. Because `settings` passes through, new app keys need no archix change at all.

## Bringing your existing dotfiles in

`archix init` can import the dotfiles you already have, but only in the interactive wizard, where you see every pre-ticked entry and can untick it. `archix init --defaults` never imports dotfiles: it writes no `home` section and leaves your home untouched. In the wizard's Dotfiles step, pick the user under `Whose dotfiles` if init captured more than one, then choose entries in `Dotfiles to bring in`. It looks at each entry in `~/.config` and the top-level dotfiles in the home:

- **Recommended** entries start ticked and are tagged `recommended`: shell rc files such as `.bashrc` and `.zshrc`, `.gitconfig` and `.config/git`, editors such as `nvim` and `helix`, terminals (`kitty`, `alacritty`, `foot`, `wezterm`, `ghostty`), window managers, bars and launchers (`hypr`, `sway`, `i3`, `niri`, `waybar`, `rofi` and similar), tools such as `btop` and `fastfetch`, and `mimeapps.list` and `user-dirs.dirs`.
- **Everything else** is offered unticked, such as most KDE `*rc` files.
- **Caches and app state** aren't offered: `.cache`, `.local`, toolchains such as `.cargo` and `.rustup`, browsers, IDEs, chat apps and shell histories.
- **Credentials** are never offered, only counted: `.ssh`, `.gnupg`, `.aws`, `.kube`, `.docker`, `.cloudflared`, `.config/gh`, password stores, `.claude.json`, `.config/sunshine` and anything else that looks like a secret, by its name or by what's in it. An entry whose files contain a private key, an access token from GitHub, GitLab, Slack or Anthropic, an AWS secret key, an OAuth refresh token, or a `psk=` or `password=` line counts too, such as a `~/.config` directory holding an app's login token. Files over 1 MiB aren't read, and a directory is only read up to 200 files. The step says so: `· 1 entry that holds a secret stays out of the config and isn't listed`, or `· 4 entries that hold secrets stay out of the config and aren't listed`.

`space` or `enter` toggles an entry, `a` and `n` select all or none, and `r` goes back to the recommended ones. The full lists are in `src/capture.json` in the archix repository.

Each one you pick is copied into the config's `home/` directory, owned by that user, and declared in one of three ways:

- **Directories** become [live links](#live-links-for-app-written-configs).
- **Files their app replaces on save**, such as KDE's `*rc` files, `kdeglobals`, `mimeapps.list`, `user-dirs.dirs` and `.gitconfig`, become writable copies with `mutable = true`. Such an app writes a new file and renames it over the old one, which would replace a link with a plain file, so a link wouldn't last. The full list is `replaced` in `src/capture.json`.
- **Other files** become live links, or read-only copies if you turned on `Write files read-only, like the Nix store (apps can't change them; a switch puts them back)` in the Config step.

```lua title="lua/hosts/desktop/init.lua"
return {
  -- everything else init captured stays above
  home = {
    user = "me",
    files = {
      [".config/hypr"] = { source = "home/.config/hypr", live = true },
      [".config/kdeglobals"] = { source = "home/.config/kdeglobals", mutable = true },
      [".config/nvim"] = { source = "home/.config/nvim", live = true },
      [".zshrc"] = { source = "home/.zshrc", live = true },
    },
  },
}
```

The first switch moves the originals, directories included, to `/var/lib/archix/backups` and replaces them with links into the config, so from then on editing a dotfile through a link edits your repo. A copy, like `kdeglobals` above, doesn't: what the app saves there is [flagged as edited](#home-files) by `archix diff`, and the next switch puts the declared version back. To keep only some of its settings, switch it to [`merge`](#setting-single-keys). That makes imported dotfiles the one thing the first switch after `init` changes. init's Result section lists them:

```
── Result ──────────────────────────────────────
  ! the first switch would still change these:
      replace    /home/me/.config/hypr
      replace    /home/me/.config/nvim
      replace    /home/me/.zshrc
```

## Which layer to use

- One machine, one file, no logic: a format writer in `home.files`, or `source`.
- A file an app rewrites itself: a live link.
- A file an app writes, where you only care about some of its settings: `merge`.
- The same app on several machines, with per-host tweaks: `archix.program`, or the shipped module if there is one.
- An app archix ships a module for: use the module. It knows how the app's files fit together and renders them exactly as Home Manager would.
