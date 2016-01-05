---
layout: post
title: "How to do jQuery UI Autocomplete with capybara 2"
date: 2013-09-12 14:04
comments: true
tags: rails capybara ruby
author: "Trung Lê"
---

{{ post.title }}

In this tutorial, I'll show you how to do jQuery UI Autocomplete the better way with Capybara 2!
That is getting rid of `sleep` that many solutions on the Internet suggested. Furthermore this
solution works both with selenium and polstergeist driver.

<!--more-->

If you search on the Internet on how to do autocomplete with capybara, you'll normally find
solutions for Capybara 1.x like below:

```ruby
s.fill_in "Make", :with => "Make"
sleep 3
page.execute_script("$('.ui-menu-item a:contains(\"Make Two\")').trigger('mouseenter').click();")
```

It was a fine solution however the `sleep` call is not good. It tells the driver to passively
sleep while waiting for JS loading up. As of Capybara 2, there is no need to call `sleep` as
finder API will actively look for existence of the node and raise error if not found. Here is
a better solution:

```ruby
def fill_autocomplete(field, options = {})
  fill_in field, with: options[:with]

  page.execute_script %Q{ $('##{field}').trigger('focus') }
  page.execute_script %Q{ $('##{field}').trigger('keydown') }
  selector = %Q{ul.ui-autocomplete li.ui-menu-item a:contains("#{options[:select]}")}

  page.should have_selector('ul.ui-autocomplete li.ui-menu-item a')
  page.execute_script %Q{ $('#{selector}').trigger('mouseenter').click() }
end
```

To use it, you call:

```ruby
fill_autocomplete('field_id', with: 'term')
```

Please be noted that `field` parameter must be the CSS ID, not label nor name.

Let me digest the above code, first, I fill in the text field and I want the autocomplete
to kick in by trigger focus and keydown event on the field using JS. Then I check
the existence of `li.ui-menu-item a` instead of doing `sleep` to ensure that the JS
has popped the dropdown correctly. The final step is just to trigger mouseentter event to
do mouse hover then click.

Using `have_selector` is definitely a nicer way to check for JS initialization and I
encourage you apply this technique as often as possible. You'll thank me later for saving
you from randomly failed test. That's all for now. Good day!
