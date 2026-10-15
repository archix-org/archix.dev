---
title: Activation hooks
description: Run scripts at the end of every switch and rollback with hooks, and react to a changed file with on_change, like NixOS activation scripts and Home Manager's onChange.
sidebar:
  order: 6
---

Most of a system fits in files, units and services. Some of it doesn't: a daemon keeps settings you can only change with its own CLI, a service has to restart to read a new config file, a running app has to be told to reload. archix has two ways to do those things as part of a switch:

- **`hooks`** are scripts that run at the end of every switch and rollback, like NixOS activation scripts.
- **`on_change`** is a script on a [`files`](/docs/reference/options/#files) or [`home.files`](/docs/reference/options/#homefiles) entry that runs only when that file changed, like Home Manager's `onChange`.

## Hooks

Each hook has a name and a bash script:

```lua title="lua/hosts/desktop/init.lua"
return {
  packages = { "tailscale" },
  services = { ["tailscaled.service"] = true },
  hooks = {
    tailscale = { run = "tailscale set --operator=alice" },
  },
}
```

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `run` | `lines` | *required* | The bash script to run. |
| `user` | `null or str` | `null` | Run as this user instead of root. See [User hooks](#user-hooks). |
| `after` | `list of str` | `[]` | Hooks that must run before this one. See [Order](#order). |

`run` is a `lines` value, like a file's `text`, so several modules can each add lines to the same hook. A script runs as `bash -e -o pipefail -c`, so the first command that fails stops it and fails the hook.

The switch preview lists the hooks it will run, in order, on a `run hooks` line. Hooks aren't counted as changes in the prompt:

```console
$ sudo archix switch
install    tailscale 1.88.3-1
enable     tailscaled.service
boot entry for the new generation
run hooks  tailscale
apply 3 changes? [Y/n]
:: installing packages
:: removing packages
:: setting install reasons
:: updating users and groups
:: stopping and disabling services
:: writing managed files
:: reloading systemd, sysctl, udev and locales
:: enabling and starting services
:: restarting what uses replaced files
:: verifying the result
:: writing the boot entry
:: making the new generation current
:: running activation hooks
   hook tailscale
switched to generation 45 (snapshot 2026-10-15)
```

## When they run

Hooks run last, once the switch itself is done: after files are written, services started and units using replaced files [restarted](/docs/concepts/safety/#restarting-what-uses-replaced-files), after the result is verified, the boot entry written and the new generation made current, and after Flatpak apps are installed and pinned. A hook sees the finished system, Flatpak apps the same switch added included. Hooks aren't a step of the [journal](/docs/concepts/safety/#the-journal); by the time they run, the switch is complete, and nothing a hook does can leave it unfinished.

They run on every switch, including one that has nothing to change, exactly like NixOS runs its activation scripts on every `nixos-rebuild switch`:

```console
$ sudo archix switch
nothing to do: the system matches generation 45 (snapshot 2026-10-15)
run hooks  tailscale
:: running activation hooks
   hook tailscale
```

`install`, `remove` and `upgrade` end with a switch, so they run the hooks too; `install --temp` doesn't switch, and runs none. Hooks are part of the generation, so adding or changing one creates a new generation, like any other change to the config.

Every rollback runs the hooks of the generation it goes back to, not the ones of the generation it leaves, and its preview has the same `run hooks` line. That includes a rollback with nothing to put back, which prints `generation 41 (snapshot 2026-10-09) is current; no package changes were needed` and then runs them. So rolling back to the current generation, `sudo archix rollback 45` with 45 current, reruns its hooks.

`archix recover` runs hooks too, since the interrupted switch never got to them. `--continue` runs the hooks of the generation it finishes the switch to, and `--rollback` the hooks of the state from before the switch. archix can't tell which files the interrupted switch had already rewritten, so a recovery runs every `on_change` script of the state it finishes in, not only those of files that changed.

A rollback from the [boot menu](/docs/concepts/generations/#rolling-back-from-the-boot-menu) happens early in boot, before the network is up or anyone has logged in, so it skips hooks and `on_change` scripts, the way it skips pinning Flatpak apps, and says so:

```
archix: hooks don't run at boot; the next switch or `sudo archix rollback 41` runs them once you're online
```

The `on_change` scripts it was due to run aren't lost. archix records each script, exactly as that generation had it, in `/var/lib/archix/pending-on-change`, and the next switch, rollback or recovery runs them together with its own, before the hooks, even if its own config no longer has an `on_change` for that file. When that switch is due to run an `on_change` for the same file itself, only its own, newer script runs, once. After several boot rollbacks in a row, the latest one's script is kept for each file. `sudo archix rollback 41` once you're logged in runs generation 41's hooks, those `on_change` scripts, and pins its Flatpak apps.

## Make them idempotent

Because hooks run on every switch, a hook has to be safe to run again and again. The same goes for `on_change` scripts: they usually run only when their file changed, but a [recovery](#when-they-run) runs all of them. Write it as "make it so", not "do this once":

| Instead of | Write |
| --- | --- |
| `echo 'alias ll="ls -l"' >> /etc/bash.bashrc` | A [`files`](/docs/reference/options/#files) entry, which archix keeps in step. |
| `mkdir /srv/backups` | `mkdir -p /srv/backups` |
| `ln -s /srv/media /srv/www/media` | `ln -sfn /srv/media /srv/www/media` |
| `useradd builder` | A [`users`](/docs/reference/options/#users) entry. |

Commands that set state, such as `tailscale set`, `gsettings set` or `flatpak override`, are idempotent already. For anything that should only happen when a file changed, such as a restart, use [`on_change`](#on_change) instead of a hook.

## Order

Hooks run in name order, unless `after` says otherwise. A hook with `after` waits until every hook it names has run:

```lua title="lua/hosts/desktop/init.lua"
return {
  users = { alice = { groups = { "wheel" } } },
  hooks = {
    tailscale = { run = "tailscale set --operator=alice" },
    lan = { user = "alice", run = "tailscale set --exit-node-allow-lan-access", after = { "tailscale" } },
  },
}
```

By name, `lan` would go first. With `after`, it waits until root has made `alice` the operator, so `alice` can change Tailscale's settings without root:

```
run hooks  tailscale, lan
```

`after` can only name hooks, and can't go in a circle. Both are config errors, so `diff` and `switch` stop before anything changes. A cycle error names just the hooks in the cycle, not the ones waiting behind it:

```console
error: hooks.lan.after names "tailscal", which isn't a hook
error: hooks lan -> tailscale -> lan wait on each other in a cycle
error: hook lan lists itself in after
```

## Environment

A hook gets a clean environment, not the one you ran `sudo archix switch` from:

| | Root hooks | User hooks |
| --- | --- | --- |
| `PATH` | `/usr/local/sbin:/usr/local/bin:/usr/bin` | The same |
| `HOME` | `/root` | The user's home |
| `USER`, `LOGNAME` | `root` | The user's name |
| `LANG` | archix's own, if it has one | The same |
| `XDG_RUNTIME_DIR` | Not set | `/run/user/<uid>`, while the user is logged in |
| `DBUS_SESSION_BUS_ADDRESS` | Not set | `unix:path=/run/user/<uid>/bus`, while the user has a session bus |
| Working directory | `/` | The user's home, or `/` if it doesn't exist |

Nothing else comes through, so there's no `DISPLAY`, `WAYLAND_DISPLAY` or `SUDO_USER`. The script's output goes to your terminal, under its `hook <name>` line.

## User hooks

With `user`, the hook runs as that user, with their uid, primary group and supplementary groups. The user has to exist by the time hooks run; one declared in [`users`](/docs/reference/options/#users) is created earlier in the same switch. It doesn't have to be [`home.user`](/docs/reference/options/#homeuser).

archix switches to the user without opening a login session, so a hook never starts a systemd user manager that lingers after it. When the user is logged in, their runtime directory and session bus are there, and the hook gets `XDG_RUNTIME_DIR` and `DBUS_SESSION_BUS_ADDRESS`, so `systemctl --user` reaches their running session. When they aren't logged in, both are missing and `systemctl --user` fails, so check first:

```lua title="lua/modules/common.lua"
return {
  users = { alice = { groups = { "wheel" } } },
  home = {
    user = "alice",
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
  hooks = {
    ["notes-sync"] = {
      user = "alice",
      run = [[
[ -n "$DBUS_SESSION_BUS_ADDRESS" ] || exit 0
systemctl --user daemon-reload
systemctl --user restart notes-sync.timer
]],
    },
  },
}
```

When `alice` is logged in, her user manager picks up the new units and restarts the timer. When she isn't, the hook does nothing, and her user manager reads the units when she next logs in.

## `on_change`

`on_change` belongs to one file. It runs when a switch or rollback changed that file:

```lua title="lua/hosts/server/init.lua"
return {
  files = {
    ["/etc/foo.conf"] = { text = "port = 8080", on_change = "systemctl try-restart foo" },
  },
}
```

`systemctl try-restart` restarts `foo` only if it's running, so the script is safe even before the service is enabled.

On a [`files`](/docs/reference/options/#files) entry, `on_change` runs as root. On a [`home.files`](/docs/reference/options/#homefiles) entry, it runs as [`home.user`](/docs/reference/options/#homeuser), in the same environment as a [user hook](#user-hooks). It works on files a `programs.*` module writes too, since those are `home.files` entries:

```lua title="lua/modules/desktop.lua"
return {
  users = { alice = { groups = { "wheel" } } },
  home = {
    user = "alice",
    files = {
      [".config/waybar/config.jsonc"] = { source = "home/waybar/config.jsonc", on_change = "pkill -SIGUSR2 -x waybar || true" },
      [".config/kitty/kitty.conf"] = { on_change = "pkill -USR1 -x kitty || true" },
    },
  },
  programs = {
    kitty = { enable = true, settings = { font_size = 11 } },
  },
}
```

waybar and kitty reload their config on those signals. `pkill` exits non-zero when nothing matched, so `|| true` keeps a closed app from failing the script.

### When it runs

`on_change` runs when the switch or rollback changed the file, in either of two ways:

- **Its entry changed.** The file's entry in the new generation differs from the one in the generation the system is leaving: its contents, mode, owner and group, or its link target. That covers the first switch that manages the file, a new `text`, an edited `source` or a new `url`, and a rollback to a generation where the file was different. The script that runs is the one from the generation you end up in.
- **archix rewrote it on disk.** The entry is the same, but the file on disk didn't match it, so the switch or rollback wrote it again: a file [edited outside archix](/docs/concepts/safety/#managed-files-backups-and-hand-edits) that it put back, or a file someone deleted that it recreated.

Two more cases run it:

- **After a boot-menu rollback.** That rollback runs no scripts, but records the files whose `on_change` was due. The next switch, rollback or recovery runs those scripts along with its own.
- **On `archix recover`.** `--continue` and `--rollback` can't know which files the interrupted switch had already rewritten, so they run every `on_change` script of the state they finish in.

So write `on_change` scripts to be idempotent, like hooks. `systemctl try-restart` and reload signals are.

It doesn't run:

- when nothing about the file changed and it already matched on disk, even though hooks run on that switch;
- when you only change the `on_change` script itself;
- when the file leaves the config;
- when you edit a file behind a [live link](/docs/guides/dotfiles-and-apps/#live-links-for-app-written-configs). The entry is the link, and archix never writes what's behind it.

`on_change` scripts run in the final `:: running activation hooks` phase, before any hook, in path order, so hooks see whatever they restarted or reloaded.

## Failures

A failing hook or `on_change` script is a [soft failure](/docs/concepts/safety/#hard-and-soft-failures), like a service that doesn't start. The rest of the hooks still run. Hooks run after the switch is complete, so nothing is undone: archix lists what failed and exits non-zero:

```console
:: enabling and starting services
:: restarting what uses replaced files
:: verifying the result
:: writing the boot entry
:: making the new generation current
:: running activation hooks
   hook tailscale
   hook lan
switched to generation 46 (snapshot 2026-10-15)
failed: hook lan: exited with exit status: 1
error: the new generation is current, but 1 step failed (listed above)
```

The last line says what did happen before the failures:

| When | Last line |
| --- | --- |
| A switch that made a new generation | `error: the new generation is current, but 1 step failed (listed above)` |
| A switch that had nothing to do | `error: nothing was changed, but 1 step failed (listed above)` |
| A rollback | `error: generation 41 (snapshot 2026-10-09) is current, but 1 step failed (listed above)` |
| `archix recover --continue` | `error: generation 46 (snapshot 2026-10-15) is current, but 1 step failed (listed above)` |
| `archix recover --rollback` | `error: the rollback finished, but 1 step failed (listed above)` |

With more than one failure, it's `2 steps failed`.

An `on_change` script is listed by its file, `failed: on_change of /etc/foo.conf: exited with exit status: 1`, and a user hook whose user doesn't exist as `failed: hook lan: user alice doesn't exist`.

This is where archix differs from Home Manager, which stops activation when an `onChange` script or activation script fails. Undoing a whole generation because a restart failed would be worse than the failure. Fix the script and switch again; since hooks run on every switch, the next one runs it again even if nothing else changed.
