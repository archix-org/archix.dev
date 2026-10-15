---
title: Environment and firewall
description: Set environment variables and shell aliases for every user, and run an nftables firewall that leaves Docker's and Tailscale's rules alone.
sidebar:
  order: 7
---

Two things most machines need, declared like everything else: the environment every user starts with, and a firewall.

## Environment variables

[`environment.variables`](/docs/reference/options/#environmentvariables) sets variables for every user:

```lua title="lua/modules/common.lua"
return {
  environment = {
    variables = {
      EDITOR = "nvim",
      MOZ_ENABLE_WAYLAND = 1,
      XDG_DATA_DIRS = { "/usr/local/share", "/usr/share", "/var/lib/flatpak/exports/share" },
    },
  },
}
```

A list is joined with `:`, so it works for path-like variables; numbers and other values are written as text. archix writes each variable twice, because nothing on Linux reads one file for every program:

- **`/etc/profile.d/archix-environment.sh`**, for login shells. Arch's bash and zsh read `/etc/profile`, which sources it:

  ```sh title="/etc/profile.d/archix-environment.sh"
  export EDITOR='nvim'
  export MOZ_ENABLE_WAYLAND='1'
  export XDG_DATA_DIRS='/usr/local/share:/usr/share:/var/lib/flatpak/exports/share'
  ```

- **`/etc/environment.d/90-archix.conf`**, for systemd user services and graphical sessions started by them:

  ```ini title="/etc/environment.d/90-archix.conf"
  EDITOR=nvim
  MOZ_ENABLE_WAYLAND=1
  XDG_DATA_DIRS=/usr/local/share:/usr/share:/var/lib/flatpak/exports/share
  ```

The shell file quotes every value, so a `$` in it stays a literal `$` there, while `environment.d` expands `${VAR}` references. Write values that don't depend on other variables, and they mean the same in both. New values reach a session the next time you log in.

## Shell aliases

[`environment.shell_aliases`](/docs/reference/options/#environmentshell_aliases) gives every user's login shells the same aliases, from `/etc/profile.d/archix-aliases.sh`:

```lua title="lua/modules/common.lua"
return {
  environment = {
    shell_aliases = { ll = "ls -l", gs = "git status" },
  },
}
```

```sh title="/etc/profile.d/archix-aliases.sh"
alias gs='git status'
alias ll='ls -l'
```

Only login shells read `/etc/profile.d`. A terminal emulator usually starts a shell that isn't a login shell, so for aliases in every interactive zsh, use [`programs.zsh.shell_aliases`](/docs/reference/options/#programszsh), which goes into `~/.zshrc`.

## Firewall

[`firewall.enable = true`](/docs/reference/options/#firewallenable) turns on an nftables firewall. Incoming traffic is dropped unless it answers something the machine sent, or you allow it. Outgoing traffic isn't filtered.

### A server with SSH and a web server

```lua title="lua/hosts/server/init.lua"
return {
  firewall = {
    enable = true,
    allowed_tcp_ports = { 22, 80, 443 },
    allowed_udp_ports = { 443 },
  },
}
```

The switch adds the `nftables` package, writes the rules to `/etc/nftables-archix.conf`, and writes, enables and starts `archix-firewall.service`, which loads them before the network comes up at boot:

```console
$ sudo archix switch
install    nftables 1:1.1.5-1
create     /etc/nftables-archix.conf
create     /etc/systemd/system/archix-firewall.service
enable     archix-firewall.service
apply 4 changes? [Y/n]
```

The rules are an `inet archix` table, for IPv4 and IPv6 alike:

```nft title="/etc/nftables-archix.conf"
#!/usr/bin/nft -f
# Written by archix from the firewall options.
table inet archix
delete table inet archix

table inet archix {
  chain rpfilter {
    type filter hook prerouting priority mangle + 10; policy accept;
    meta nfproto ipv4 udp sport . udp dport { 68 . 67, 67 . 68 } accept
    fib saddr . iif oif missing drop
  }

  chain input {
    type filter hook input priority filter; policy drop;
    ct state vmap { established : accept, related : accept, invalid : drop }
    iifname "lo" accept
    meta l4proto ipv6-icmp accept
    ip6 saddr fe80::/10 udp dport 546 accept
    icmp type echo-request accept
    tcp dport { 22, 80, 443 } accept
    udp dport { 443 } accept
  }
}
```

In order, the input chain accepts replies to connections the machine made, everything on loopback, your trusted interfaces, all ICMPv6, which IPv6 needs to work at all, DHCPv6 replies from link-local addresses, ping unless `allow_ping = false`, your ports, and finally your extra rules. Everything else is dropped.

The `rpfilter` chain checks reverse paths: a packet whose source address wouldn't be routed back out the interface it arrived on is dropped, which stops spoofed traffic. DHCPv4 is let through it, since a client asking for an address has none yet. `check_reverse_path = "loose"` only requires a route back out of any interface, and `"off"` leaves the chain out.

When you change the firewall options, the next switch writes the new rules and reloads the service with `systemctl try-reload-or-restart archix-firewall.service`, as an [`on_change`](/docs/guides/hooks/#on_change) script. The new table replaces the old one in one step, so there's no moment without rules. Setting `enable = false`, or removing the options, stops and disables the service, which deletes only the `archix` table.

### Tailscale and a port range

```lua title="lua/hosts/desktop/init.lua"
return {
  firewall = {
    enable = true,
    allowed_tcp_ports = { 22 },
    allowed_udp_port_ranges = { { from = 60000, to = 61000 } },
    trusted_interfaces = { "tailscale0" },
    check_reverse_path = "loose",
  },
}
```

Everything arriving on `tailscale0` is accepted, so your tailnet reaches every service while the internet only reaches SSH and mosh's UDP range:

```nft
    iifname { "tailscale0" } accept
    tcp dport { 22 } accept
    udp dport { 60000-61000 } accept
```

VPNs that route asymmetrically, such as Tailscale with an exit node or subnet routes, need `check_reverse_path = "loose"`; with `"strict"`, their replies can look spoofed. Ports have to be 1 to 65535, and a range needs `from` no greater than `to`; otherwise `diff` and `switch` stop with `firewall ports must be 1-65535, not 70000` or `firewall port ranges need from <= to, both 1-65535`.

### Extra nftables rules

For anything the options don't cover, write nftables rules. `extra_input_rules` are added to the end of the input chain, after everything above:

```lua title="lua/hosts/server/init.lua"
return {
  firewall = {
    enable = true,
    allowed_tcp_ports = { 22 },
    extra_input_rules = [[
ip saddr 192.168.1.0/24 tcp dport 8080 accept
ip6 saddr fd00::/8 tcp dport 8080 accept
]],
    extra_rules = [[
set blocked {
  type ipv4_addr; flags interval;
  elements = { 203.0.113.0/24 }
}
chain blocklist {
  type filter hook input priority filter - 10; policy accept;
  ip saddr @blocked drop
}
]],
  },
}
```

Here port 8080 is open to the local network only. `extra_rules` adds more sets and chains inside the `archix` table; this `blocklist` chain runs before the input chain, so the blocked range is dropped even on port 22. archix doesn't check the syntax of either: a mistake makes `nft` refuse the file when the service loads it, which the switch reports as a [soft failure](/docs/concepts/safety/#hard-and-soft-failures), and the rules from before stay loaded.

### Other tables are left alone

Docker, Tailscale, libvirt and others manage their own nftables tables. archix never touches them: it doesn't use `nftables.service`, whose stop flushes every table on the machine, and its own service only ever deletes the `archix` table.

Each table's input chain still applies to every packet, though, so a packet another table accepts can still be dropped by `archix`'s. Docker's published ports are forwarded to containers rather than delivered to the host's input chain, so they keep working; a service on the host itself needs its port in `allowed_tcp_ports`. If `nftables.service` is also enabled, the rules in `/etc/nftables.conf` apply too; Arch's default file drops everything but SSH, so disable it with `services = { ["nftables.service"] = false }` when archix's firewall takes over.
