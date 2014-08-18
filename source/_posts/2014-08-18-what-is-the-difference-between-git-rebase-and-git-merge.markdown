---
layout: post
title: "What is the difference between git rebase and git merge"
date: 2014-08-18 13:52
comments: true
tags: git
author: "Trung Lê"
---

{{ post.title }}

There are many people who asked me about the differences between `git rebase` vs `git merge`.
Read on if you want to know :-)

<!--more-->


Here is my simple explanation with real examples:

1) Create an empty project:

```
mkdir demo
cd demo
git init .
```

2) Create first file

```
touch README.md
git add README.md
git commit -m 'Add README.md'
```

3) Create 2 separate branches from master branch, ie 1 for rebase testing and other for merge testing

```
git checkout master -b rebase-test
git checkout master -b merge-test
```

double check and you shall see 2 new branches:

```
git checkout master
git branch
* master
  merge-test
  rebase-test
```

4) Now let's make some unique changes in our new branches

```
git checkout rebase-test
touch hello.txt
git add hello.txt
git commit -m 'A hello from rebase branch'
```

```
git checkout merge-test
touch bye.txt
git add bye.txt
git commit -m 'A goodbye from merge branch'
```


5) Okay, now let's go back to our master branch and change
our README.md

```
git checkout master
echo "WTF" >> README.md
git add README.md
git commit -m 'WTF README'
```

6) Now we have a new changes in master and we need to rebase
our 2 branches to be up-to-date with changes made in master.
The wording 'rebase' here does not mean `git rebase`, it means
to sync feature branch with master branch

```
git checkout rebase-test
git rebase master

git checkout merge-test
git merge master
# accept the default new commit message and save
```

7) Let's compare the log tree:

```
git checkout rebase-test
git log --graph --pretty=oneline --abbrev-commit
* 933dae8 A hello from rebase branch
* 8142960 WTF README
* 0497b20 Add README
```

```
git checkout merge-test
git log --graph --pretty=oneline --abbrev-commit
*   1af5ec5 Merge branch 'master' into merge-test
|\
| * 8142960 WTF README
* | 278a115 A goodbye from merge branch
|/
* 0497b20 Add README
```

The commit that is at the top of the commit is the latest commit.

Paying close attention to log tree of `rebase-test`, we could see that
commit `933dae8 A hello from rebase branch` is pushed to the top, higher
than the master (which is created later) `8142960 WTF README`.

Compared to the log tree of `merge-test`, we can clearly see that it
is the other way around, that is the master commit `8142960 WTF README`
is placed to the top and higher than the commit `278a115 A goodbye from merge branch`.
Plus, an additional commit (or what I called meta-commit) `1af5ec5 Merge branch 'master' into merge-test`
is also created.

Okay, now you might ask WTF? Why people come up with 2 different ways
to do rebase? Isn't the world too complicated already? Well, the answer
lies in how you want to rewrite history (Every crazy men who wants to
take on the world do). You use `git rebase` if you want your changes in
the feature branch to always the latest. And you use `git merge` if you
want to reflect the true ordering of commits.

For mosts of cases, I use `git merge` because of following cons of `git rebase`:

* Cannot push to remote feature branch because the history of local and remote is mistmached.
And the only way to push it to remote branch is to use `git push --force` or being
explictily `git push origin <name-of-branch> --force` for the sake of avoiding pushing
into wrong branch. And this could cause lots of issue if you. (I did, so I know)
* Recursive pain and pain and pain when resolving conflicts

The only cons that I know about `git merge` is that you have a long list of many
commits which could be hard on eyes. However I find it is not really an issue because
the log tree show the merged trees very clearly.

I guess that's it for today folks. See you in another how to for dummies from me!





