---
layout: post
title: "How to write custom serializer for ActiveRecord#serialize"
date: 2014-04-21 07:49
comments: true
tags: rails routing
author: "Trung Lê"
---

{{ post.title }}

Rails comes with a powerful and convenient `serialize` method that would do the serialization/deserializtion
for a specify column of an ActiveRecord model. In today tutorial, I'll walk you through on how to write a
custom serializer that would encrypt/decrypt your serialized value for extra security.

<!--more-->

## The Basic

ActiveRecord model has access to class method `serialize` which by default would serialize/deserialize the
specified column value with YAML.

```ruby
class Product < ActiveRecord::Base
  serialize :properties
end
```

As you can see in the example above, our `Product` class has column `properties` which we will store an
array of properties hash:

```ruby
o = Product.new(name: 'The Uber Kubik')
o.properties = [length: '10cm', width: '20cm']
o.save!
```

and you might be wondering what sort of magic has happened. Let's check the output log:

```text
 (0.1ms)  begin transaction
SQL (0.4ms)  INSERT INTO "products" ("name", "created_at", "properties", "updated_at") VALUES (?, ?, ?)  [["name", "The Uber Kubik"], ["created_at", "2014-04-20 21:59:47.486719"], ["properties", "---\n- :length: 10cm\n  :width: 20cm\n"], ["updated_at", "2014-04-20 21:59:47.486719"]]
 (1.2ms)  commit transaction
```

clearly we learn that Rails store a YAML string `---\n- :length: 10cm\n  :width: 20cm\n` into DB. Again, Rails
also does the magic to deserialize this YAML string back to object:

```ruby
o.properties
#=> [length: '10cm', width: '20cm']
```

## Alternative serializer

Rails takes in two arguments for class method `serialize`, the first is the column name and the second is the _coder
classname_. The coder class is the name of the serializer class. Rails comes with two serialzers, YAML and JSON.

```ruby
class Product < ActiveRecord::Base
  serialize :properties, JSON
end
```

By specifying `JSON` as the second argument for `serialize` we tell Rails to use serialize/deseralize the column
value with JSON.

Again, you might be wondering how YAML and JSON serializers work? Well, the basically they are class that have two
class methods, that is `load` for serialization and `dump` for deserialization.

Let's write a JSON serializer that encrypt/decrypt the value with RSA key.

We create a very naive and primitive cipher that simply un/reverse the string:

```ruby
# lib/protected_json_serializer.rb
class ProtectedJsonSerializer
  def self.load(value)
    JSON.load(decrypt(value))
  end

  def self.dump(value)
    encrypt(JSON.dump(value))
  end

  private

  def self.encrypt(value)
    value.reverse
  end

  def self.decrypt(value)
    value.reverse
  end
end
```

and set the coder to `ProtectedJsonSerializer`:

```ruby
require 'protected_json_serializer'

class Product < ActiveRecord::Base
  serialize :properties, ProtectedJsonSerializer
end
```

and now let's see the magic:

```ruby
o = Product.new(name: 'The Uber Kubik')
o.properties = [length: '10cm', width: '20cm']
o.save!
```

would yield log output:

```text
 (0.1ms)  begin transaction
SQL (0.4ms)  INSERT INTO "products" ("name", "created_at", "properties", "updated_at") VALUES (?, ?, ?)  [["name", "The Uber Kubik"], ["created_at", "2014-04-20 21:59:47.486719"], ["properties", "]}\"mc02\":\"htdiw\",\"mc01\":\"htgnel\"{["], ["updated_at", "2014-04-20 21:59:47.486719"]]
 (1.2ms)  commit transaction
```

the output clearly shows that our `properties` value is encryted `["properties", "]}\"mc02\":\"htdiw\",\"mc01\":\"htgnel\"{["]`

so now we can rest assured that even RSA has access to our DB, they could not read our data (well, you need
to do a better job with the encryption than my naive implementation).

### Summary

For most of common cases, you probably don't have to worry about custom serializer. But the power is there
if you ever need it. For example, having custom data type column like Array or Set which comes very handy.

Please remember that `serialzie` takes the serializer class name as the second argument
and the serializer must have 2 methods `load` and `dump`.

That's all folks. Keep on learning!