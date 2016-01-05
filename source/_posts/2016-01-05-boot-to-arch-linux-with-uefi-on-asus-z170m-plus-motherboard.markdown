---
layout: post
title: "Setup Arch Linux to boot UEFI on Asus Z170M-PLUS motherboard"
date: 2016-01-05 14:21
comments: true
categories: linux
author: "Trung Lê"
---

{{ post.title }}

There are many ways to get Arch Linux boot on ASUS Z170M-PLUS, one of which is with EFIStub but sadly it _does not_ work easily like other motherboards I've encountered before. The first issue is that the annoying Secure Boot that complicates the set up alot, and the second is the issue with `efibootmgr`'s newly added entries are not persisted in NVRAM (thus not appearing in UEFI menu). But I did have one succes with using `systemd-boot` added in UEFI, then it's just a matter of adding a Arch Linux boot entry with systemd-boot.

## Prerequisites

* Arch Linux Live USB
* Internet (via Ethernet cable or Wifi)
* Time and patience
* Secure Boot is off

## Disable Windows Secure Boot

Because I don't run Windows, it's much better to disable Secure Boot. To do so:

* Get into your BIOS by pressing Del on boot up
* Then press F7 to get into Advance Mode, move to Boot menu, then select Secure Boot
* Choose OS Type from Windows to Other type OS
* Make sure you save this changes, then reboot

## Partition Structure

In my case, I have 2 partitions:

* sda1: `/boot`: Size 500MB, partition type is EFI System and filesystem is FAT32
* sda2: `/`: <whatever you like>

What we care here is the EFI System Partition (ESP), here is the `/boot` partition.
This is the placeholder for my Linux kernel and all EFI-related stuffs.

## Install systemd-boot EFI

Boot into your Arch Linux LiveUSB, then mount your partition to `/mnt`:

```
$ mount /dev/sda2 /mnt
$ mount /dev/sda1 /mnt/boot
```

then change root into it by

```
$ arch-chroot /mnt
```

now we need to install `systemd-boot`:

```
$ bootctl --path=/mnt install
```

It will copy the systemd-boot binary to your EFI System Partition (/mnt/EFI/systemd/systemd-bootx64.efi and /mnt/EFI/Boot/BOOTX64.EFI - both of which are identical - on x64 systems) and add systemd-boot itself as the default EFI application (default boot entry) loaded by the EFI Boot Manager. The next time when you boot up your machine, you would see `Linux Boot Manager` option appears in UEFI Boot Menu.

Next, we add a new boot entry for our Arch Linux kernel inside systemd-boot. To do so, simple create a new file under `/boot/loader/entries/my_arch_linux.conf with following content:

```
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux
options root=/dev/sda2 rw
```

All done, now we need to exit out to our Live USB env by typing `exit` then unmount the mounted sda by `umount -R /mnt` and reboot with command: `reboot`.

Upon seeing the Asus logo, press F8 to see UEFI Boot Menu, choose Linux Boot Manager and you now can see the systemd-boot EFI boot manager screen, select Arch Linux and your set to go.

```