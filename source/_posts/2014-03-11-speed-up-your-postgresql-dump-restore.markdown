---
layout: post
title: "Speed up your PostgreSQL dump restore"
date: 2014-03-11 16:20
comments: true
tags: postgres
author: "Trung Lê"
---

{{ post.title }}

Restore a DB dump with pg_restore is always a time-consuming process. However if
you are using Postgres 8.4 or newer, you could speed it up easily by having
multiple concurrent process do it for you.

<!--more-->

Version 8.4 introduced the `-j` or `--jobs` parameter for the pg_restore.

This speed up the restore process drastically on a multi-cores machine.

Here is how to use it:

```
pg_restore [connection-options...] -j <number_of_jobs> [other-options...] [filename]
```

Says, you specify `-j 4`, this tells pg_restore to run 4 jobs. Each job
is one process or one thread depending on the OS and uses a separate
connection to the server.

So what is the optimal number? If you have to work this out by yourself,
a good start is number of core minus 2, and then try to bump the number up
while measuring the time taken to run. If the number is too high, it actually
slow down the process due to thrashing.

FYI, restoring a 500MB dump on 4 core with `-j 4` takes 6min instead of 17'.

One more important thing that I should mention is that, this feature only
works with custom and directory archive dump.

That's it for today, folks. Keep on learning!
