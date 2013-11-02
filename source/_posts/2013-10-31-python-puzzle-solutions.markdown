---
layout: post
title: "Python Puzzle Solutions"
date: 2013-10-31 10:47
comments: true
categories: 
---

I really enjoyed seeing all the clever solutions to the [python puzzle I posted](http://akaptur.github.io/blog/2013/10/29/a-python-puzzle/).  You're all very creative!  Here's a discussion of the solutions I've seen, plus some clarifications.  All spoilers are below the fold.

First, clarifications.  (These weren't always clear in the problem statement, particularly if you got the problem off of twitter, so award yourself full marks as desired.)

#### Order doesn't matter
"Order doesn't matter" means that the three-line version *always* returns `False`, and the semicolon version *always* returns `True`.

#### You control only the contents of the lines
Several people, including [Pepijn De Vos](https://twitter.com/pepijndevos), [David Wolever](https://twitter.com/wolever), and [diarmuidbourke](https://twitter.com/diarmuidbourke) suggested something like the following:

``` python
>>> """a; b; c""" == 'a; b; c'
True
>>> """a
... b
... c""" == 'a; b; c'
False
```

I'm being pedantic here, but I rule this cheating, since (a) each line has to be a valid python expression or statement, and a multi-line string literal is only one expression, and (b) the string `"""a; b; c"""` is not the same as the string `"""a\nb\nc"""`.

Solutions appear below the fold.

<!-- more -->


###Solutions!
#### Jessica McKellar
[Jessica](https://twitter.com/jessicamckellar) suggests the following solution:

``` python
>>> global a
>>> a = a + "a" if "a" in globals() else ""
>>> print(bool(len(a) % 3))
False
>>> global a; a = a + "a" if "a" in globals() else ""; print(bool(len(a) % 3))
True
>>> def my_function():
...     global a
...     a = a + "a" if "a" in globals() else ""
...     print(bool(len(a) % 3))
... 
>>> my_function()
True
``` 
However, Jessica's solution ~~fails the "order doesn't matter" test, and it~~ is stateful:

``` python
>>> global a; a = a + "a" if "a" in globals() else ""; print(bool(len(a) % 3))
False
>>> global a; a = a + "a" if "a" in globals() else ""; print(bool(len(a) % 3))
True
```

_Edit: As Jessica points out, I'm wrong here: her solution does pass the order test.  She also notes that the restriction against state wasn't present in the blog post (and she didn't see the [original tweet](https://twitter.com/akaptur/status/395252265117687808)).  Full credit to Jessica, then!_

#### Javier Novoa Cataño
[Javier](https://twitter.com/JaviStitch) suggests a solution for Python 2 that fails the order test:

``` python
>>> True = not True
>>> a = True
>>> print a
True
>>> True = not True; a = True; print a
False
```
Again, this depends on order and is stateful.

```
>>> True = not True; a = True; print a
False
>>> True = not True; a = True; print a
True
```

#### Alex Gaynor
[Alex](https://twitter.com/alex_gaynor) suggests using `sys._getframe(0)`.  I might quibble that `sys._getframe` constitutes introspection, but Alex is on to something.  He notices that each line of code executed in the interpreter gets its own frame.  (_foreshadowing_)  

Alex gets full credit because I just discovered that the behavior underlying the original puzzle works in CPython, but is different in PyPy.  Sorry, Alex!

_Edit: Never mind - there is a simpler solution in PyPy. See below._

``` python
>>> a = sys._getframe(0)
>>> b = sys._getframe(0)
>>> a == b
False
>>>
>>> a = sys._getframe(0);b = sys._getframe(0);a == b
True
>>>
>>> def my_function():
...  a = sys._getframe(0)
...  b = sys._getframe(0)
...  return a == b
...
>>> my_function()
True
```

#### Anton Dubrau
For sheer inventiveness and creativity I have to hand it to [Anton](https://twitter.com/ant6n), working in the iPython REPL:

``` python
In [145]: for n in range(-1,1):x=n or x-1;x+=2;sys.stdout.write("" if x == 1 else ("False" if x == 0 else "True"))
True
In [142]: for n in range(-1,1):x=n or x-1;
In [143]: x+=2;
In [144]: sys.stdout.write("" if x == 1 else ("False" if x == 0 else "True"))
False
```

He's exploiting the fact that when executed on a single line, all three lines are considered part of the `for` loop. When the lines are broken up, only the `x=n or x-1` part belongs to the loop.  (This pretty much works in the standard CPython REPL too, but you have to throw in an extra line break.)

#### Alexey Bezhan
[Alexey](https://twitter.com/allait) was the only person outside of Hacker School to hit on my solution.

``` python
>>> a = 257
>>> b = 257
>>> a is b
False
>>> a = 257; b = 257; a is b
True
```
Nicely done, Alexey!

(For the pedantic among us, like me: this was the version I had in mind when I described my solution as 14 non-whitespace characters. Of course, to make the function version work too, we have to add `print` to the last line, taking us up to 19.  I've edited the original post.)

So what's going on here, and why is this interesting?

## Analyzing this solution
I described Alex Gaynor's solution as foreshadowing earlier.  Alex appears to have started from a good question: what's different about executing three lines of code versus executing three statements as one line of code?  

One thing to note is that each time you get an interpreter prompt back (`>>>`), your python has compiled and executed some code.  Every prompt corresponds to at least one code object.

How can we use the difference in code objects to generate this behavior? One way is to use constants: in CPython, two instances of the same number (larger than 256) are different objects. This explains why `a is b` returns `False` in the REPL (in the extended version, without semicolons).  In the semicolon version, all three statements are part of one code object.  We're giving the CPython compiler a chance to optimize.  It notices that we have two constants with the same value, and only hangs on to one copy.

Let's take a look.  It's easy to get a handle on the code object corresponding to a function - use `f.func_code` in Python 2, and `f.__code__` in Python 3.  Getting a handle on a code object corresponding to a line in the REPL is a little bit trickier, but we can do it if we compile them by hand.  In Python 2.x:

``` python
>>> code_obj_a = compile("a = 257", '', "exec")
>>> code_obj_a.co_consts
(257, None)
>>> code_obj_b = compile("b = 257", '', "exec")
>>> code_obj_b.co_consts
(257, None)
>>> code_obj_both = compile("a = 257; b = 257", '', "exec")
>>> code_obj_both.co_consts
(257, None)

>>> def f():
...     a = 257
...     b = 257
...     print a is b
... 
>>> f()
True
>>> f.func_code
<code object f at 0x107133cb0, file "<stdin>", line 1>
>>> f.func_code.co_consts
(None, 257)
```

The compiler's being a little smart here, and we only get one occurence of the constant `257` per code object.  When we execute the assignment statements on two different lines, we get two different code objects with two different instances of the integer `257`. This explains why `is`, which is comparing object identity, sees one as the same and the others as different.

This came up originally at Hacker School when [Steve Katz](https://github.com/phsteve) stumbled across that thing with small integers and `is` versus `==` in python.  (If you're not familiar with it, you can read [many questions on Stack Overflow about it](http://stackoverflow.com/questions/306313/python-is-operator-behaves-unexpectedly-with-integers).  As a general rule, don't use `is` for integer comparisons.)  Steve went on to notice that the behavior changes when run from a script rather than in the REPL.

I didn't realize when posing this problem that it was an implementation detail of CPython, and [PyPy behaves differently](http://pypy.readthedocs.org/en/latest/cpython_differences.html#object-identity-of-primitive-values-is-and-id) (and arguably correctly). That's why Alex Gaynor had to hook into the frame object.

### _Edit: Late submissions_
More cleverness in my twitter feed!

[Ned Batchelder](https://twitter.com/nedbat) suggests using `-9` instead of `257` to shave off a few characters.

#### Zev Benjamin
[Zev](https://github.com/zbenjamin) came up with my favorite so far, in 14 characters *including* the print.  ~~He exploits the intricacies of floating-point arithmetic~~ 

``` python
>>> x = .1
>>> print x is .1
False
>>> x = .1; print x is .1
True
```

If you've never dug around with floating-point math, do yourself a favor: it's really interesting stuff.  The Julia language suggests some good [background reading](http://docs.julialang.org/en/latest/manual/integers-and-floating-point-numbers/#background-and-references).

_Edit 2: Zev emails that floating-point precision isn't involved here. Instead, he's exploiting the fact that floating-point numbers - even those that can be precisely represented - are not interned.  We'd get the same results using `1.` instead of `.1`. This suggests a broader point: many solutions here could have used `a = thing; a is thing`, omitting LINE_B._

#### Nick Olson-Harris
[Nick](https://twitter.com/TheNyktos) has a fix to the PyPy integer handling: use a string instead of an int.

``` python
>>>> a = "f"
>>>> b = "f"
>>>> a is b
False
>>>> a = "f"; b = "f"; a is b
True
```

### Thanks, everyone!
This was fun!  If I missed your solution and you want it to be included, [ping me on twitter](https://twitter.com/akaptur).