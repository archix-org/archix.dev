---
title: Secrets
description: Keep passwords, keys and tokens in the config encrypted with age, decrypted only at boot into a ramfs, with links, templates, credentials and password hashes for the services that need them.
sidebar:
  order: 10
---

A WireGuard private key, a Wi-Fi password, an API token or a password hash doesn't belong in a config you push to git. archix keeps each one as an [age](https://age-encryption.org) file in the config, encrypted to the machine that uses it and to you, and decrypts it only when the deployment boots, into a `ramfs` at `/run/archix/secrets` that never reaches a disk or swap. Services read it from there, through a link, a rendered file, `EnvironmentFile=` or `LoadCredential=`.

## What it protects

Encrypted secrets protect the **config**: its repository and every clone, fork and backup of it, a leaked `archix.lock` or generation record, package caches shared between machines, and screenshots of `archix diff` and `status`. The value is never in a deployment, a btrfs snapshot archix takes, its caches, its logs or the journal, its `/etc` merge state or the backups it makes.

They don't protect against:

- **root on the machine**, which can read the identity in `/var/lib/archix/secrets` and the decrypted values in `/run/archix/secrets`, as with agenix and sops-nix;
- **anyone who reads an unencrypted disk**, because the machine's identity is a file on it. Put the root on LUKS: `archix status` warns when secrets are declared and the root isn't on dm-crypt;
- **services that copy what they read**: an app that writes its token into its own state, logs it, or keeps it in memory that gets swapped;
- **old values**: git history keeps every earlier ciphertext, so when a key leaks, change the credential itself, not only its encryption.

Two things put a value where the rest of the system can read it, and only when you ask for them: a password hash from a secret goes into `/etc/shadow`, as `passwd` would put it, since logins read it there; and [`eject --secrets materialize`](#leaving-archix) writes each secret to a real file, as plain Arch keeps them.

## Set up

Make this machine's key, on the machine:

```console
$ sudo archix secrets init
made this machine's secrets identity in /var/lib/archix/secrets
put this in the host's config:

  secrets = { machine_key = "age1qyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqs3290gq", admins = { "<your age or ssh-ed25519 key>" } }

archix never copies this identity anywhere: your admin recipients are the backup, so keep the admin key safe (on paper, a YubiKey or in a password manager). Without one, losing this machine's key loses every secret. Leave /var/lib/archix/secrets out of backups, or encrypt them.
```

The identity is a plain age X25519 key in `/var/lib/archix/secrets/identity` (0400, in a 0700 directory). `/var` is shared by every deployment, so a rollback or an older entry in the boot menu finds the same key. `archix secrets pubkey` prints its recipient again.

Then add your own key as an admin. Every secret is encrypted to it too, and it's what gets them back when a machine's key is lost. An age key (`age-keygen -o ~/.config/archix/identity`), an `age-plugin-yubikey` recipient, or your `~/.ssh/id_ed25519.pub` all work:

```lua
-- lua/hosts/laptop.lua
return {
  secrets = {
    machine_key = "age1qyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqszqgpqyqs3290gq",
    admins = { "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... me@desk" },
    items = {
      wg0 = { file = "secrets/wg0.conf.age" },
      api_key = { file = "secrets/api.age", owner = "me", group = "me", mode = "0400" },
    },
  },
}
```

A config that declares secrets with no admin is refused: `secrets.admins is empty: an admin recipient (your own age or ssh-ed25519 key) is the only way back to these secrets if this machine's key is lost; add one, or set secrets.without_admin = true to go on knowing that losing the key loses every secret`.

## Write a secret

`archix secrets edit NAME` decrypts the secret, or starts it empty, opens your editor on it, and encrypts what you save to every recipient:

```console
$ archix secrets edit wg0
encrypted secret wg0 to 2 recipients into secrets/wg0.conf.age
```

The plain text lives in a fresh directory on a private `ramfs`, mounted in a mount namespace of archix's own, so it never reaches a disk or swap and is gone when archix and the editor exit, by a kill too. Where no private ramfs can be had, it goes in `$XDG_RUNTIME_DIR`, which must be a tmpfs or ramfs, or archix refuses. tmpfs can swap, and so can the editor's own memory, so archix warns when swap isn't encrypted. With vim and nvim it turns off undo files, swap files, backups and viminfo/shada; with other editors it warns that it can't, so check that yours keeps no copies.

`archix secrets set NAME` encrypts what comes on stdin, or `--file F`, for piping from a password manager; at a terminal it reads one line without echo:

```console
$ pass show wifi/home | archix secrets set home_wifi
encrypted secret home_wifi to 2 recipients into secrets/home.nmconnection.age
```

Files are ASCII-armored age v1, so `age -d -i ~/.config/archix/identity secrets/wg0.conf.age` reads them by hand and agenix's files work as they are. Each write goes through a temporary file and a rename, so a kill leaves the old file.

`edit` and `rekey` decrypt with your own identities: `$ARCHIX_AGE_IDENTITY`, `~/.config/archix/identity` (which may name age plugins such as `age-plugin-yubikey`), then `~/.ssh/id_ed25519` and `~/.ssh/id_rsa`, asking for a passphrase if one is set. As root on a machine, `--machine` decrypts with its own key instead.

### Who a secret is encrypted to

archix encrypts each file to the `machine_key` of every host whose config declares it, and to those hosts' admins: it evaluates every host in `lua/hosts/`, so there's no separate recipients list to keep. It records who each file was encrypted to, and the ciphertext's sha256, in `secrets/recipients.toml`, since an age file doesn't say. After adding a host or an admin:

```console
$ archix secrets rekey
re-encrypted secrets/wg0.conf.age to 3 recipients
the old ciphertexts stay in the config's git history: if a key was lost or leaked, change the credentials themselves, not only their encryption
```

`archix secrets check` lists, for each secret, who it should be encrypted to against who it was, whether this machine decrypts it, and what uses it, and fails when one needs a rekey:

```console
$ sudo archix secrets check
wg0  secrets/wg0.conf.age
  recipients: the 2 the config names
  this machine: decrypts it
  used by: /etc/wireguard/wg0.conf
api_key  secrets/api.age
  recipients: unknown, as it was encrypted outside archix; `archix secrets rekey api_key` encrypts it to the config's
  this machine: decrypts it
  used by: /run/archix/secrets/api_key
```

## Use a secret

Evaluation never sees a value. `archix.secret("name")` is a placeholder and `archix.secret_path("name")` is the path `/run/archix/secrets/name`, which every declared secret gets at boot, with its `owner`, `group` and `mode` (root's and 0400 by default). There are four ways to hand one to a service.

**A link.** `files[path] = { secret = "name" }` makes the file a link to the secret. The deployment holds only the link:

```lua
files = { ["/etc/wireguard/wg0.conf"] = { secret = "wg0" } },
```

**A template.** A file's `text` may hold placeholders. The text, with placeholders and no values, goes into the deployment, and the boot renders it into `/run/archix/secrets/files/`, with the file's owner and group and its mode without the bits for others; the file is a link to that:

```lua
files = {
  ["/etc/netbird/env"] = { text = "NB_SETUP_KEY=" .. archix.secret("nb_key") .. "\n", mode = "0640" },
},
```

**A path under `/run`.** For apps that won't follow a link, `path` puts the secret at a file under `/run` at each boot, in a ramfs mounted on that file's directory, so the value never sits on `/run`'s swappable tmpfs, and an app that saves by writing a new file and renaming it over the old one edits it in place. NetworkManager reads profiles from `/run/NetworkManager/system-connections`, and `nmcli connection modify --temporary` edits one there, in the ramfs. Without `--temporary`, and from GUI editors, NetworkManager saves a `/run` profile by moving it to `/etc/NetworkManager/system-connections`, key and all, on disk; change the profile in your config instead. If that happens, the next switch finds the value in `/etc` and refuses, naming the file, which you then delete. A directory that already holds other files at boot gets the ramfs file bound over the path instead, which an app can read but not replace:

```lua
secrets.items.home_wifi = {
  file = "secrets/home.nmconnection.age",
  path = "/run/NetworkManager/system-connections/home.nmconnection",
  mode = "0600",
}
```

**A unit.** A unit file is loaded before anything is decrypted, so it can't hold a secret; give the service its path with `EnvironmentFile=` or `LoadCredential=`, which fail the unit when the file is missing:

```lua
units["backup.service"] = "[Service]\nLoadCredential=key:" .. archix.secret_path("api_key") .. "\nExecStart=/usr/local/bin/backup\n",
```

`LoadCredential=` copies the value into the service's own credentials directory and suits `DynamicUser=`. A placeholder anywhere else is refused with the option's name: `kernel_params.[1]: a secret can't go here; ...`, `units.x.service: a unit file is loaded before any secret is decrypted, so it can't hold one; give the service EnvironmentFile= or LoadCredential=NAME:/run/archix/secrets/<secret> (archix.secret_path)`. A merged file (`merge`) can't take one either, as it's an app's own file in a deployment's `/etc`. Home files can't hold secrets yet.

### A password

A secret of `kind = "crypt_hash"` holds a crypt(3) hash, such as `mkpasswd` prints, and sets a user's password:

```lua
secrets.items.me_password = { file = "secrets/me.hash.age", kind = "crypt_hash" },
users.me = { groups = { "wheel" }, hashed_password = archix.secret("me_password") },
```

```console
$ mkpasswd | archix secrets set me_password
```

The switch checks that it's a whole yescrypt, SHA-512 or bcrypt hash, without printing it, and the deployment's boot sets it in `/etc/shadow`, by the same rules as [`users.<name>.hashed_password`](/docs/reference/options/#users): a locked or expired account stays so, and a password set in the [rescue entry](/docs/guides/recovery/#without-a-live-usb-the-rescue-entry) holds, with a login note, until a switch changes the secret's value. A `passwd` change lasts until the next boot.

## What a switch checks

When this machine's identity is there, a switch decrypts every secret into locked memory before it builds anything, and refuses one that doesn't decrypt here, a hash that doesn't parse, or a value above 1 MiB (16 MiB together), with nothing changed:

```console
$ sudo archix switch
error: secret wg0 isn't encrypted to this machine (age1qyqszqgp...3290gq); run `archix secrets rekey` where an admin key is available, or switch with --without-secrets; nothing was changed
```

The decryption and these scans run with all of archix's memory locked, the age library's own buffers included, so none of it can be swapped out. With the values in memory, it also refuses a managed file that holds one, and, just before it commits, looks for each value of 8 bytes or more in the new deployment's `/etc` and its UKIs and initramfs images, compressed streams opened. A find in a boot image or a managed file fails the switch; in `/etc`, a value of 16 bytes or more fails it and a shorter one only warns, since it may be chance: `secret pin's value, under 16 bytes, also occurs in /etc/archix-test-pin; that may be chance, but check that nothing copied it there, and prefer a longer secret`. The deployment's `/etc/shadow` is the exception for password hashes. That catches a value copied into the initramfs by mkinitcpio's `FILES`, or written into a config file by hand.

`archix diff` and the switch name each secret and what uses it, never its value: `secret     wg0: new (/etc/wireguard/wg0.conf)`, `changed`, `re-encrypted only` (the same value, encrypted again) or `removed`. Telling `changed` from `re-encrypted only` needs root and the machine's key; without them `diff` says it can't tell.

A regular file already at a secret's path, as on a machine you set up by hand, is replaced only when it holds exactly the value: `replacing  /etc/wireguard/wg0.conf, which holds secret wg0's value, with a link to it; the old file isn't backed up`. A file that holds anything else is refused, since archix won't keep a copy of what may be a secret: move it away yourself. The deployment it was in keeps it until gc drops that deployment.

## At boot

Each deployment carries its secrets' ciphertexts and a manifest in `/usr/lib/archix/secrets`, and needs nothing else to boot: a rollback, an older entry in the boot menu, or a rebuilt dropped generation decrypts the values that deployment was built with. Rebuilding one needs no identity, so it works from a live USB.

`archix-secrets.service` runs early, after `/var` is mounted, swap is on and system users exist, and before `sysinit.target`, so services and first-boot hooks find the secrets in place. It mounts a `ramfs` at `/run/archix/secrets` (0711, so names can't be listed), which never swaps, writes each secret whole or not at all, owner and mode set before a byte is in it, renders the templates, puts the `path` files in place and sets the password hashes. It runs with its memory locked, out of core dumps, and with no core file limit, so a crash leaves nothing behind. Nothing is decrypted again until the next boot.

A secret that can't be decrypted is left out and named, and the others are written; what uses it fails closed: a dangling link, a missing `EnvironmentFile=` or credential. The unit fails, a login note and `archix status` say why:

```console
$ archix status
...
secret nb is missing at this boot: its ciphertext in the deployment is damaged
```

Without the identity, nothing is decrypted, and the login note says how to get it back. Secrets aren't part of [boot counting](/docs/reference/options/#boottries), since every deployment shares the identity and falling back wouldn't help; list `archix-secrets.service` in [`boot.health_check`](/docs/reference/options/#boothealth_check) to make them count.

### Links you edit

Editing through a link, as vim does by default, changes the copy in the ramfs, which is gone at the next boot; edit the secret instead. An editor that replaces the link with a regular file puts the plain text into the running deployment's `/etc`. archix can't stop that, but doesn't spread it: the [`/etc` merge](/docs/concepts/safety/#changes-to-etc-before-the-reboot) never carries a secret's path, `archix status` warns, `/etc/wireguard/wg0.conf should be a link to secret wg0 but is a file; it holds plain text on disk in deployment 42, and the next switch refuses until it's moved away`, and the next switch refuses until it is.

## A new machine

On a new machine, `archix secrets init` makes its key, which goes into the host's config; then `archix secrets rekey` where an admin key is, such as the machine itself with your YubiKey plugged in or your ssh key there. Until then, a switch refuses, or with `--without-secrets` commits a deployment that leaves out the secrets this machine can't decrypt; what uses them fails until a switch after the rekey, and `status` lists them. A password hash from a secret counts as a login for the [lockout check](/docs/reference/options/#allow_no_password_login) only when the switch decrypted and checked it, so not with `--without-secrets` or in a chroot.

Restoring the old `identity` file from your own backup needs no rekey.

## Rotating the machine's key

```console
$ sudo archix secrets rotate-machine-key
this machine's new key is age1zv8yjzyh...
1. put it in the host's secrets.machine_key, in place of age1qyqszqgp...
2. `archix secrets rekey` where an admin key is (or `sudo archix secrets rekey --machine` here, which decrypts with the retired key)
3. `archix switch` and reboot
the old key stays in /var/lib/archix/secrets/identity.retired while a kept deployment needs it, and gc deletes it after
```

The old key is retired, not deleted: deployments and generations built before still decrypt with it, so a rollback across the rotation boots with its secrets. Once no kept deployment or generation has a ciphertext that only the retired key opens, gc deletes it: `deleted the retired secrets identity age1qyqszqgp..., which no kept deployment needs`. If the key leaked, `--compromised` deletes it at once, and you should change every credential it protected: its ciphertexts are in the config's history for whoever has it.

## Losing the machine's key

With an admin key, nothing is lost: `sudo archix secrets init` makes a new key, put it in `machine_key`, `archix secrets rekey` with the admin key, switch and reboot. Deployments built before the loss decrypt nothing until a switch replaces them; `status` names their secrets. Without an admin key, every secret is gone, which is why archix asks for one.

archix never copies the identity into a backup, archive or cache. A backup of `/var` you make yourself, and snapper or Timeshift snapshots of the root, do hold it: leave `/var/lib/archix/secrets` out, or encrypt them.

## Swap and hibernation

The ramfs and archix's own buffers never swap, but the services reading the secrets can. `archix status` warns while secrets are declared and a swap, active or in fstab, isn't dm-crypt or zram. Hibernation writes all of memory to swap, the ramfs included, so archix turns hibernation, suspend-then-hibernate and hybrid sleep off while secrets are declared and a swap isn't encrypted, as it does while a deployment waits for its reboot: `hibernation is off while secrets are declared and swap isn't encrypted (/dev/nvme0n1p3): it would write every decrypted secret to it; encrypt the swap, or set secrets.allow_hibernation = true`. Suspend still works. `secrets.allow_hibernation = true` lifts only this refusal: hibernation stays off while a deployment waits for its reboot, and in a deployment that swaps on a file its initramfs can't resume from.

## Leaving archix

Plain Arch has nothing to decrypt secrets at boot, so [`archix eject`](/docs/reference/cli/#eject) asks what becomes of them:

- `--secrets drop` removes their links; what used them goes without. The identity and ciphertexts stay in `/var/lib/archix/secrets`, so `age -d -i /var/lib/archix/secrets/identity` decrypts one by hand.
- `--secrets materialize` writes each secret, as this boot decrypted it, to its path as a real file with its owner and mode, as plain Arch keeps credentials. Units that read `/run/archix/secrets` lose theirs.

The [rescue entry](/docs/guides/recovery/#without-a-live-usb-the-rescue-entry) never decrypts anything, so its `eject` drops them.

## Builds never see them

An AUR package's PKGBUILD is code from the internet that runs during a switch. Every build, in the clean chroot, on the host as the build user, or in the build view of a laid-out system, runs where `/run/archix/secrets` and `/var/lib/archix/secrets` are covered by an empty, unreadable tmpfs or don't exist, whatever their permissions, and no value is ever in a build's environment or command line.

## Testing

Every secrets scenario in the container tests and the `secrets` VM suite ends with a leak sweep for fresh random markers, each of at least 8 bytes: every deployment and staging leftover, `/var` with archix's state, blobs, generations and backups, the package caches and a `package_cache_push` directory, the config but its `.age` files, the ESP with its boot images' compressed streams opened, the journal, core dumps and the output of every archix command. The VM suite ends with a grep of the raw root and swap partitions after memory pressure, freed blocks included. See [Feature status](/docs/concepts/status/).
