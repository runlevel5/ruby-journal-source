---
layout: post
title: "How to uninstall all Ruby gems"
date: 2013-10-11 12:01
comments: true
tags: ruby rubygems gem
author: "Trung Lê"
---

{{ post.title }}

In order to uninstall all gems, you have to loop through all entries in `gem list` with bash scripting. This method is very inconveninent. Thanks to Rubygems 2.1.0, you now could do it with one command.

<!--more-->

Firstly, please make sure you upgrade your Rubygems to 2.1.0 or newer:

```ruby
gem update --system
gem --version
# 2.1.8
```

and in order to uninstall all gems:

```
gem uninstall --all
```

That's it folks!
