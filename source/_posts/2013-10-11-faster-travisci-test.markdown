---
layout: post
title: "Faster TravisCI test"
date: 2013-10-11 12:37
comments: true
tags: ruby rails test spec rspec
author: "Trung Lê"
---

{{ post.title }}

Running test suite is time consuming. There are various techniques to optimize the runtime performance of the CI by stub/mock, parallel_test, etc. In this short tutorial, I'll show you how to optimize your TravisCI test suite by splitting your test suit into concurrent jobs, which drastically improve the build time.

<!--more-->

I have a Rails app with rspec. Here are specs that are run sequentially:

* models
* controllers
* helpers
* routing
* views
* features

Running specs sequentially is slow. Travis can break these specs into concurrent jobs for you. You need to modify your `.travis.yml`:

```
language: ruby
rvm:
  - 2.0.0
bundler_args: --without development production --quiet
env:
  - TEST_SUITE=features
  - TEST_SUITE=controllers
  - TEST_SUITE=helpers
  - TEST_SUITE=models
  - TEST_SUITE=routing
  - TEST_SUITE=views
script:
  - bundle exec rspec spec/$TEST_SUITE/*
```

As you can see in the above YAML file, we set `$TEST_SUITE` env variable with the name of our spec folders. This variable will get substitute into `bundle exec rspec spec/$TEST_SUITE/*` call.

Once you push new change to your GitHub, TravisCI now run 6 concurrent jobs at the same time. My build before take 45mins, now reduce to 18mins. That's a great improvement. Please be noted that, please upgrade to 10 concurrent build plan if you are using Travis Pro service.

Furthmore, you could cut at least 3 more mins by enabling the experimental bundler caching (which is only available for Travis Pro plan now). Append to your `.travis.yml`:

```
cache: bundler
```

Travis will cache your bundler and next build will not waste their time with bundler install anymore which save time.

By using `env` and `cache` option, you could reduce the build time by at least 60%. You could push further by breaking your slow specs and group them together and have them run separately too. Or you could introduce parallel_test gem into your app, this would save you at least 30% more. I hope you enjoy the tutorial and see you next time.
