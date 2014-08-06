---
layout: post
title: "The CPython Peephole Optimizer and You"
date: 2014-08-02 11:25
categories: python python-internals
---

Last Thursday I gave a lightning talk at [Hacker School](//www.hackerschool.com) about the peephole optimizer in Python.  A "peephole optimization" is a compiler optimization that looks at a small chunk of code at a time and optimizes in that little spot. This post explains one surprising side-effect of an optimization in CPython.

## Writing a test coverage tool

Suppose that we're setting out to write a test coverage tool. Python provides an easy way to trace execution using `sys.settrace`, so a simple version of a coverage analyzer isn't too hard.

Our code to test is one simple function:

``` python example.py
def iffer(condition):
    if condition:
        return 3
    else:
        return 10
```

Then we'll write the world's simplest testing framework:

``` python tests.py
from example import iffer

def test_iffer():
    assert iffer(True) == 3
    assert iffer(False) == 10

def run_tests():
    test_iffer()
```

Now for the simplest possible coverage tool. We can pass `sys.settrace` any tracing function, and it'll be called with the arguments `frame`, `event`, and `arg` every time an event happens in the execution.  Lines of code being executed, function calls, function returns, and exceptions are all events. We'll filter out everything but `line` and `call` events, then keep track of what line of code was executing.[^1] Then we run the tests while the trace function is tracing, and finally report which (non-empty lines) failed to execute.

``` python coverage.py
import sys
import tests
import inspect

class TinyCoverage(object):
    def __init__(self, file_to_watch):
        self.source_file = file_to_watch
        self.source_code = open(file_to_watch).readlines()
        self.executed_code = []

    def trace(self, frame, event, arg):
        current_file = inspect.getframeinfo(frame).filename

        if self.source_file in current_file and \
            (event == "line" or event == "call"):

            self.executed_code.append(frame.f_lineno)

        return self.trace

    def unexecuted_code(self):
        skipped = []
        for line_num in range(1, len(self.source_code)+1):
            if line_num not in self.executed_code:
                src = self.source_code[line_num - 1]
                if src != "\n":
                    skipped.append(line_num)
        return skipped

    def report(self):
        skipped = self.unexecuted_code()
        percent_skipped = float(len(skipped)) / len(self.source_code)
        if skipped:
            print "{} line(s) did not execute ({:.0%})".format(len(skipped), percent_skipped)
            for line_num in skipped:
                print line_num, self.source_code[line_num - 1]
        else:
            print "100% coverage, go you!"

if __name__ == '__main__':
    t = TinyCoverage('example.py')
    sys.settrace(t.trace)
    tests.run_tests()
    sys.settrace(None)
    t.report()
```

Let's try it.  We're pretty confident in our test coverage - there are only two branches in the code, and we've tested both of them.

``` text
peephole [master *] ⚲ python coverage.py
1 line(s) did not execute (9%)
4     else:
```

Why didn't the `else` line execute? To answer this, we'll run our function through the disassembler.[^2]

```
>>> from example import iffer
>>> import dis
>>> dis.dis(iffer)
  2           0 LOAD_FAST                0 (condition)
              3 POP_JUMP_IF_FALSE       10

  3           6 LOAD_CONST               1 (3)
              9 RETURN_VALUE

  5     >>   10 LOAD_CONST               2 (10)
             13 RETURN_VALUE
             14 LOAD_CONST               0 (None)
             17 RETURN_VALUE

```

You don't need to follow exactly what's going on in this bytecode, but note that the first column is the line numbers of source code and line 4, the one containing the `else`, doesn't appear. Why not? Well, there's nothing to _do_ with an else statement - it's just a separator between two branches of an `if` statement. The second line in the disassembly, `POP_JUMP_IF_FALSE   10`, means that the interpreter will pop the top thing off of the virtual machine stack, jump to bytecode index ten if that thing is false, or continue with the next instruction if it's true.

From the bytecode's perspective, there's no difference at all between writing this:
``` python
if a:
    ...
else:
    if b:
       ...
    else:
        ...
```

and this:

```python
if a:
    ...
elif b:
   ...
else:
    ...
```
(even though the second is better style).

We've learned we need to special-case `else` statements in our code coverage tool.  Since there's no logic in them, let's just drop lines that only contain `else:`. We can revise our `unexecuted_code` method accordingly:

``` python coverage.py
    def unexecuted_code(self):
        skipped = []
        for line_num in range(1, len(self.source_code)+1):
            if line_num not in self.executed_code:
                src = self.source_code[line_num - 1]
                if src != "\n" and "else:\n" not in src:  # Add "else" dropping
                    skipped.append(line_num)
        return skipped
```

Then run it again:

``` text
peephole [master *] ⚲ python coverage.py
100% coverage, go you!
```

Success!

## Complications arise

Our previous example was really simple. Let's add a more complex one.

``` python example.py
def iffer(condition):
    if condition:
        return 3
    else:
        return 10

def continuer():
    a = b = c = 0
    for n in range(100):
        if n % 2:
            if n % 4:
                a += 1
            continue
        else:
            b += 1
        c += 1

    return a, b, c
```

`continuer` will increment `a` on all odd numbers and increment `b` and `c` for all even numbers. Don't forget to add a test:

``` python tests.py
import sys
import inspect
from example2 import iffer, continuer

def test_iffer():
    assert iffer(True) == 3
    assert iffer(False) == 10

def test_continuer():
    assert continuer() == (50, 50, 50)

def run_tests():
    test_iffer()
    test_continuer()

```

``` text
peephole [master *] ⚲ python coverage2.py
1 line(s) did not execute (4%)
13             continue
```

Hmm. The test we wrote certainly did involve the `continue` statement - if the interpreter hadn't skipped the bottom half of the loop, the test wouldn't have passed. Let's use the strategy we used before to understand what's happening: examining the output of the disassembler.

``` python
>>> dis.dis(continuer)
  8           0 LOAD_CONST               1 (0)
              3 DUP_TOP
              4 STORE_FAST               0 (a)
              7 DUP_TOP
              8 STORE_FAST               1 (b)
             11 STORE_FAST               2 (c)

  9          14 SETUP_LOOP              79 (to 96)
             17 LOAD_GLOBAL              0 (range)
             20 LOAD_CONST               2 (100)
             23 CALL_FUNCTION            1
             26 GET_ITER
        >>   27 FOR_ITER                65 (to 95)
             30 STORE_FAST               3 (n)

 10          33 LOAD_FAST                3 (n)
             36 LOAD_CONST               3 (2)
             39 BINARY_MODULO
             40 POP_JUMP_IF_FALSE       72

 11          43 LOAD_FAST                3 (n)
             46 LOAD_CONST               4 (4)
             49 BINARY_MODULO
             50 POP_JUMP_IF_FALSE       27

 12          53 LOAD_FAST                0 (a)
             56 LOAD_CONST               5 (1)
             59 INPLACE_ADD
             60 STORE_FAST               0 (a)
             63 JUMP_ABSOLUTE           27

 13          66 JUMP_ABSOLUTE           27
             69 JUMP_FORWARD            10 (to 82)

 15     >>   72 LOAD_FAST                1 (b)
             75 LOAD_CONST               5 (1)
             78 INPLACE_ADD
             79 STORE_FAST               1 (b)

 16     >>   82 LOAD_FAST                2 (c)
             85 LOAD_CONST               5 (1)
             88 INPLACE_ADD
             89 STORE_FAST               2 (c)
             92 JUMP_ABSOLUTE           27
        >>   95 POP_BLOCK

 18     >>   96 LOAD_FAST                0 (a)
             99 LOAD_FAST                1 (b)
            102 LOAD_FAST                2 (c)
            105 BUILD_TUPLE              3
            108 RETURN_VALUE
```

There's a lot more going on here, but you don't need to understand all of it to proceed. Here are the things we need to know to make sense of this:

  * The second column in the output is the index in the bytecode, the third is the byte name, and the fourth is the argument.  The fifth, when present, is a hint about the meaning of the argument.
  * `POP_JUMP_IF_FALSE`, `POP_JUMP_IF_TRUE`, and `JUMP_ABSOLUTE` have the jump target as their argument. So, e.g. `POP_JUMP_IF_TRUE 27` means "if the popped expression is true, jump to position 27."
  * `JUMP_FORWARD`'s argument specifies the distance to jump forward in the bytecode, and the fifth column shows where the jump will end.
  * When an iterator is done, `FOR_ITER` jumps forward the number of bytes specified in its argument.

Unlike the `else` case, the line containing the `continue` does appear in the bytecode. But trace through the bytecode using what you know about jumps: no matter how hard you try, you can't end up on bytes 66 or 69, the two that belong to line 13.

The `continue` is unreachable because of a compiler optimization. In this particular optimization, the compiler notices that two instructions in a row are jumps, and it combines these two hops into one larger jump. So, in a very real sense, the `continue` line didn't execute - it was optimized out - even though the logic reflected in the `continue` is still reflected in the bytecode.

What would this bytecode have looked like without the optimizations? There's not currently an option to disable the peephole bytecode optimizations, although there will be in a future version of Python (following an [extensive debate](//mail.python.org/pipermail/python-ideas/2014-May/027893.html) on the python-dev list). For now, the only way to turn off optimizations is to comment out the relevant line in `compile.c`, the call to `PyCode_Optimize`, and recompile Python. Here's the diff, if you're playing along at home.

``` diff
cpython ⚲ hg diff
diff -r 77f36cdb71b0 Python/compile.c
--- a/Python/compile.c  Fri Aug 01 17:48:34 2014 +0200
+++ b/Python/compile.c  Sat Aug 02 15:43:45 2014 -0400
@@ -4256,10 +4256,6 @@
     if (flags < 0)
         goto error;

-    bytecode = PyCode_Optimize(a->a_bytecode, consts, names, a->a_lnotab);
-    if (!bytecode)
-        goto error;
-
     tmp = PyList_AsTuple(consts); /* PyCode_New requires a tuple */
     if (!tmp)
         goto error;
@@ -4270,7 +4266,7 @@
     kwonlyargcount = Py_SAFE_DOWNCAST(c->u->u_kwonlyargcount, Py_ssize_t, int);
     co = PyCode_New(argcount, kwonlyargcount,
                     nlocals_int, stackdepth(c), flags,
-                    bytecode, consts, names, varnames,
+                    a->a_bytecode, consts, names, varnames,
                     freevars, cellvars,
                     c->c_filename, c->u->u_name,
                     c->u->u_firstlineno,
```

``` python
>>> dis.dis(continuer)
  8           0 LOAD_CONST               1 (0)
              3 DUP_TOP
              4 STORE_FAST               0 (a)
              7 DUP_TOP
              8 STORE_FAST               1 (b)
             11 STORE_FAST               2 (c)

  9          14 SETUP_LOOP              79 (to 96)
             17 LOAD_GLOBAL              0 (range)
             20 LOAD_CONST               2 (100)
             23 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
             26 GET_ITER
        >>   27 FOR_ITER                65 (to 95)
             30 STORE_FAST               3 (n)

  10         33 LOAD_FAST                3 (n)
             36 LOAD_CONST               3 (2)
             39 BINARY_MODULO
             40 POP_JUMP_IF_FALSE       72

  11         43 LOAD_FAST                3 (n)
             46 LOAD_CONST               4 (4)
             49 BINARY_MODULO
             50 POP_JUMP_IF_FALSE       66

  12         53 LOAD_FAST                0 (a)
             56 LOAD_CONST               5 (1)
             59 INPLACE_ADD
             60 STORE_FAST               0 (a)
             63 JUMP_FORWARD             0 (to 66)

  13    >>   66 JUMP_ABSOLUTE           27
             69 JUMP_FORWARD            10 (to 82)

  14    >>   72 LOAD_FAST                1 (b)
             75 LOAD_CONST               5 (1)
             78 INPLACE_ADD
             79 STORE_FAST               1 (b)

  15    >>   82 LOAD_FAST                2 (c)
             85 LOAD_CONST               5 (1)
             88 INPLACE_ADD
             89 STORE_FAST               2 (c)
             92 JUMP_ABSOLUTE           27
        >>   95 POP_BLOCK

  16    >>   96 LOAD_FAST                0 (a)
             99 LOAD_FAST                1 (b)
            102 LOAD_FAST                2 (c)
            105 BUILD_TUPLE              3
            108 RETURN_VALUE
```

Just as we expected, the jump targets have changed. The instruction at position 50, `POP_JUMP_IF_FALSE`, now has 66 as its jump target - a previously unreachable instruction associated with the `continue`. Instruction 63, `JUMP_FORWARD`, is also targeting 66. In both cases, the only way to reach this instruction is to jump to it, and the instruction itself jumps away.[^3]

Now we can run our coverage tool with the unoptimized Python:

``` text
peephole [master *+] ⚲ ../cpython/python.exe coverage2.py
100% coverage, go you!
```

Complete success!

## So is this a good idea or not?

Compiler optimizations are often a straightforward win.  If the compiler can apply simple rules that make my code faster without requiring work from me, that's great. Almost nobody requires a strict mapping of code that they write to code that ends up executing. So, peephole optimization in general: yes! Great!

But "almost nobody" is not nobody, and one kind of people who _do_ require strict reasoning about executed code are the authors of test coverage software. In the [python-dev thread](//mail.python.org/pipermail/python-ideas/2014-May/027893.html) I linked to earlier, there was an extensive discussion over whether or not serving this demographic by providing an option to disable to optimizations was worth increasing the complexity of the codebase. Ultimately it was decided that it was worthwhile, but this is a fair question to ask.

## Further reading

Beyond the interesting Python-dev thread linked above, my other suggestions are mostly code. [CPython's `peephole.c`](//hg.python.org/cpython/file/118d6f49d6d6/Python/peephole.c) is pretty readable C code, and I encourage you to take a look at it. ("Constant folding" is a great place to start.) There's also a website [compileroptimizations.com](//www.compileroptimizations.com/) which has short examples and discussion of 45 different optimizations. If you'd like to play with these code examples, they're all available on my [github](//github.com/akaptur/peephole-optimization).

[^1]: We need to include `call` events to capture the first line of a function declaration, `def fn(...):`
[^2]: I've previously written an introduction to the disassembler [here](/blog/2013/11/17/introduction-to-the-python-interpreter-3/).
[^3]: You may be wondering what the `JUMP_ABSOLUTE` instruction at position 66 is doing.  This instruction does nothing unless a particular compiler optimization is turned on. The optimization support faster loops, but creates restrictions on what those loops can do. See `ceval.c` for more. _Edit: This footnote previously incorrectly referenced `JUMP_FORWARD`._
