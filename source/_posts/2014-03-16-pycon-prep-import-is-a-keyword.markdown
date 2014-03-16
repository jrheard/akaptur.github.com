---
layout: post
title: "PyCon prep: import is a keyword"
date: 2014-03-16 16:08
comments: true
categories:
---

Last week I had a ton of fun working with [Amy Hanlon](https://twitter.com/amygdalama) on her Harry Potter themed fork of Python, called Nagini.  Nagini is full of magic and surprises. It implements the things you'd hope for out of a Harry Potter Python, like making `quit` into `avada_kedavra`, and many analogous jokes.

Amy also had the idea to replace `import` with `accio`! Replacing `import` is a much harder problem than renaming a builtin.  Python doesn't prevent you from overwriting builtins, whereas to change keywords you have to edit the grammar and recompile Python. You should [go read Amy's post](http://mathamy.com/import-accio-bootstrapping-python-grammar.html) on making this work.

This brings us to an interesting question: why *is* `import` a keyword, anyway? There's a function, `__import__`, that does (mostly) the same thing:

``` python
>>> __import__('random')
<module 'random' from '/path/to/random.pyc'>
```

The function form requires the programmer to assign the return value - the module - to a name, but once we've done that it works just like a normal module:

``` python
>>> random = __import__('random')
>>> random.random()
0.32574174955668145
```

The `__import__` function can handle all the forms of import, including `from foo import bar` and `from baz import *` (although it never modifies the calling namespace).  There's no technical reason why `__import__` couldn't be the regular way to do imports.[^1]

As far as I can tell, the main argument against an `import` function is aesthetic. Compare:

``` python
import foo
from foo import bar
import longmodulename as short

foo = __import__('foo')
bar = __import__('random').bar
short = __import__('longmodulename')
```

The first way certainly feels much cleaner and more readable.[^2]

Part of my goal in my [upcoming PyCon talk](https://us.pycon.org/2014/schedule/presentation/229/) is to invite Pythonistas to consider decisions they might not have thought about before. `import` is a great vehicle for this, because everyone learns it very early on in their programming development, but most people don't ever think about it again. Here's another variation on that theme: `import` doesn't have to be a keyword!

[^1]: I think all keywords *could* be expressed as functions, except those used for flow control (which I loosely define as keywords that generate any `JUMP` instructions when compiled). For example, between Python 2 and 3, two keywords did become functions - `print` and `exec`.

[^2]: I realize this is a slightly circular argument - if the function strategy were the regular way to import, it probably wouldn't be so ugly.
