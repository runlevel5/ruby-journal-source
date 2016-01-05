---
layout: post
title: "How to DRY your Rails routes"
date: 2014-04-18 09:29
comments: true
tags: rails routing
author: "Trung Lê"
---

{{ post.title }}

So you have duplicated routes in your resources? In today tutorial, I'll show you how to DRY it up abit.

<!--more-->

## Sample code

Just imagine we have following routes:

```ruby
# config/routes.rb

resources :pamphlets do
  member do
    post :print
  end
end

resources :posters do
  member do
    post :print
  end

  collection do
    post :bulk_print
  end
end
```

As we could see in the code above, we have two resources `:pamphlets` and `:posters` that both share same
`post :print` route.

## DRY with Rails 3

In order to DRY it, we could extract the block `member { post :print }` to a shared proc `printable`:

```ruby
printable = Proc.new do
  member do
    post :print
  end
end

resources :pamphlets, &printable

resources :posters do
  printable.call

  collection do
    post :bulk_print
  end
end
```

`resources` method take in a block, so for `pamphlets` we parse the whole `printable` in as block. It is
a bit different with `posters` because this resource already have a block, here what we do is we call
the proc `printable` within the block by using `call`.

### DRY with Rails 4

With Rails 4, it is much easier by using routing concern. Here's how:

```ruby
concern :printable do
  member do
    post :print
  end
end

resources :pamphlets, concerns: :printable

resources :posters, concerns: :printable do
  collection do
    post :bulk_print
  end
end
```

We create a routing concern by using method `concern` and specify which concern we want to use for each resource
via argument `concerns`.

### Summary

Rails routing is very powerful and there are many magic that I want to talk to you in near future. For now, you learn
how to extract shared routes into concern to DRY your routes. Go on, share it with everyone and keep on learning!