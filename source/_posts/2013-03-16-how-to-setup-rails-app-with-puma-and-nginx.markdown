---
layout: post
title: "How to setup Rails app with puma and NGINX"
date: 2013-03-16 17:51
comments: true
categories:
tags: nginx, capistrano, deployment, rails, ruby, puma
author: "Trung Lê"
---

{{ post.title }}

In this tutorial, I'll walk you through the concept behind using puma + NGINX, plus thorough instructions on setting them up on CentOS and Ubuntu.

<!--more-->

## Concept

Many people who come from the old Apache HTTPd day often ask me how reverse proxy + web server work?

> _"It's different paradigm"_

Reverse proxy software (such as Varnish or NGINX) would acts as a load balancer that routes all external requests to a pool of web apps. The below diagram depicts simply how it works:

```
                                               +---> web app process 1 --> threads
                                               |
[requests] <------>  [reverse proxy server]  --+---> web app process 2 --> threads
                                               |
                                               +---> web app process 3 --> threads
```

In constrast to Apache way, which is depicted below:

```
                    +--- web process fork
                    |
[requests] ------>  +--- web process fork
                    |
                    +--- web process fork
```

in which Apache HTTPd will act as both load balancer that route request to only 1 dedicated web app and automatically tell itself to fork more processes to meet demand.

I am NOT going to dwelve into which way is better than which. My 5cent on this is, reverse proxy + web server is new way to do things, it offers distinct advantages on scalability on multi-tenancy scenario. In which you could up-scale and down-scale more web processes on demand without affecting other apps. The downside is you have to deal with process monitoring which requires understanding of UNIX processes.

## Installation

### puma

puma is a multi-threaded high performance webserver written in Ruby. It is new in the market yet it has gained lots of traction. It can be used to server any ruby web app that support rack such as Sinatra or Ruby On Rails.

As a first class Ruby project, you could install `puma` via RubyGems.

With Rails 3+ app, simply append to `Gemfile`:

```
gem 'puma', '~> 2.1.0'
```

then `bundle install`

You can now start your app with puma with `rails s`.  You should see output if it is started correctly:

```
Puma 2.1.1 starting...
* Min threads: 0, max threads: 16
* Environment: development
...
```


### NGINX

NGINX is utilised as reverse proxy server for its `HttpProxyModule` could perform proxy passing request to many virtual hosts.

Firstly, we need to get the software installed on our server:

__CentOS 5+__

Create file name `/etc/yum.repos.d/nginx.repo` and paste following:

```
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1
```

Then we can install NGINX with:

```
sudo yum install nginx
```

and the server could be started with:

```
sudo /etc/init.d/nginx start
```

__Ubuntu 12.04__

The NGINX version in the Ubuntu repo is quite old (ie. 1.2.6), you could install
newer version by adding the official nginx.org repo:

```
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys ABF5BD827BD9BF62
```

then add following line into the end of file `/etc/apt/sources.list`:

```
deb http://nginx.org/packages/ubuntu/ precise nginx
```

NOTE: If you have installed NGINX before, make sure you remove it first:

```
sudo apt-get purge nginx*
```

then you can install the package with:

```
sudo apt-get update
sudo apt-get install nginx
```

Once successfully installed, you could verify with:

```
nginx -v
# nginx version: nginx/1.4.0
```

you can manually start the daemon using Upstart:

```
sudo service nginx start
```

## Configuration

Due to the differences in file locations between CentOS and Ubuntu, I divide this section into two, please read the section that match your OS.

Before continuing, there are few assumptions I would like you to be aware of:

* You are running on Ruby 1.9.3 or newer
* Your Rails app is 3.x or newer
* You are running your app under `RAILS_ENV=production`
* Your rails app should be placed in `/var/www` folder.
* You have setup correctly all permissions and firewall settings for your environment

### NGINX configuration

We are going to configure NGINX to have an `upstream` directive, this directive tell NGINX where to proxy parse the request to.
Next we will add a virtual host and use `proxy_pass` directive to tell NGINX to pass the request to the pool of processes defined in `upstream` section.

__CentOS__

The very first thing I normally do is to delete all default virtual hosts files:

```
sudo rm /etc/nginx/conf.d/default.conf
```

Next, let's create a new host config file at `/etc/nginx/conf.d/my_app.conf` for our rails app, paste following:

```
upstream my_app {
  server unix:///var/run/my_app.sock;
}

server {
  listen 80;
  server_name my_app_url.com; # change to match your URL
  root /var/www/my_app/public; # I assume your app is located at this location

  location / {
    proxy_pass http://my_app; # match the name of upstream directive which is defined above
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```

then you can restart your NGINX server:

```
sudo /etc/init.id/nginx restart
```


__Ubuntu__

The very first thing I normally do is to disable default site by removing the symlink:

```
sudo rm /etc/nginx/conf.d/sites-enabled/default
```

Next, let's create a new virtual host config file at `/etc/nginx/sites-available/my_app.conf` for our rails app, paste following:

```
upstream my_app {
  server unix:///var/run/my_app.sock;
}

server {
  listen 80;
  server_name my_app_url.com; # change to match your URL
  root /var/www/my_app/public; # I assume your app is located at that location

  location / {
    proxy_pass http://my_app; # match the name of upstream directive which is defined above
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```

and we need to enable it by creating symlink in `/etc/nginx/sites-enabled`

```
sudo ln -sf /etc/nginx/sites-available/my_app.conf /etc/nginx/sites-enabled/my_app.conf
```

then you can restart your nginx server:

```
sudo service nginx restart
```

### start your app server

Now we need to tell puma to start our app and bind it to a Unix socket:

```
cd /var/www/my_app
RAILS_ENV=production bundle exec puma -e production -b unix:///var/run/my_app.sock
```

if nothing goes wrong, you should see this:

```
Puma 2.1.1 starting...
* Min threads: 0, max threads: 16
* Environment: development
* Listening on unix:///var/run/my_app.sock
Use Ctrl-C to stop
```

and now try to surf your site with Firefox or Chrome, open `my_app_url.com`. Please substitute it with your site domain name instead here. And you should not see any error page like 403.

Once you have verified that our puma has correctly serve the request, we now can run the puma server as daemon.

Press `CTRL-C` to stop the foreground running puma processing and run command with `-d` parameter:

```
RAILS_ENV=production bundle exec puma -e production -d -b unix:///var/run/my_app.sock
```

You could verify if the puma process is in background or not:

```
ps aux | grep puma
# 9594 92.8  1.4 496844 117280 ?       Rl   17:25   0:25 ruby /usr/lib/ruby/1.9.1/bin/puma -e production -d -b unix:///var/run/my_app.sock
```

### restart/stop your daemon puma server

In order to restart our server, we use command `kill` to send in `SIGUSR2` signal
to the puma PID, in our case the PID is 9594:

```
kill -s SIGUSR2 9594
```

The process kills itself and fork a new process with new PID

If you want to stop the server, simply send `SIGTERM`:

```
kill -s SIGTERM 9594
```

you can verify if the process has been killed or not with `ps`.

If you want UNIX fetch you the PID, you can start puma server with `--pidfile` params:

```
RAILS_ENV=production bundle exec puma -e production -d -b unix:///var/run/my_app.sock --pidfile /var/run/puma.pid
```

The PID will be store in `/var/run/puma.pid` and now you could just `cat` the PID out easily:

To restart:

```
kill -s SIGUSR2 `cat /var/run/puma.pid`
```

To stop:

```
kill -s SIGTERM `cat /var/run/puma.pid`
```


### a better way to manage your puma processes

Dealing with UNIX PID is not something you want to deal with everyday. There are
many tools to manage processes such as `god` or `monit`. You need to spend time
to configure them correctly. Thanksfully, puma comes with a built-in process
manager that help ease your admin job, introducing the awesome `pumactl` command line tool.

`pumactl` is the puma processes monitor and controller, it allows you to start/restart/stop the server with 3 ways:

* PID
* State file
* Control Server

#### PID (RECOMMENDED)

This is exactly like the traditional way of sending signal to process with `kill`. Please be noted that we could not start our server with this way because `pumactl` has no idea what our configuration, so you have to start the server the normal way with `puma` command.

However, for existing running puma process, you could stop it with:

```
bundle exec pumactl -p <pid> stop
# Command stop sent success
```

and to restart it:

```
bundle exec pumactl -p <pid> resart
# Command restart sent success
```

And in the case where you tell puma to store its PID in pid file:

```
RAILS_ENV=production bundle exec puma -e production -d -b unix:///var/run/my_app.sock --pidfile /var/run/puma.pid
```

and then we could restart the process with:

```
bundle exec pumactl -P /var/run/puma.pid restart
```

and stop process with:

```
bundle exec pumactl -P /var/run/puma.pid stop
```

Isn't that more civilized? At the moment of this writing, I find dealing with PID is very reliable. The downside is that you have to manually manage the process. I highly recommend you to use `monit` or `god` to manage the process instead of `pumactl` because `pumactl` suffers few bugs. 

#### State file (AVOID)

Dealing with PID is painful, you need to type the whole `puma` start line over and over again. However, there is a better way, by storing all the configurations into one state file, `pumactl` would be able to manage processes easily.

So firstly, we need to kill all puma processes running:

```
sudo killall puma
```

then we start our server with `-S` parameter:

```
RAILS_ENV=production bundle exec puma -e production -d -b unix:///var/run/my_app.sock -S /var/run/my_app.state
```

puma would generate `/var/run/my_app.state` file with content:

```
---
pid: 9654
config: !ruby/object:Puma::Configuration
  options:
    :min_threads: 0
    :max_threads: 16
    :quiet: true
    :debug: false
    :binds:
    - unix:///var/run/my_app.sock
    :workers: 0
    :daemon: true
    :worker_boot: []
    :environment: production
    :worker_directory: /home/jr_deploy/staging/current
    :state: /var/run/my_app.state
```

this is a serialized `Puma::Configuration` object, and `pumactl` would read this object to figure out where you bind your puma process and other details.

Now to restart our puma, we do:

```
bundle exec pumactl -S /var/run/my_app.state restart
```

and if you want to stop puma:

```
bundle exec pumactl -S /var/run/my_app.state stop
# Command stop sent success
```

The interesting thing is that `/var/run/my_app.state` will be peristent and you could start the server with all configurations retained:

```
bundle exec pumactl -S /var/run/my_app.state start
```

the output I get is:

```
Puma 2.1.1 starting...
* Min threads: 0, max threads: 16
* Environment: development
* Listening on tcp://0.0.0.0:9292
Use Ctrl-C to stop
```

which is not the way I expected because the server is started in non-daemon mode and use TCP mode. I believe this is an issue with puma. So for now, please refrain from starting the server with pumactl. Instead, remove file `/var/run/my_app.state` and repeat the whole process again. I've lodged this issue with @evanphx at https://github.com/puma/puma/issues/287

#### Control Server (AVOID)

puma also comes with a built-in control server which will be started on localhost port 9293 when you run puma with `--control` params:

```
RAILS_ENV=production bundle exec puma -e production -d -b unix:///var/run/my_app.sock --control unix:///var/run/my_app_pumactl.sock --control-token foo
```

If nothing goes wrong, you should see:

```
Puma 2.1.1 starting...
* Min threads: 0, max threads: 16
* Environment: development
* Listening on unix:///var/run/my_app.sock
* Starting status server on unix:///var/run/my_app_pumactl.sock
Use Ctrl-C to stop
```

Please pay attention to `--control-token` param, this is auth token which is required by `pumactl` to talk to this control server.

You can query status of the server with:

```
pumactl -C unix://var/run/my_app_pumactl.sock [status|restart|halt|stop]
```

You should see:

```
Puma is started
```

Now let's try to halt/stop/restart the server:

```
pumactl -C unix://var/run/my_app_pumactl.sock -T foo [restart|halt|restart]
``` 

Please be noted, if your `-T` token does not match the token the server used, in our case `foo`, you would get this error:

```
Unauthorized access to server (wrong auth token)
```

Okay, so far all goes well, let's try to stop the server first then start it with:

```
pumactl -C unix://var/run/my_app_pumactl.sock -T foo start
```

and the output:

```
Puma 2.1.1 starting...
* Min threads: 0, max threads: 16
* Environment: development
* Listening on tcp://0.0.0.0:9292
Use Ctrl-C to stop
```

which is wrong, it seems to me puma suffers the same issue as of the State file. So please refrain from using State File and Control Server methods to monitor puma process. Instead, just go for the PID way and use `monit` or `god` instead. I'll update this section once the issue is resolved.

### Capistrano deployment

Now you know how puma works, you could adapt what you learned for your deployment. If you use `capistrano`, you could use the `puma/capistrano` tasks by append to your `config/deploy.rb`:

```
require 'puma/capistrano`
```

and then

```
bundle exec cap puma:start
bundle exec cap puma:restart 
bundle exec cap puma:stop
```


