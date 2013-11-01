---
layout: post
title: "A Python puzzle"
date: 2013-10-29 19:40
comments: false
categories: 
---

A couple of Hacker Schoolers were discussing an interesting corner of python today.  We discovered a nice bit of trivia: there exist three lines of python code that display the following behavior:

``` python
>>> LINE_A
>>> LINE_B
>>> LINE_C
False
>>> LINE_A; LINE_B; LINE_C
True
>>> def my_function():
...     LINE_A
...     LINE_B
...     LINE_C
>>> my_function()
True
```

What are the lines?

Some ground rules:

* Introspection of any kind is cheating (e.g. noting the line number).
* No dunder (`__foo__`) methods allowed.
* Each line is a valid python expression.
* You can't rely on order: while the lines will always execute A -> B -> C, a complete solution behaves identically if e.g. the semicolon version happens before the separate-line version.
* No cheating with the function: e.g. you can't add a `return` unless you add it everywhere.
* _Edit: And nothing stateful._

For bonus points, code golf!  My solution to this is ~~14~~ 19 characters long, not counting whitespace.