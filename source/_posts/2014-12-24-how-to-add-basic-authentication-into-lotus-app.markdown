---
layout: post
title: "How to add basic authentication into Lotus app"
date: 2014-12-24 08:28
comments: true
categories: lotusrb
tags: lotusrb, auth
author: "Trung Lê"
---

{{ post.title }}

Today Lotus Framework officially release its 0.2.0 milestone with many new features.
One is the standardised container architecture app generator. What does that mean?
That means you could seriously make app with Lotus now.

I know that one of the frequently asked question is how to add basic HTTP authentication
for an app. In this tutorial, I'll show you how.

<!--more-->

## Prequisites

* Lotus 0.2.0
* Understanding of Rack is a plus

## Concept

Lotus Framework is based on Rack, so is Rails. Rack is a web protocol spec and also a
gem which provides a minimal and unified interface between webservers. Rack introduces
the concept of middleware stacking, in which many middlewares (each is a Rack app of itself)
are bound together to create a full-stack framework. There are middlewares for anything
you want, from request santization to caching. And suprise Rack also provides middleware
for basic authentication.

Introducing [Rack::Auth::Basic](https://github.com/rack/rack/blob/master/lib/rack/auth/basic.rb).

Before we move on to the implementation, please take caution if you want to implement this
authentication. It is not very secure for production environment because the password is
hardcoded and it does not support multiple-users access authentication.

## Setup

Create a new lotus container:

```
bundle exec lotus new demo
```

That's it, you are ready to rock!

## Configuration

By default, Lotus would create for you 1 app, that is `Web` which resides in `apps/web`. To put
authentication on our `Web` application, we add to `apps/web/application.rb`:


```ruby
module Web
  class Application < Lotus::Application
    configure do

      #..whatever there already

      middleware.use Rack::Auth::Basic, "Protected Area" do |username, password|
        username == 'admin' && password == 'password'
      end

    end
  end
end

```

As you can see in the example above, we tell Lotus to add `Rack::Auth::Basic` to existing Lotus middlewares.
Lotus provides `middleware.use` DSL which push the middleware to the bottom of the stack. This DSL
is the same to Rails's `config.middleware.use`.

Please be mindful that this Rack middleware is very limited because username and password are stored
in clear text, this could pose great security risk.

To test it, we start server: `bundle exec lotus server`

Browse to `http://0.0.0.0:2300`, if things are working correctly, you should be prompted with a login dialog.

If you want to put protection only on development environment, you can move the whole code to:

```ruby
module Web
  class Application < Lotus::Application
    configure do

    #..whatever there already

      configure :development do
        #..whatever there

        middleware.use Rack::Auth::Basic, "Protected Area" do |username, password|
          username == 'admin' && password == 'password'
        end
      end
    end
  end
end
```

Now you might be wondering why there is no production environment? It is because the default environment
is production.

I don't really like the way we hardcode our password, we could refactor to store the password in ENV variables.
Thanksfully, Lotus comes with dotenv bundled, and ENV variables can be configured in
`config/.env`, `config/.env.development` and `config/.env.test`. We can change our code to:

```ruby
middleware.use Rack::Auth::Basic, "Protected Area" do |username, password|
  username == ENV["AUTH_USER"] && password == ENV["AUTH_PASSWORD"]
end
```

and set `AUTH_USER` and `AUTH_PASSWORD` in `config/.env` or `config/.env.development`:

```
AUTH_USER="admin"
AUTH_PASSWORD="password"
```

## Conclusion

Lotus is based on Rack and can leverage all mighty power of Rack. By inserting `Rack::Auth::Basic` into
Lotus middleware stack, we could easily implement a basic HTTP authentication.

If you have any questions, please leave it the comment.