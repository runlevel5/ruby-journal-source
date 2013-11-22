---
layout: post
title: "Digesting pumactl"
date: 2013-11-22 16:39
comments: true
categories:
---

Puma is multi-threaded web server which is implemented in Ruby and has become a popular choice
for many production servers in the world. Given its short time of development, you'll likely
to see issues. One of the issue that I and many people often bump into is processes management.
By default, puma offers `pumactl`, yet this utitlity hasn't lived up to expectation (this is a year
ago), thus people seeks different approaches with custom bash script by calling `puma` directly,
upstart, monit, etc. However, today when I revisit `pumactl`, this tool has been polished and
now does exactly what it promises. In this short article, I'll go through with you how to use
`pumactl` to manage your puma processes.

## Anatomy of puma processes

Like any other UNIX web server, puma is run as daemon, spawning child processes (in puma term,
they called in puma cluster worker) to handle requests from the outside world.

For example:

```
$ ps aux | grep puma
1000      2527  0.0  0.2  80824 20312 ?        Sl   Nov21   0:02 puma -C config/puma.rb
1000      2530  0.0  2.2 870928 184180 ?       Sl   Nov21   0:33 puma: cluster worker: 2527
1000      2533  0.0  2.3 870868 188284 ?       Sl   Nov21   0:32 puma: cluster worker: 2527
```

You can see above that process with PID 2527 is our mother process which spawn two children.
Both children with PID 2530 and 2533 respectively clearly states that its mother is PID 2527.

Next, we are going to look into how to manage puma process manually.

### Stop puma process

To stop this process, we send a SIGTERM signal to PID 2557

```
$ kill -s SIGTERM 2557
```

And we could check if the process has been cleanly killed with:

```
$ ps aux | grep puma | grep 2557
# yields no results
```

### Hard restart puma process

To restart this process, we send a SIGUSR2 signal to PID 2557.

```
$ kill -s SIGUSR 2557
```

Please note, this action is equivalent to killing the mother process and start a new one. Avoid this thing in production environment at all cost. Because we do not want downtime.

Checking the ps yields:

```
$ ps aux | grep puma
1000      3001  0.0  0.2  80824 20312 ?        Sl   Nov21   0:02 puma -C config/puma.rb
1000      3010  0.0  2.2 880321 184180 ?       Sl   Nov21   0:33 puma: cluster worker: 3001
1000      3020  0.0  2.3 700828 188284 ?       Sl   Nov21   0:32 puma: cluster worker: 3001
```

The output shows that our PID 2557 is killed and a new PID 3001 is created.

### Graceful restart puma process

In order to achieve zero downtime, we only kill these 2 children and respawn
with 2 new one instead of killing the mother process. In puma term, they call it `phased-restart`,
that is sending SIGUSR1 signal to PID 2557.

```
$ kill -S SIGUSR1 2557
```

Let verify the result:

```
$ ps aux | grep puma
1000      2527  0.0  0.2  80824 20312 ?        Sl   Nov21   0:02 puma -C config/puma.rb
1000      3001  0.0  2.2 870928 184180 ?       Sl   Nov21   0:33 puma: cluster worker: 2527
1000      3002  0.0  2.3 870868 188284 ?       Sl   Nov21   0:32 puma: cluster worker: 2527
```

The 2 children are killed and two new spawn children appears. Just what we expect.

### Checking status of puma process

In order to check process, you have to start puma with `-S` parameter, this points to a state
file which stores all statuses of our puma process. Example:

```
$ puma -S /var/run/app.state
```

Now you could hook this file into any monitoring tools. I find this state file idea not helpful
as many of us relies on other system to manage status such as monit or god.

## Introduction to pumactl

As we can see that above operations can be tedious and error prone and definitely not fun to work
with a big deployment scale. Introducing pumactl, this utility automates all of above tasks. Let's
see how we could reproduce all above steps

### Start puma process

We setup a config file that asks puma to store its PID in a pid file and run in daemon mode. We'll
carry out a test on an Rails app. Create `config/puma.rb`:

```
environment ENV['RAILS_ENV'] || 'production'
daemonize

workers    2 # should be same number of your CPU core
threads    1, 6

pidfile    "/var/run/puma_app1.pid"
```

We now can start puma with:

```
$ bundle exec pumactl -F config/puma.rb start
```

Verify with:

```
$ ps aux | grep puma
1000      2527  0.0  0.2  80824 20312 ?        Sl   Nov21   0:02 puma -C config/puma.rb
1000      2530  0.0  2.2 870928 184180 ?       Sl   Nov21   0:33 puma: cluster worker: 2527
1000      2533  0.0  2.3 870868 188284 ?       Sl   Nov21   0:32 puma: cluster worker: 2527
```

we can see above that pumactl will start our server with `puma -C config/puma.rb`. Sweet. Next
we check to see if pid file store correct PID:

```
$ cat /var/run/puma_app1.pid
2527
```

Just as expected.

### Stop puma process

Simply with:

```
$ bundle exec pumactl -F config/puma.rb stop
```

There is no output yielded with

```
$ ps aux | grep puma | grep 2557
```

indicates that the process is cleanly killed

### Hard restart puma process

```
$ bundle exec pumactl -F config/puma.rb restart
```

```
$ ps aux | grep puma
1000      3001  0.0  0.2  80824 20312 ?        Sl   Nov21   0:02 puma -C config/puma.rb
1000      3010  0.0  2.2 880321 184180 ?       Sl   Nov21   0:33 puma: cluster worker: 3001
1000      3020  0.0  2.3 700828 188284 ?       Sl   Nov21   0:32 puma: cluster worker: 3001
```


The output shows that 2557 is killed and new PID 3001 is created. We could check to see if
our pid file is updated with 3001:

```
$ cat /var/run/puma_app1.pid
3001
```

Magic! It's working!

### Graceful restart puma process

```
$ bundle exec pumactl -F config/puma.rb phased-restart
```

and check again:

```
$ ps aux | grep puma
1000      2527  0.0  0.2  80824 20312 ?        Sl   Nov21   0:02 puma -C config/puma.rb
1000      3001  0.0  2.2 870928 184180 ?       Sl   Nov21   0:33 puma: cluster worker: 2527
1000      3002  0.0  2.3 870868 188284 ?       Sl   Nov21   0:32 puma: cluster worker: 2527
```

PID 2527 is still there and only its children are respawned. Perfect!

### Checking status of puma process

Now we do not need other tool to tell if our PID is running, we could do with one command:

```
$ bundle exec pumactl -F config/puma.rb status
```

if PID is running we would get output:

```
PID 2557 is running
```

else

```
No puma process
```

## Conclusion

That's it folk. Now you know how to use pumactl, why don't you delete your custom script and replace
it with pumactl. See you in the next article.

