---
layout: post
title: Spree 1.0 deployment on Heroku
date: 2012-02-21 06:40
comments: true
categories:
tags: spree, heroku, ruby, deployment, cedar stack
author: "Trung Lê"
---

{% img left /images/spree-1.0.0-release-ribbon.png %}

# {{ post.title }} #

Spree is without a doubt a de-facto eCommerce stack of Ruby world. Yet to many Spree
is still a pandora box. In this tutorial, I will try to go through step by step
from how to set up a Spree sandbox app on a local box to deployment to Heroku.

Please note that all instructions are written for Apple MacOSX 10.7.x system. However I
believe it applies to other UNIX and Linux systems too (maybe with minor adaptations).

<br/>
<br/>
<br/>
<!--more-->

## Prerequisites ##

#### Heroku ####

Install Heroku toolbelt via Rubygems:

```
$ gem install heroku
```

If you have installed Heroku, please ensure you update to version 2.1.0 or higher
for Cedar support:

```
$ heroku update
$ heroku --version
heroku-gem/2.28.7 (x86_64-darwin12.0.0) ruby/1.9.3
```

#### Ruby ####

By default, Heroku Cedar stack uses Ruby 1.9.2. However, I will use Ruby 1.9.3
for this tutorial.

If you haven't installed Ruby 1.9.3-p194 on your box, please do so:

```
$ rvm install 1.9.3-p194
```

I use RVM to manage rubies versions on my box, you can adapt this technique
if you use rbenv or precompiled binary package

#### Ruby On Rails ####

Spree 1.0.x leaves the choice of rails version to you. You can choose version
`3.1.1` to `3.1.6`. It is highly recommended that you go for `3.1.6`.

```
$ gem install rails -v=3.1.6
```

#### Spree ####

```
$ gem install spree -v=1.0.4
```

Check installed spree gems:

```
$ gem list | grep 'spree'
spree (1.0.4)
spree_api (1.0.4)
spree_auth (1.0.4)
spree_cmd (1.0.4)
spree_core (1.0.4)
spree_dash (1.0.4)
spree_promo (1.0.4)
spree_sample (1.0.4)
```

`spree` gem consists of many components, however please note that you only need `spree_core`
to build an online store.


#### PostgreSQL ####

Heroku only supports PostgreSQL. It is a good practice to have you development
environment use the same DB.

PostgreSQL can be installed on OSX with either of these methods below:

##### 1. Postgres.app (RECOMMENDED) #####

You can install the Postgres.app from Heroku guys at [http://postgresapp.com/][postgres.app]

##### 2. Homebrew #####

If you aren't in a rush, PostgeSQL can be installed with Homebrew:

```
$ brew install postgresql
```

Though I do not recommend this method because you might bump into issues with compilation.
Please make sure you read the Build Notes after the installation.

#### pg gem ####

Now we need to install `pg` gem too:

```
$ gem install pg
```

#### ImageMagick ####

Spree uses `paperclip` gem which in turn require imagemagick. You search on Google
for binary DMG package or for my case, I install it with Homebrew:

```
$ brew install imagemagick
```

## Prepare local application ##

Create a new rails app default to postgreSQL

```
rails _3.1.6_ new fool-man-chew -d postgresql
```
Configure database setting by editing `config/database.yml`.

```
development:
  adapter: postgresql
  encoding: unicode
  database: fool-man-chew_development
  pool: 5

test:
  adapter: postgresql
  encoding: unicode
  database: fool-man-chew_development
  pool: 5

production:
  adapter: postgresql
  encoding: unicode
  database: fool-man-chew_development
  pool: 5
```

then create the DB tables:

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
gem 'spree', '~> 1.0.4'
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

By default, Heroku use the `thin` webserver. However in this tutorial, we are going to
use `puma` instead, just to show you the great new process types system that
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

The great thing about Cedar stack is that Heroku introduces a new way to scale your app,
that is [Process Model][1], now you could define a custom list of process type
that you want to run in the `Procfile` file.

We configure our unicorn which is of type `web` by creating new file in `Rails.root`
folder `Procfile`:

```
web: bundle exec rails server puma -p $PORT -e $RACK_ENV
```

### Assets Storage ###

Because Heroku is disk-less therefore assets like images are not persistently
stored. The workaround is to use Cloud storage service like Amazon S3.

#### spree-heroku extension ####

The `spree_heroku` gem lets you store images and data to Amazon S3, to install it
we append to `Gemfile`:

```
gem 'spree_heroku', :git => 'git://github.com/joneslee85/spree-heroku.git', :branch => '1-0-stable'
```

then

```
$ bundle install
```

Next, we create a new bucket 'fool-man-chew_production' under US Standard region via AWS Management Console.

We need to tell Spree how to access our bucket, there are 2 ways to configure S3
settings.

First one is to create Heroku config vars (recommended way):

```
$ heroku config:add S3_KEY='your_access_key'
$ heroku config:add S3_SECRET='secret_access_key'
$ heroku config:add S3_BUCKET='fool-man-chew_production'
```

The second is to create a new file under `config/s3.yml` and modify the key in accordance to your S3 account:

```
production:
  bucket: fool-man-chew_production
  access_key_id: your_access_key
  secret_access_key: secret_access_key
```

#### Heroku boostrap ####

We are going to create an Cedar stack app:

```
$ heroku apps:create smooth-autumn-7451
```

If success, you would see below output:

```
Creating smooth-autumn-7451... done, stack is cedar
http://smooth-autumn-7451.herokuapp.com/ | git@heroku.com:smooth-autumn-7451.git
Git remote heroku added
```

and double check git remote you would see heroku remote listed:

```
$ git remote show
heroku
```

#### Use ruby-1.9.3 on Heroku ####

Cedar stack default to ruby-1.9.2, however Spree has been tested with
`ruby-1.9.3` so I highly recommended you to use same version for production.

*UPDATE*: After March 19, 2012, Heroku has deprecated installing Ruby 1.9.3 using
heroku-labs plugin. So it is uncesssary to setup RUBY_VERSION variable or to set up
PATH config variable to include 'bin'.

The new way is to specify the Ruby version in the `Gemfile`.
Unfortunately, as of 3 July 2012, the unreleased version 1.2.0.pre.1 is the only
version that support the feature. So we will go with this unrelease version.

You could read more about [Selecting a version of Ruby] [55].

First, we install latest version of Rubygems-bundler:

```
$ gem update rubygems-bundler
```

Then install Bundler 1.2.0 or newer, we have to uninstall the
current version before installing:

```
$ gem uninstall -ax bundler
$ gem install bundler
```

Now we specify Ruby version in the `Gemfile`:

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


#### Add SSL certificate ####

Spree production mode always enforce SSL. This step is very optional,
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
$ heroku domains:add smooth-autumn-7451.herokuapp.com
$ heroku ssl:add final.crt site.key
```

#### Deploy Spree to Heroku ####

Now we could push our app to Heroku:

```
git init
git add -A
git commit -m "Initial commit"
git push heroku master
```

*OPTIONAL*: If you *ever* bump into issues where Bundler fails to locate gems, the best workaround is to cache the bundle:

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
       http://smooth-autumn-7451.herokuapp.com deployed to Heroku
```

Next we could repeat the same bootstraping step on our remote heroku:

```
$ heroku run rails g spree:install
```

Now we could open app:

```
$ heroku apps:open
```

However to your surprise you could see that there are no images displayed correctly.
This is expected behavior and worry not, I have your back covered, please read section
[Assset Pipeline] [assets-precompiling]

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

So we have to disable precompile on initialization by set `config.assets.initialize_on_precompile` to `false` in `config/application.rb`

{% codeblock config/application.rb lang:ruby %}
config.assets.initialize_on_precompile = false
{% endcodeblock %}

To workaround this issue, there are 2 ways:


##### 1. Locally Precompiling Assets #####

We precompile the assets on localbox:

```
$ RAILS_ENV=production bundle exec rake assets:precompile
```

What will happen next is Sprocket will compile our assets and place them in `public/assets` folder. What Heroku really care is the `public/assets/manifest.yml`. This file contains all MD5 checksums of our assets and Heroku will check the existence of the file to tell if we compile our assets locally or not. Make sure you double check your `.gitignore` and remove the `public/assets` if there is one so git won't omit this path.

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

Again, I do not recommend you using this method, because you pollute your source tree
with precompiled assets. Please avoid this at all cost and instead use a CDN like S3 or Rackspace
to serve our precompiled assets.

##### 2. Serve precompiled assets on CDN (RECOMMENDED) #####

First, we install `asset_sync` gem by appending to your `Gemfile`:

```
group :assets do
  # asset_sync is required as needed by application.rb
  gem 'asset_sync', :require => nil
end

```

then

```
$ bundle install
```

Then we have to enable `user_env_compile` plugin first:

```
heroku labs:enable user_env_compile
```

Next, we configure asset_sync to sync with Amazon S3 by adding following variables into Heroku env:

```
$ heroku config:add AWS_ACCESS_KEY_ID=xxxx
$ heroku config:add AWS_SECRET_ACCESS_KEY=xxxx
$ heroku config:add FOG_DIRECTORY=xxxx
$ heroku config:add FOG_PROVIDER=AWS
# and optionally:
$ heroku config:add FOG_REGION=eu-west-1
$ heroku config:add ASSET_SYNC_GZIP_COMPRESSION=true
$ heroku config:add ASSET_SYNC_MANIFEST=true
$ heroku config:add ASSET_SYNC_EXISTING_REMOTE_FILES=keep
```

Configure `config/environments/production.rb` to use Amazon S3 as the asset host and ensure precompiling is enabled:

{% codeblock config/environments/production.rb lang:ruby %}
config.action_controller.asset_host = "//#{ENV['FOG_DIRECTORY']}.s3.amazonaws.com"
{% endcodeblock %}

Also, ensure the following are defined (in `production.rb` or `application.rb`)

* config.assets.digest is set to true.
* config.assets.enabled is set to true.

That's done. Commit changes and push to remote Heroku:

```
git commit -am "Added asset_sync gem and configure asset_host to serve from S3 bucket in production"
git push heroku master
```

The `asset_sync` override `rake assets:precompile` deploy task and will automatically synchronized all modified and
new assets to S3.

You should see output like below if success:

```
....
Preparing app for Rails asset pipeline
Running: rake assets:precompile
AssetSync: using default configuration from built-in initializer
AssetSync: using default configuration from built-in initializer
AssetSync: Syncing.
Using: Directory Search of /tmp/build_z7t7ti7lk5up/public/assets
Uploading: assets/admin/all-190f577bede460d23825559e6cfdd106.js
Uploading: assets/admin/all-190f577bede460d23825559e6cfdd106.js.gz
AssetSync: Done.
Asset precompilation completed (91.32s)
....
```

#### Custom Domain ####

Now we push a bit further by setting up custom domain for our shop, first we need
to set up Heroku to respond to requests at custom domains:

```
$ heroku addons:add custom_domains
Adding custom_domains to smooth-autumn-7451...done.
```

And inform Heroku our beautiful `fool-man-chew.com` domain

```
$ heroku domains:add www.fool-man-chew.com
Added www.example.com as a custom domain name to smooth-autumn-7451.heroku.com
$ heroku domains:add fool-man-chew.com
Added example.com as a custom domain name to smooth-autumn-7451.heroku.com
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


### Extra ###

#### Disable SSL in Production mode ####

Edit file `config/initializers/spree.rb`:

{% codeblock config/initializers/spree.rb lang:ruby %}
Spree.config do |config|
  config.allow_ssl_in_production = false
end
{% endcodeblock %}


Make sure you commit the changes to app repository.


#### Deploy to Heroku using heroku_san ####

It is much quicker to setup and deploy to Heroku by using heroku_san gem.

Simply install `heroku_san` by appending to your `Gemfile`:

```
group :development do
  gem 'heroku_san'
end
```

then

```
$ bundle install
```

Configure heroku_san by creating `config/heroku.yml` file with content:

```
production:
  stack: cedar
  app: smooth-autumn-7451
  config:
    BUNDLE_WITHOUT: 'development:test'
  config:
    # vars for asset_sync
    AWS_ACCESS_KEY_ID: xxx
    AWS_SECRET_ACCESS_KEY:xxx
    FOG_DIRECTORY: 'skateshop'
    FOG_PROVIDER: 'AWS'
    # vars for spree_heroku
    S3_KEY='your_access_key'
    S3_SECRET='secret_access_key'
    S3_BUCKET='fool-man-chew_production'
```

We deploy to heroku by running following rake task:

```
$ rake production deploy
```

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
[postgres.app]: http://postgresapp.com/