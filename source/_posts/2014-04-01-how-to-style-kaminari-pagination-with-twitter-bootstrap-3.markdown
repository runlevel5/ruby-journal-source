---
layout: post
title: "How to style kaminari pagination with Twitter Bootstrap 3"
date: 2014-04-01 19:04
comments: true
tags: rails, bootstrap
author: "Trung Lê"
---

{{ post.title }}

Kamiari is an awesome that would do all heavy-lifting work if you want to do pagination.
Yet Kaminari's default layout does not fit well with Twitter Bootstrap pagination styling.
In this tutorial, I'll show you how to make Kaminari play well with Bootstrap v3.

<!--more-->

First thing, we need to tell bootstrap to generate template files:

```
rails generate kaminari:views bootstrap
```

which create various fields under `app/views/kaminari`.

Next, we need to edit `app/views/kaminari/_paginator.html.haml` and replace the
content with:

```haml
= paginator.render do
  %ul.pagination.pagination-lg
    = first_page_tag unless current_page.first?
    = prev_page_tag unless current_page.first?
    - each_page do |page|
      - if page.left_outer? || page.right_outer? || page.inside_window?
        = page_tag page
      - elsif !page.was_truncated?
        = gap_tag
    = next_page_tag unless current_page.last?
    = last_page_tag unless current_page.last?
```

What we did above is getting rid of `div.pagination` and adding class
`pagination` to `ul` tag

That's it, now you have Twitter Bootstrap 3 pagination powered by Kaminari!

Keep on learning guys!
