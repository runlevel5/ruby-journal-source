---
layout: post
title: "Exposing ActiveRecord model attributes to Liquid dynamically"
date: 2012-03-12 12:52
comments: true
categories: rails
tags: liquid, template, activerecord, rails, module, ruby
author: "Trung Lê"
---

# {{ post.title }} #

[Liquid](http://liquidmarkup.org/) is a powerful templating tool especially when
used with rails. It is quite common that you have to expose ActiveRecord attributes
to liquid. You can achieve that by implement `to_liquid` method in your ActiveRecord
model so it acts as if it were `Liquid::Drop`, OR you can use the helper `liquid_methods`
to tell which attributes / call-able methods of the instance that could be passed
with the `liquid_methods` call. In most of cases, people tend to use the latter method
because they could narrow the exposure scope to liquid.

However, what if your model has so many attributes and typing all them out for
the `liquid_methods` seems arduous, you can dynamically mapping attributes by
creating a module:

{% codeblock lib/attributes_to_liquid_methods_mapper.rb lang:ruby %}
module AttributesToLiquidMethodsMapper
  def self.included(base)
    base.class_eval do
      base.attribute_names.each do |attribute|
        liquid_methods attribute.to_sym
      end
    end
  end
end
{% endcodeblock %}

And you also need to include the above module in the ActiveRecord classes that
you want to be exposed to Liquid:

```ruby
class MyObject < ActiveRecord::Base
  require 'attributes_to_liquid_methods_mapper'
  include AttributesToLiquidMethodsMapper
end
```

Hope you enjoy this tutorial. Comments and feedbacks are greatly welcomed :)
