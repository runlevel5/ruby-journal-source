---
layout: post
title: "How to integrate Sidekiq with ActiveJob"
date: 2014-10-03 08:39
comments: true
categories:
tags: ruby, rails
author: "Trung Lê"
---

{{ post.title }}

One of the hot thing in Rails 4.2 is the brand new ActiveJob gem, this gem consolidate
the API for background job gems on the market such as DelayedJob, Resque, etc. Today I
am going to guide you through how to integrate Sidekiq with ActiveJob, and you will learn:

* Set up Sidekiq adapter for ActiveJob
* Basic of ActiveJob class
* Advanced usage of multiple queues
* ActiveJob callback
* ActiveJob exception catch

<!--more-->

## The concept

Before we move into details on implementation, I want to clarify that ActiveJob is not
Sidekiq but Sidekiq can act like ActiveJob. Because ActiveJob does not care how a job
is processed (that is forking processes for eg), it only does job queuing and delegate
job crunching to adapter, that is Sidekiq. However Sidekiq does both, it could act
as job queuer and job processor at same time.

So what's so good about ActiveJob then? ActiveJob standardises the API interface for
job queuer. This helps changing from one job backend to the other much easily.

## Install Rails 4.2

ActiveJob is only available in Rails 4.2 and you need to install the latest version of
4.2 (at the time of the writing is 4.2.0-beta2).


And in your Gemfile of your app, change the version to

```
gem 'rails', '4.2.0.beta2'
```

And do the `bundle install`

## Install Sidekiq

It is just as simple as append this line to your Gemfile

```
gem 'sidekiq', '3.2.5'
```

and fire away `bundle install`

Then we create new config file `config/sidekiq.yml` for our sidekiq

```
---
:concurrency: 25
:pidfile: ./tmp/pids/sidekiq.pid
:logfile: ./log/sidekiq.log
:queues:
  - default
  - [high_priority, 2]
```

The above config file tell Sidekiq where to store PID and log file.
Plus I also configure Sidekiq to create 2 queues, default and high priority queue.
FYI, the number 2 in high_priority line is the weight of 2, it means
that it will be check twice as often. I'll demonstrate how can we delegate our job to
right queue later.

You are free to configure Sidekiq in whatever way you see fit your app.

We can start our sidekiq daemon:

```
$ sidekiq -C config/sidekiq.yml
```

if nothing goes wrong, you should see similar output:

```
2014-10-03T01:23:24.775Z 13918 TID-oxapc5ias INFO: Running in ruby 2.1.3p242 (2014-09-19 revision 47630) [x86_64-darwin14.0]
2014-10-03T01:23:24.775Z 13918 TID-oxapc5ias INFO: See LICENSE and the LGPL-3.0 for licensing details.
2014-10-03T01:23:24.776Z 13918 TID-oxapc5ias INFO: Upgrade to Sidekiq Pro for more features and support: http://sidekiq.org/pro
2014-10-03T01:23:24.776Z 13918 TID-oxapc5ias INFO: Starting processing, hit Ctrl-C to stop
2014-10-03T01:23:24.777Z 13918 TID-oxaphajos DEBUG: {:queues=>["default", "high_priority", "high_priority"], :labels=>[], :concurrency=>25, :require=>".", :environment=>nil, :timeout=>8, :error_handlers=>[#<Sidekiq::ExceptionHandler::Logger:0x007fe76b1cfcc8>], :lifecycle_events=>{:startup=>[], :quiet=>[], :shutdown=>[]}, :verbose=>true, :daemon=>false, :pidfile=>"./tmp/pids/sidekiq.pid", :logfile=>"./log/sidekiq.log", :strict=>false, :config_file=>"config/sidekiq.yml", :tag=>"demo_app"}
```

That's a good, quit out the sidekiq by Ctrl-C and we could run sidekiq in daemon mode by
modifying our `config/sidekiq.yml` by appending so our file is:

```
---
:concurrency: 25
:pidfile: ./tmp/pids/sidekiq.pid
:logfile: ./log/sidekiq.log
:queues:
  - default
  - [high_priority, 2]
:daemon: true
```

Let's run out sidekiq again:

```
$ sidekiq -C config/sidekiq.yml

# we then can check if it's started correctly
$ ps aux | grep sidekiq
trung_le        13534  22.6  0.7  2667148 118288   ??  S    10:31am   0:02.62 sidekiq 3.2.5 demo_app [0 of 25 busy]
```

## Set up our Job

Let's imagine that our job is to run a CSV Importer in background, here is our CSVImporter class:

```ruby
class CsvImporter
  attr_reader :filepath

  def initialize(filepath)
    @filepath = filepath
  end

  def run
    puts "Prepare to import #{filepath} ..."
    # some crunching code
    puts "Import #{filepath} completed"
  end
end
```

Now we have our gems installed, we'll need to configure ActiveJob to use sidekiq as backend by creating following initializer:

```ruby
#config/initializers/active_job.rb

ActiveJob::Base.queue_adapter = :sidekiq
```

By convention, we place our Job class under `app/jobs`

Here is our uber-cool CSV importing job (I try to make the most boring thing on Earth...interesting)

```ruby
# app/jobs/csv_import_job.rb

class CsvImportJob < ActiveJob::Base
  queue_as :default

  # we assume that we have a class CsvImporter to handle the import
  def perform(filepath)
    CsvImporter.new(filepath).run
  end
end
```

As you can see above, we have `queue_as` and `perform`, these two methods are required by convention
when creating ActiveJob class.

As you might have known that sidekiq supports multiple queues, which we configure with `queues` option
in the sidekiq config file. In the above example, I tell Rails to delegate this CSV job to default queue by
specifying the queue name using `queue_as` API.

The `perform` method is the logic of job handler, what you want to do with the job. Please be noted that
you could customize the params to be whatever you like, in our case, the param is the CSV file path.

## The art of enqueuing

We could tell Rails to queue a job and run it as soon as the queue is free with `#perform_later`:

```ruby
CsvImportJob.perform_later('/tmp/my_file.csv')
```

and if you peek into `log/development.log` you should see:

```
[ActiveJob] Enqueued CsvImportJob (Job ID: 525b3f7b-adab-41de-afe7-bee229188501) to Sidekiq(csv) with arguments: "/tmp/my_file.csv"
```

the output indicates that our job has been successfully queued and processed

## Run the queue in the future!

What's cool about ActiveJob is that it allows you to schedule the time to run enequeued jobs.

we could delay the running till tomorrow noon by using `#set` method with `wait_until` option:

```ruby
CsvImportJob.set(wait_until: Date.tomorrow.noon).perform_later('/tmp/my_file.csv')
```

which generates log line:

```
[ActiveJob] Enqueued CsvImportJob (Job ID: 08f113f4-d12c-4401-a84e-1b0e55f194d6) to Sidekiq(csv) at 2014-10-04 12:00:00 UTC with arguments: "/tmp/my_file.csv"
```

the above output clearly point out that the job is scheduled to run on 2014-10-04 12:00:00 UTC. Cool, isn't it?

Furthermore, the option `wait` is also very cool too, it takes in human idomatic syntax from now on:

```ruby
CsvImportJob.set(wait: 2.week).perform_later('/tmp/my_file.csv')
```

the above code will tell the worker to run the job after 2 weeks from now.

## Prioritise with multi-queues

Let's assume that the business people want file that are located in folder `/tmp/urgent` to be processed first.
How could we go about tackling this? Introducing multi-queues, by specifying queues with higher weight, in our
case, we configure:

```
:queues:
  - default
  - [high_priority, 2]
```

the `high_priority` queue will have higher precedence than default queue, thus it'll be run first.

To tell ActiveJob to use this high priority queue on condition, we could parse in a block into our `queue_as` line.

```ruby
class CsvImportJob < ActiveJob::Base
  queue_as do
    if urgent_job?
      :high_priority
    else
      :default
    end
  end

  # we assume that we have a class CsvImporter to handle the import
  def perform(filepath)
    CsvImporter.new(filepath).run
  end

  private

  def urgent_job?
    self.arguments.first =~ /\/urgent\//
  end
end
```

the code in the block will be evaluated under the context of the job. I guess I don't have to explain much
about the code above, it's just a simple condition. But you might be puzzled about the usage of `self.arguments`.
This method returns an array of parameters that are parsed into `#perform`.

Let's give our code a trial:

```ruby
CsvImportJob.perform_later('/tmp/urgent/my_file.csv')
```

and pay close attention to out log output:

```
[ActiveJob] [CsvImportJob] [30488ea0-108c-484f-bf12-8341da38e76b] Performed CsvImportJob from Sidekiq(high_priority) in 0.2ms
```

you could see that ActiveJob tells Sidekiq to run the job from `Sidekiq(high_priority)`.

However, should you want to run non-urgent file in high priority queue, you could override with:

```ruby
CsvImportJob.set(queue: :high_priority).perform_later('/tmp/my_file.csv')
```

## The powerful callbacks!

I love ActiveRecord callbacks, it is one of the best pattern ever in Rails!!! (I will kill you if you use it in your app!)

So just like ActiveRecord and ActionController, Rails also offers callbacks for ActiveJob. Below is the list of available
callbacks:

* before/after/around_enqueue
* before/after/around_perform

Those callbacks hook into the enqueuing and performing steps of the job.

We can use callbacks to do job logging and notification. Saying that we want to notify our manager once the job is finished, we could
code do:

```ruby
class CsvImportJob < ActiveJob::Base
  queue_as :default

  after_perform :notify_manager

  # we assume that we have a class CsvImporter to handle the import
  def perform(filepath)
    CsvImporter.new(filepath).run
  end

  private

  def notify_manager
    NotificationMailer.job_done(User.find_manager).deliver_later
  end
end
```

The above code tells ActiveJob to execute `#notify_manager` after the CsvImporter has finished.

FYI, I use method `NotificationMailer#deliver_later`, this would tell ActiveJob to deliver email in the background too.

Furthermore, you could tell AJ to delay the deliver of mailer by specifying `wait` option:

```ruby
def notify_manager
  NotificationMailer.job_done(User.find_manager).deliver_later(wait: 2.minutes)
end
```

How cool is that!

## Catching the Exceptions!

There is no guarantee that our CsvImporter won't run into error, thus we should notify our manager should
the import job fails too! How can we do that? Introducing the `#rescue_from` method.

```ruby
class CsvImportJob < ActiveJob::Base
  queue_as :default

  rescue_from(StandardErrors) do |exception|
    notify_failed_job_to_manager(exception)
  end

  # we assume that we have a class CsvImporter to handle the import
  def perform(filepath)
    CsvImporter.new(filepath).run
  end

  private

  def notify_failed_job_to_manager(exception)
    NotificationMailer.job_failed(User.find_manager, exception).deliver_later
  end
end
```

The above code tells ActiveJob to listen to job execution's exception and then catch and
notify the manager.

## Conclusion

ActiveJob is surely a nice addition to Rails stack, it makes scheduling background jobs easier and more intuitive.
It is also a great abstraction for your app, you don't have to worry about the under layer.

Again, good luck and keep on learning folks!
