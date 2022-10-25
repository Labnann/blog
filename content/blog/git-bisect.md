---
title: git bisect - Which commit caused this bug?
date: 2022-09-24
description: git has this amazing tool that lets you find out which part of code might be causing a bug.

 
---


`git` has this amazing tool that lets you find out which part of code might be causing a bug.

When we use `git`, to manage the source, we are keeping track of

- When a patch of code has been added

- Who added which part of code

- Which part of the code was added at the same time


And thus we get a time-machine to observe what happened when. So in theory, you should be able to binary-search and find out which commit caused a bug. `git-bisect` is a feature of `git`, which lets us do exactly that.

Here is a simple guide on we do it:

Let's say we have a bug in one of our `git` repo and we want to find out which commit introduced it. So, lets cd into our `git` repository and run:

```
$ git bisect start
```
This will start the bisection process. Now run:
```
$ git bisect bad
```
To mark the current commit as 'bad'.

Now let's mark a certain commit as a good commit. Choose a commit when the bug was not there, aka the good commit.
```
$ git bisect good <commit -sha/id/tag when the bug was non-existent>
```
We have specified one good and one bad commit. Now `git` will automatically select and checkout a commit in the middle of that range. What we need to do now is:

Test if the bug exists. If the bug exists, run

```
$ git bisect bad
```
Otherwise run,
```
$ git bisect good
```
And `git` will check out another commit for you to test. Repeat this process and eventually `git` will output the culprit commit id and the commit message, as well as the person who committed this bug ;).


Extra tip:

Let's say we messed up trying to do this process, or just don't wanna bisect anymore. In that case we can run,

```
$ git bisect reset
```
With this, `git` will checkout the commit before we ran "git bisect start" and abort the bisection process.

Here is the official docs for git bisection if you want to know more:

https://git-scm.com/docs/git-bisect

That's all for today. Happy hunting!

