---
layout: post
title: "The difference between ActiveRecord scope DSL and class method"
date: 2014-10-27 12:14
comments: true
categories: rails
author: "Trung Lê"
---

{{ post.title }}

Now and then I get question from people should they use class method or the
scope DSL for ActiveRecord scope. At first I thought they are the same but
in fact there are minor differences.

<!--more-->

## How to define scope?

There are 2 ways to define ActiveRecord model scope:

* scope DSL
* class method

## Simple example

Let's see both of them in action with following example:

```ruby
# DSL
class Taxi < ActiveRecord::Base
  scope :by_driver, ->(driver) { where(driver_id: driver.id) }
end


# class method
class Taxi < ActiveRecord::Base
  def self.by_driver(driver)
    where(driver_id: driver.id)
  end
end
```

Both ways returns us ActiveRelation scope which seems the same. In fact
in this case, they are alike with one minor difference, that is the DSL
version doc comment won't be picked up by RDoc.

## Complex example

Please be careful if you have a complex scope like below:


```ruby
# DSL
class Driver < ActiveRecord::Base
  scope :by_shift, ->(shift) do
    return if shift == 'night'

    where(shift: shift)
  end
end


# A wrong or naive version using class method
class Driver < ActiveRecord::Base
  def self.by_shift(shift)
    return if shift == 'night'

    where(shift: shift)
  end
end
```

As you can see above, the scope involves condition check and in the DSL
would `return` if it is night shift. You might be wondering what object
is returned implicitly. For the DSL version, the chain scope is returend,
that is the ActiveRelation object.

I naively copy 1:1 the same implementation of DSL to my class method implementation
and I got `nil` as reuturn value instead. This breaks the chain (if we use our scope in combination
with other scopes). The class method must be rewritten as:


```ruby
class Driver < ActiveRecord::Base
  def self.by_shift(shift)
    return scoped if shift == 'night'

    where(shift: shift)
  end
end
```

Please be noted that I returned a `scoped` which is an anonymous scope. This object
is a ActiveRelation object and would keep the chain intact.

## Conclusion

What would you use? I myself prefer class method for I find sugar syntax of Rails (which
is nice) does not add in much value. Yet having said so, I do use DSL from time to time
if the scope is tiny.