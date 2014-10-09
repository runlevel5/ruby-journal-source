---
layout: post
title: "Quick guide to ActiveSupport::HashWithIndifferentAccess"
date: 2014-10-09 10:49
comments: true
categories:
tags: ruby, rails
author: "Trung Lê"
---

{{ post.title }}

Hash is beautiful. It is one of many things why I love Ruby. As you might have known,
Hash is identified by a key, this key could be either a String or a Symbol. For most
of cases, people tend to go for symbol because it would take up less memory (though it
might come with a side effect that is Symbol is not GC-colletable (Ruby 2.2.0 does clean
it up though)).

There are access usecases that requires our hash key to be interchangable between String and Symbol
key. For example, web application request parameter processing.

We could typecast the key to either String or Symbol but it would soon emerge an
annoying pattern. Instead, with the help of ActiveSupport, you can create a hash with
no differences if accessing using String or Symbol key. Introducing ActiveSupport::HashWithIndifferentAccess.

<!--more-->

## Basic Hash

A String hash is like below:

```ruby
my_string_hash = { 'special_key' => 'the string value' }
```

Or with Symbol:

```ruby
my_symbol_hash = { :special_key => 'the symbol value' }
```

Please be noted that the key might looks similar but they are different to each other in concept.


```ruby
my_mix_hash = { 'special_key' => 'the string value', :special_key => 'the symbol value' }

my_mix_hash['special_key']
#=> 'the string value'

my_mix_hash[:special_key]
#=> 'the symbol value'
```

## The Indifferent Access Hash

If we want to consolidate two type keys into one, ActviveSupport provides a convenient way to do it:


Ensure you install `actives_support` gem first:

```ruby
gem install activesupport
```

Fire up your IRB:

```ruby
require 'active_support/core_ext/hash/indifferent_access'
my_mix_hash = HashWithIndifferentAccess.new

my_mix_hash['special_key'] = 'value'
my_mix_hash['special_key']
# => value
my_mix_hash[:special_key]
# => value

my_mix_hash[:special_key] = 'value'
my_mix_hash['special_key']
# => value
my_mix_hash[:special_key]
# => value
```

As you can see above, our `my_mix_hash` now both points to same value regardless the type of the key.

## Application in Rails

You can find traces of HashWithIndifferentAccess in Rails controller, the magical `params` method.