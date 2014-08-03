---
layout: post
title: "Getting Started with Python Internals"
date: 2014-08-03 09:17
comments: false
categories:
---

I talk to a lot of people at Hacker School and elsewhere who have been programming Python for some time and want to get a better mental model of what's happening under the hood.  The words "really" or "why" often features in these questions - "What's _really_ happening when I write a list comprehension?" "Why are function calls considered expensive?" If you've seen any of the rest of this blog, you know I love digging around in Python internals, and I'm always happy to share that with others.

## Why do this?

First off, I reject the idea that you _have_ to understand the internals of Python to be a good Python developer. Many of the things you'll learn about Python won't help you write better Python. The "under the hood" construction is specious, too - why stop at Python internals? Do you also need to know C perfectly, and the C compiler, and the assembly, and ...

That said, I think you should dig around in Python - it sometimes will help you write better Python, you'll be more prepared to contribute to Python if you want to, and most importantly, it's often really interesting and fun.

## Setup

Follow the instructions in the [Python dev guide](https://docs.python.org/devguide/setup.html) under "Version Control Setup" and "Getting the Source Code". You now have a Python that you can play with.

## Strategies

#### 1. Naturalism
Peter Seibel has a [great blog post](//www.gigamonkeys.com/code-reading/) about reading code. He thinks that "reading" isn't how most people interact with code - instead, they dissect it. From the post:

>But then it hit me. Code is not literature and we are not readers. Rather, interesting pieces of code are specimens and we are naturalists. So instead of trying to pick out a piece of code and reading it and then discussing it like a bunch of Comp Lit. grad students, I think a better model is for one of us to play the role of a 19th century naturalist returning from a trip to some exotic island to present to the local scientific society a discussion of the crazy beetles they found: “Look at the antenna on this monster! They look incredibly ungainly but the male of the species can use these to kill small frogs in whose carcass the females lay their eggs.”

>The point of such a presentation is to take a piece of code that the presenter has understood deeply and for them to help the audience understand the core ideas by pointing them out amidst the layers of evolutionary detritus (a.k.a. kluges) that are also part of almost all code. One reasonable approach might be to show the real code and then to show a stripped down reimplementation of just the key bits, kind of like a biologist staining a specimen to make various features easier to discern.

#### 2. Science!
I'm a big fan of hypothesis-driven debugging, and that also applies in exploring Python.  I think you _should not_ just sit down and read CPython at random. Instead, enter the codebase with (1) a question and (2) a hypothesis. For each thing you're wondering about, make a guess for how it might be implemented, then try to confirm or refute your guess.

#### 3. Guided tours
Follow a step-by-step guide to changing something in Python. I like [Amy Hanlon's post](//mathamy.com/import-accio-bootstrapping-python-grammar.html) on changing a keyword in Python and [Eli Bendersky's](//eli.thegreenplace.net/2010/06/30/python-internals-adding-a-new-statement-to-python/) on adding a keyword.

#### 4. Reading recommendations.
I don't think you should sit down and read CPython at random, but I do have some suggestions for my favorite modules that are implemented in Python. I think you should read the implementation of

##### 1. `timeit` in `Lib/timeit.py`

##### 2. `namedtuple` in `Lib/collections.py`.

If you have a favorite module implemented in Python, [tweet at me](//twitter.com/akaptur) and I'll add it to this list.

#### 5. Blog & talk
Did you learn something interesting? Write it up and share it, or present at your local meetup group! It's easy to feel like everyone else already knows everything you know, but trust me, they don't.

#### 6. Rewrite
Try to write your own implementation of `timeit` or `namedtuple` before reading the implementation.  Or read a bit of C and rewrite the logic in Python. [Byterun](//github.com/nedbat/byterun) is an example of the latter strategy.

## Tools
I sometimes hesitate to recommend tooling because it's so easy to get stuck on installation problems. If you're having trouble installing something, get assistance (IRC, StackOverflow, a Meetup, etc.)  These problems are challenging to fix if you haven't seen them before, but often straightforward once you know what you're looking for. If you don't believe me, [this thread](//mail.python.org/pipermail/python-dev/2014-February/132313.html) features Guido van Rossum totally misunderstanding a problem he's having with a module that turns out to be related to upgrading to OS X Mavericks. _This stuff is hard._

#### 1. Ack
I'd been using `grep` in the CPython codebase, which was noticeably slow. (It's especially slow when you forget to add the `.` at the end of the command and grep patiently waits on stdin, a mistake I manage to make _all the time_.) I started using [ack](//beyondgrep.com/) a few months ago and really like it.

If you're on a Mac and use homebrew, you can `brew install ack`, which takes only a few seconds. Then do `ack string_youre_looking_for` and you get a nicely-formatted output. I imagine you could get the same result with `grep` if you knew the right options to pass it, but I find ack fast and simple.

Try using `ack` on [the text of an error message](/blog/2014/06/11/of-syntax-warnings-and-symbol-tables/) or [a mysterious constant](//acmonette.com/here-there-be-pydras.html). You may be surprised how often this leads you directly to the relevant source code.

#### 2. timeit
Timing & efficiency questions are a great place to use the "Science!" strategy. You may have a question like "Which is faster, X or Y?" For example, is it faster to do two assignment statements in a row, or do both in one tuple-unpacking assignment? I'm guessing that the tuple-unpacking will take longer because of the unpacking step. Let's find out!

``` text
~ ⚲ python -m timeit "x = 1; y = 2"
10000000 loops, best of 3: 0.0631 usec per loop
~ ⚲ python -m timeit "x, y = 1, 2"
10000000 loops, best of 3: 0.0456 usec per loop
```

I'm wrong! Interesting! I wonder why that is. What if instead of unpacking a tuple, we did ...

A lot of people I talk to like using IPython for timing. IPython is pip-installable, and it usually installs smoothly into a virtual environment. In the IPython REPL, you can use `%timeit` for timing questions. There are also other [magic functions](//ipython.org/ipython-doc/dev/interactive/tutorial.html#magic-functions) available in IPython.

``` python
In [1]: %timeit x = 1; y = 2;
10000000 loops, best of 3: 82.3 ns per loop

In [2]: %timeit x, y = 1, 2
10000000 loops, best of 3: 47.3 ns per loop
```

One caveat on timing stuff - use `timeit` to enhance your understanding, but unless you have real speed problems, you should write code for clarity, not for miniscule speed gains like this one.

#### 3. Disassembling
Python compiles down to bytecode, an intermediate representation of your Python code used by the Python virtual machine.  It's sometimes enlightening and often fun to look at that bytecode using the built-in `dis` module.

``` python
>>> def one():
...     x = 1
...     y = 2
...
>>> def two():
...     x, y = 1, 2
...
>>> import dis
>>> dis.dis(one)
  2           0 LOAD_CONST               1 (1)
              3 STORE_FAST               0 (x)

  3           6 LOAD_CONST               2 (2)
              9 STORE_FAST               1 (y)
             12 LOAD_CONST               0 (None)
             15 RETURN_VALUE
>>> dis.dis(two)
  2           0 LOAD_CONST               3 ((1, 2))
              3 UNPACK_SEQUENCE          2
              6 STORE_FAST               0 (x)
              9 STORE_FAST               1 (y)
             12 LOAD_CONST               0 (None)
             15 RETURN_VALUE
```

The implementation of the various operations are in `Python/ceval.c`.

#### 4. Inspect/cinspect
You can get into the habit of trying to call `inspect` on anything you're curious about to see the source code.

``` python
>>> import inspect
>>> import collections
>>> print inspect.getsource(collections.namedtuple)
def namedtuple(typename, field_names, verbose=False, rename=False):
    """Returns a new subclass of tuple with named fields.
    ...
```

However, `inspect` will only show the source code of things that are implemented in Python, which can be frustrating.

``` python
>>> print inspect.getsource(collections.defaultdict)
Traceback (most recent call last):
   [... snip ...]
IOError: could not find class definition
>>> :(
```

To get around this, [Puneeth Chaganti](//github.com/punchagan) wrote a tool called [cinspect](//github.com/punchagan/cinspect) that extends `inspect` to work reasonably consistently with C-implemented code as well.

#### 5. K&R
I think C is about a hundred times easier to read than it is to write, so I encourage you to read C code even if you don't totally know what's going on. That said, I think an afternoon spent with the first few chapters of [K&R](//www.amazon.com/The-Programming-Language-2nd-Edition/dp/0131103628) would take you pretty far.  [Hacking: The Art of Exploitation](//www.amazon.com/Hacking-The-Art-Exploitation-Edition/dp/1593271441) is another fun, if less direct, way to learn C.

## Get started!

CPython is a huge codebase, and you should expect that building a mental model of it will be a long process. Download the source code now and begin poking around, spending five or ten minutes when you're curious about something. Over time, you'll get faster and more rigorous, and the process will get easier.

Do you have recommended strategies and tools that don't appear here? [Let me know!](//twitter.com/akaptur)
