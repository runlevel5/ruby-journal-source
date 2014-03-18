---
layout: post
title: "How to switch between selenium and poltergeist in capybara for Rails integration RSpec test"
date: 2014-03-14 09:20
comments: true
tags: ruby
author: "Trung Lê"
---

{{ post.title }}

Debugging integration is still a pain with headless driver like poltergeist. In this tutorial
I'll show you a good tip how to switch to selenium for debugging with Firefox.

<!--more-->

Testing with selenium-webdriver is always slower than headless poltergeist, abeit easier to debug.
Debugging with selenium brings up Firefox browser window with CSS and JS correctly loaded whilst
it's not the same case with poltergeist.

I am going to show you to have both of drivers in same Rails app and you could switch to selenium
for debugging easily!

First thing is to add into your `Gemfile`:

```ruby
group :test do
  gem 'selenium-webdriver', require: false
  gem 'poltergeist', require: false
  gem 'capybara'
end
```

I want to point out that we explicitly set `reauire: false` so that capybara won't load any driver.
This help avoid conflicts when two drivers are loaded at same time.

Please make sure you run 'bundle install'  before moving to the next step.

Now I assume that you are using default RSpec's helper `spec/spec_helper` for your integration test. Please ensure
that this line is in your file:

```ruby
require 'capybara/rspec'

Dir[Rails.root.join("spec/support/**/*.rb")].each { |f| require f }
```

The above line tells RSpec to pick up any extensions/helpers that we place under `spec/support` folder.

Next we create new `spec/support/feature_spec_extensions.rb` with content:

```ruby
module FeatureSpecExtensions
  def hang
    print "Waiting.... Press return if you wanna continue"

    if ENV['FF'] == 'true'
      STDIN.getc
    else
      page.driver.debug
    end
  end
end

RSpec.configure do |config|
  config.include FeatureSpecExtensions, type: :feature

  config.before :suite, type: :feature do
    if ENV['FF'] == 'true' # in case you wanna run it with selenium
      require 'selenium-webdriver'
    else
      require 'capybara/poltergeist'
      Capybara.register_driver :poltergeist do |app|
        Capybara::Poltergeist::Driver.new(app, {
          js_errors: true,
          inspector: true,
          phantomjs_options: ['--load-images=no', '--ignore-ssl-errors=yes'],
          timeout: 120
        })
      end
    end
  end

  config.before :each, type: :feature do
    Capybara.current_driver = ENV['FF'] == 'true' ? Capybara.javascript_driver : :poltergeist
  end
end
```

The above RSpec helper sets default driver to poltergeist and should ENV variable FF is set, it will
switch to selenium-webdriver.

So to run a feature spec in poltergeist, we do:

```
bundle exec rspec spec/features/the_uber_spec.rb
```

and to test with selenium-webdriver, we do:

```
FF=true bundle exec rspec spec/features/the_uber_spec.rb
```

I also provide a helper method `hang` which will hang the page. Simply place it to the line
that you want and Firefox browser will open up if selenium is used. Eg:

```
scenario 'some uber case' do
  click_on 'Sign In'

  hang

  ...
end
```

That's it for now folks. Credits to Nikolai Nemshilov, my ex-colleague who came up with this brilliant hybrid idea.