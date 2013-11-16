---
layout: post
title: "Introduction to the Python Interpreter, Part 1: Function Objects"
date: 2013-11-15 18:50
comments: true
categories: python, python-internals
---

Over the last three months, I've spent a lot of time working with Ned Batchelder on [byterun](https://github.com/nedbat/byterun), a python bytecode interpreter written in python.  Working on byterun has been tremendously educational and a lot of fun for me.  At the end of this series, I'm going to attempt to convince you that it would be interesting and fun for you to play with byterun, too.  But before we do that, we need a bit of a warm-up: an overview of how python's internals work, so that we can understand what an interpreter is, what it does, and what it doesn't do.

This series assumes that you're in a similar position to where I was three months ago: you know python, but you don't know anything about the internals.

One quick note: I'm going to work in and talk about Python 2.7 in this post.  The interpreter in Python 3 is mostly pretty similar.  There are also some syntax and naming differences, which I'm going to ignore, but everything we do here is possible in Python 3 as well.

### How does it python?
We'll start out with a really (really) high-level view of python's internals.  What happens when you execute a line of code in your python REPL?

```python
~ $ python
Python 2.7.2 (default, Jun 20 2012, 16:23:33) 
[GCC 4.2.1 Compatible Apple Clang 4.0 (tags/Apple/clang-418.0.60)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> a = "hello"
```

There are four steps that python takes when you hit return: lexing, parsing, compiling, and interpreting. Lexing is breaking the line of code you just typed into tokens.  The parser takes those tokens and generates a structure that shows their relationship to each other (in this case, an Abstract Syntax Tree).  The compiler then takes the AST and turns it into one (or more) code objects.  Finally, the interpreter takes each code object executes the code it represents.

I'm not going to talk about lexing, parsing, or compiling at all today, mainly because I don't know anything about these steps yet.  Instead, we'll suppose that all that went just fine, and we'll have a proper python code object for the interpreter to interpret.  

Before we get to code objects, let me clear up some common confusion.  In this series, we're going to talk about function objects, code objects, and bytecode. They're all different things.  Let's start with function objects.  We don't really have to understand function objects to get to the interpreter, but I want to stress that function objects and code objects are not the same - and besides, function objects are cool.

### Function objects
You might have heard of "function objects."  These are the things people are talking about when they say things like "Functions are first-class objects," or "Python has first-class functions."  Let's take a look at one.


``` python
>>> def foo(a):
...     x = 3
...     return x + a
... 
>>> foo
<function foo at 0x107ef7aa0>
```

"Functions are first-class objects," means that function are objects, like a list is an object or an instance of `MyObject` is an object.  Since `foo` is an object, we can talk about it without invoking it (that is, there's a difference between `foo` and `foo()`).  We can pass `foo` into another function as an argument, or we could bind it to a new name (`other_function = foo`). With first-class functions, all sorts of possibilities are open to us!

In Part 2, we'll dive down a level and look at the code object.
