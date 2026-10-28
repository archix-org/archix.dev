---
title: Users, SSH and sudo
description: Declare users with passwords and SSH keys, sudo rules and tmpfiles lines, so a fresh machine comes up ready to log in and reachable over SSH.
sidebar:
  order: 7
---

A machine set up from a config should come up ready to use: its people able to log in, an administrator reachable over SSH, sudo working and the directories its services need in place. All of it goes into the new deployment and takes effect at the next boot, like every `/etc` change.

## Passwords

A user gets its password from one of two options, given as a crypt(3) hash, never the password itself. Make one with `mkpasswd` (yescrypt, from the `whois` package) or `openssl passwd -6` (SHA-512):

```sh
$ mkpasswd
Password:
$y$j9T$u2f4nll1JxfH3hjxKg.8P1$iQmwHBe.iDFgnhAJVdpekC.aTQ8uIWtwoFNGILtAHV9
```

[`initial_hashed_password`](/docs/reference/options/#users) is set when archix creates the account and never again: the machine comes up with it, and `passwd` changes it for good. This is what a personal machine wants, and what NixOS calls `initialHashedPassword`.

[`hashed_password`](/docs/reference/options/#users) is enforced, like NixOS's `hashedPassword` with `mutableUsers = false`: every switch sets it again, so a password changed with `passwd` lasts until the next switch. archix never does that silently; `archix diff` and the switch both say `password   resetting alice's password to the declared one`, and no hash is ever printed. It suits servers and shared machines whose passwords should be what the config says.

```lua
users = {
  alice = { groups = { "wheel" }, initial_hashed_password = "$y$j9T$..." },
  deploy = { hashed_password = "$y$j9T$..." },
},
```

Some things win over an enforced hash:

- **Locks.** An account locked with `passwd -l` (after a leak, say) or expired with `chage -E` stays so; the switch warns and leaves it alone until you unlock it. Only a declared `"!"` replaces it. A lock made after a switch and before its reboot reaches the waiting deployment through the [`/etc` merge](/docs/concepts/safety/#changes-to-etc-before-the-reboot).
- **The rescue entry.** A password reset in the [rescue entry](/docs/guides/recovery/#without-a-live-usb-the-rescue-entry) holds until the config's hash for that user changes, with a login note saying so, so recovering doesn't get undone by the next switch.
- **Rollbacks.** A rollback carries the running system's passwords, as it does for every account, and never brings back an older generation's declared hash.

A hash in the config can be cracked offline by whoever reads the config, and a config usually lives in git: `archix switch` and `archix diff` warn when it has a git remote. Use a long passphrase, keep the repository private, or leave passwords to `passwd`. A secrets option, coming later, will supply hashes from an encrypted file instead.

A switch refuses to leave the machine with nobody able to log in with a password, root or a member of `wheel` or `sudo`, when the running system has someone, since the rescue entry needs one; [`allow_no_password_login`](/docs/reference/options/#allow_no_password_login) allows it.

## SSH keys

[`ssh_authorized_keys`](/docs/reference/options/#users) takes the lines of each user's `.pub` files:

```lua
users = {
  alice = { groups = { "wheel" }, ssh_authorized_keys = { "ssh-ed25519 AAAAC3Nza... alice@laptop" } },
},
services = { ["sshd.service"] = true },
```

They're written to `/etc/ssh/authorized_keys.d/alice` in the deployment, owned by root so the user can't change them, and an sshd drop-in adds that file to `AuthorizedKeysFile`. `~/.ssh/authorized_keys` keeps working beside it, so a key added by hand isn't locked out. Declaring a key installs openssh; enable `sshd.service` yourself, and open port 22 with [`firewall.allowed_services = { "ssh" }`](/docs/guides/environment-and-firewall/) if the firewall is on.

## sudo

```lua
sudo = {
  enable = true,
  wheel_needs_password = true,
  extra_rules = { "alice ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx" },
},
```

[`sudo`](/docs/reference/options/#sudo) installs sudo and writes `/etc/sudoers.d/archix`, letting `wheel` run anything (with `NOPASSWD` when `wheel_needs_password = false`) plus your lines. The build runs `visudo -c` on it inside the new deployment, so a typo fails the switch with nothing changed instead of leaving sudo broken.

## Directories at boot

[`tmpfiles`](/docs/reference/options/#tmpfiles) takes [tmpfiles.d](https://man.archlinux.org/man/tmpfiles.d.5) lines, which systemd applies at every boot:

```lua
tmpfiles = { "d /var/lib/foo 0750 foo foo -" },
```

They apply at the new deployment's boot, not at the switch, and the build checks them with `systemd-tmpfiles --dry-run`, so an owner that doesn't exist fails the switch. Lines touching archix's own paths are refused.

## From an existing machine

[`archix init`](/docs/reference/cli/#init) turns hand-made `/etc/tmpfiles.d` files into `tmpfiles` lines and keys in `/etc/ssh/authorized_keys.d` into the users it declares. It never captures password hashes: they're secrets, and the machine keeps the passwords it has.
