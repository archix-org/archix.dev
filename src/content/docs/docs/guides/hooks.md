---
title: Activation hooks
description: Run scripts after every switch and revert with hooks, and react to a changed file with on_change, like NixOS activation scripts and Home Manager's onChange; both run on the running system once the switch has applied.
sidebar:
  order: 6
---

Most of a system fits in files, units and services. Some of it doesn't: a directory with the right owner, an override only a tool's own CLI writes, a cache that has to be rebuilt after a file changes. archix has two ways to do those things as part of a switch:

- **`hooks`** are scripts that run after every switch and revert, like NixOS activation scripts.
- **`on_change`** is a script on a [`files`](/docs/reference/options/#files) or [`home.files`](/docs/reference/options/#homefiles) entry that runs only when that file changed, like Home Manager's `onChange`.

Both run on the running system, at the end of the switch, once its packages, users, files and services are in place. See [When they run](#when-they-run).

## Hooks

Each hook has a name and a bash script:

```lua title="lua/hosts/desktop/init.lua"
return {
  users = { alice = { groups = { "wheel" } } },
  hooks = {
    backups = { run = "install -d -m 0750 -o alice -g alice /srv/backups" },
  },
}
```

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `run` | `lines` | *required* | The bash script to run. |
| `user` | `null or str` | `null` | Run as this user instead of root. See [User hooks](#user-hooks). |
| `after` | `list of str` | `[]` | Hooks that must run before this one. See [Order](#order). |

`run` is a `lines` value, like a file's `text`, so several modules can each add lines to the same hook. A script runs as `bash -e -o pipefail -c`, so the first command that fails stops it and fails the hook.

The switch preview lists the hooks, in order, on a `run hooks` line. Hooks aren't counted as changes in the prompt:

```console
$ sudo archix switch
add user   alice
run hooks  backups
apply 2 changes? [Y/n]
:: applying generation 45
...
:: running hooks
   hook backups
switched to generation 45 (snapshot 2026-10-15)
```

## When they run

A switch applies in place, in this order: the package transaction, users and passwords, home directories, files, services enabled and disabled, then on the running system `systemctl daemon-reload`, stopping what the config turned off, starting what it newly enables and restarting what changed. Then it runs the `on_change` scripts that are due, in path order, and then every hook. So users the same switch declared exist, services it enabled are started, and the network is as the running system has it.

- **A switch** runs every hook, and the `on_change` scripts of files it changed.
- **A switch that has nothing to do** runs the hooks too, like NixOS runs its activation scripts on every `nixos-rebuild switch`:

  ```console
  $ sudo archix switch
  nothing to do: the system matches generation 45 (snapshot 2026-10-15)
  :: running hooks
     hook backups
  ```

- **`archix revert`** runs the hooks of the generation it goes back to, not the ones of the generation it leaves, and its preview has the same `run hooks` line.
- **Keeping a snapshot** with `archix rollback N`, booting one from the boot menu, and an automatic fallback run no hooks: the snapshot holds the system as it was, hooks' results included. See [Keeping a snapshot](/docs/concepts/snapshots/#keeping-a-snapshot).

`install`, `remove` and `upgrade` end with a switch, so they run the hooks too.

## Make them idempotent

Because hooks run after every switch, a hook has to be safe to run again and again. The same goes for `on_change` scripts, which run again whenever archix rewrites their file. Write it as "make it so", not "do this once":

| Instead of | Write |
| --- | --- |
| `echo 'alias ll="ls -l"' >> /etc/bash.bashrc` | A [`files`](/docs/reference/options/#files) entry, which archix keeps in step. |
| `mkdir /srv/backups` | `mkdir -p /srv/backups` |
| `ln -s /srv/media /srv/www/media` | `ln -sfn /srv/media /srv/www/media` |
| `useradd builder` | A [`users`](/docs/reference/options/#users) entry. |

Commands that set state, such as `flatpak override` or `setfacl -m`, are idempotent already. For anything that should only happen when a file changed, such as rebuilding a cache, use [`on_change`](#on_change) instead of a hook.

## Order

Hooks run in name order, unless `after` says otherwise. A hook with `after` waits until every hook it names has run:

```lua title="lua/hosts/desktop/init.lua"
return {
  users = { alice = { groups = { "wheel" } } },
  hooks = {
    media = { run = "install -d -o alice -g alice /srv/media" },
    link = { user = "alice", run = "ln -sfn /srv/media ~/Media", after = { "media" } },
  },
}
```

By name, `link` would go first. With `after`, it waits until root has created `/srv/media`:

```
run hooks  media, link
```

`after` can only name hooks, and can't go in a circle. Both are config errors, so `diff` and `switch` stop before anything changes. A cycle error names just the hooks in the cycle, not the ones waiting behind it:

```console
error: hooks.link.after names "medai", which isn't a hook
error: hooks link -> media -> link wait on each other in a cycle
error: hook link lists itself in after
```

## Environment

A hook gets a clean environment:

| | Root hooks | User hooks |
| --- | --- | --- |
| `PATH` | `/usr/local/sbin:/usr/local/bin:/usr/bin` | The same |
| `HOME` | `/root` | The user's home |
| `USER`, `LOGNAME` | `root` | The user's name |
| `LANG` | archix's own, if it has one | The same |
| `XDG_RUNTIME_DIR` | Not set | `/run/user/<uid>`, while the user is logged in |
| `DBUS_SESSION_BUS_ADDRESS` | Not set | `unix:path=/run/user/<uid>/bus`, while the user has a session bus |
| Working directory | `/` | The user's home, or `/` if it doesn't exist |

Nothing else comes through, so there's no `DISPLAY`, `WAYLAND_DISPLAY` or `SUDO_USER`. The output goes to your terminal.

## User hooks

With `user`, the hook runs as that user, with their uid, primary group and supplementary groups. The user has to exist when the hook runs; one declared in [`users`](/docs/reference/options/#users) is created by the same switch. It doesn't have to be [`home.user`](/docs/reference/options/#homeuser).

archix switches to the user without opening a login session, so a hook never starts a systemd user manager that lingers after it. While the user is logged in, the hook gets their runtime directory and session bus, so `systemctl --user` and `gsettings` work; when nobody is, they aren't there, and `systemctl --user` fails. User units archix writes with [`home.systemd`](/docs/reference/options/#homesystemd) are read when the user's manager starts at login, so they need no hook.

## `on_change`

`on_change` belongs to one file. It runs when a switch or revert changed that file:

```lua title="lua/hosts/desktop/init.lua"
return {
  files = {
    ["/etc/dconf/db/local.d/00-archix"] = {
      text = "[org/gnome/desktop/interface]\ncolor-scheme='prefer-dark'",
      on_change = "dconf update",
    },
  },
}
```

`dconf update` compiles the file into the database GNOME reads.

On a [`files`](/docs/reference/options/#files) entry, `on_change` runs as root. On a [`home.files`](/docs/reference/options/#homefiles) entry, it runs as [`home.user`](/docs/reference/options/#homeuser), in the same environment as a [user hook](#user-hooks). It works on files a `programs.*` module writes too, since those are `home.files` entries. Every `on_change` runs on the running system once the switch has applied, so `pkill -USR1 -x kitty` reaches the running kitty; see [Run a script when a file changes](/docs/guides/dotfiles-and-apps/#run-a-script-when-a-file-changes).

### When it runs

`on_change` runs when the switch or revert changed the file, in either of two ways:

- **Its entry changed.** The file's entry in the new generation differs from the one in the generation the system is leaving: its contents, mode, owner and group, or its link target. That covers the first switch that manages the file, a new `text`, an edited `source` or a new `url`, and a revert to a generation where the file was different. The script that runs is the one from the generation you end up in.
- **archix rewrote it.** The entry is the same, but the file didn't match it, so the switch or revert wrote it again: a file [edited outside archix](/docs/concepts/safety/#managed-files-backups-and-hand-edits) that it put back, or a file someone deleted that it recreated.

It doesn't run:

- when nothing about the file changed and it already matched, even though hooks run on that switch;
- when you only change the `on_change` script itself;
- when the file leaves the config;
- when you edit a file behind a [live link](/docs/guides/dotfiles-and-apps/#live-links-for-app-written-configs). The entry is the link, and archix never writes what's behind it.

`on_change` scripts run before any hook, in path order, so hooks see whatever they rebuilt.

## Failures

A failing hook or `on_change` script is a soft failure. The rest of the hooks still run, nothing is undone, and the generation stays applied and current. archix lists each failure, and the switch exits non-zero:

```console
:: running hooks
   hook media
   hook link
failed: hook link: exited with exit status: 1
error: generation 46 is applied and current, but 1 step failed (listed above)
```

An `on_change` script is listed by its file, `on_change of /etc/dconf/db/local.d/00-archix: exited with exit status: 1`, and a user hook whose user doesn't exist as `hook link: user alice doesn't exist`. On a switch with nothing to do, the last line is `error: nothing was changed, but 1 step failed (listed above)`. Nothing runs them again by itself: fix what they need, such as the config or the network, and switch again, which runs every hook.

This is where archix differs from Home Manager, which stops activation when an `onChange` script or activation script fails. Undoing a whole generation because a script failed would be worse than the failure.
