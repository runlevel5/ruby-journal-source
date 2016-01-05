---
layout: post
title: "Avoid unless syntax in a chain of conditions statement"
date: 2013-11-23 23:11
comments: true
categories:
tags: ruby
author: "Trung Lê"
---

{{ post.title }}

Ruby gives us a nice human friendly `unless` which is equivalent to negation of `if`. Yet if we abuse using this method in a long complex statements, it could add more confusion for normal readers. In this article, I'll give you one example to prove that you should not use `unless` in a chain of conditions statement.

<!--more-->

I have been using `unless` for a long time and to my surprise using this method in a chain of conditions is confusing for human brain as many of my colleagues misinterpret the code all the time. Here is one example:

```ruby
unless @employee.on_annual_leave? || @employee.on_parental_leave?
  puts "You have no leave yet!"
end
```

Majority of normal human would read it as "if employee is not on annual leave or employee is on parental leave". Which is completely wrong, this ambiguous code is supposed to read "if employee is _not_ annual leave and employee is _not_ on parental leave". And for a complex piece of code, misreading this condition could yield detrimental issues. So I'd suggest not to use `unless` and instead to use `if !condition`. Let's refactor our code:

```ruby
if !@employee.on_annual_leave? || !@employee.on_parental_leave?
  puts "You have no leave yet!"
end
```

This code would always read as "if employee is not on annual leave or employee is not on parental leave", which is precise to the context. So be wise before using `unless`, in cases negation with `if` is much more concise.

I hope you enjoy this short article, and please leave comments if you like it.