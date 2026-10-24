---
title: Secure Boot
description: Enroll your own Secure Boot keys with archix, keep every boot entry signed, check what the firmware and anti-cheat software see, and add a protected boot entry.
sidebar:
  order: 8
---

archix boots every generation as its own deployment, from a unified kernel image on the ESP. With Secure Boot, the firmware only starts images signed by a key it trusts. archix signs every entry, and systemd-boot itself, with your own keys through `sbctl`, and it won't make an entry the default if the firmware would refuse it.

This needs [`boot.loader = "systemd-boot"`](/docs/reference/options/#bootloader), which every switch does.

## Set up your keys

`archix secure-boot setup` creates `sbctl` keys if you don't have any, and enrolls them in the firmware. What it can do depends on the firmware's state.

### Firmware in setup mode

Enrolling keys needs the firmware in setup mode, where it has no platform key of its own. Most firmware settings have an option to reset or clear the Secure Boot keys, which puts it there. Then:

```console
$ sudo archix secure-boot setup
enrolled your Secure Boot keys; the next `archix switch` signs its boot entries with them
then turn Secure Boot on in the firmware settings
```

Your keys go into the firmware's own `db`, together with Microsoft's, with `sbctl enroll-keys --microsoft`. Run `sudo archix switch` to build a deployment with signed entries, then reboot into the firmware settings and turn Secure Boot on.

### Firmware not in setup mode

Setup still creates the keys, but can't enroll them, and tells you what to do:

```console
$ sudo archix secure-boot setup
the keys exist, but the firmware isn't in setup mode, so they can't be enrolled
reboot into the firmware settings, reset or clear the Secure Boot keys (setup mode), then run `sudo archix secure-boot setup` again
```

On a machine that didn't boot through UEFI, it stops with `this machine didn't boot through UEFI, so it has no Secure Boot`. `sbctl` has to be installed: [`boot.secure_boot = true`](/docs/reference/options/#bootsecure_boot) declares it, or install it with `pacman -S sbctl`.

### Without Microsoft's keys

`--no-microsoft` enrolls only your own keys:

```console
$ sudo archix secure-boot setup --no-microsoft
warning: without Microsoft's keys, GPU option ROMs and Windows may fail to start, and some boards won't boot at all
```

Many graphics cards and network cards carry firmware (option ROMs) signed by Microsoft, which the firmware runs before any boot loader. Without Microsoft's keys in `db`, those don't load, and on some boards that leaves no display or no way to boot. Leave Microsoft's keys in unless you know your hardware doesn't need them.

## Signing

[`boot.secure_boot`](/docs/reference/options/#bootsecure_boot) decides whether archix signs. With the default, `null`, it signs whenever `sbctl` has keys; `true` always signs and declares `sbctl`; `false` never signs. Every entry archix writes is signed with `sbctl sign`, one UKI per installed kernel, and so is systemd-boot's own binary, `EFI/systemd/systemd-bootx64.efi` and `EFI/BOOT/BOOTX64.EFI`. The [fallback image](/docs/concepts/atomicity/#the-fallback-boot-image) is a copy of the signed UKI of `current`.

archix enrolls your keys directly in the firmware. It doesn't use shim or a Machine Owner Key (MOK). Booting through shim needs a kernel signed by a distribution key that Microsoft's shim trusts, and Arch doesn't ship one, so there's nothing for shim to chain to; with your own keys in `db`, archix's signed entries boot directly.

### A switch never makes an unbootable entry the default

With Secure Boot enforcing, which means on and not in setup mode, a boot entry that isn't signed would be refused by the firmware. archix checks this twice.

Before a switch builds anything, archix checks that the new entry will be signed. If it won't be, because `boot.secure_boot = false` or because `sbctl` has no keys, the switch stops:

```console
error: Secure Boot is on, but this switch's boot entry wouldn't be signed (sbctl has no keys), so the firmware would refuse it; nothing was changed. Run `sudo archix secure-boot setup`, or set boot.secure_boot = true if sbctl already has keys
```

With `boot.secure_boot = false`, the parenthesis reads `(boot.secure_boot = false)`.

As a backstop, once the new deployment's boot image is built and before it's committed, archix checks the image, and the systemd-boot loader on the ESP, for a signature. An unsigned one stops the build:

```console
error: Secure Boot is on, but 7.2.9-arch1-1-aedff12d49d2.efi isn't signed, so the firmware would refuse it; nothing was committed. Run `sudo archix secure-boot setup` (or set boot.secure_boot = true if sbctl already has keys)
```

The image is built from the kernel in the new deployment, so it can't be checked earlier, and the check before the switch already refuses every case where archix knows it won't sign. The staging tree is deleted, and the running system and its boot entries are as they were. Fix the keys and switch again. See [Safety](/docs/concepts/safety/#secure-boot).

### In a chroot

Inside a chroot, such as `arch-chroot` from a live USB, the firmware belongs to the machine running the live system, not to the system you're setting up. So both checks only warn: `warning: this host's Secure Boot is on and the boot entry won't be signed; ignored, since this runs in a chroot` before the switch, and `warning: not checking the new boot entry's signature against this host's Secure Boot, since this runs in a chroot` when the entry is written.

### Other boot entries

[`boot.extra_entries`](/docs/reference/options/#bootextra_entries) adds systemd-boot entries archix doesn't build, such as memtest86+. archix writes the entry file but doesn't sign what it starts, and the firmware refuses an unsigned image like any other. The image has to be on the ESP. Sign it with `sbctl`; `-s` also adds it to the files `sbctl` re-signs:

```console
$ sudo sbctl sign -s /efi/memtest86+/memtest.efi
```

## Check the state

`archix secure-boot check` reports on the boot you're running, one line per requirement, and exits non-zero if any isn't met. It doesn't need root or a config.

```console
$ archix secure-boot check
  ok  Secure Boot is on
  ok  the firmware isn't in setup mode (keys are enrolled)
```

`archix status` has a line for it too, `Secure Boot: on`, `Secure Boot: off`, or ``Secure Boot: setup mode (no keys enrolled; `sudo archix secure-boot setup` enrolls them)``. On a machine that didn't boot through UEFI, `status` leaves the line out, and the check reports `no  the machine didn't boot through UEFI`.

### Anti-cheat requirements

Some games' anti-cheat, Easy Anti-Cheat's Linux v2 among them, only runs on a boot it can trust. `--anticheat` checks what it asks for:

```console
$ archix secure-boot check --anticheat
  ok  Secure Boot is on
  ok  the firmware isn't in setup mode (keys are enrolled)
  no  the kernel is signed with a well-known key (archix signs with your own key, and Arch has no well-known-signed kernel)
  ok  kernel lockdown is in integrity mode
  ok  kexec is disabled
  ok  the BPF LSM is active
  ok  the IOMMU is on, without passthrough
  ok  a TPM 2.0 is present
  ok  the boot was measured into the TPM
  no  no out-of-tree or unsigned kernel modules are loaded (NVIDIA and DKMS modules count)
Easy Anti-Cheat Linux v2 would refuse this boot: 2 requirements aren't met
```

| Requirement | What it reads |
| --- | --- |
| the kernel is signed with a well-known key | A kernel signed by a key the anti-cheat vendor trusts. archix entries are signed with your own key, and Arch has no well-known-signed kernel, so this is never met on an archix entry. Booted another way, it says `(not booted from an archix entry, so archix can't tell)`. |
| kernel lockdown is in integrity mode | `/sys/kernel/security/lockdown` shows `[integrity]`. |
| kexec is disabled | `kernel.kexec_load_disabled` is `1`. |
| the BPF LSM is active | `bpf` is in `/sys/kernel/security/lsm`. |
| the IOMMU is on, without passthrough | An IOMMU is present and no IOMMU group is in `identity` (passthrough) mode. |
| a TPM 2.0 is present | The first TPM reports major version 2. |
| the boot was measured into the TPM | The firmware's measurement log exists. |
| no out-of-tree or unsigned kernel modules are loaded | The kernel isn't tainted by an out-of-tree or unsigned module. NVIDIA's driver and DKMS modules count. |

The summary only appears with `--anticheat`, and says `1 requirement isn't met` for a single one.

## The protected entry

Most of those requirements are kernel command line settings you don't want on every boot. [`boot.protected_entry = true`](/docs/reference/options/#bootprotected_entry) adds a separate entry for them:

```lua title="lua/hosts/desktop/init.lua"
return {
  boot = { loader = "systemd-boot", secure_boot = true, protected_entry = true },
}
```

Next to its normal entry, each kept deployment then gets `archix-<n>-protected.conf`, titled "Arch Linux (archix protected, generation 43)", signed like the others. Its command line adds:

```
lockdown=integrity sysctl.kernel.kexec_load_disabled=1 lsm=landlock,lockdown,yama,integrity,bpf iommu.passthrough=0
```

Pick it in the boot menu when you want it. It covers lockdown, kexec, the BPF LSM and the IOMMU. It can't make the kernel well-known-signed, so the first anti-cheat requirement stays unmet, and loaded NVIDIA or DKMS modules still count against the last one. `archix secure-boot check --anticheat` shows what's still missing.

## Revoking old boot images

:::caution[Experimental]
Revoking is tested on OVMF, QEMU's firmware, not on real machines, whose variable stores differ. Try it where you can reach the firmware settings to clear `dbx` if something goes wrong.
:::

Every UKI archix signed boots wherever it's put, including ones `gc` deleted, with the older kernel and initramfs they carry, unless the firmware's `dbx` lists them. With your own keys there's no shim to enforce SBAT, so `dbx` is what revokes: `archix secure-boot revoke` adds the hashes of the UKIs archix signed and no longer uses, and [`boot.revoke_unused = true`](/docs/reference/options/#bootrevoke_unused) does that after every switch.

It only ever appends: the update is signed with your KEK and written with the firmware's append flag, so the entries Windows and Microsoft's own `dbx` updates put there stay. archix's entries carry its own owner GUID, and it caps them at 128, since `dbx` lives in the same small NVRAM store Windows needs room in for its updates. It never revokes a hash any `.efi` file on the ESP has, and once it has revoked anything, every UKI it builds gets a random `.archix` section, so a rebuild can never come out identical to a revoked one.

It needs the KEK as a file, as sbctl creates it by default, and `efitools` for the signed write. The `dbx` from before is saved in `/var/lib/archix/boot/dbx-before-<time>`.

## TPM-bound secrets

:::caution[Experimental]
A VM suite unlocks a LUKS volume bound this way with swtpm, a software TPM, under every UKI archix signs with the key, an older kept deployment's included, but no test has used a real TPM yet. Keep a passphrase or recovery key on any volume you bind this way.
:::

A disk key or other secret sealed to PCR 11 breaks whenever the UKI changes, since a new kernel, initramfs or command line measures differently. systemd's answer is a signed PCR policy: the UKI carries a policy for its own PCR 11 values, signed with your key, and the secret is bound to the key instead of the values. archix signs one into every UKI it builds once systemd's PCR signing key exists, at `/etc/systemd/tpm2-pcr-private-key.pem` with its public half at `/etc/systemd/tpm2-pcr-public-key.pem`, where `systemd-cryptenroll` looks for the public key. Create the key yourself; `ukify genkey` needs `python-cryptography`:

```console
$ sudo ukify genkey --pcr-private-key=/etc/systemd/tpm2-pcr-private-key.pem --pcr-public-key=/etc/systemd/tpm2-pcr-public-key.pem
```

From the next switch that builds a deployment on, UKIs carry `.pcrsig` and `.pcrpkey` sections, since the key is part of what a UKI's name hashes; a switch with nothing to do builds none. Then bind the secret to the key, for a LUKS volume:

```console
$ sudo systemd-cryptenroll --tpm2-device=auto --tpm2-public-key=/etc/systemd/tpm2-pcr-public-key.pem --tpm2-public-key-pcrs=11 /dev/nvme0n1p2
```

Every later switch signs the new UKI's policy with the same key, so the secret keeps unlocking. Anyone who can read the private key can sign a policy for any UKI, so it's as sensitive as your Secure Boot keys. Package scriptlets and hooks don't see it while a switch stages a deployment; only `ukify` does.

### Rotating the PCR key

Every UKI signed with the key carries a valid policy for as long as the volume is bound to it, older ones included. To retire them, or a key that may have leaked, replace the key and bind the volume to the new one:

```console
$ sudo archix secure-boot rotate-pcr-key /dev/nvme0n1p2
wrote a new PCR policy signing key to /etc/systemd in all 2 deployments
bound /dev/nvme0n1p2's TPM2 slot to the new key
run `archix switch` before the next restart: until a UKI built with the new key boots, TPM2-bound volumes unlock only with a passphrase or recovery key, and UKIs built with the old key no longer unlock them
$ sudo archix switch
```

`systemd-cryptenroll` asks for each volume's passphrase or recovery key, and replaces its TPM2 slot. The switch after builds a new deployment whose UKIs carry policies signed with the new key, even when nothing else changed. Until you boot it, the volume asks for its passphrase. See [`secure-boot rotate-pcr-key`](/docs/reference/cli/#secure-boot-rotate-pcr-key).

