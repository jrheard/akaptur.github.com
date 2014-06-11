---
layout: post
title: "Of Syntax Warnings and Symbol Tables"
date: 2014-06-11 18:30
comments: false
categories: python, python-internals
---

A Hacker Schooler hit an interesting bug today: her program would sometimes emit the message `SyntaxWarning: import * only allowed at module level`.  I had never seen a `SyntaxWarning` before, so I decided to dig in.

The wording of the warning is strange: it says that star-import is only *allowed* at the module level, but it's not a syntax error, just a warning.  In fact, you can use a star-import in a scope that isn't a module (in Python 2):

```python
>>> def nope():
...     from random import *
...     print randint(1,10)
...
<stdin>:1: SyntaxWarning: import * only allowed at module level
>>> nope()
7
```

The Python [spec](https://docs.python.org/2/reference/simple_stmts.html?highlight=import#the-import-statement) gives some more details:
>The from form with * may only occur in a module scope. If the wild card form of import — import * — is used in a function and the function contains or is a nested block with free variables, the compiler will raise a SyntaxError.

Just having `import *` in a function isn't enough to raise a syntax error - we also need free variables. The [Python execution model](https://docs.python.org/2.7/reference/executionmodel.html) refers to three kinds of variables, 'local,' 'global,' and 'free', defined as follows:
>If a name is bound in a block, it is a local variable of that block. If a name is bound at the module level, it is a global variable. (The variables of the module code block are local and global.) If a variable is used in a code block but not defined there, it is a free variable.

Now we can see how to trigger a syntax error from our syntax warning:

``` python
>>> def one():
...     def two():
...         from random import *
...         print randint(1,10)
...     two()
...
<stdin>:2: SyntaxWarning: import * only allowed at module level
  File "<stdin>", line 3
SyntaxError: import * is not allowed in function 'two' because it is a nested function
```

and similarly,

```python
>>> def one():
...     from random import *
...     def two():
...         print randint(1,10)
...     two()
...
<stdin>:1: SyntaxWarning: import * only allowed at module level
  File "<stdin>", line 2
SyntaxError: import * is not allowed in function 'one' because it contains a nested function with free variables
```

As Python programmers, we're used to our lovely dynamic language, and it's unusual to hit compile-time constraints.  As [Amy Hanlon](https://twitter.com/amygdalama) points out, it's particularly weird to hit a compile-time error for code that wouldn't raise a NameError when it ran - `randint` would indeed be in `one`'s namespace if the import-star had executed.

But we can't run code that doesn't compile, and in this case the compiler doesn't have enough information to determine what bytecode to emit. There are different opcodes for loading and storing each of global, free, and local variables. A variable's status as global, free, or local must be determined at compile time and then stored in the symbol table.

To investigate this, let's look at minor variations on this code snippet and disassemble them.

```python
>>> code_type = type((lambda: 1).__code__) # just a handle on the code type, which isn't exposed as a builtin
>>> # A helper function to disassemble nested functions
>>> def recur_dis(fn):
...     print(fn.__code__)
...     dis.dis(fn)
...     for const in fn.__code__.co_consts:
...         if isinstance(const, code_type):
...             print
...             print
...             print(const)
...             dis.dis(const)
```

First, when `x` is local, the compiler emits `STORE_FAST` in the assignment statement and `LOAD_FAST` to load it, marked with arrows below.

```python
>>> def one():
...     def two():
...         x = 1
...         print x
...     two()
>>> recur_dis(one)
<code object one at 0x10e246730, file "<stdin>", line 1>
  2           0 LOAD_CONST               1 (<code object two at 0x10e246030, file "<stdin>", line 2>)
              3 MAKE_FUNCTION            0
              6 STORE_FAST               0 (two)

  5           9 LOAD_FAST                0 (two)
             12 CALL_FUNCTION            0
             15 POP_TOP
             16 LOAD_CONST               0 (None)
             19 RETURN_VALUE

<code object two at 0x10e246030, file "<stdin>", line 2>
  3           0 LOAD_CONST               1 (1)
              3 STORE_FAST               0 (x)          <---- STORE_FAST

  4           6 LOAD_FAST                0 (x)          <----- LOAD_FAST
              9 PRINT_ITEM
             10 PRINT_NEWLINE
             11 LOAD_CONST               0 (None)
             14 RETURN_VALUE
```

When `x` is global, the compiler emits `LOAD_GLOBAL` to load it.  I think the assignment is `STORE_FAST` again, but it's not pictured here because the assignment is outside the function and thus not disassembled.

```python
>>> x = 1
>>> def one():
...     def two():
...         print x
...     two()
...
>>> recur_dis(one)
<code object one at 0x10e246730, file "<stdin>", line 1>
  2           0 LOAD_CONST               1 (<code object two at 0x10e2464b0, file "<stdin>", line 2>)
              3 MAKE_FUNCTION            0
              6 STORE_FAST               0 (two)

  4           9 LOAD_FAST                0 (two)
             12 CALL_FUNCTION            0
             15 POP_TOP
             16 LOAD_CONST               0 (None)
             19 RETURN_VALUE

<code object two at 0x10e2464b0, file "<stdin>", line 2>
  3           0 LOAD_GLOBAL              0 (x)          <----- LOAD_GLOBAL
              3 PRINT_ITEM
              4 PRINT_NEWLINE
              5 LOAD_CONST               0 (None)
              8 RETURN_VALUE
```

Finally, when `x` is nonlocal, the compiler notices that we'll need a closure, and emits the opcodes `LOAD_CLOSURE`, `MAKE_CLOSURE`, and later `LOAD_DEREF`.

```python
>>> def one():
...     x = 1
...     def two():
...        print x
...     two()
...
>>> recur_dis(one)
<code object one at 0x10e246e30, file "<stdin>", line 1>
  2           0 LOAD_CONST               1 (1)
              3 STORE_DEREF              0 (x)          <----- STORE_DEREF

  3           6 LOAD_CLOSURE             0 (x)          <----- LOAD_CLOSURE
              9 BUILD_TUPLE              1
             12 LOAD_CONST               2 (<code object two at 0x10e246d30, file "<stdin>", line 3>)
             15 MAKE_CLOSURE             0
             18 STORE_FAST               0 (two)

  5          21 LOAD_FAST                0 (two)
             24 CALL_FUNCTION            0
             27 POP_TOP
             28 LOAD_CONST               0 (None)
             31 RETURN_VALUE

<code object two at 0x10e246d30, file "<stdin>", line 3>
  4           0 LOAD_DEREF               0 (x)          <----- LOAD_DEREF
              3 PRINT_ITEM
              4 PRINT_NEWLINE
              5 LOAD_CONST               0 (None)
              8 RETURN_VALUE
```

Let's now return to a case that throws a syntax error.

```
>>> def one():
...      from random import *
...      def two():
...          print x
...      two()
...
<stdin>:1: SyntaxWarning: import * only allowed at module level
  File "<stdin>", line 2
SyntaxError: import * is not allowed in function 'one' because it contains a nested function with free variables
```

I'd love to show what the disassembled bytecode for this one looks like, but we can't do that because there is no bytecode! We got a compile-time error, so there's nothing here.

### Further reading
Everything I know about symbol tables I learned from [Eli Bendersky's blog](http://eli.thegreenplace.net/2010/09/18/python-internals-symbol-tables-part-1/). I've skipped some complexity in the implementation that Eli covers.

`ack`ing through the source code of CPython for the text of the error message leads us right to `symtable.c`, which is exactly where we'd expect this message to be emitted. The function `check_unoptimized` shows where the syntax error gets thrown (and shows another illegal construct, too - but we'll leave that one as an exercise for the reader).

p.s. In Python 3, `import *` anywhere other than a module is just an unqualified syntax error - none of this messing around with the symbol table.
