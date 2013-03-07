---
layout: post
title: "Define Fixtures with Polymorphic Association"
date: 2012-03-09 15:16
comments: true
categories: rails
tags: fixtures, testing, rails, polymorphic
author: "Trung Lê"
---

# {{ post.title }} #

It is more than often that you have to write a fixtures for models with polymorphic
association. In this short tutorial, I will show you how to use few shortcut trick
to create fixtures.

Assume we have following code:

```ruby
class Car < ActiveRecord::Base
  belongs_to :borrowable, :polymorphic => true
end

class Company < ActiveRecord::Base
  has_many :cars, :as => :borrowable
end

```

Above is just a very simple code declaring a polymorphic association between
Company to Car. Next, we will write some fixtures for the sake
of testing.

```
# Company fixtures
google:
  id: 1
  name: Google

# car fixtures
ferrari:
  name: F430
  borrowable_id: 1
  borrowable_type: Company
```

We could do better by referencing to Company fixture via named fixture, so
the above car fixture would be:

```
ferrari:
  name: F430
  borrowable: google (Company)
```

Please be noted that the syntax of line `borrowable` now has an additonal `(Company)`
keyword. This is to tell rails that the `borrowable_type` is of `Company` class.
Without that, the fixtures will set `borrowable_type` to `nil`, thus invalid.

Hope you find this tutorial useful. And please avoid using fixures as much as you can. Use factory_girl or machinist. See you in the next tutorial.