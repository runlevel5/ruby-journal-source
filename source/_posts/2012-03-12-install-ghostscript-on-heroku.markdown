---
layout: post
title: "Install Ghostscript on Heroku"
date: 2012-03-12 16:57
comments: true
categories: heroku
tags: install, heroku, gs, ghostscript
author: "Trung Lê"
---

# {{ post.title }} #

Today I found out that Heroku does ship an old version of ghotscript:

```
$ heroku run bash
$ /usr/bin/gs --version
8.71
```

However, my app requires version 9.05, so I decided to compile version 9.05 on
Heroku. Here's how:

```
$ heroku run bash
$ curl -O http://downloads.ghostscript.com/public/ghostscript-9.05.tar.gz
$ tar xzvf ghostscript-9.05.tar.gz
$ cd ghostscript-9.05
$ ./configure --disable-cups --disable-gtk --with-drivers=FILES
$ make
$ cp bin/gs ~/bin
```

Let me break it down those above steps, I fetch the source and compile the source
without printers devices by specifying `--with-drivers=FILES`. Once the compilation
is completed, I copy the binary `ghostscript-9.05/bin/gs` to `~/bin`.

All binaries in `~/bin` will be available for your Heroku app now. You can verify
if the binary works by:

```
$ gs --version
9.05
```

And please do not forget to clean up:

```
$ cd ~
$ rm -rf ghostscript-9.05*
```

If you feel lazy, you could download my Ruby-wrapper of gs at [https://github.com/joneslee85/ruby-ghostscript](https://github.com/joneslee85/ruby-ghostscript).
