---
title: Activation hooks
description: Run scripts after every switch and rollback with hooks, and react to a changed file with on_change, like NixOS activation scripts and Home Manager's onChange; both run at the new deployment's first boot.
sidebar:
  order: 6
---

Most of a system fits in files, units and services. Some of it doesn't: a directory with the right owner, an override only a tool's own CLI writes, a cache that has to be rebuilt after a file changes. archix has two ways to do those things as part of a switch:

- **`hooks`** are scripts that run after every switch and rollback, like NixOS activation scripts.
- **`on_change`** is a script on a [`files`](/docs/reference/options/#files) or [`home.files`](/docs/reference/options/#homefiles) entry that runs only when that file changed, like Home Manager's `onChange`.

Both run at the first boot of the [deployment](/docs/concepts/atomicity/) the switch built, early, before the network is up and before anyone logs in. See [When they run](#when-they-run).

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
a new deployment with its own boot entry; it boots at the next restart
run hooks  backups, when the deployment boots
apply 2 changes? [Y/n]
:: building deployment 45 from deployment 44
switched to generation 45 (snapshot 2026-10-15); its deployment boots at the next restart
```

At the next boot, the console shows them running:

```
archix: running deployment 45's hooks
   hook backups
```

## When they run

A switch records the hooks and `on_change` scripts its deployment is due to run, and `archix-deploy-activate.service` runs them at that deployment's first boot: after local filesystems are mounted and `systemd-sysusers` has run, once the deployment's `/var` changes are applied, and before `sysinit.target`. So users the same switch declared exist, but no service has started: no network, no D-Bus, no session. A hook that needs those, such as `tailscale set` or `gsettings set`, fails there.

They run once per deployment. A record is cleared only once every script in it succeeded; until then all of them run again at each boot.

- **A switch** records the generation's hooks, and the `on_change` scripts of files it changed. A switch built on a deployment that never booted carries that deployment's pending `on_change` scripts along, so none are lost.
- **A switch that has nothing to build** runs the hooks right away, on the running system, like NixOS runs its activation scripts on every `nixos-rebuild switch`:

  ```console
  $ sudo archix switch
  nothing to do: the system matches generation 45 (snapshot 2026-10-15)
  :: running hooks
     hook backups
  ```

- **A rollback** records the hooks of the generation it goes back to, not the ones of the generation it leaves, and its preview has the same `run hooks` line. They run when that deployment next boots.
- **Picking an older entry in the boot menu** runs nothing for a deployment that has booted before.
- **`install --temp`** records no hooks.

`install`, `remove` and `upgrade` end with a switch, so they record the hooks too. Hooks are part of the generation, so adding or changing one builds a new deployment, like any other change to the config.

## Make them idempotent

Because hooks run after every switch, a hook has to be safe to run again and again. The same goes for `on_change` scripts: they usually run only when their file changed, but a failure anywhere in the record runs them again at the next boot. Write it as "make it so", not "do this once":

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
run hooks  media, link, when the deployment boots
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

Nothing else comes through, so there's no `DISPLAY`, `WAYLAND_DISPLAY` or `SUDO_USER`. At boot the output goes to the console and the journal, under `archix-deploy-activate.service`; on a switch with nothing to build, to your terminal.

## User hooks

With `user`, the hook runs as that user, with their uid, primary group and supplementary groups. The user has to exist when the hook runs; one declared in [`users`](/docs/reference/options/#users) is created by the same switch. It doesn't have to be [`home.user`](/docs/reference/options/#homeuser).

archix switches to the user without opening a login session, so a hook never starts a systemd user manager that lingers after it. At boot nobody is logged in, so the user's runtime directory and session bus aren't there, and `systemctl --user` fails. User units archix writes with [`home.systemd`](/docs/reference/options/#homesystemd) are read when the user's manager starts at login, so they need no hook.

## `on_change`

`on_change` belongs to one file. It runs when a switch or rollback changed that file:

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

`dconf update` compiles the file into the database GNOME reads, offline, so it works before anything has started.

On a [`files`](/docs/reference/options/#files) entry, `on_change` runs as root. On a [`home.files`](/docs/reference/options/#homefiles) entry, it runs as [`home.user`](/docs/reference/options/#homeuser), in the same environment as a [user hook](#user-hooks). It works on files a `programs.*` module writes too, since those are `home.files` entries. Home files themselves are written as soon as the switch commits, but their `on_change` waits for the boot like the rest, so a signal to reload a running app reaches nothing there; see [Run a script when a file changes](/docs/guides/dotfiles-and-apps/#run-a-script-when-a-file-changes).

### When it runs

`on_change` runs when the switch or rollback changed the file, in either of two ways:

- **Its entry changed.** The file's entry in the new generation differs from the one in the generation the system is leaving: its contents, mode, owner and group, or its link target. That covers the first switch that manages the file, a new `text`, an edited `source` or a new `url`, and a rollback to a generation where the file was different. The script that runs is the one from the generation you end up in.
- **archix rewrote it.** The entry is the same, but the file didn't match it, so the switch or rollback wrote it again: a file [edited outside archix](/docs/concepts/safety/#managed-files-backups-and-hand-edits) that it put back, or a file someone deleted that it recreated.

It doesn't run:

- when nothing about the file changed and it already matched, even though hooks run on that switch;
- when you only change the `on_change` script itself;
- when the file leaves the config;
- when you edit a file behind a [live link](/docs/guides/dotfiles-and-apps/#live-links-for-app-written-configs). The entry is the link, and archix never writes what's behind it.

`on_change` scripts run before any hook, in path order, so hooks see whatever they rebuilt.

## Failures

A failing hook or `on_change` script is a soft failure. The rest of the hooks still run, nothing is undone, and the boot goes on. archix prints each failure on the console and leaves a note for your next login, in login shells and as a desktop notification:

```
archix: running deployment 46's hooks
   hook media
   hook link
archix: hook link: exited with exit status: 1; it runs again at the next boot
```

An `on_change` script is listed by its file, `on_change of /etc/dconf/db/local.d/00-archix: exited with exit status: 1`, and a user hook whose user doesn't exist as `hook link: user alice doesn't exist`. Every script in that deployment's record runs again at the next boot, until all of them succeed; a fixed config needs a new switch.

On a switch with nothing to build, where hooks run right away, a failure is listed as `failed: hook link: exited with exit status: 1` and the switch ends with `error: nothing was changed, but 1 step failed (listed above)`.

This is where archix differs from Home Manager, which stops activation when an `onChange` script or activation script fails. Undoing a whole generation because a script failed would be worse than the failure.
