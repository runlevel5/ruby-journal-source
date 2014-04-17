---
layout: post
title: "How to write perfect Rails controller spec with RSpec"
date: 2014-04-17 20:10
comments: true
tags: test, ruby, ci, travis
---

{{ post.title }}

More than few occasions, I get asked this very question: "How to write a Rails controller spec? What do we test?".
Obviously there are many articles and books on the world wide web that have answered this question since
the very first version of Rails. However most of them are varied in test style or simple too abstract which
aren't very helpful for beginners.

Today I am going to show you how to write a perfect controller spec with RSpec (well, with my convention). So
if you are a confused Rails begineer, I hope this would be your last stop.

<br/>
<!--more-->

## Prerequisites

We are going to use rspec-rails gem. You might ask why I don't show you how to it with stock Rails testing framework?
Well, I've been using trying out stock Rails testing framework and I really hate the Rails concept of so called functional test and integration test. I'd much prefer the RSpec style. I am going deep in this topic and this would
burn down few houses in the progress.

## Sample app

For the demo purpose, we are going to create a simple blog app. A blog should start with a Post right?

```
rails g scaffold post title content:text
```

Rails would generate a resource Post with PostsController that consists of 7 CRUD actions: index, create, update, show, edit, new, destroy which are to be tested.

## Controller spec

Firstly, we remove the auto-generate controller spec file:

```
rm spec/controllers/posts_controller_spec.rb
```