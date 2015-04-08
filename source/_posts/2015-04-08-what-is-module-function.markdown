---
layout: post
title: "How to consolidate module functions?"
date: 2015-04-08 09:22
comments: true
categories: ruby
tags: ruby
author: "Trung Lê"
---

{{ post.title }}

If you have touched module before, you would know that function can be called
with module as receiver or as instance method when mixed in class. Today I am
going to show you a nitfy trick to consolidate your module function so it can
be used for both purposes at same time by using `Module#module_function`.

<!--more-->

## Concept

### Instance functions

Module functions can be mixed into classes and thus are invoked as instance
methods for the object of that classes. For example:

```ruby
module MyLibrary
  def hello
    puts "Hello world"
  end
end

class Greeter
  include MyLibrary
end

Greeter.new.hello # => Hello world
```

### Module functions

Module functions can be invoked with module as receiver. For example:

```ruby
module MyLibrary
  def self.hello
    puts "Hello world"
  end
end

MyLibrary.hello # => Hello world
```

### I want both, how?

Well, you can, here is my ugly version:

```ruby
module MyLibrary
  def self.hello
    puts "Hello world"
  end

  def hello
    MyLibrary.hello
  end
end
```

As you can see from the code above, we reference `hello` method to module
function `MyLibrary.hello`.

It looks duplicating, isn't it? Thanks to Ruby, there is an alternative:

```ruby
module MyLibrary
  def hello
    puts "Hello world"
  end

  module_function :hello
end

class Greeter
  include MyLibrary
end

Greeter.new.hello # => Hello world
MyLibrary.hello # => Hello world
```

Ruby gives us `Module#module_function` which takes the method name and automatically
generate the module function that is equivalent to `MyLibrary.hello` for us.
