---
layout: post
title: "Python Bytecode: Fun with dis"
date: 2013-08-14 16:04
comments: true
categories: 
---

Last week at [Hacker School](https://www.hackerschool.com/) I did a quick presentation on python bytecode and the `dis` module.  The disassembler is a very powerful tool with a gentle learning curve - that is, you can get a fair amount out of it without really knowing much about what's going on.  This post is a quick introduction to how and why you should use it.

### What's bytecode?
Bytecode is the internal representation of a python program in the compiler.  Here, we'll be looking at bytecode from cpython, the default compiler.  If you don't know what compiler you're using, it's probably cpython.

### How do I get bytecode?
You already have it!  Bytecode is what's contained in those .pyc files you see when you import a module.  It's also created on the fly by running any python code.  

### Disassembling
Ok, so you have some bytecode, and you want to understand it.  Let's look at it without using the `dis` module first.

``` python 
>>> def foo():
...     a = 2
...     b = 3
...     return a + b
... 
>>> foo.func_code
<code object foo at 0x106353530, file "<stdin>", line 1>
>>> foo.func_code.co_code
'd\x01\x00}\x00\x00d\x02\x00}\x01\x00|\x00\x00|\x01\x00\x17S'
>>> print [ord(x) for x in foo.func_code.co_code]
[100, 1, 0, 125, 0, 0, 100, 2, 0, 125, 1, 0, 124, 0, 0, 124, 1, 0, 23, 83]
```

Hmm, that was ... not very enlightening.  We can see that we have a bunch of bytes (some printable, others not), but we have no idea what they mean.  

Let's run it through `dis.dis` instead.

``` python
>>> import dis
>>> dis.dis(foo)
  2           0 LOAD_CONST               1 (2)
              3 STORE_FAST               0 (a)

  3           6 LOAD_CONST               2 (3)
              9 STORE_FAST               1 (b)

  4          12 LOAD_FAST                0 (a)
             15 LOAD_FAST                1 (b)
             18 BINARY_ADD          
             19 RETURN_VALUE  
```

`dis` takes each byte, finds the opcode that corresponds to it in `opcodes.py`, and prints it as a nice, readable constant.  If we look at `opcodes.py` we see that `LOAD_CONST` is 100, `STORE_FAST` is 125, etc. `dis` also shows the line numbers on the left and the values or names on the right.  So without ever seeing something like before, we have an idea what's going on: we first load a constant, 2, then somehow store it as `a`.  Then we repeat this with 3 and `b`.  We load `a` and `b` back up, do `BINARY_ADD`, which presumably adds the numbers, and then do `RETURN_VALUE`.

Examining the bytecode can sometimes increase your understanding of python code.  Here are two examples.

### elif
`elif` is identical in bytecode to `else ... if`.  Take a look:

``` python
>>> def flat(num):
...     if num % 3 == 0:
...         print "Fizz"
...     elif num % 5 == 0:
...         print "Buzz"
...     else:
...         print num
>>>
>>> def nested(num):
...     if num % 3 == 0:
...         print "Fizz"
...     else:
...         if num % 5 == 0:
...             print "Buzz"
...         else:
...             print num
```

We've read [PEP 8](http://www.python.org/dev/peps/pep-0008/) so we know that _flat is better than nested_ for style and readability.  But is there a performance difference?  Not at all - in fact, these two functions have identical bytecode.

```
  2           0 LOAD_FAST                0 (num)
              3 LOAD_CONST               1 (3)
              6 BINARY_MODULO       
              7 LOAD_CONST               2 (0)
             10 COMPARE_OP               2 (==)
             13 POP_JUMP_IF_FALSE       24

  3          16 LOAD_CONST               3 ('Fizz')
             19 PRINT_ITEM          
             20 PRINT_NEWLINE       
             21 JUMP_FORWARD            29 (to 53)

  4     >>   24 LOAD_FAST                0 (num)
             27 LOAD_CONST               4 (5)
             30 BINARY_MODULO       
             31 LOAD_CONST               2 (0)
             34 COMPARE_OP               2 (==)
             37 POP_JUMP_IF_FALSE       48

  5          40 LOAD_CONST               5 ('Buzz')
             43 PRINT_ITEM          
             44 PRINT_NEWLINE       
             45 JUMP_FORWARD             5 (to 53)

  7     >>   48 LOAD_FAST                0 (num)
             51 PRINT_ITEM          
             52 PRINT_NEWLINE       
        >>   53 LOAD_CONST               0 (None)
             56 RETURN_VALUE    
```

That makes sense - `else` just means "start executing here if the `if` was false" - there's no more computation to do.  `elif` is just [syntactic sugar](https://en.wikipedia.org/wiki/Syntactic_sugar).

### Further reading:
This just scratches the surface of what's interesting about python bytecode.

If you enjoyed this, you might enjoy diving into Yaniv Aknin's [series](http://tech.blog.aknin.name/category/my-projects/pythons-innards/) on python internals.  If you're excited about bytecode, you should contribute to Ned Batchelder's [byterun](https://github.com/nedbat/byterun).
