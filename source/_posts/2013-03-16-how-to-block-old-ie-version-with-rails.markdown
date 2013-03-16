---
layout: post
title: "How to block old IE version with Rails"
date: 2013-03-16 17:56
comments: true
tags: rails
author: "Trung Lê"
---

{{ post.title }}

There are many ways to detect browser agent, it could be front-end side with Javascript or backend.
In this short tutorial, I'll walk you through on how to detect browser version with Ruby On Rails

This applies for Rails > 2.x

<!--more-->

## How does thing work?

When you surf the site, your browser name and versions are stored in `HTTP_USER_AGENT` in the request. We need to process this string to work out the browser version and name to decide to greet viewers with warning text or not.

## Implementation

We will use `ActionDispatch`'s `request.user_agent` to grab the `HTTP_USER_AGENT` string. An example of user agent string:

```
Mozilla/4.0 (compatible; MSIE 5.00; Windows 98)
```

With a bit of text processing, we could work out the browser version, OS platform. Sound simple, isn't it? A bit of regex magic and you could write your own detect method. However, we will not do reinvent the wheel in this tutorial, instead we will use the `useragent` gem for the sake of convenience.

Install `useragent` gem by appending to `Gemfile`:

```
gem 'useragent'
```

then `bundle install`

Now we will inject a filter in `ApplicationController` to detect user agent. Below is the source code:

```ruby
class ApplicationController < ActionController::Base

  before_filter :check_browser

  private

    Browser = Struct.new(:browser, :version)

    SupportedBrowsers = [
      Browser.new('Safari', '6.0.2'),
      Browser.new('Firefox', '19.0.2'),
      Browser.new('Internet Explorer', '9.0'),
      Browser.new('Chrome', '25.0.1364.160')
    ]

    def check_browser
      user_agent = UserAgent.parse(request.user_agent)
      unless SupportedBrowsers.detect { |browser| user_agent >= browser }      
        render text: 'Your browser is not supported!'
      end
    end
end
```

Let's digest what's happening the code above.

`Browsers` is a `Struct` object with two attributes `:browser` and `:version`. This models after the way `UserAgent` create browser object. Pay attention to `SupportedBrowsers` closely, this array defines a stack of supported browsers.

`check_browser` will get called before any action, this method compare your current user agent with `SupportedBrowsers`. If the condition is not met, we render a simple text to warn the user. You could extend it to use HTML template if you like.

Please be noted that I assume all your controllers are subclass of `ApplicationController`. Please adapt the source code accordingly should your controllers extends different parent controller class.

