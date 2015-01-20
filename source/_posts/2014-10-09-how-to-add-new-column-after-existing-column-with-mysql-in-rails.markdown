---
layout: post
title: "How to add new column after existing column with MySQL in Rails?"
date: 2014-10-09 15:27
comments: true
categories:
tags: sql, rails
author: "Trung Lê"
---

{{ post.title }}

By default, SQL `ADD COLUMN` would add new column to the tail of columns. What if you
want to add a new column and append this new column after an existing column? Read on
and I'll show you how to do that with Rails in MySQL

<!--more-->

## Add new column after existing column in SQL

MySQL provides `AFTER` contrainst for `ADD COLUMN`:

```sql
ALTER TABLE mytable
ADD COLUMN  new_column <type>
AFTER       existing_column
```

The example above tell MySQL to create new `new_column` after `existing_column`.

## Doing it with Rails migration

With Rails, we could use ActiveRecord::Migration helper for the job. Let's create
a new migration with content:

```ruby
def change
  add_column :mytable, :new_column, after: 'existing_column'
end
```

OR if you prefer the SQL way:

```ruby
def change
  add_column :mytable, :new_column, '<type> AFTER existing_column'
end
```

That's all for today folks