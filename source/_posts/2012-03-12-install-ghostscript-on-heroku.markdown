---
layout: post
title: "Install Ghostscript on Heroku Cedar"
date: 2012-03-12 16:57
comments: true
categories: heroku
tags: install, heroku, gs, ghostscript, cedar
author: "Trung Lê"
---

{{ post.title }}

In this tutorial, I will show you how to install ghostscript on Heroku Cedar.
As you might have known that Heroku virtual machine does come with a system-wide
ghostscript version which is located at `/usr/bin/gs`. You can find out the
location of this version:

```
$ heroku run bash
$ /usr/bin/gs --version
8.71
```

However, explicit dependencies is not recommended, you could read 12 Factor Approach
on dependencies at [http://www.12factor.net/dependencies](http://www.12factor.net/dependencies). Credit to [Ryan Daigle](https://github.com/rwdaigle) who pointed it out for me and I agree with him.

To install ghostscript, we fetch the source under heroku console, fetch the source,
configure and compile the software:

```
$ heroku run bash
$ curl -O http://downloads.ghostscript.com/public/ghostscript-9.05.tar.gz
$ tar xzvf ghostscript-9.05.tar.gz
$ cd ghostscript-9.05
$ ./configure --disable-cups --disable-gtk --with-drivers=FILES
$ make
$ cp bin/gs ~/bin
```

You might notice that I only specify configuration parameters `--with-drivers=FILES`.
It is because I don't need printer drivers for my app which only does think like
images and PDF manipulation.

Once the compilation is completed, copy the binary `ghostscript-9.05/bin/gs` to `~/bin`.
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
