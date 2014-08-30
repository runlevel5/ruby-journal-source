---
layout: post
title: "How to track changes of a model in a `after_callbacks` in Rails 3 or newer"
date: 2013-09-23 16:43
comments: true
tags: rails activerecord
author: "Trung Lê"
---

{{ post.title }}

You can track changes to a ActiveRecord model with `ActiveRecord::Dirty#changed?`. Yet there is one caveat though, the tracking is lost after the model is saved. In this tutorial I'll show how to track the changes to a record even the record has been saved, this comes handy when being used with ActiveRecord `after` callbacks

<!--more-->

Imagine that we have a `Product` model with `name`, we make change to `name` with:

```ruby
@product.name # => "Diamond"
@product.name = 'Ruby'
@product.changed? # => true
@product.name_changed? # => true
```

By using ActiveRecord::Dirty, we could tell if the model or attribute `name` is changed.

If we save our model

```ruby
@product.save
@product.changed? # => false
@product.name_changed? # => false
```

The `false` value indicates that ActiveRecord::Dirty is not persisent after the model is saved. What if now you are being asked to add in a code to notify the inventory system if product name is changed? We can
simply workaround by using `around` callback. Here is details:

```ruby
class Product < ActiveRecord::Base
  around_update :notify_system_if_name_is_changed

  private

  def notify_system_if_name_is_changed
    name_changed = self.name_changed?

    yield

    notify_system if named_changed
  end

end
```

`around` callback is introduced in Rails 3. This callback consists of 3 parts:

* The part before `yield` is codes prior to the action is called
* The `yield` is action that gets called
* The part after `yield` is codes after the action is called

In our example above, we store the changing state of attribute `name` into `named_change` variable. The `yield` executes the `update` action. The changing state then can be referenced afterward and will send out notification if name is truly changed.

The `around` callbacks are very powerful features of ActiveRecord. I hope that you should take advantage of it more. See you in the next tutorial.

