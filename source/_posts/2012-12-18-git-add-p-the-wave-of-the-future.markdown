---
layout: post
title: "Git add -p: The Wave of the Future"
date: 2012-12-18 16:12
comments: false
categories: git
published: false
---

I recently started using git's patch mode via `git add -p`. Patch mode makes my commits more granular, which means my commit messages are more accurate.  Better still, patch mode allows you to abstract away from adding "files" - you're adding changes to be committed.  This is a closer mental model to what git is actually doing.  I'm finding it particularly useful for code review: make several changes in a friend's program, group the changes by concept, and commit one concept at a time. 

That's the why - here's the how.

`git add -p` launches what git calls the "interactive hunk selector."  This leads me to my other favorite reason to use -p: besides being useful, all the docs sound like an episode of The Bachelorette.  "Once you have decided the fate of all the hunks, if there is any hunk that is chosen, the index is updated with the selected hunks." And if not, our bachelorette takes the million dollars!

In practice, I only end up using three of the interactive hunk selector options: [y]es, [n]o, and [s]plit.  Split takes the current hunk and divides it into smaller hunks, then presents them for selection one at a time.  (If you're attempting to continue the Bachelorette metaphor, now is a good time to stop.)

Let's look at an example.  Suppose I'm working on a function that recursively calculates the sum of element of a list.  My first draft looks like this:
``` [python]
def sum(list_in):
    """ Returns the sum of elements of a list."""
    if list_in == []:
        return 0
    else:
        return list_in.pop() + sum(list_in)
```

Not bad, but it's a little verbose.  Let's take advantage of the fact that an empty list is falsey.  And, oops, we're overwriting the built-in `sum` function in python - we probably don't want to do that. 
``` [python]
def summer(list_in):
    """ Returns the sum of elements of a list."""
    if not list_in:
        return 0
    else:
        return list_in.pop() + summer(list_in)
```
Ok, these are two different thoughts, but we didn't commit in between.  Interactive mode to the rescue - let's stage those hunks.
```
sum [master]\ ⚲git add -p
diff --git a/sum.py b/sum.py
index fdccd3f..6d000f7 100644
--- a/sum.py
+++ b/sum.py
@@ -1,6 +1,6 @@
-def sum(list_in):
+def summer(list_in):
     """ Returns the sum of elements of a list."""
-    if list_in == []:
+    if not list_in:
         return 0
     else:
-        return list_in.pop() + sum(list_in)
+        return list_in.pop() + summer(list_in)
Stage this hunk [y,n,q,a,d,/,s,e,?]? 
```
Our changes aren't in the right hunks, so we'll pick `s` to split them.  
```
Stage this hunk [y,n,q,a,d,/,s,e,?]? s
Split into 3 hunks.
@@ -1,2 +1,2 @@
-def sum(list_in):
+def summer(list_in):
     """ Returns the sum of elements of a list."""
Stage this hunk [y,n,q,a,d,/,j,J,g,e,?]? 
```
Let's deal with the naming issue first.  `y`.  Next up:
```
@@ -2,4 +2,4 @@
     """ Returns the sum of elements of a list."""
-    if list_in == []:
+    if not list_in:
         return 0
     else:
```
This one's different. `n`.
```
Stage this hunk [y,n,q,a,d,/,K,j,J,g,e,?]? n
@@ -4,3 +4,3 @@
         return 0
     else:
-        return list_in.pop() + sum(list_in)
+        return list_in.pop() + summer(list_in)
```
Back to the naming issue.  `y`.
```
sum [master *+]\ ⚲git commit -m "fix naming to not conflict with builtin"
[master caaa300] fix naming to not conflict with builtin
 1 file changed, 2 insertions(+), 2 deletions(-)
 ```
 Now we can repeat this process for our other change. 
 ```
 sum [master *]\ ⚲git add -p
diff --git a/sum.py b/sum.py
index 77676b1..6d000f7 100644
--- a/sum.py
+++ b/sum.py
@@ -1,6 +1,6 @@
 def summer(list_in):
     """ Returns the sum of elements of a list."""
-    if list_in == []:
+    if not list_in:
         return 0
     else:
         return list_in.pop() + summer(list_in)
Stage this hunk [y,n,q,a,d,/,e,?]? 
```
This is our only hunk now, so `y`.
```
sum [master +]\ ⚲git commit -m "use falseyness of empty list"
[master 5209728] use falseyness of empty list
 1 file changed, 1 insertion(+), 1 deletion(-)
 ```

Of course, you may not want your commits to be quite *this* granular - but hopefully the example has demonstrated the strength of `git add -p` in creating single-concept commits. 

