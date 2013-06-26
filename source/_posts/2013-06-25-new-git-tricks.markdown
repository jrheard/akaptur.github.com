---
layout: post
title: "New git tricks"
date: 2013-06-25 14:14
comments: false
categories: 
---

I picked up a couple of new tricks at Hacker School's intermediate/advanced Git seminar, led by Peter Bell. 
```
$ git checkout -
```
This checks out whatever you had checked out last.  Thanks, <a href="https://github.com/nnja">Nina</a>!

Draw a graph of the commit history:

```
git-advanced [master]\ $ git log --graph --oneline --all
*   b2da179 Merge branch 'test-graph'
|\  
| * d71e4fc testing graph thing
* | 8535311 still testing graph
|/  
* b90b105 Added contact form
* b36ba51 added gitignore to ignore generated log files
* 70461dc added about
* c57f995 added homepage
```

You can alias this with `git config --global alias.plog "log --graph --oneline --all" `.  (Looking at this graph may motivate you to start using `git merge --no-ff`.)  You can substitute whatever you want for `plog` above.

If you get really screwed up, it's reflog time!  (This is pronounced ref-log, not re-flog as I'd been reading it.  On the other hand, as my colleague <a href="https://github.com/happy4crazy">Alan</a> notes, "You damn well ought to feel penitent if you're wedged enough to need the reflog.")  Here, we're using `git reset` to undo the last commit.  Even though the commit doesn't appear in `git log` anymore, there's still a references to it in the reflog.

```
git-advanced [master %]\ $ git add tester.css
git-advanced [master +]\ $ git commit -m "add some file"
git-advanced [master]\ $ git reset HEAD~1
git-advanced [master %]\ $ git add tester.css
git-advanced [master +]\ $ git commit -m "redoing some file"

git-advanced [master]\ $ git log --oneline
8039ec5 redoing some file
b2da179 Merge branch 'test-graph'
8535311 still testing graph
d71e4fc testing graph thing

git-advanced [master]\ $ git reflog
8039ec5 HEAD@{0}: commit: redoing some file
b2da179 HEAD@{1}: reset: moving to HEAD~1
848874d HEAD@{2}: commit: add some file
```

It turns out that there is a better way to get a commit sha than running `git log` and copying and pasting the relevant shas.  You can use `rev-parse` instead.

```
git-advanced [master]\ $ git rev-parse HEAD~2
0fafe464f4b7d524c8ba707ea2115950d6dd4bdf
git-advanced [master]\ $ git rev-parse master
8039ec584485540f07aac1209389022f627bc8c5
```

Then you can inline with backticks!
```
git-advanced [master]\ $ git diff `git rev-parse HEAD~2` `git rev-parse HEAD`
```

Now go forth and git in peace.

