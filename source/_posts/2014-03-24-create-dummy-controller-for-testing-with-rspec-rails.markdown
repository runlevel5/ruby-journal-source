---
layout: post
title: "Create dummy controller for testing with rspec-rails"
date: 2014-03-24 09:21
comments: true
tags: rspec, rails, testing, controller
author: "Trung Lê"
---

{{ post.title }}

Today I learned a nitfy trick from Jon Rowe on how to to create anonymous controller
with rspec-rails for controller spec. This would aid testing action callbacks testing
in controller.

<!--more-->

Say you are testing `before_action :authenticate!` in your ApplicationController.

Here is my `ApplicationController`:

```ruby
class ApplicationController < ActionController::Base

  before_action :authenticate!

end
```

Now, I want to test this callback. There are many ways to tackle this, I could create
a dummy controller like below:

```ruby
class DummyController < ApplicationController
  def dummy
    render text: 'Hello World'
  end
end

# adding dummy routing dynamically - prepare for the hacky hacky!
begin
  _routes = Rails.application.routes
  _routes.disable_clear_and_finalize = true
  _routes.clear!
  Rails.application.routes_reloader.paths.each { |path| load(path) }
  _routes.draw do
    get '/dummy' => 'dummy#dummy'
  end
  ActiveSupport.on_load(:action_controller) { _routes.finalize! }
ensure
  _routes.disable_clear_and_finalize = false
end


describe ApplicationController do
  # your test
end
```

Oh gosh, that's surely ugly! We could refactor all above with:

```ruby
describe ApplicationController do
  controller do
    def dummy
      render text: 'Hello world'
    end
  end

  describe 'authenticate! action callback' do
    it 'does redirect if user not sign in' do
      # ...
      get :dummy

      # ...
    end
  end
```

Thanks to rspec-rails, we have the `controller` helper which creates
an anonymous controller (that inherits from ApplicationController).
It saves you many hacky lines and plus you don't need to care about
routing.

See? You know know a powerful trick! So go on, try it! Keep on learning!

