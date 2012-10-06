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

## Set up DHCP server

In concept, PXE-bootable device will look for DHCP service in order to receive available PXE boot server. If you don't have a DHCP servive running locally in router or in your LAN, you have to set up a DHCP server.

### With isc-dchpd

1. Install isc-dhcpd with Homebrew:

```
$ brew install isc-dhcp
```

2. Create configuration file at `/usr/local/etc/dhcpd.conf`:

```
default-lease-time 600;
max-lease-time 7200;

subnet X.X.X.0 netmask Y.Y.Y.0 {
  range X.X.X.151 X.X.X.205;
}

option domain-name-servers 8.8.8.8;

host netbook {
  hardware ethernet ??:??:??:??:??:??;
  filename "pxelinux.0";
  next-server Z.Z.Z.Z; # the IP address of your TFTP server
  fixed-address X.X.X.202;
  option subnet-mask Y.Y.Y.0;
  option broadcast-address X.X.X.255;
  option routers X.X.X.1;
}
```

in which `X.X.X` is your network address, `Y.Y.Y` is your subnet mask, `??:??:??:??:??:??` is the MAC address of the box you want to install to and finally `Z.Z.Z.Z` is the address of TFTP server.

3. Start the server with:

```
$ sudo /usr/local/sbin/dhcpd -f -d -cf /usr/local/etc/dhcpd.conf
```

4. Once the installation finished, clean up with:

```
$ brew uninstall isc-dhcp
```

### With built-in bootpd

OSX does come with a built-in BOOTP server called `bootpd`, which offer also offer DHCP service. This technology is known as NetBoot and used to install OSX on CD/DVD-less machines like MacBook Air or Mac Mini. I adapt instructions at [Jacques Fortier's blog][0] for this tutorial.

1. Create `/etc/bootpd.plist` with content:

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>bootp_enabled</key>
    <false/>
    <key>detect_other_dhcp_server</key>
    <integer>1</integer>
    <key>dhcp_enabled</key>
    <array>
        <string>en0</string>
    </array>
    <key>reply_threshold_seconds</key>
    <integer>0</integer>
    <key>Subnets</key>
    <array>
        <dict>
            <key>allocate</key>
            <true/>
            <key>lease_max</key>
            <integer>86400</integer>
            <key>lease_min</key>
            <integer>86400</integer>
            <key>name</key>
            <string>192.168.1</string>
            <key>net_address</key>
            <string>192.168.1.0</string>
            <key>net_mask</key>
            <string>255.255.255.0</string>
            <key>net_range</key>
            <array>
                <string>192.168.1.101</string>
                <string>192.168.1.202</string>
            </array>
        </dict>
    </array>
</dict>
</plist>
```

The config file assume that the network address is 192.168.1.0 and the DHCP allocation pool is from .101 to .102.

2. To assign static IP address to our to-be-installed host, we create file `/etc/bootptab`:

```
%%
# machine entries have the following format:
#
# hostname      hwtype  hwaddr              ipaddr          bootfile
client1         1       00:1f:16:08:61:96   192.168.1.105   pxelinux.0
```

2. To start the server, run the following command:

```
$ sudo /bin/launchctl load -w /System/Library/LaunchDaemons/bootps.plist
```

3. Once done, stop the server with command:

```
$ sudo /bin/launchctl unload -w /System/Library/LaunchDaemons/bootps.plist
```

## Booting Ubuntu

To boot from TFTP, you need to configure your PC to boot from the network interface in the BIOS.

Once booting into the installer, you can install Ubuntu by having sources download for a mirror.

[netboot-ubuntu]: http://cdimage.ubuntu.com/netboot/
[tftp-server]: http://ww2.unime.it/flr/tftpserver/
[pxe]: http://en.wikipedia.org/wiki/Preboot_Execution_Environment "Preboot Execution Environment on Wikipedia"
[0]: http://www.jacquesf.com/2011/04/mac-os-x-dhcp-server/ "Mac OSX DHCP Server"