---
layout: post
title: "Debug your failed test in Travis CI"
date: 2012-08-24 13:04
comments: true
tags: test, ruby, ci, travis
author: "Trung Lê"
---

# {{ post.title }} #

Ever wondering why some tests passed locally but failed on Travis?
Ever questioning how could I go about to debug why that failed test on remote Travis?
Read on as I show you how

<br/>
<br/>
<br/>
<!--more-->

In one occassion, I had a failed Ruby test on Travis which infact passed on my local box. So I tried
to put in `debugger` and see if Travis could let me drop into Irb or not. It turns out Travis hang
for quite a while until timed out. So debug remotely doesn't work.

Right, plan B then. I spent some time dig into Travis documentation and learned that Travis server
provisioned new VM image for every test. So if I could get my hand on the VMs, I could replicate the
same environment that Travis run the test locally. Sound rational, doesn't it? Unfortunately, I could
not find any traces nor URL where to download those VMs. Thanks to Josh at Travis, he sent me the URL
to VMs.


## Download VM

Travis VM are packaged Vagrant box.

You coud download VM here at:

```
files.travis-ci.org/boxes/provisioned/travis-(box_name).box
```

in which `(box_name)` is the language you use, `ruby`, `php`, etc.

For my case, I was testing a Rails app, so I downloaded the ruby box:

```
wget files.travis-ci.org/boxes/provisioned/travis-ruby.box
```

Please note that those files are large in size ~> 3GB


## Install Vagrant

Head to http://vagrantup.com/ and download the package for your OS.

# Install Virtualbox

Head to https://www.virtualbox.org/ and download the app for your OS.

# Import the VM box

Once you have downloaded the VM box, the image can be imported to your system. In my case:

```
$ vagrant box add travis-ruby travis-ruby.box
```

If imported successfully, you should be able to see `travis-ruby` in the box list with:

```
$ vagrant box list
```

# Bootstrap

Let's get our box up so we could SSH and start playing around with it:

```
$ vagrant init travis-ruby
```

A new `Vargrantfile` will be created for you in the current folder.
We could bring the box up with:

```
$ vagrant up
```

And now you can SSH into with:

```
$ vagrant ssh
```

Once you are in the box terminal, we run the post-install script:

```
$ sh ./postinstall.sh
```

This script install all essential packages like compilers, libraries.

# Debug your test

Copy your application to this VM box via `scp` or `git clone` and debug the test like you debug it on your local box.

