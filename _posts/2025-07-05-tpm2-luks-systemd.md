---
title: "update: 'secure' automatic system decryption - tpm2 + LUKS + systemd-cryptenroll"
categories:
  - linux
tags:
  - security
  - tpm 2.0
  - systemd-cryptenroll
---

# 'Secure' automatic system decrpytion with tpm and LUKS - update

As I have written about [here]({% link _posts/2023-12-10-tpm2-luks.md %}), I
sort of have accepted a compromise in security for added convenience,
auto-unlocking my [LUKS](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup)
encrypted root partition making use of my system's built in TPM in order to
only auto-unlock the system under specific (secure) conditions.

I've used `clevis` so far to manage the setup, using the trusty
`initramfs-tools` to configure and bake the _initramfs_ with the required
modules and configuration, relying on `cryptsetup` to manage and reset the
specifics of the configuration, e.g. whenever a bios update invalidated a _PCR
measurement_.

Not that elegant. And there are new, more elegant and capable mechanisms
available!

- There is `systemd-cryptenroll` to enroll physical tokens to LUKS2 encrypted
  volumes, simplifying the setup and management of this part of the process
- There is `dracut` for the configuration and management of _initramfs_ images,
  the designated successor of `initramfs-tools`

While my previous setup worked, I wanted to try setting it up with the
(designated) successor systems.

Well. To be honest, first I just wanted to try out `systemd-cryptenroll`
instead of `clevis`, realized that the intended way of configuring
auto-unlocking didn't work with `initramfs-tools`, realized that (due to that,
amongst other things) there have been discussions about replacing
`initramfs-tools` with `dracut` as default and then decided to redo the whole
process.

## The Problem

Well, it's not really a new problem, it's an itch and a mental problem, I guess
😅. The original challenge, shifting the compromise of convenience vs. security
a little by auto-unlocking the encrypted system with the help of the TPM has
been working for a while.

But:

- `systemd-cryptenroll` exists, potential successor to `clevis`
- `dracut` exists, potential successor to `initramfs-tools`
- Both of those systems are very capable
- My setup is using their (potential) predecessors

## The solution

### systemd-cryptenroll

Installing `systemd-cryptenroll` on Debian is easy enough. It comes with
`systemd-cryptsetup`, which might be on your system already?
If not:

```sh
sudo apt-get install systemd-cryptsetup
```

Then, enrolling the TPM with the encrypted partition is pretty straightforward:

```sh
sudo systemd-cryptenroll --tpm2-device=auto /dev/nvme0n1p3
```

Depending on your needs/level-of-paranoia, you might want to add other PCRs, as
`systemd-cryptenroll` per default only uses PCR 7, maybe you even want to use
TPM + PIN:

```sh
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7+11 --tpm2-with-pin=yes /dev/nvme0n1p3
```

You follow the guided process, and then can verify success. Hopefully.

```sh
sudo systemd-cryptenroll /dev/nvme0n1p3
SLOT TYPE
   0 password
   1 tpm2
```

Configuration should (™️) work by configuring the tpm in `/etc/crypttab` and
rebuilding the _initramfs_ with it:

```sh
cat /etc/crypttab
nvme0n1p3_crypt UUID=e010d6c1-6016-4f2d-bd05-1be8f383ada9 none tpm2-device=auto,luks,discard
```

### building the initramfs with initramfs-tools (fail)

But, alas, rebuilding the initial ram disk fails on Debian and should also fail
on Ubuntu:

```sh
sudo update-initramfs -u -k all
...
cryptsetup: WARNING: nvme0n1p3_crypt: ignoring unknown option 'tpm2-device'
...
```

After a bit of research: `initramfs-tools` doesn't support tpm2 devices. There
is a [bug report](https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=1031254)
and even an open [merge request with a
fix](https://salsa.debian.org/cryptsetup-team/cryptsetup/-/merge_requests/39),
but the situation appears to be stale.

However, there is the designated successor of `initramfs-tools`:
[`dracut`](https://en.wikipedia.org/wiki/Dracut_\(software\)), which has
replaced `initramfs-tools` in a bunch of linux distributions already.

### building the initramfs with dracut

On debian and Ubuntu, it can be installed from the package sources like this,
replacing `initramfs-tools` in the process:

```sh
sudo apt-get install dracut
```

Configuration of dracut happens in `/etc/dracut.conf.d/*.conf`.

I configured dracut to run the host-only initramfs creation with tpm2 support
and compress with `lz4`, all in separate files:

```bash
cat /etc/dracut.conf.d/compression.conf
compress="lz4"

cat /etc/dracut.conf.d/hostonly.conf
hostonly="yes"
hostonly_cmdline="no"

cat /etc/dracut.conf.d/tpm.conf
add_dracutmodules+=" tpm2-tss crypt "
add_drivers+=" tpm tpm_tis_core tpm_tis tpm_crb "
```

With that configuration in place, the initramfs can be (re)built with

```bash
dracut -f
```

After that, a reboot should end up with an  automatically unlocked encrypted
partition.

## tl;dr

If you trust your system firmware and tpm enough, you can set your system up to
decrypt automatically without providing a passphrase using
`systemd-cryptenroll` and `dracut`.

There are also [other methods]({% link _posts/2023-12-10-tpm2-luks.md %}), but
this one appears to be the most future proof and easy to set-up and maintain.
