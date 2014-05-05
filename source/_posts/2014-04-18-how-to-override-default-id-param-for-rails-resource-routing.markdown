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
the `Order#to_param`:

```ruby
# app/models/order.rb
def to_param
  number.parameterize
end

# app/controllers/orders_controller.rb

@order = Order.find(params[:id])
```

Now you might question, what is an `id`? Is it a reference to `id` column or a metaphor
to an entity that acts as record ID? I don't know, as far as I can tell, Rails does
not want you to use other primary key.

IMHO, I would leave `params[:id]` as is and would use `params[:number]`.

If you are on Rails 4, we can modify the resource ID param by specifying `param`:

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

```ruby
# app/controllers/orders_controller.rb
@order = Order.find(params[:number])
```

Please be noted that, we still need to override the `Order#to_param` to ensure that
it comply to URI.

That's all for today!