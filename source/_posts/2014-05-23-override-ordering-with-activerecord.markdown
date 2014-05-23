---
layout: post
title: "Override ordering with ActiveRecord"
date: 2014-05-23 14:33
comments: true
tags: rails, activerecord
author: "Trung Lê"
---

{{ post.title }}

If your ActiveRecord models happen to have default ordering scope, you could override
this ordering scope in queries by using `reorder` method.

<!--more-->

Imagine we have a model like this:

```ruby
class User < ActiveRecord::Base
  default_scope { order(:updated_at) }
end
```

All queries would include this scope by default. If you want to override this scope,
you can apply `reorder` to the query chain:

```ruby
query_chain.reorder(:order_index)
```

That's it for today folks!