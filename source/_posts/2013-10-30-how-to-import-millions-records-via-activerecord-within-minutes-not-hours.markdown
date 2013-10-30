---
layout: post
title: "How to import millions records via ActiveRecord within minutes not hours"
date: 2013-10-30 17:23
comments: true
categories:
tags: rails activerecord unix bash
author: "Trung Lê"
---

{{ post.title }}

In today tutorial, I'll show you how to optimise a ActiveRecord import script by 300%. My solution is better than other solution as it doesn't use any SQL hack, thus you can retain the integrity with the data by running it through ActiveRecord normally.

<!--more-->

At work, I am assigned a task to import millions rows of records from a 300MB CSV file into Rails app. The rake task takes in FILE and process it with ActiveRecord.

```
FILE=/tmp/big_file.csv rake data:import
```

And soon I bumped into performance issue because ActiveRecord could not release garbage effectively. The script tooks *~2hrs* to complete. This is unacceptable to my standard.

There are various workarounds on the net such as using `ar_import` gem which uses SQL INSERT. However I do not like these SQL solutions as there are so many callbacks with my models and data integrity is very important. So I come up with an alternative solution:

* Split the big_file.csv into smaller files
* Loop through these smaller chunks and recursively run rake task on each

So now you wonder how the above solution works better. It is because now we run many small processes in which Rails won't have to deal much with big GC. Once a process is completed, memory will be instantly released. Now, let's code this up using shell script, I chose bash as example (please adapt to fit your purpose):

```bash
#! /bin/bash

NUMBER_OF_SPLIT_LINES=50000
SPLIT_FILE_PREFIX='small_'
BIG_FILE_PATH=/tmp/big_file.csv
SPLIT_FILES=/tmp/$SPLIT_FILE_PREFIX*

temp_home () {
  cd /tmp
}

rails_app_home () {
  cd /your_app
}

split_big_csv_into_small_chunks () {
  echo "Split $BIG_FILE_PATH file into small chunks with size $NUMBER_OF_SPLIT_LINES lines..."
  temp_home && split -l $NUMBER_OF_SPLIT_LINES $BIG_FILE_PATH $SPLIT_FILE_PREFIX
}

process_split_files () {
  for f in $SPLIT_FILES
  do
    echo "Processing $f file..."
    rails_app_home && FILE=$f rake data:import
  done
}

split_big_csv_into_small_chunks
process_split_files
```

Let's go through the above script. I use `split` UNIX command to split the big file into many smaller files, each with 50000 lines. Then I loop through these small files and parse it to rake task to run.

Now, how many minutes you think our bash script would take to finish? It is *3 mintutes* - no kidding! This is a massive gain compared to 2hrs.

Ruby/Rails are not the best for dealing with huge chunk of memory. So before deciding to try some SQL way, you can be pragmatic and abuse UNIX by spawning as many processes as your computer can handle and you'll be surprised on how much gain you would achieve. Good luck!