---
layout: post
title: "Introduction to the Python Interpreter, Part 4: It's Dynamic!"
date: 2013-12-03 11:25
comments: false
categories: python python-internals
---

[Edit: A significantly expanded version of this series appears as a chapter in The Architecture of Open Source Applications, volume 4, as [A Python Interpreter Written in Python](//www.aosabook.org/en/500L/a-python-interpreter-written-in-python.html).]

_This is Part 4 in a [series](/blog/categories/python-internals) on the Python interpreter. Read [Part 1](/blog/2013/11/15/introduction-to-the-python-interpreter/), [Part 2](/blog/2013/11/15/introduction-to-the-python-interpreter-2/), and [Part 3](/blog/2013/11/17/introduction-to-the-python-interpreter-3/). If you're enjoying this series, consider applying to [Hacker School](https://www.hackerschool.com/), where I work as a facilitator._

One of the things I was confused about when I started digging into python internals was how python could be "dynamic" if it was also "compiled." Often, in casual coversation, those two words are used as antonyms - there are "dynamic languages,"[^1] like Python, Ruby, and Javascript, and "compiled languages," like C, Java, and Haskell.

Most of the time, when people talk about a "compiled" language, they mean one that compiles down to native x86/ARM/etc instructions[^2] - instructions for an actual machine made of metal. An "interpreted" language either doesn't have any compilation at all[^3], or compiles to an intermediate representation, like bytecode.  Bytecode is instructions for a virtual machine, not a piece of hardware. Python falls into this latter category: the Python compiler's job is to generate bytecode for the Python interpreter.[^4]

The Python interpreter's job is to make sense of the bytecode via the virtual machine, which turns out to be a lot of work. We'll dig in to the virtual machine in Part 5.

So far our discussion of compiling versus interpretation has been abstract. These ideas become more clear with an example.

``` python
>>> def modulus(x, y):
...     return x % y
... 
>>> [ord(b) for b in modulus.func_code.co_code]
[124, 0, 0, 124, 1, 0, 22, 83]
>>> dis.dis(modulus.func_code)
  2           0 LOAD_FAST                0 (x)
              3 LOAD_FAST                1 (y)
              6 BINARY_MODULO       
              7 RETURN_VALUE
```

Here's a function, its bytecode, and its bytecode run through the disassembler. By the time we get the prompt back after the function definition, the `modulus` function has been compiled and a code object generated. That code object will never be modified.

This seems pretty easy to reason about. Unsurprisingly, typing a modulus (`%`) causes the compiler to emit the instruction `BINARY_MODULO`. It looks like this function will be useful if we need to calculate a remainder.

```python
>>> modulus(15,4)
3
```

So far, so good. But what if we don't pass it numbers?

``` python
>>> modulus("hello %s", "world")
'hello world'
```

Uh-oh, what happened there? You've probably seen this before, but it usually looks like this: 

``` python
>>> print "hello %s" % "world"
hello world
```

Somehow, when `BINARY_MODULO` is faced with two strings, it does string interpolation instead of taking a remainder. This situation is a great example of dynamic typing. When the compiler built our code object for `modulus`, it had no idea whether `x` and `y` would be strings, numbers, or something else entirely. It just emitted some instructions: load one name, load another, `BINARY_MODULO` the two objects, and return the result. It's the interpreter's job to figure out what `BINARY_MODULO` actually means.

I'd like to reflect on the depth of our ignorance for a moment. Our function `modulus` can calculate remainders, or it can do string formatting ... what else?  If we define a custom object that responds to `__mod__`, then we can do _anything_.

``` python
>>> class Surprise(object):
...     def __init__(self, num):
...         self.num = num
...     def __mod__(self, other):
...         return self.num + other.num
... 
>>> seven = Surprise(7)
>>> four = Surprise(4)
>>> modulus(seven, four)
11
>>> modulus(7,4)
3
>>> modulus("hello %s", "world")
'hello world'
```

The same function `modulus`, with the same bytecode, has wildly different effects when passed different kinds of objects.  It's also possible for `modulus` to raise an error - for example, a `TypeError` if we called it on objects that didn't implement `__mod__`. Heck, we could even write a custom object that raises a `SystemExit` when `__mod__` is invoked.  Our `__mod__` function could have written to a file, or changed a global variable, or deleted another attribute of the object. We have near-total freedom.

This ignorance is one of the reasons that it's hard to optimize Python: you don't know when you're compiling the code object and generating the bytecode what it's going to end up doing. The compiler has no idea what's going to happen. As Russell Power and Alex Rubinsteyn wrote in ["How fast can we make interpreted Python?"](http://arxiv.org/pdf/1306.6047v2.pdf), "In the general absence of type information, almost every instruction must be treated as INVOKE_ARBITRARY_METHOD."

While a general definition of "compiling" and "interpreting" can be difficult to nail down, in the context of Python it's fairly straightforward. Compiling is generating the code objects, including the bytecode. Interpreting is making sense of the bytecode in order to actually make things happen. One of the ways in which Python is "dynamic" is that the same bytecode doesn't always have the same effect. More generally, in Python the compiler does relatively little work, and the intrepreter relatively more.

In Part 5, we'll look at the actual virtual machine and interpreter.

[^1]: You sometimes hear "interpreted language" instead of "dynamic language," which is usually, mostly, synonymous.
[^2]: Thanks to David Nolen for this definition. The lines between "parsing," "compiling," and "interpreting" are not always clear. 
[^3]: Some languages that are usually not compiled at all include R, Scheme, and binary, depending on the implementation and your definition of "compile."
[^4]: As always in this series, I'm talking about CPython and Python 2.7, although most of this content is true across implementations.
