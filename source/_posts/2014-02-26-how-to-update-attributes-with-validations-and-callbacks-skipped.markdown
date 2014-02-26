---
layout: post
title: "How to update attributes with validations and callbacks skipped"
date: 2014-02-26 18:20
comments: true
categories: rails
tags: rails
author: "Trung Lê"
---

{{ post.title }}

In this short tutorial, I'll show you how to update attributes for ActiveRecord model with:


* Validations are skipped.
* Callbacks are skipped.
* `updated_at`/`updated_on` are not updated.

Solutions for both Rails 3 and 4 are provided.

<!--more-->

## Rails 4

Rails 4 comes with method `update_columns` which takes in a hash parameter.

Here is one example how to use it:

```ruby
user.update_columns(last_request_at: Time.current. name: 'Trung Le')
```

At the background, one SQL UPDATE is run and will update both two attributes.

## Rails 3

Rails 3 does not have `update_columns` method, however there is `update_column` method, so we could loop through the parameters like below:

```ruby
{ last_request_at: Time.current. name: 'Trung Le' }.each do |k, v|
  user.update_column(k, v)
end
```

However this solution sucks because there are 2 SQL UPDATE queries involved!

There is a work-around by using `update_all`

```ruby
User.where(id: user.id).update_all(params)
```

Which only generate only 1 SQL UPDATE query.