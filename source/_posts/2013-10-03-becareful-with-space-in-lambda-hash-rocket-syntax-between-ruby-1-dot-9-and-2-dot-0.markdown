---
layout: post
title: "Becareful with space in lambda rocket syntax between Ruby 1.9 and 2.0"
date: 2013-10-03 19:45
comments: true
tags: ruby
author: "Trung Lê"
---

{{ post.title }}

Majority of Ruby 2.0 syntaxes are backward-compatible with Ruby 1.9. Yet there is one tiny change in the way that Ruby 2.0 does lambda rocket that would break backward-compatability with Ruby 1.9. That is...a single space. Yes, you heard it correctly.

<!--more-->

So, in Ruby 2.0, you could declare shorthand `lambda` with `->`. Here is example:

```ruby
hello_world = -> (message) { puts message }
```

The above code runs fine under 2.0 yet failed spectacularly with 1.9:

```
SyntaxError: (irb):1: syntax error, unexpected tLPAREN_ARG, expecting keyword_do_LAMBDA or tLAMBEG
hello_world = -> (message) { puts message }
                  ^
(irb):1: syntax error, unexpected tLAMBEG, expecting $end
hello_world = -> (message) { puts message }
                            ^
  from /Users/trung_le/.rubies/ruby-1.9.3-p448/bin/irb:12:in `<main>'
```

In order to fix this issue, we have to rid of the space between `->` and `(message)`:

```ruby
hello_world = ->(message) { puts message }
```

Weird, isn't it? At least you are aware of this discrepancy now.