---
layout: post
title: "Debugging with pstree"
date: 2014-09-21 17:46
comments: false
categories: ruby systems
---

I hit a very fun bug yesterday while trying to run a script that sends emails to certain subsets of Hacker Schoolers. When I tried to test the script locally, I discovered that one of the tables of the database, `Batch`, was missing from my local version.  After briefly panicking and making sure that the actual site was still up, I could dig in.

It turns out that my local version of psql was way out of date, and as of a few days ago we'd started using a data type that wasn't present in my old version. Because of that, creating that particular table failed when I pulled from the production database the night before. The failure was logged, but the output is so verbose that I didn't notice the problem. Both the diagnosis and the fix here were easy - I went back and read the logs, googled the data type that was raising an error, and then upgraded Postgres.app and psql. That's when the real trouble started.

The new version of Postgres.app was placed in a new spot on the $PATH, as you'd expect, and the upgrade prompted me to change my `.bashrc`, which I did. But the rake tasks we use to manage local copies of the database errored out with this message:
~~~~.txt
$ pg_restore --verbose --clean --no-acl --no-owner -h localhost -U `whoami` -d hackerschool latest.dump
sh: pg_restore: command not found
~~~~

This was pretty clearly a $PATH problem. I tried the usual things first, like sourcing my `.bashrc` in the terminal I was using, closing the terminal and opening a new one, etc. None of that worked.

One thing that jumped out to me was the `sh` in the error message. That was an indicator that rake wasn't using bash as a shell - it was using `sh` - which means my `.bashrc` wasn't setting the environment. Reading the rake task showed that it was a thin wrapper around lots of system calls via Ruby's `system("cmd here")`. I added the line `system("echo $PATH")` and verified that the new location of `pg_restore` wasn't in it.

At this point I found I had lots of questions about the execution context of the rake task. Since I was making system calls and could easily edit the rakefile, I added in the line `system("sh")` to drop me into a shell mid-execution. This turned out to be an efficient way to figure out what was going on (and made me feel like a badass hacker).

From within in that shell, I could do `$$` to get that process's PID, then repeatedly do `ps -ef | grep [PID]` to find the parent process.

~~~~.sh
sh-3.2$ $$
sh: 34652: command not found
sh-3.2$ ps -ef | grep 34652
  501 34652 34639   0  4:18PM ??         0:00.04 sh
    0 34881 34652   0  4:26PM ??         0:00.01 ps -ef
  501 34882 34652   0  4:26PM ??         0:00.01 grep 34652
sh-3.2$ ps -ef | grep 34639
  501 34639  2914   0  4:18PM ??         0:00.41 rake db:drop db:create db:pull
  501 34652 34639   0  4:18PM ??         0:00.04 sh
  501 34885 34652   0  4:28PM ??         0:00.00 grep 34639
sh-3.2$ ps -ef | grep 2914
  501  2914  2913   0 10Sep14 ??        27:11.72 spring app    | hackerschool | started 244 hours ago | development mode
  501 34639  2914   0  4:18PM ??         0:00.41 rake db:drop db:create db:pull
  501 34889 34652   0  4:28PM ??         0:00.01 grep 2914
sh-3.2$ ps -ef | grep 2913
  501  2914  2913   0 10Sep14 ??        27:11.98 spring app    | hackerschool | started 244 hours ago | development mode
  501 34892 34652   0  4:29PM ??         0:00.00 grep 2913
  501  2913     1   0 10Sep14 ttys001    0:00.94 spring server | hackerschool | started 244 hours ago
~~~~

Aha! The parent process of the rake task I was running is the spring server, which starts on boot - several days ago, at the time - and doesn't have the new and updated $PATH information.[^1] A kick to the spring server (with `kill 2913`) forced the server process to restart with the new environment.

It turns out there's a handy utility called `pstree`[^2] (brew installable) to visualize the tree of processes. This would have saved me a couple of steps of grepping. For example:

~~~~.txt
hackerschool [master] $ pstree -p 35351
-+= 00001 root /sbin/launchd
 \-+- 35129 afk spring server | hackerschool | started 25 hours ago
   \-+= 35130 afk spring app    | hackerschool | started 25 hours ago | development mode
     \--- 35351 afk rails_console
~~~~

This bug and some related ones have gotten me more interested in operating systems, and I've started reading the book [Operating Systems: Three Easy Pieces](http://pages.cs.wisc.edu/~remzi/OSTEP/). I'm only a few chapters in, but so far it's readable, clear, and entertaining. I look forward to building up my mental model of processes and environments as I keep reading it.

[^1]: We can tell it (probably) starts on boot because the parent process ID is 1. This means that rebooting my computer would have solved the problem.
[^2]: Thanks to [Paul Tag](//twitter.com/paultag) for the pointer to `pstree`.
