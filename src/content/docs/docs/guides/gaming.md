---
title: Gaming
description: Give gamescope and other tools the file capabilities a read-only /usr won't let you set by hand, and hold a kernel, a scheduler or a driver at a version that works while the rest of the system updates.
sidebar:
  order: 6
---

A gaming machine is where a read-only `/usr` and a locked snapshot most often get in the way: a compositor wants a capability `setcap` would give it, and a new kernel or driver breaks a game. Both have an option.

## Capabilities: gamescope

gamescope raises its own scheduling priority when it has `CAP_SYS_NICE`, and the Arch wiki tells you to `setcap 'CAP_SYS_NICE=eip' $(which gamescope)`. On archix that fails: `/usr` is read-only, and the next switch would start from a deployment without it anyway. Declare it instead:

```lua title="lua/hosts/desktop/init.lua"
return {
  packages = { "gamescope", "steam" },
  capabilities = {
    ["/usr/bin/gamescope"] = "cap_sys_nice+ep",
  },
}
```

The next switch sets it in the new deployment, `setcap     /usr/bin/gamescope cap_sys_nice=ep`, and every switch after keeps it, also when gamescope is upgraded. It's set on a copy of the file, so the deployments before keep theirs, and a [rollback](/docs/concepts/generations/) goes back to what that deployment had. `getcap /usr/bin/gamescope` shows it once you've rebooted into it, and [`archix verify`](/docs/reference/cli/#verify) checks it.

Some packages set their own: `gpu-screen-recorder` gives `gsr-kms-server` `cap_sys_admin=ep` when it's installed, and archix leaves that alone. See [`capabilities`](/docs/reference/options/#capabilities).

## Holding a kernel, a scheduler or a driver

When an update breaks a game, the quickest fix is to [roll back](/docs/concepts/generations/#rolling-back-from-a-running-system). To keep updating everything else while one package stays where it worked, hold it:

```lua
holds = {
  ["scx-scheds"] = "1.1.2-1",
  ["v4l2loopback-dkms"] = ">= 0.15.4",
},
```

`archix update` then moves the snapshot but keeps `scx-scheds` at 1.1.2-1, fetched from the Arch Linux Archive, and refuses any snapshot that would take `v4l2loopback-dkms` below 0.15.4. `archix status` says how far behind each hold is: `hold: scx-scheds held at 1.1.2-1, behind the snapshot's 1.1.3-2`.

A hold is a partial upgrade, which Arch doesn't support, so archix refuses an update where the held version no longer meets what the rest needs, sonames included, and names both packages. Holding the kernel is allowed but warned about in `status`, since it keeps security fixes out; keep the `linux` and `linux-headers` holds together, and drop them once a fixed version lands. See [Holds](/docs/concepts/lockfile/#holds).
