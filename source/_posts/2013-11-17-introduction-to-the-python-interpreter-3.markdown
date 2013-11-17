---
layout: post
title: "Introduction to the Python Interpreter, Part 3: Understanding Bytecode"
date: 2013-11-17 09:56
comments: false
categories: python python-internals
---

_This is Part 3 in a series on the Python interpreter.  Part 1 [here](/blog/2013/11/15/introduction-to-the-python-interpreter/), Part 2 [here](/blog/2013/11/15/introduction-to-the-python-interpreter-2/).  If you're enjoying this series, consider applying to [Hacker School](https://www.hackerschool.com/), where I work as a facilitator._

### Bytecode

When we left our heroes, they had come across some odd-looking output:

```python 
>>> foo.func_code.co_code
'd\x01\x00}\x01\x00|\x01\x00|\x00\x00\x17S'
```

This is python _bytecode_.  

You recall from Part 2 that "python bytecode" and "a python code object" are not the same thing: the bytecode is an attribute of the code object, among many other attributes.  Bytecode is found in the `co_code` attribute of the code object, and contains instructions for the interpreter.

So what is bytecode?  Well, it's just a series of bytes.  They look wacky when we print them because some bytes are printable and others aren't, so let's take the `ord` of each byte to see that they're just numbers.

``` python
>>> [ord(b) for b in foo.func_code.co_code]
[100, 1, 0, 125, 1, 0, 124, 1, 0, 124, 0, 0, 23, 83]
```

Here are the bytes that make up python bytecode.  The interpreter will loop through each byte, look up what it should do for each one, and then do that thing.  Notice that the bytecode itself doesn't include any python objects, or references to objects, or anything like that.

One way to understand python bytecode would be to find the CPython interpreter file (it's `ceval.c`), and flip through it looking up what `100` means, then `1`, then `0`, and so on.  We'll do this later in the series!  For now, there's a simpler way: the `dis` module.

### Disassembling bytecode

Disassembling bytecode means taking this series of bytes and printing out something we humans can understand.  It's not a step in python execution; the `dis` module just helps us understand an intermediate state of python internals. I can't think of a reason why you'd ever want to use `dis` in production code - it's for humans, not for machines.

Today, however, taking some bytecode and making it human-readable is exactly what we're trying to do, so `dis` is a great tool.  We'll use the function `dis.dis` to analyze the code object of our function `foo`.

``` python
>>> def foo(a):
...     x = 3
...     return x + a
... 
>>> import dis
>>> dis.dis(foo.func_code)
  2           0 LOAD_CONST               1 (3)
              3 STORE_FAST               1 (x)

  3           6 LOAD_FAST                1 (x)
              9 LOAD_FAST                0 (a)
             12 BINARY_ADD          
             13 RETURN_VALUE
```

(You usually see this called as `dis.dis(foo)`, directly on the function object.  That's just a convenience: `dis` is really analyzing the code object.  If it's passed a function, it just gets its code object.)

The numbers in the left-hand column are line numbers in the original source code.  The second column is the offset into the bytecode: `LOAD_CONST` appears at position 0, `STORE_FAST` at position 3, and so on.  The middle column shows the names of bytes. These names are just for our (human) benefit - the interpreter doesn't need the names.

The last two columns give details about the instructions's argument, if there is an argument.  The fourth column shows the argument itself, which represents an index into other attributes of the code object. In the example, `LOAD_CONST`'s argument is an index into the list `co_consts`, and `STORE_FAST`'s argument is an index into `co_varnames`.  Finally, in the fifth column, `dis` has looked up the constants or names in the place the fourth column specified and told us what it found there. We can easily verify this:

``` python 
>>> foo.func_code.co_consts[1]
3
>>> foo.func_code.co_varnames[1]
'x'
```

This also explains why the second instruction, `STORE_FAST`, is found at bytecode position 3.  If a bytecode has an argument, the next two bytes are that argument. It's the interpreter's job to handle this correctly.

(You may be surprised that `BINARY_ADD` doesn't have arguments. We'll come back to this in a future installment, when we get to the interpreter itself.)

People often say that `dis` is a disassembler of python bytecode.  This is true enough - the `dis` module's docs say it - but `dis` knows about more than just the bytecode, too: it uses the whole code object to give us an understandable printout.  The middle three columns show information actually encoded in the bytecode, while the first and the last columns show other information.  Again, the bytecode itself is really limited: it's just a series of numbers, and things like names and constants are not a part of it.

How does the `dis` module get from bytes like `100` to names like `LOAD_CONST` and back?  Try to think of a way you'd do it.  If you thought "Well, you could have a list that has the byte names in the right order," or you thought, "I guess you could have a dictionary where the names are the keys and the byte values are the values," then congratulations!  That's exactly what's going on.  The file `opcode.py` defines the list and the dictionary.  It's full of lines like these (`def_op` inserts the mapping in both the list and the dictionary):

``` python
def_op('LOAD_CONST', 100)       # Index in const list
def_op('BUILD_TUPLE', 102)      # Number of tuple items
def_op('BUILD_LIST', 103)       # Number of list items
def_op('BUILD_SET', 104)        # Number of set items
```

There's even a friendly comment telling us what each byte's argument means.

Ok, now we understand what python bytecode is (and isn't), and how to use `dis` to make sense of it. In Part 4, we'll look at another example to see how Python can compile down to bytecode but still be a dynamic language.