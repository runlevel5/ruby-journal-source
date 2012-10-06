---
layout: post
title: "Install Ubuntu with PXE via OSX"
date: 2012-10-06 14:21
comments: true
tags: linux, osx, dhcp, deployment, ubuntu, tftp
author: "Trung Lê"
---

# {{ post.title }} #

In this tutorial, I'll guide you through how to to setup OSX as PXE server to install Ubuntu on other hosts.

<!--more-->

## Introduction

[The Preboot Execution Environment (PXE)][pxe] is widely used in enterprise environment for mass deployment, however it is not well-known in home and office environment because it is always easier to install Ubuntu Linux using traditional CD/DVD or USB storage devices method.

If your box doesn't have CD/DVD or USB storage devices, you can install Ubuntu using PXE. The concept is very simple, a computer that host TFTP server and the to-be-installed host support PXE.

I adapt the concept to OSX to show you that you could achieve the same result with Mac OSX. In this tutorial, I use OSX 10.8.2. However, it should also work with older versions.

## Prerequisites

* [Netboot installer for Ubuntu][netboot-ubuntu]
* TFTP Server
* A PC that is to be installed to supports PXE
* Fast and reliable Internet connection
* Time and patience

## Download Ubuntu Netboot installer

We only need the netboot installer of Ubuntu, you don't have to download a full ISO for the purpose. The file that we need to download is the `netboot.tar.gz` which can be found at [http://cdimage.ubuntu.com/netboot/][netboot-ubuntu].

1. Open your *Terminal.app*
2. Use *curl* command to download the netboot installer:

```
$ cd /tmp
$ curl -O http://archive.ubuntu.com/ubuntu/dists/precise/main/installer-i386/current/images/netboot/netboot.tar.gz
```

3. Extract `netboot.tar.gz` to suitable folder that we are going to use as TFTP root folder

```
$ mkdir ~/Downloads/tftp
$ tar -xvzf netboot.tar.gz -C ~/Downloads/tftp
```

4. Fix folder permissions:

```
$ sudo chown -R nobody:nogroup ~/Downloads/tftp
```

## Set up TFTP server

### Easy setup with TFTP Server software

Though OSX comes with `tftp` command line, it still takes more steps to setup manually. Alternatively, you can download the TFTP Server. It provides a GUI to configure the built-in tftp server on OSX.

1. Download the [TFTP Server][tftp-server].
2. Open the DMG file and drag the application into your Applications folder.
3. Open the *TftpServer.app*.
4. We will NOT use the default `/private/tftpboot` as root for TFTP server. From *TftpServer.app*'s screen, click on *Change Path* icon and select `Downloads/tftp` and you should see the path bar change to `/Users/yourusername/Downloads/tftp`.

5. Fixing folder permission by click on *Fix* buttons of both `Working path permission` and `Parentals folders permissions`. You should see `Attributes OK` if all goes well.

6. Start the server by clicking on *Start TFTP*. You should see *Server Status* change to `Running`.


### Hard setup using built-in tftp server

For whom who want to set up the `tftp` manually via CLI.

1. Configure the server by modify the `tftp.plist` into a suitable folder for modification:

```
$ cp /System/Library/LaunchDaemons/tftp.plist /tmp
```

2. Edit the file `/tmp/tftp.plist` and change:

```
<key>Disabled</key>
<true/>
```

to

```
<key>Disabled</key>
<false/>
```

also modify your TFTP root folder into the `<array>` section:

```
<array>
  <string>/usr/libexec/tftpd</string>
  <string>-s</string>
  <string>/Users/yourusername/Downloads/tftp</string>
</array>
```

3. Backup the existing configuration file:

```
$ sudo mv /System/Library/LaunchDaemons/tftp.plist /System/Library/LaunchDaemons/tftp.plist.backup
```

4. Copy our configuration file into `/System/Library/LaunchDaemons/`:

```
$ sudo mv /tmp/tftp.plist /System/Library/LaunchDaemons
```

5. Start the server with:

```
$ sudo launchctl load -F /System/Library/LaunchDaemons/tftp.plist
```

OR

```
$ sudo launchctl start com.apple.tftpd
```

6. Once installation is finished, you could disable it with:

```
$ sudo launchctl unload -F /System/Library/LaunchDaemons/tftp.plist
```

OR

```
$ sudo launchctl stop com.apple.tftpd
```

## Set up DHCP server (Optional)

In concept, PXE-bootable device will look for DHCP service in order to receive available PXE boot server. If you don't have a DHCP servive running locally in router or in your LAN, you have to set up a DHCP server.

If you are using OSX Server, it'll be relatively easy to setup DHCP server. If you are using non-server version, OSX does come with a built-in DHCP server called `bootpd`, which offer services for DHCP and BOOTP. In fact, this technology is known as NetBoot and used to install OSX on CD/DVD-less machines like MacBook Air or Mac Mini. You can read more about this from [Jacques Fortier's blog][0].

## Booting Ubuntu

To boot from TFTP, you need to configure your PC to boot from the network interface in the BIOS.

Once booting into the installer, you can install Ubuntu by having sources download for a mirror.

[netboot-ubuntu]: http://cdimage.ubuntu.com/netboot/
[tftp-server]: http://ww2.unime.it/flr/tftpserver/
[pxe]: http://en.wikipedia.org/wiki/Preboot_Execution_Environment "Preboot Execution Environment on Wikipedia"
[0]: http://www.jacquesf.com/2011/04/mac-os-x-dhcp-server/ "Mac OSX DHCP Server"