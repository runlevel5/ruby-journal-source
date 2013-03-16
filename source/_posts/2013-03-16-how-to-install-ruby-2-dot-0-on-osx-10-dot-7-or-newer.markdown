---
layout: post
title: "How to install Ruby 2.0 on OSX 10.7 or newer"
date: 2013-03-16 17:58
comments: true
tags: ruby
---

{{ post.title }}

This short tutorial, I will show you how to install Ruby 2.0.0 on OSX 10.7+ or newer

<!--more-->


## RVM:

```
rvm get stable && rvm reload
```

Firstly, the OpenSSL comes with your OSX 10.7+ is outdated, you need to get latest version

```
rvm pkg install openssl
```

Secondly, the readline library on OSX 10.7+ suffers UTF-8 issue in which method in UTF-8 is converted to characters under `irb`, the 
fix is to use latest version

```
rvm pkg install readline
```

And you can install Ruby 2.0.0-p0 with:

```
rvm reinstall 2.0.0 --with-readline-dir=$rvm_path/usr --with-openssl-dir=$rvm_path/usr
```

Please note that `clang-425.0.24` that comes with Xcode successfully compile the source, should you bump into any compilation issue, you could try compiling it with `gcc-422`

## rbenv

Make sure your `ruby-build` is up-to-date:

```
cd ~/.rbenv/plugins/ruby-build && git pull
```

rbenv does not fetch fixed readline lib, thus you are required to install them manually with `brew`:

```
brew update
brew install readline openssl
env CONFIGURE_OPTS=--with-readline-dir=`brew --prefix readline` rbenv install 2.0.0-p0
```