---
layout: post
title: Spree 1.1 deployment on Heroku
date: 2012-07-6 07:31
comments: true
categories: 
tags: spree, heroku, ruby, deployment, cedar stack
author: "Trung Lê"
---

# {{ post.title }} #

In this tutorial, I will show you how to create a Spree application on your local
box, configure and push it to Heroku.

<br/>
<br/>
<br/>
<!--more-->

## Prerequisites ##

All instructions are written for OSX 10.8.x system. However it
also works to UNIX and Linux systems with minor adaptations.

#### Heroku ####

If you have not yet installed Heroku, you can install it via Rubygem:

```
$ gem install heroku
```

If you have installed Heroku, please make sure you update to version 2.1.0 or higher
for Cedar support:

```
$ heroku update
$ heroku --version
heroku-gem/2.28.7 (x86_64-darwin12.0.0) ruby/1.9.3
```

#### Ruby ####

Because we are going to deploy on Heroku Cedar stack with `ruby-1.9.3-p194`,
we should use the same Ruby version on our local box for consistency.

```
$ rvm install 1.9.3-p194
```

#### Ruby On Rails ####

Spree 1.0.x leaves the choice of rails version to you. You can choose version any
minor version of `3.2.x`. It is highly recommended that you go for `3.2.6`
unless you have reasons not to.

```
$ gem install rails -v=3.2.6
```

#### Spree ####

```
$ gem install spree -v=1.1.2
```

Check installed spree gems:

```
$ gem list | grep 'spree'
spree (1.1.2)
spree_api (1.1.2)
spree_auth (1.1.2)
spree_cmd (1.1.2)
spree_core (1.1.2)
spree_dash (1.1.2)
spree_promo (1.1.2)
spree_sample (1.1.2)
```

`spree` gem consists of many components, however you only need `spree_core`
to build an online store.


#### PostgreSQL ####

Heroku only support PostgreSQL and the software can be installed with Homebrew:

```
$ brew install postgresql
```

Please make sure you read the Build Notes after the installation.

Additionally, `pg` is installed to provide DB adapter:

```
$ gem install pg
```

#### Other dependencies ####

```
$ brew install imagemagick
```

## Prepare local application ##

Create a new rails app default to postgreSQL

```
rails _3.2.6_ new fool-man-chew -d postgresql
```
Configure database setting by editing `config/database.yml`.

```
development:
  adapter: postgresql
  encoding: unicode
  database: fool-man-chew_development
  pool: 5
  username: your_username
  password: your_password

test:
  adapter: postgresql
  encoding: unicode
  database: fool-man-chew_development
  pool: 5
  username: your_username
  password: your_password

production:
  adapter: postgresql
  encoding: unicode
  database: fool-man-chew_development
  pool: 5
  username: your_username
  password: your_password
```

In fact, you could remove `production` from `config/database.yml` because Heroku
doesn't create db based on local box `config/database.yml` file though.

Don't forget to create databases with:

```
$ bundle exec rake db:create:all
```

## Bootstraping on local box ##

There are two ways to bootstrap Spree, I prefer the latter method as it gives me
more control of bootstraping process.

Both ways runs Asset Precompiling rake task which fix an issue where Heroku could
not precompile asset, you could read more about this issue at [Assets Precompiling section][assets-precompiling]


#### 1. Wizard mode ####

`spree_cmd` gem provides the convenient `spree` command that add the Spree gem, create initializers, copy migrations and optionally generate sample products and orders.


```
$ RAILS_ENV=development spree install fool-man-chew
```

You can notice that I explicitly declare `RAILS_ENV=development` here. If not,
`spree install` will assume your `RAILS_ENV=production`


The wizard will guide you through a list of questions, I opt `no` for Default Gateway
because I am not going to use skrill gateway for this tutorial.

```
Would you like to install the default gateways? (yes/no) [yes] no
Would you like to run the migrations? (yes/no) [yes] yes
Would you like to load the seed data? (yes/no) [yes] yes
Would you like to load the sample data? (yes/no) [yes] yes
Admin Email [spree@example.com] fool@man.ch
Admin Password [spree123] foo123
```

if nothing goes wrong, you would see:

```
...
     loading  seed data
     loading  sample data
      insert  config/routes.rb
**************************************************
We added the following line to your application's config/routes.rb file:
 
    mount Spree::Core::Engine, :at => '/'
**************************************************
Spree has been installed successfully. You're all ready to go!
 
Enjoy!
precompiling  assets
```

#### 2. Manual mode ####

You could manually append `spree` gem into the end of your `Gemfile`:

```
gem 'spree', '~> 1.1.2'
```

If you have not yet run `bundle install`, please run it now:

```
$ bundle install
```

Next we invoke Spree install generator to copy migrations, initializers and
generate sample data:

```
$ rails g spree:install
```

OR

Bootstraping manually with command:

```
$ bundle exec rake spree:install:migrations
$ bundle exec rake db:migrate
$ bundle exec rake db:seed
$ bundle exec rake spree_sample:load
```

Once the bootstrap is finished, we need to precompile our assets too:

```
$ bundle exec rake assets:precompile:nondigest
```

## Deploy to Heroku ##

### Configure web server ###

By default, Heroku use the Thin server. However in this tutorial, we are going to
use Puma instead, just to show you the great new process types system that
Cedar support.

Append to `Gemfile`:

```
group :production do
  gem 'puma'
end
```

and install the gem with:

```
$ bundle install
```

Then we set up Puma to use minium 4 threads. You can scale up to more Dynos should the app
need more processing power. Create a new file `config/puma.rb`:

```
threads 4, 16
```

The great about Cedar stack is that Heroku introduces a new way to scale your app,
that is [Process Model][1], now you could define a custom list of process type
that you want to run in the `Procfile` file.

We configure our unicorn which is of type `web` by creating new file in `Rails.root`
folder `Procfile`:

```
web: bundle exec puma -p $PORT -C ./config/puma.rb
```

### Heroku setup ###

#### Create Heroku app ####

We are going to create an Cedar stack based app:

```
$ heroku apps:create fool-man-chew
```

If success, you would see below output:

```
Creating fool-man-chew... done, stack is cedar
http://fool-man-chew.herokuapp.com/ | git@heroku.com:fool-man-chew.git
Git remote heroku added
```

and double check git remote you would see heroku remote listed:

```
$ git remote show
heroku
```

#### Install ruby-1.9.3 for Heroku ####

Cedar stack default to ruby-1.9.2, however Spree has been tested with
`ruby-1.9.3` so I highly recommended you to use same version for production.

*UPDATE*: After March 19, 2012, Heroku has deprecated installing Ruby 1.9.3 using
heroku-labs plugin. So it is uncesssary to setup RUBY_VERSION variable or to set up
PATH config variable to include 'bin' when you specify the default Ruby 1.9.3 with:

```
$ heroku config:add RUBY_VERSION=ruby-1.9.3-p194
$ heroku config:add PATH=bin:vendor/bundle/ruby/1.9.1/bin:/usr/local/bin:/usr/bin:/bin
```

To configure Heroku to use Ruby 1.9.3, we specify the Ruby version in the Gemfile.
Unfortunately, as of 3 July 2012, the unreleased version 1.2.0.pre.1 is the only
version that support the feature. So we will go with this unrelease version.

You could read more about [Selecting a version of Ruby] [55].

First, we install latest version of Rubygems-bunder:

```
$ gem update rubygems-bundler
```

Then install pre-release version of Bunlder 1.2.0, we have to uninstall the
current version before installing:

```
$ gem uninstall -ax bundler
$ gem install bundler --pre
```

Now we specify Ruby version in the Gemfile:

```
source 'http://rubygems.org'

ruby '1.9.3'
```

then

```
bundle install
```

*NOTE*: It is no longer possible to explicitly specify a patch level for a Ruby
version (such as ruby-1.9.3-p194), Heroku provides the most secure patch level of
whatever minor version you expect.

#### Set up Amazon S3 ####

Heroku is diskless, thus assets storage is delegated to third-party cloud storage
service like Amazon S3. 


#### Add SSL certificate ####

By default, Spree production mode enforce SSL. This step is very optional, 
please read [Disable SSL in Production] [disable_ssl] section if you want to disable SSL in Production mode. 

A Piggyback SSL is a now standard feature on all Heroku apps so you don't have
to enable. We are not going to buy a certificate for this test app. Instead, we are
going to set up a [Self-Signed SSL Certificate] [self-signed-ssl].

A private key and certificate signing request can be generated:

```
$ openssl genrsa -des3 -out site.key 2048
    ...
   Enter pass phrase for site.key:
   Verifying - Enter pass phrase for site.key:
$ mv site.key site.orig.key
$ openssl rsa -in site.orig.key -out site.key
   Enter pass phrase for site.orig.key:
   writing RSA key
$ openssl req -new -key site.key -out site.csr
   ...
   Country Name (2 letter code) [AU]:US
   State or Province Name (full name) [Some-State]:California
   ...
```

and now the self-signed SSL certificate is generated from the `site.key` private key and `site.csr` files:

```
$ openssl x509 -req -days 365 -in site.csr -signkey site.key -out final.crt
```

The `final.crt` file is your site certificate suitable for use with Heroku’s SSL add-on along with the `site.key` private key.

Now we upload those two files to Heroku:

```
$ heroku domains:add fool-man-chew.herokuapp.com
$ heroku ssl:add final.crt site.key
```

#### Bootstraping Spree on Heroku ####

Now we could push our app to Heroku:

```
git init
git add -A
git commit -m "Initial commit"
git push heroku master
```

*OPTIONAL*: If you ever bump into issues where Bundler fails to locate gems, the best workaround is to cache the bundle:

```
bundle cache
git add -A
git commit -m 'Bundle cache'
```

If all goes well, you would see following output:

```
-----> Heroku receiving push
-----> Ruby/Rails app detected
-----> Using RUBY_VERSION: ruby-1.9.3-p194
-----> Installing dependencies using Bundler version 1.1.rc.7
       Running: bundle install --without development:test --path vendor/bundle --binstubs bin/ --deployment
       Fetching gem metadata from http://rubygems.org/.......
       Fetching gem metadata from http://rubygems.org/..
       Fetching git://github.com/joneslee85/spree-heroku.git
       Using rake (0.9.2.2)
       ....
       Writing config/database.yml to read from DATABASE_URL
-----> Preparing app for Rails asset pipeline
       Detected manifest.yml, assuming assets were compiled locally
-----> Rails plugin injection
       Injecting rails_log_stdout
       Injecting rails3_serve_static_assets
-----> Discovering process types
       Procfile declares types      -> web
       Default types for Ruby/Rails -> console, rake, worker
-----> Compiled slug size is 39.4MB
-----> Launching... done, v9
       http://fool-man-chew.herokuapp.com deployed to Heroku
```

Next we could repeat the same bootstraping step on our remote heroku:

```
$ heroku run rails g spree:install
```

Now we could open app:

```
$ heroku apps:open
```




#### Custom Domain ####

Now we push a bit further by setting up custom domain for our shop, first we need
to set up Heroku to respond to requests at custom domains:

```
$ heroku addons:add custom_domains
Adding custom_domains to fool-man-chew...done.
```

And inform Heroku our beautiful `fool-man-chew.com` domain

```
$ heroku domains:add www.fool-man-chew.com
Added www.example.com as a custom domain name to fool-man-chew.heroku.com
$ heroku domains:add fool-man-chew.com
Added example.com as a custom domain name to fool-man-chew.heroku.com
```

Then I point the domain DNS to Heroku. Please read more at [Heroku Custom Domain] [2]

We also need to let Spree know of our custom domain by append `site_url` in our
`config/initializers/spree.rb`

{% codeblock config/initializers/spree.rb lang:ruby %}
Spree.config do |config|
  config.site_url = 'fool-man-chew.com'
end
{% endcodeblock %}

Add, commit and push again:

```
$ git add config/initializers/spree.rb
$ git commit -m 'Use custom domain'
$ git push heroku master
$ git heroku:restart
```



### Issues ###

#### Disable SSL in Production mode ####

Edit file `config/initializers/spree.rb`:

{% codeblock config/initializers/spree.rb lang:ruby %}
Spree.config do |config|
  config.allow_ssl_in_production = false
end
{% endcodeblock %}


Make sure you commit the changes to app repository.

#### Assets Precompiling ####

Heroku would fail precompiling assets in slug compilation. Following output show
the error:

```
       Injecting rails_log_stdout
       Injecting rails3_serve_static_assets
-----> Preparing app for Rails asset pipeline
       Running: rake assets:precompile
       rake aborted!
       could not connect to server: Connection refused
       Is the server running on host "127.0.0.1" and accepting
       TCP/IP connections on port 5432?

       Tasks: TOP => environment
       (See full trace by running task with --trace)
       Precompiling assets failed, enabling runtime asset compilation
       Injecting rails31_enable_runtime_asset_compilation
       Please see this article for troubleshooting help:
       http://devcenter.heroku.com/articles/rails31_heroku_cedar#troubleshooting
```


It make some sense though because Spree requires access to DB to complete this task and yet before you push to Heroku the environment config is not present. 

So we have to disable precompile on intialize by set `config.assets.initialize_on_precompile` to `false` in `config/application.rb`

```
config.assets.initialize_on_precompile = false
```

Then workaround this issue by locally precompile assets before deployment:

```
$ bundle exec rake assets:precompile RAILS_ENV=development
```

What will happen next is Sprocket will compile our assets and place them in `public/assets` folder. What Heroku really care is the `public/assets/manifest.yml`. This file contains all MD5 checksums of our assets and Heroku will check the existence of the file to tell if we compile our assets locally or not.

If we push this file to our server:

```
$ git add -A public/assets
$ git commit -m 'Added precompiled assets'
$ git push heroku master
```

you would see:

```
....
-----> Preparing app for Rails asset pipeline
       Detected manifest.yml, assuming assets were compiled locally
...
```

You could read more on [Rails 3.1 on Heroku] [a]

### Conclusion ###

Spree 1.0 is a big major leap to previous versions. It is faster, more robust and
much easier to install. Outstanding issue such as 'Superclass mistmach bug with Calculator::PriceBucket', 'Bootstraping migration run failed' are resolved. Yet
there are possibly issues that I am not aware of, so please file a ticket on [GitHub Issues] [3] and I'll make sure it has my utmost attention.

I'd like to extend my gratitude to the Spree community for the hardwork. 

[0]: http://michaelvanrooijen.com/articles/2011/06/01-more-concurrency-on-a-single-heroku-dyno-with-the-new-celadon-cedar-stack/ "More concurrency on a single Heroku dyno with the new Celadon Cedar stack"
[1]: http://adam.heroku.com/past/2011/5/9/applying_the_unix_process_model_to_web_apps/ "Heroku Process Model"
[2]: http://devcenter.heroku.com/articles/custom-domains "Heroku Custom Domains"
[3]: https://github.com/spree/spree/issues "Spree Issues"
[a]: http://devcenter.heroku.com/articles/rails31_heroku_cedar/ "Rails 3.1 Heroku Cedar"
[55]: https://devcenter.heroku.com/articles/ruby-versions#selecting_a_version_of_ruby "Selecting a version of Ruby"
[assets-precompiling]: #assets-precompiling
[disable_ssl]: #disable-ssl-in-production-mode
[self-signed-ssl]: http://devcenter.heroku.com/articles/ssl-certificate-self "Creating a Self-Signed SSL Certificate"