---
layout: post
title: "How to override :id param for Rails resource routing"
date: 2014-04-18 10:28
comments: true
tags: ruby
author: "Trung Lê"
---

{{ post.title }}

By default, every resource routing is generated with `:id` as the identifier param.
What if you want to change this `:id` to something? Read on and I'll show you how.

<!--more-->

When I was working with Spree, there is a requirement to find Order by its `number`
column. The way Spree solved this problem (this is pre-Rails 4 era), is to override
the `Order#to_param` like this:

```ruby
# app/models/order.rb
def to_param
  number.to_s.to_url.upcase
end
```

Yet I find the solution more problematic as we now have a unorthodox `to_param` which
does not refer to our default integer `:id` column. Furthermore, our `OrdersController`
still find object with `params[:id]`, which might be confusing for users.

There is a better solution if you are on Rails 4, that is modifying the routing.

We have a routing like below:

```ruby
# config/routes.rb
resources :orders
```

which would generates:

```text
bundle exec rake routes
            Prefix Verb  URI Pattern                   Controller#Action
           orders GET    /orders(.:format)             orders#index
                  POST   /orders(.:format)             orders#create
        new_order GET    /orders/new(.:format)         orders#new
       edit_order GET    /orders/:id/edit(.:format)    orders#edit
            order GET    /orders/:id(.:format)         orders#show
                  PATCH  /orders/:id(.:format)         orders#update
                  PUT    /orders/:id(.:format)         orders#update
                  DELETE /orders/:id(.:format)         orders#destroy
```

Now if the business logic is to use `:number` instead of `:id`, we could override the
default routing by using `param` argument:

```ruby
# config/routes.rb
resources :orders, param: :number
```

and we double-check with:

```text
bundle exec rake routes
            Prefix Verb  URI Pattern                       Controller#Action
           orders GET    /orders(.:format)                 orders#index
                  POST   /orders(.:format)                 orders#create
        new_order GET    /orders/new(.:format)             orders#new
       edit_order GET    /orders/:number/edit(.:format)    orders#edit
            order GET    /orders/:number(.:format)         orders#show
                  PATCH  /orders/:number(.:format)         orders#update
                  PUT    /orders/:number(.:format)         orders#update
                  DELETE /orders/:number(.:format)         orders#destroy
```

Please be noted that you need to modify the `OrdersController` to find Order record
with `params[:number]` instead of `params[:id]`.

For most of the cases, the Rails convention of using `:id` as primary key does the job
well. However should you need to use something different, please do not hesistate to
use custom column for the job. And IMHO, please refrain from overriding the `to_param`.

That's it folks, keep on learning!
