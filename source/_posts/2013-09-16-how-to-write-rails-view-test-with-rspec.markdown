---
layout: post
title: "How to write Rails View test with RSpec"
date: 2013-09-16 16:18
comments: true
tags: rails capybara ruby test
author: "Trung Lê"
---

{{ post.title }}

In this short tutorial, I will show you how to do a View test with Rspec + capybara

<!--more-->

To my surprise that not many people is aware of View test provided by rspec-rails gem. It is partially
because they migrate from default Rails stack testing in which functional test performs both controllers
and views testing.

Testing View is to assert the template contain the piece of expected informations that is parsed from the controllers.


The default Rails stack ships with ActionDispatch::Assertions::SelectorAssertions which consits of method 'assert_select'
to traverse through our DOM nodes whilst default rspec does not provide any CSS or XPath selector method, the only available
method is `contain`.

So if we have our view like the example below:

```erb
# app/views/products/show.html.erb

<table id="product_<%= @product.id %>">
  <thead>
    <tr>
      <th>Name</th>
      <th>Price</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><%= @product.name %></td>
      <td><%= @product.price %></td>
    </tr>
  </tbody>
```

we could write our spec:

```ruby
# spec/views/products/show.html.erb_spec.rb

require 'spec_helper'

describe 'products/show.html.erb' do
  it 'displays product details correctly' do
    assign(:product, Product.create(name: 'Shirt', price: 50.0))

    render

    rendered.should contain('Shirt')
    rendered.should contain('50.0')
  end
end
```

the View spec is not associated with controller, thus we have to assign the @product into
`products/show.html.erb` template with method `#assign`. The `#render` method is the same
as `ActionView#render`. Lastly, the `#rendered` returns the HTML response, of which assertions
can be performed upon.

The `contain` matcher is not suffice to perform explicit check on DOM level. Therefore I
use capybara. Some might prefer webrat. Please ensure you install capybara with Gemfile
and add this line into your `spec/spec_helper.rb`:

```
# spec/spec_helper.rb

require 'capybara/rspec'
```

that inject all rspec matchers that capybara provides, in which we are interested in `#has_selector` matcher.

and the above spec can be refactored to:

```ruby
# spec/views/products/show.html.erb_spec.rb

require 'spec_helper'

describe 'products/show.html.erb' do
  it 'displays product details correctly' do
    assign(:product, Product.create(name: 'Shirt', price: 50.0))

    render

    rendered.should have_selector("table#product_#{@product.id} tbody tr:nth-of-type(1) td:nth-of-type(1)", text: 'Shirt')
    rendered.should have_selector("table#product_#{@product.id} tbody tr:nth-of-type(1) td:nth-of-type(2)", text: '50.0')
  end
end
```

The `#has_selector` accept CSS and XPath rule and comes with many userful options, it fills in the gap of ActionDispatch::Assertions::SelectorAssertions.

I hope you like this short tutorial, comments are welcome.
