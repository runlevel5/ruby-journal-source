---
layout: post
title: "Genereate gems dependency graph with Bundler"
date: 2014-03-18 13:34
comments: true
author: "Trung Lê"
---

{{ post.title }}

One of the features I like about RubyMine is the gems dependency graph.
This graps shows you all explicit/implicit dependencies of gems defined in
your Gemfile. In this short tutorial, I'll show how to generate this graph
with Bundler.

<!--more-->

You'll need graphviz installed first. On my Mac, I install it with Homebrew:

```
brew install graphviz
```

You could find packages for your flavor on the Internet if you are on Linux.

In order to generate graph, you need to change directory to the app folder
where the Gemfile file resides then use command:

```
bundle viz
# Resolving dependencies...
# /app_path/gem_graph.png
```

If there is no errors, you should see the image file path in the output.

Open it up and you'll see something like this for a stock Rails 4 app:

{% img left /images/2014-03-18-genereate-gems-dependency-graph-with-bundler/gem_graph.png %}

That's it for today folks! Keep on learning!