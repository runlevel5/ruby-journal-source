---
layout: post
title: "How to open/override/monkey-patch a class in Ruby?"
date: 2014-10-09 11:18
comments: true
categories:
tags: ruby
author: "Trung Lê"
---

{{ post.title }}

So you search for how to moneykey-patch a class in Ruby? Read on, I'll show you how.

## Note

Ruby is very powerful and flexible. Before I show you how to override a class, I want to
ensure that you understand that monkey-patching is not considered a good practice. It might
be the source of hard to trace bugs.

## Open existing class by re-defining class

Imagine we have following class:

```ruby
class Greeter
  def hello
    "hello world"
  end
end
```

You can open your existing class easily by re-declaring the class and its methods:

```ruby
class Greeter
  def hello
    "hello Uncle!"
  end
end
```

The `Greeter#hello` would be overriden with new message _if_ class `Greeter` does exist.

If class `Greeter` has not been defined, then our above code would define a new Greeter class. And
that could lead to side effect if we are not careful.

## Open existing class with class_eval

Alternatively, Ruby provide `class_eval` method which would check for the existence of the to-be overriden
class. If the class does exist, the block of code that is parsed to `class_eval` would be evaluated and
replace the existing methods. So our example above will be:

```ruby
Greeter.class_eval do
  def hello
    "hello WORLD!"
  end
end
```

If class does not exist, an NameError exeception will be raised:

```ruby
NonExistingGreeter.class_eval do
  def hello
    "hello WORLD!"
  end
end
#=> NameError: uninitialized constant NonExistingGreeter
```

## Conclusion

Be aware of your action before you override a class and I highly recommend you to use `class_eval` because
it does gives you extra class existence check.
