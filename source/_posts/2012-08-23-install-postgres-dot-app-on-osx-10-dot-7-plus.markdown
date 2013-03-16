---
layout: post
title: Install Postgres.app on OSX 10.7+
date: 2012-08-23 16:21
comments: true
categories: 
tags: postgresql, ruby, rails
author: "Trung Lê"
---

{% img left /images/netsuke.png %}

{{ post.title }}

Traditionally, pogstgresql is installed manually with MacPort or Homebrew on Mac OSX 10.7+. I used
to have lots of problem with the setup for the installation as it requires Xcode, this libs and that libs, etc.
In summary, it is not convenient enough and I want something as simple as dragging an OS app to my /Application.
Thanks to Heroku, they took the heed and create Postgres.app. A wrapper bundled with binary postgresql server.
It is not only easy to install but also easy to setup config file if you are using Rails.


<br/>
<br/>
<br/>
<!--more-->

## Installation

If you have installed postgres using Homebrew or Macport, please make sure you uninstall it first.

Head to [http://postgresapp.com/][postgres.app] and download DMG into your localbox. Mount the DMG and drag the Postgres.app
icon into your Applications folder.

## Configuration

No configuration at all! Simply click on Postgres.app to start it and the app will reside in your top bar tray (elephant icon).
You can set it to start on start up by click on Elephant Icon on top bar tray and click Automatically Start on Login. That's it, dead simple compare to
setting up plist launcher file.

PostgreSQL ships with a constellation of useful binaries, like `pg_dump` or `pg_restore`, that you will likely want to use.
Go ahead and add the `/bin` directory that ships with Postgres.app to your `PATH`. Add this to your `.bash_profile`:

```
PATH="/Applications/Postgres.app/Contents/MacOS/bin:$PATH"
```

Once setup, you could refresh your terminal session with:

```
$ source .bash_profile
```

And try to run `psql` without a host, if everything is correct, you should be able to get into the the postgres console

## Configure Rails database connection

Use following settings in `config/database.yml`:

```
development:
  adapter: postgresql
  database: [YOUR_DATABASE_NAME]
  host: localhost
```

## Install pg gem

In order to install `pg` gem, we need to uninstall `pg` first and re-installed with:

```
$ gem uninstall pg
$ gem install pg -- --with-pg-lib=/Applications/Postgres.app/Contents/MacOS/lib
```

[postgres.app]: http://postgresapp.com/